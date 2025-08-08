# 📝 Template para Issue 

```markdown

name: 🐞 Bug Report
about: Reporte um problema ou comportamento inesperado
title: '[BUG] Título resumido do problema'
labels: bug
assignees: ''

## Descrição
Descreva o problema de forma clara e objetiva.

## Passos para reproduzir
1. Vá para '...'
2. Clique em '...'
3. Veja o erro

## Comportamento esperado
Descreva o que deveria acontecer.

## Logs / Prints / Stacktrace
Inclua logs relevantes ou prints da tela.

## Ambiente
- Sistema Operacional: [e.g. Windows, Linux, macOS]
- Node.js versão: [e.g. 16.x]
- Versão do serviço: [e.g. v1.0.0]

## Informações adicionais
Qualquer outra informação que possa ajudar a resolver o problema.
```

---

# 📝 Template para Pull Request 

```markdown
name: 🚀 Pull Request
about: Descreva suas mudanças para revisão
title: '[FEATURE/BUGFIX/REFACTOR] Título resumido'
labels: ''
assignees: ''

## Descrição
Explique resumidamente o que foi feito.

## Tipo de mudança
- [ ] Bugfix
- [ ] Nova feature
- [ ] Refatoração
- [ ] Documentação
- [ ] Testes

## Como testar?
Descreva os passos para validar as mudanças.

## Checklist
- [ ] Código segue padrões do projeto (ESLint/Prettier)
- [ ] Testes unitários criados/atualizados
- [ ] Testes de integração criados/atualizados
- [ ] Documentação atualizada
- [ ] Build e testes passaram no CI

## Issue relacionada
Fechando #[número da issue]
```

---

# 📦 Padrão de Commit (Conventional Commits)

Exemplos de commits:

* feat(auth): adicionar suporte a MFA no login
* fix(pix): corrigir cálculo do status da transação
* docs(readme): atualizar seção de deploy
* test(account): criar testes para validação de saldo
* chore(ci): configurar GitHub Actions para lint e testes
