# Infraestrutura essencial:

* VPC, Subnets, Security Groups
* Aurora PostgreSQL cluster
* MSK Kafka Cluster
* DynamoDB tables básicas
* S3 Buckets (para bank reconciliation)
* AWS Lambda (esqueleto)
* Cognito User Pool e App Client
* Redis via Elasticache
* AWS Batch (config básica para reconciliation)
* IAM Roles e Policies para serviços principais

---

# terraform/main.tf (infraestrutura geral FinBank)

```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  default = "sa-east-1"
}

##################################
# VPC + Networking
##################################

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "finbank-vpc" }
}

resource "aws_subnet" "public_1" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "public_2" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "${var.aws_region}b"
  map_public_ip_on_launch = true
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = { Name = "public-route-table" }
}

resource "aws_route_table_association" "public_1_assoc" {
  subnet_id      = aws_subnet.public_1.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_route_table_association" "public_2_assoc" {
  subnet_id      = aws_subnet.public_2.id
  route_table_id = aws_route_table.public_rt.id
}

##################################
# Security Groups
##################################

resource "aws_security_group" "aurora_sg" {
  name        = "aurora-sg"
  description = "Allow Postgres access"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "msk_sg" {
  name        = "msk-sg"
  description = "Allow Kafka ports"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 9094
    to_port     = 9094
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "elasticache_sg" {
  name        = "elasticache-sg"
  description = "Allow Redis access"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 6379
    to_port     = 6379
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

##################################
# Aurora PostgreSQL Cluster
##################################

resource "aws_db_subnet_group" "aurora_subnet" {
  name       = "aurora-subnet-group"
  subnet_ids = [aws_subnet.public_1.id, aws_subnet.public_2.id]
}

resource "aws_rds_cluster" "aurora" {
  cluster_identifier      = "finbank-aurora-cluster"
  engine                  = "aurora-postgresql"
  engine_version          = "13.6"
  database_name           = "finbankdb"
  master_username         = "finbankadmin"
  master_password         = var.db_master_password
  backup_retention_period = 7
  preferred_backup_window = "07:00-09:00"
  vpc_security_group_ids  = [aws_security_group.aurora_sg.id]
  db_subnet_group_name    = aws_db_subnet_group.aurora_subnet.name
  skip_final_snapshot     = true
}

resource "aws_rds_cluster_instance" "aurora_instances" {
  count               = 2
  identifier          = "finbank-aurora-instance-${count.index + 1}"
  cluster_identifier  = aws_rds_cluster.aurora.id
  instance_class      = "db.r5.large"
  engine              = aws_rds_cluster.aurora.engine
  engine_version      = aws_rds_cluster.aurora.engine_version
  publicly_accessible = false
  db_subnet_group_name = aws_db_subnet_group.aurora_subnet.name
  depends_on          = [aws_rds_cluster.aurora]
}

variable "db_master_password" {
  description = "Password for Aurora master user"
  type        = string
  sensitive   = true
}

##################################
# MSK Kafka Cluster
##################################

resource "aws_msk_cluster" "kafka" {
  cluster_name           = "finbank-msk-cluster"
  kafka_version          = "3.3.1"
  number_of_broker_nodes = 2

  broker_node_group_info {
    instance_type   = "kafka.m5.large"
    client_subnets  = [aws_subnet.public_1.id, aws_subnet.public_2.id]
    security_groups = [aws_security_group.msk_sg.id]
  }

  encryption_info {
    encryption_in_transit {
      client_broker = "TLS"
      in_cluster    = true
    }
  }

  tags = {
    Environment = "dev"
    Project     = "finbank"
  }
}

##################################
# DynamoDB Tables (Pix Service)
##################################

resource "aws_dynamodb_table" "pix_transactions" {
  name           = "pix_transactions"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "transactionId"

  attribute {
    name = "transactionId"
    type = "S"
  }

  tags = {
    Environment = "dev"
    Project     = "finbank"
  }
}

##################################
# S3 Buckets (Bank Reconciliation)
##################################

resource "aws_s3_bucket" "cnab_files" {
  bucket = "finbank-cnab-files-${random_id.bucket_suffix.hex}"
  acl    = "private"

  tags = {
    Environment = "dev"
    Project     = "finbank"
  }
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}

##################################
# AWS Lambda (placeholder)
##################################

resource "aws_iam_role" "lambda_exec_role" {
  name = "finbank-lambda-exec-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_policy_attachment" "lambda_basic_execution" {
  name       = "lambda-basic-execution-attach"
  roles      = [aws_iam_role.lambda_exec_role.name]
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_lambda_function" "bank_reconciliation_processor" {
  filename         = "lambda/bank_reconciliation_processor.zip" # coloque o zip pronto
  function_name    = "bank-reconciliation-processor"
  role             = aws_iam_role.lambda_exec_role.arn
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  timeout          = 300
  memory_size      = 512
  source_code_hash = filebase64sha256("lambda/bank_reconciliation_processor.zip")

  environment {
    variables = {
      AURORA_CLUSTER_ARN  = aws_rds_cluster.aurora.arn
      AURORA_SECRET_ARN   = "TODO: colocar ARN do secret manager"
      S3_BUCKET           = aws_s3_bucket.cnab_files.bucket
    }
  }
}

##################################
# AWS Batch (bank reconciliation)
##################################

resource "aws_batch_compute_environment" "batch_env" {
  compute_environment_name = "finbank-batch-env"
  compute_resources {
    type              = "EC2"
    max_vcpus         = 16
    subnets           = [aws_subnet.public_1.id, aws_subnet.public_2.id]
    instance_types    = ["optimal"]
    security_group_ids = [aws_security_group.aurora_sg.id]
  }
  service_role = aws_iam_role.batch_service_role.arn
  type         = "MANAGED"
}

resource "aws_iam_role" "batch_service_role" {
  name = "finbank-batch-service-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect    = "Allow",
      Principal = {
        Service = "batch.amazonaws.com"
      },
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_batch_job_queue" "batch_queue" {
  name                 = "finbank-batch-queue"
  compute_environments = [aws_batch_compute_environment.batch_env.arn]
  priority             = 1
  state                = "ENABLED"
}

resource "aws_batch_job_definition" "bank_reconciliation_job" {
  name       = "bank-reconciliation-job"
  type       = "container"
  container_properties = jsonencode({
    image        = "amazonlinux"
    vcpus        = 1
    memory       = 2048
    command      = ["sh", "-c", "echo Run bank reconciliation logic here"]
    environment  = [
      {
        name  = "S3_BUCKET"
        value = aws_s3_bucket.cnab_files.bucket
      },
      {
        name  = "AURORA_CLUSTER_ARN"
        value = aws_rds_cluster.aurora.arn
      }
    ]
  })
  retry_strategy {
    attempts = 3
  }
}

##################################
# AWS Cognito User Pool + Client
##################################

resource "aws_cognito_user_pool" "user_pool" {
  name = "finbank-user-pool"

  mfa_configuration = "OPTIONAL"

  sms_configuration {
    external_id = "your-sms-external-id"
    sns_caller_arn = "arn:aws:iam::123456789012:role/SNSRole"
  }

  password_policy {
    minimum_length    = 8
    require_uppercase = true
    require_numbers   = true
    require_symbols   = false
  }
}

resource "aws_cognito_user_pool_client" "user_pool_client" {
  name         = "finbank-user-pool-client"
  user_pool_id = aws_cognito_user_pool.user_pool.id
  generate_secret = false
  allowed_oauth_flows_user_pool_client = true
  explicit_auth_flows = [
    "ALLOW_USER_PASSWORD_AUTH",
    "ALLOW_REFRESH_TOKEN_AUTH",
    "ALLOW_USER_SRP_AUTH",
    "ALLOW_CUSTOM_AUTH"
  ]
  supported_identity_providers = ["COGNITO"]
  prevent_user_existence_errors = "ENABLED"
}

##################################
# Redis Elasticache Cluster
##################################

resource "aws_elasticache_subnet_group" "redis_subnet_group" {
  name       = "finbank-redis-subnet-group"
  subnet_ids = [aws_subnet.public_1.id, aws_subnet.public_2.id]
}

resource "aws_elasticache_cluster" "redis_cluster" {
  cluster_id           = "finbank-redis-cluster"
  engine               = "redis"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 1
  subnet_group_name    = aws_elasticache_subnet_group.redis_subnet_group.name
  security_group_ids   = [aws_security_group.elasticache_sg.id]
  parameter_group_name = "default.redis6.x"
  port                 = 6379
}

##################################
# Outputs úteis
##################################

output "aurora_endpoint" {
  value = aws_rds_cluster.aurora.endpoint
}

output "aurora_reader_endpoint" {
  value = aws_rds_cluster.aurora.reader_endpoint
}

output "msk_bootstrap_brokers_tls" {
  value = aws_msk_cluster.kafka.bootstrap_brokers_tls
}

output "dynamodb_pix_table_name" {
  value = aws_dynamodb_table.pix_transactions.name
}

output "s3_cnab_bucket" {
  value = aws_s3_bucket.cnab_files.bucket
}

output "redis_endpoint" {
  value = aws_elasticache_cluster.redis_cluster.cache_nodes[0].address
}
```

---

# Como usar esse arquivo único

1. Ajuste variáveis sensíveis (ex: `db_master_password`, ARNs de roles SNS, segredos Cognito, etc) usando `terraform.tfvars` ou secrets manager integrados.

2. Configure AWS CLI localmente ou perfil IAM com permissão para criar recursos.

3. Rode:

```bash
terraform init
terraform plan
terraform apply
```
