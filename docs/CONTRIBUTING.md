# Como contribuir no CampusHub

Este documento define o fluxo de trabalho do grupo. Siga antes de criar qualquer código ou documento novo.

## Regras gerais

1. **Nunca trabalhe direto na `main`.** Ela contém a base estável do projeto.
2. **Toda funcionalidade nova vai em uma worktree própria**, com uma branch própria. Assim, o trabalho de um integrante não entra em conflito com o do outro.
3. **Toda vez que fizer push, atualize o progresso** no `docs/PRD.md` (seção 12) e no `docs/ADR.md` (seção "Progresso de implementação").
4. **O Pull Request é aberto só quando a implementação estiver validada** (build passando e testes da funcionalidade rodando).

## Criar uma worktree para uma funcionalidade

Dentro da pasta do repositório (`C:\Users\Ryan Lima\CampusHub`):

```
git worktree add -b feat/nome-da-funcionalidade "C:\Users\Ryan Lima\worktrees\nome-da-funcionalidade" main
```

Exemplos de nomes de branch: `feat/login`, `feat/reservas`, `fix/conflito-de-horario`, `docs/progresso`.

Para trabalhar, abra a pasta da worktree no Android Studio. Cada worktree é uma cópia independente do projeto, com a própria branch.

## Enviar a branch

Dentro da worktree:

```
git add -A
git commit -m "tipo: descrição curta"
git push -u origin feat/nome-da-funcionalidade
```

Tipos de commit: `feat` (funcionalidade), `fix` (correção), `docs` (documentação), `chore` (configuração), `test` (testes).

## Abrir um Pull Request

Quando a funcionalidade estiver validada:

1. Confirme que o build passa e que os testes da funcionalidade rodam.
2. Confirme que o progresso no PRD e no ADR está atualizado.
3. Abra o PR no GitHub, da branch da funcionalidade para a `main`.
4. Peça revisão ao outro integrante.

## Remover uma worktree depois do merge

```
git worktree remove "C:\Users\Ryan Lima\worktrees\nome-da-funcionalidade"
git branch -d feat/nome-da-funcionalidade
```

## Consultar as worktrees ativas

```
git worktree list
```

## Documentos de referência

- `docs/PRD.md`: o que o app faz, requisitos e progresso.
- `docs/ADR.md`: as decisões de arquitetura e o progresso de implementação.
