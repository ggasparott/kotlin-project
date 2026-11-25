# ADR — CampusHub (versão 1, histórica)

> **Esta versão está superada.** Ela descreve o escopo com banco local em Room e um único aparelho, de 2026-11-11. A versão vigente está em `docs/ADR.md`. Este arquivo é mantido apenas como registro da evolução das decisões.

Autor: @Gaspa · 2026-11-11

## Contexto e índice

Este documento registra as decisões arquiteturais do CampusHub, o app Android de reservas de espaços e eventos do campus descrito no PRD. Cada decisão segue o formato contexto, decisão, alternativas e consequências. Todas estão como Proposta até o grupo confirmar.

| ID | Decisão | Status |
|---|---|---|
| ADR-001 | Banco local com Room (SQLite) | Proposta |
| ADR-002 | Kotlin, Jetpack Compose e MVVM com Hilt | Proposta |
| ADR-003 | Conflito de reservas evitado por transação no Room | Proposta |
| ADR-004 | Perfis de acesso por campo role verificado na camada de domínio | Proposta |
| ADR-005 | Ocupação calculada por horários e confirmação manual de presença | Proposta |
| ADR-006 | Escopo de um único aparelho, sem sincronização | Proposta |

---

## ADR-001: Banco local com Room (SQLite)

**Status:** Proposta

**Contexto.** O trabalho de semestre pede um app Android com persistência de dados, e o SQLite já vem no Android, sem conta externa nem servidor. O Room é a camada oficial do Android por cima do SQLite e reduz o código repetitivo. O CampusHub precisa guardar usuários, espaços, reservas e eventos e gerar relatórios com consultas.

**Decisão.** Guardar todos os dados em um banco local com Room (SQLite): entidades, DAOs com consultas SQL e Flow para a tela reagir às mudanças. O app funciona em um único aparelho, sem internet. Os perfis (aluno, organizador, administrador) são usuários cadastrados no próprio banco, e o login troca de perfil na mesma instalação.

**Alternativas consideradas.**
- Firebase (Auth + Firestore): dados compartilhados entre aparelhos em tempo real, mas exige conta, regras de segurança e aprender um banco NoSQL. Ficaria como evolução futura.
- API própria com banco relacional (por exemplo Spring ou Node com PostgreSQL): dados compartilhados e SQL de verdade, porém soma servidor e deploy ao escopo.
- SQLiteOpenHelper puro: o mesmo banco, mas com muito mais código manual e sem verificação das consultas em tempo de compilação.

**Consequências.**
- Positivo: tudo em um só lugar, sem conta externa, funcionando offline, e as consultas SQL (JOIN, GROUP BY) facilitam os relatórios.
- Positivo: o SQLite executa uma gravação por vez, o que simplifica a regra de não sobreposição (ADR-003).
- Negativo: os dados ficam em um aparelho só. Reservas feitas em um celular não aparecem em outro, e os três perfis são demonstrados trocando de usuário no mesmo app (ADR-006).
- Negativo: sem servidor, a segurança por perfil depende do código do app (ADR-004).

---

## ADR-002: Kotlin, Jetpack Compose e MVVM com Hilt

**Status:** Proposta

**Contexto.** Três perfis geram muitas telas e estados, e as regras de negócio (limites, conflitos, lista de espera) precisam ser testadas sem emulador. Premissa a confirmar com o professor: a disciplina permite Kotlin.

**Decisão.** Kotlin com interface em Jetpack Compose e Navigation Compose. Arquitetura MVVM em camadas (tela, ViewModel, Repository, fonte de dados Room), Hilt para injeção de dependências e Coroutines com Flow para dados em tempo real. As regras de negócio ficam em classes de domínio puras, testáveis com JUnit.

**Alternativas consideradas.**
- Views em XML com Fragments: funciona, mas exige mais código de tela e de estado.
- Java: possível, porém Kotlin é o padrão atual do Android e simplifica Flow e coroutines.
- Clean Architecture completa ou MVI: mais cerimônia do que o prazo comporta.
- Lógica dentro de Activities ou Composables: rápido no início, mas difícil de testar e de dividir entre pessoas do grupo.

**Consequências.**
- Positivo: regras testáveis, responsabilidades claras e estado reativo que combina com os Flows do Room.
- Negativo: curva de aprendizado de Compose e Hilt para quem ainda não usou, e algum código repetitivo entre camadas.
- Mitigação: um único módulo do app, com pacotes por funcionalidade (espaços, reservas, eventos, admin).

---

## ADR-003: Conflito de reservas evitado por transação no Room

**Status:** Proposta

**Contexto.** Uma reserva ou evento não pode se sobrepor a outro no mesmo espaço. Checar a disponibilidade e só depois gravar, em duas etapas soltas, deixa uma janela em que duas gravações passam, por exemplo quando duas coroutines tentam o mesmo horário. O SQLite não tem restrição nativa de sobreposição de intervalos.

**Decisão.** Reservas, eventos (pendentes ou aprovados) e bloqueios ficam em tabelas próprias. Uma consulta de sobreposição olha as três no mesmo espaço: há conflito quando o início pedido é menor que o fim existente e o fim pedido é maior que o início existente. Criar uma reserva, evento ou bloqueio roda dentro de uma função `@Transaction` do Room que faz essa consulta e só grava se não houver conflito. Como o SQLite executa uma gravação por vez, verificar e gravar ficam atômicos. Os horários seguem em múltiplos de 30 minutos (RN01), por regra de negócio e para simplificar a tela.

**Alternativas consideradas.**
- Checar e gravar em duas etapas: simples, mas sujeito a condição de corrida.
- Tabela de blocos de 30 minutos com índice único (espaço, data e hora): garante a regra no próprio banco, mas duplica os dados e exige manter reserva e blocos sempre em sincronia.
- Validar só na tela: não protege contra chamadas fora da interface nem contra testes.

**Consequências.**
- Positivo: regra simples de explicar e de testar, com um teste que dispara duas tentativas paralelas no mesmo horário.
- Positivo: sem tabelas duplicadas, pois os relatórios leem direto de reservas e eventos.
- Negativo: a garantia depende de toda criação passar pela função transacional. Gravar direto pelo DAO burlaria a regra, então os DAOs de escrita ficam internos ao Repository.
- Se o app evoluir para dados compartilhados (Firebase ou API), esta decisão precisa ser refeita no servidor.

---

## ADR-004: Perfis de acesso por campo role verificado na camada de domínio

**Status:** Proposta

**Contexto.** O app tem três perfis com permissões diferentes. Esconder botões na interface não basta: qualquer tela ou teste que chame a regra diretamente poderia executar uma ação que o perfil não permite. Como não há servidor, a checagem precisa estar no código do app, em um único lugar.

**Decisão.** Cada usuário tem o campo role (aluno, organizador ou administrador) na tabela de usuários. Os casos de uso (por exemplo aprovar evento, bloquear espaço, promover usuário) recebem o usuário logado e recusam a operação, com um erro de permissão, se o role não for o exigido. A interface também esconde telas por perfil, mas só por usabilidade. O usuário nunca altera o próprio role, a promoção a organizador é feita por um administrador, e o primeiro administrador entra pelos dados iniciais do banco. As senhas são guardadas apenas como hash com salt.

**Alternativas consideradas.**
- Checar o perfil só na interface: simples, mas qualquer chamada fora da tela passa.
- Cada ViewModel com a sua própria checagem: espalha a regra e facilita esquecer uma ação.
- Regras no servidor (Firebase Security Rules ou API): mais seguro de verdade, mas exige back-end, fora do escopo atual.

**Consequências.**
- Positivo: permissão testável com testes unitários simples, por exemplo "aluno não aprova evento" e "usuário não muda o próprio role".
- Negativo: sem servidor, quem tiver acesso ao aparelho e ao arquivo do banco pode alterá-lo. A proteção vale para o uso normal do app, não contra um ataque.
- Negativo: a segurança depende de todo caso de uso chamar a checagem, por isso ela fica centralizada em uma classe de permissões.

---

## ADR-005: Ocupação calculada por horários e confirmação manual de presença

**Status:** Proposta

**Contexto.** O estado de um espaço (livre, reservado, evento em andamento, bloqueado) e a taxa de faltas poderiam vir de sensores, QR code ou câmera. Tudo isso está fora do escopo desta versão, então é preciso um jeito simples e testável de chegar ao mesmo resultado.

**Decisão.** O estado é calculado no app a partir das reservas, eventos e bloqueios do dia, com esta prioridade: bloqueado, evento em andamento, reservado, livre. O cálculo fica em uma classe pura que recebe os intervalos e o instante atual. Para as faltas, o aluno toca em "Confirmar presença" na janela da regra RN07 e o app grava a hora do aparelho. A reserva que termina sem confirmação é tratada como "não compareceu" na leitura, sem processo agendado.

**Alternativas consideradas.**
- QR code com câmera na porta da sala: mede presença de verdade, mas foi retirado do escopo.
- Geolocalização por GPS: exige permissão de localização e é imprecisa dentro de prédios.
- Sensores ou BLE: depende de hardware que o grupo não tem.
- Tarefa periódica em segundo plano (WorkManager) para marcar faltas: adiciona um processo a manter e não é necessária se a falta for derivada na leitura.

**Consequências.**
- Positivo: simples, sem permissões especiais e fácil de cobrir com testes unitários.
- Negativo: "ocupado" significa reservado, não presença real. Uma sala reservada e vazia aparece ocupada, e uma sala livre pode estar em uso sem reserva.
- Negativo: nada impede o aluno de confirmar presença sem estar no local. A limitação é aceita nesta versão e deve constar na apresentação.
- Como não há servidor, o relógio do aparelho define o resultado: mudar a hora do celular altera estados e faltas. A limitação é aceita nesta versão.

---

## ADR-006: Escopo de um único aparelho, sem sincronização

**Status:** Proposta

**Contexto.** Com os dados em Room (ADR-001), cada instalação do app tem o seu próprio banco. Reservas e eventos criados em um aparelho não aparecem em outro, e não há envio de notificações entre usuários.

**Decisão.** Assumir explicitamente o escopo de um aparelho. Os três perfis são demonstrados trocando de usuário (logout e login) na mesma instalação, e o app traz dados iniciais (espaços, um administrador, alunos e eventos de exemplo) para a demonstração. Não haverá sincronização, e as migrações do Room devem preservar os dados entre versões do app.

**Alternativas consideradas.**
- Sincronizar o Room com Firebase ou uma API: entrega dados compartilhados de verdade, mas é o item de maior custo e risco do projeto.
- Exportar e importar o banco por arquivo: permite levar os dados a outro aparelho, mas não resolve concorrência entre usuários.

**Consequências.**
- Positivo: escopo fechado, sem rede, que cabe no semestre e é fácil de demonstrar.
- Negativo: o app simula um campus em um só aparelho. A apresentação deve deixar isso claro e citar a evolução para dados compartilhados como trabalho futuro.
- Ponto em aberto: se o professor esperar dados compartilhados entre aparelhos, o ADR-001 deve ser revisto para Firebase ou API própria, e os ADR-003 e ADR-004 passam a valer no servidor.

---

## Consequências gerais, pendências e histórico

O conjunto das seis decisões resulta em um app de um único aparelho, com dados locais em Room e sem servidor. O risco técnico se concentra na transação de reservas (ADR-003) e na checagem de permissões por perfil (ADR-004), que devem ser prototipadas primeiro.

**Decisões adiadas**
- Lembretes locais (alarmes ou notificações) para reservas e eventos: fora do MVP.
- Evolução para dados compartilhados entre aparelhos, com Firebase ou API própria: fora do escopo atual e registrada no ADR-006.

**A confirmar com o professor**
- Se Kotlin com Compose é aceito na disciplina.
- Se um app de um único aparelho, sem dados compartilhados, atende ao esperado para o trabalho (afeta o ADR-006).

**Histórico**

| Data | Mudança |
|---|---|
| 2026-11-11 | Versão inicial, com as seis decisões como Proposta |
| 2026-11-11 | Decisão por Room (SQLite) no lugar do Firebase: ADR-001, 003, 004 e 006 reescritos, ADR-002 e 005 ajustados |
