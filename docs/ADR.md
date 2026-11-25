# ADR — CampusHub

Autor: @Gaspa · Atualizado em 2026-11-25

## Contexto e índice

Este documento registra as decisões arquiteturais do CampusHub, o app Android de reservas de espaços e eventos do campus descrito no PRD. Cada decisão segue o formato contexto, decisão, alternativas e consequências.

Premissas confirmadas:
- A disciplina aceita Kotlin com Jetpack Compose.
- Os dados são compartilhados entre aparelhos, com servidor hospedado no Firebase (Auth + Cloud Firestore).

| ID | Decisão | Status |
|---|---|---|
| ADR-001 | Backend Firebase: Firebase Auth + Cloud Firestore | Aceita |
| ADR-002 | Kotlin, Jetpack Compose e MVVM com Hilt | Aceita |
| ADR-003 | Conflito de reservas evitado por transação com blocos de 30 min no Firestore | Proposta |
| ADR-004 | Perfis de acesso por campo `role`, verificado no domínio e nas Security Rules | Proposta |
| ADR-005 | Ocupação calculada por horários e confirmação de presença com horário do servidor | Proposta |
| ADR-006 | Escopo com dados compartilhados entre aparelhos | Aceita |

---

## ADR-001: Backend Firebase (Firebase Auth + Cloud Firestore)

**Status:** Aceita (substitui a proposta anterior de banco local com Room)

**Contexto.** O CampusHub precisa que reservas e eventos criados em um aparelho apareçam em outros, com autenticação de usuários e atualização em tempo real. Um banco local (Room) não atende a esse requisito. Uma API própria exigiria servidor, deploy e manutenção fora do escopo do semestre. O Firebase oferece autenticação e banco em tempo real sem servidor próprio, com SDK nativo para Android e plano gratuito adequado a um trabalho acadêmico.

**Decisão.**
- **Firebase Authentication** (e-mail e senha) para cadastro e login (RF01). O Firebase guarda a senha com hash e salt, então o app não armazena senha nem salt.
- **Cloud Firestore** como banco principal, com as coleções descritas no PRD (seção 7):
  - `users/{uid}`: nome, e-mail, `role`, `criadoEm`. O `uid` vem do Firebase Auth.
  - `spaces/{spaceId}`: dados do espaço e horário de funcionamento.
  - `spaces/{spaceId}/slots/{yyyyMMddHHmm}`: um documento por bloco de 30 minutos ocupado (ver ADR-003).
  - `reservations/{id}`, `events/{id}`, `events/{id}/registrations/{uid}`, `blocks/{id}`.
- **Flows**: os repositórios expõem os dados como `Flow` a partir de listeners do Firestore (`callbackFlow`), e as telas reagem às mudanças em tempo real (RNF de desempenho).
- **Security Rules** no servidor validam perfil, dono do documento e janelas de tempo (RN06 e RN07), além das checagens no domínio (ADR-004).
- **Cache offline do Firestore**: leituras funcionam sem internet. Escritas que dependem de transação (reservas, eventos, inscrições) exigem conexão.

**Alternativas consideradas.**
- **Room (SQLite) local:** rejeitada, pois os dados ficariam em um único aparelho. Era a proposta anterior.
- **API própria com Spring ou Node e PostgreSQL:** dados compartilhados e SQL de verdade, mas soma servidor, deploy e manutenção ao escopo.
- **Supabase ou outro BaaS:** também atende, mas o grupo optou pelo Firebase por familiaridade com o ecossistema Android.

**Consequências.**
- Positivo: dados compartilhados em tempo real entre aparelhos, sem servidor próprio, com autenticação pronta.
- Positivo: a demonstração pode usar dois aparelhos ou emuladores ao mesmo tempo, o que mostra o fluxo completo.
- Negativo: o Firestore é NoSQL e não tem JOIN nem GROUP BY. Os relatórios (RF20) precisam ser calculados no app a partir de consultas por período, ou com um documento de agregação.
- Negativo: não existe consulta de intervalo dentro de transação no SDK Android, o que obriga o modelo de blocos (ADR-003).
- Negativo: escrita offline não funciona, então a tela precisa mostrar mensagem clara quando não houver conexão.

---

## ADR-002: Kotlin, Jetpack Compose e MVVM com Hilt

**Status:** Aceita (a disciplina confirmou o uso de Kotlin e Compose)

**Contexto.** Três perfis geram muitas telas e estados, e as regras de negócio (limites, conflitos, lista de espera) precisam ser testadas sem emulador.

**Decisão.** Kotlin com Jetpack Compose e Navigation Compose. Arquitetura MVVM em camadas:
- **Tela (Compose)** → **ViewModel** → **Repository** → **fonte de dados Firebase** (Firestore e Auth).
- **Hilt** para injeção de dependências.
- **Coroutines e Flow** para dados em tempo real.
- As regras de negócio ficam em **classes de domínio puras** (sem dependência do Android nem do Firebase), testáveis com JUnit.

**Alternativas consideradas.**
- Views em XML com Fragments: mais código de tela e de estado.
- Java: Kotlin é o padrão atual e simplifica Flow e coroutines.
- Clean Architecture completa ou MVI: mais cerimônia do que o prazo comporta.
- Lógica dentro de Activities ou Composables: difícil de testar e de dividir entre o grupo.

**Consequências.**
- Positivo: regras testáveis, responsabilidades claras e estado reativo que combina com os Flows do Firestore.
- Negativo: curva de aprendizado de Compose, Hilt e Firebase para quem ainda não usou.
- Mitigação: um único módulo do app, com pacotes por funcionalidade (`auth`, `spaces`, `reservations`, `events`, `admin`, `domain`).

---

## ADR-003: Conflito de reservas evitado por transação com blocos de 30 min

**Status:** Proposta

**Contexto.** Uma reserva, evento ou bloqueio não pode se sobrepor a outro no mesmo espaço (RN08). Com vários aparelhos gravando ao mesmo tempo, checar disponibilidade e gravar em duas etapas deixa uma janela de condição de corrida. O Firestore não tem restrição de unicidade nem consulta de intervalo confiável dentro de transação no SDK Android.

**Decisão.** Ocupação modelada por **blocos de 30 minutos com ID determinístico**:
- Cada bloco ocupado vira um documento em `spaces/{spaceId}/slots/{yyyyMMddHHmm}`, com o tipo (`reserva`, `evento` ou `bloqueio`) e o ID do item de origem.
- Criar uma reserva, evento ou bloqueio roda em uma `runTransaction`:
  1. Calcula todos os IDs de blocos do intervalo (início a fim, de 30 em 30 min).
  2. Lê cada bloco com `get` por ID.
  3. Se algum já existir, lança erro de domínio "horário já ocupado" e não grava nada.
  4. Caso contrário, grava os blocos e o documento principal (reserva, evento ou bloqueio).
- Cancelamento ou rejeição roda em transação que remove os blocos do item.
- Como a leitura de cada bloco é por ID, a transação falha e tenta de novo quando dois aparelhos disputam o mesmo horário. Só uma gravação passa (CA01).
- RN01 é validado antes da transação: início e fim em múltiplos de 30 min.

**Os demais controles usam o mesmo padrão de documento com ID conhecido:**
- **RN04 (limite semanal):** documento `users/{uid}/weeks/{yyyyWW}` com contador de reservas ativas, incrementado na mesma transação.
- **RN11 (uma inscrição por evento):** a inscrição é `events/{eventId}/registrations/{uid}`. Como o ID é o `uid`, não há duas inscrições.
- **RN10 (vagas):** contador `inscritosCount` no evento, atualizado na transação de inscrição.
- **RF17 (promoção da fila):** o candidato é achado com consulta ordenada por `criadoEm` fora da transação. A transação revalida que ele ainda está em `espera` e que há vaga. Se não estiver, a transação falha e a consulta é refeita.

**Alternativas consideradas.**
- **Checar e gravar em duas etapas:** simples, mas sujeito a condição de corrida.
- **Consulta de sobreposição dentro da transação:** não é confiável no SDK cliente Android.
- **Cloud Function com Admin SDK e transação com consulta de intervalo:** a opção mais robusta, pois a validação roda no servidor. Exige plano Blaze para deploy. Fica como evolução, se o tempo permitir.
- **Validar só na tela:** não protege contra chamadas diretas nem contra testes.

**Consequências.**
- Positivo: a regra de não sobreposição é atômica e tem teste de concorrência simples (duas coroutines no emulador, mesmo horário).
- Positivo: RN04, RN10 e RN11 seguem o mesmo padrão, então o grupo aprende uma única forma de fazer as regras.
- Negativo: há dados duplicados (os blocos) que precisam ficar sincronizados com reservas e eventos. Toda escrita passa pelo Repository, e os testes cobrem a sincronização.
- Negativo: como a validação roda no cliente, um cliente modificado poderia gravar blocos diretamente. As Security Rules validam formato e dono, mas não conseguem checar sobreposição. Isso fica registrado como limitação aceita, a menos que a Cloud Function seja adotada.
- Negativo: os blocos crescem com o uso. Para o volume de um campus de teste, isso é irrelevante.

---

## ADR-004: Perfis de acesso por campo `role`, verificado no domínio e nas Security Rules

**Status:** Proposta

**Contexto.** O app tem três perfis com permissões diferentes. Esconder botões na interface não basta, e uma checagem só no cliente pode ser contornada. Com servidor Firebase, as regras podem ser aplicadas no próprio banco.

**Decisão.** Duas camadas de verificação:
1. **Domínio (app):** cada caso de uso recebe o usuário logado e recusa a operação com erro de permissão se o `role` não for o exigido. Todas as checagens ficam em uma classe `PermissionPolicy`.
2. **Security Rules (servidor):** repetem as regras essenciais, para que uma chamada fora do app também seja recusada.

Regras de perfil:
- Cada usuário tem o campo `role` (`aluno`, `organizador` ou `administrador`) em `users/{uid}`.
- O usuário **nunca altera o próprio `role`**. A regra de update de `users/{uid}` bloqueia qualquer mudança no campo `role` feita pelo próprio dono.
- Só o administrador altera o `role` de outros usuários (promoção, RF21).
- O primeiro administrador é criado manualmente no console do Firebase, definindo `role = administrador` no documento do `uid` dele.
- Senhas são gerenciadas pelo Firebase Auth (hash com salt). O app e o Firestore não guardam senha.
- Redefinição de senha é feita pelo fluxo de e-mail do Firebase Auth (RF02, Should).

**Alternativas consideradas.**
- **Checar só na interface:** qualquer chamada fora da tela passa.
- **Checagem em cada ViewModel:** espalha a regra e facilita esquecer uma ação.
- **Custom claims no token (`role` no token JWT via Admin SDK):** mais seguro, pois o perfil vem assinado. Exige Cloud Function para definir a claim. Fica como evolução.

**Consequências.**
- Positivo: permissão testável com testes unitários (`aluno não aprova evento`, `usuário não muda o próprio role`) e com testes das Security Rules no emulador.
- Positivo: a segurança deixa de depender só do código do app.
- Negativo: as Security Rules precisam ser mantidas em paralelo com a `PermissionPolicy`. Qualquer divergência entre as duas deve ser pega pelos testes.
- Negativo: a leitura de `role` nas regras gera uma consulta extra ao documento do usuário, o que é aceitável para o volume do projeto.

---

## ADR-005: Ocupação calculada por horários e confirmação de presença com horário do servidor

**Status:** Proposta

**Contexto.** O estado de um espaço (livre, reservado, evento em andamento, bloqueado) e a taxa de faltas (RN13) precisam ser derivados sem sensores nem QR code, que estão fora do escopo. Com o Firebase, o relógio do aparelho deixa de ser a única referência, pois o servidor pode registrar horários.

**Decisão.**
- O estado do espaço é calculado no app a partir dos blocos e itens do dia, com esta prioridade: bloqueado, evento em andamento, reservado, livre. O cálculo fica em uma classe pura que recebe os intervalos e o instante atual (RF19).
- Para a presença, o aluno toca em "Confirmar presença" na janela de RN07 (15 min antes até 15 min depois do início). O app grava `presencaEm` com `FieldValue.serverTimestamp()`, então o horário vem do servidor.
- Uma Security Rule valida a janela usando `request.time`, impedindo confirmação fora dela.
- Uma reserva cuja janela de confirmação já passou, sem `presencaEm`, é lida como "não compareceu". Não há processo agendado.
- O cancelamento pelo aluno (RN06) também é validado pela regra com `request.time`, comparando com `inicio`.

**Alternativas consideradas.**
- **Cloud Function agendada para marcar faltas:** precisa de plano Blaze e um processo a manter. Não é necessária se a falta for derivada na leitura.
- **QR code, GPS ou sensores:** fora do escopo.
- **Relógio do aparelho para tudo:** mais simples, mas o celular pode alterar o horário para confirmar presença fora da janela.

**Consequências.**
- Positivo: a janela de confirmação e a de cancelamento são impostas pelo servidor, o que reduz a fraude por relógio.
- Positivo: sem processo agendado, sem custo extra.
- Negativo: "ocupado" continua significando reservado, não presença real. Uma sala reservada e vazia aparece como ocupada.
- Negativo: nada impede o aluno de confirmar presença sem estar no local. A limitação é aceita e deve constar na apresentação.
- Negativo: o estado exibido na tela usa o relógio do aparelho para decidir "agora". Diferenças pequenas de horário são aceitáveis para o caso de uso.

---

## ADR-006: Escopo com dados compartilhados entre aparelhos

**Status:** Aceita (substitui a versão anterior de escopo de aparelho único)

**Contexto.** O grupo decidiu que os dados precisam ser compartilhados entre aparelhos. Reservas e eventos criados em um celular devem aparecer em outros, com notificações e atualização em tempo real como evolução.

**Decisão.**
- Os dados ficam no Firebase (ADR-001). Todos os aparelhos que se conectam ao mesmo projeto veem o mesmo campus.
- A demonstração usa dois aparelhos ou emuladores, com perfis diferentes logados ao mesmo tempo. Isso mostra o fluxo completo: aluno reserva, organizador cria evento, administrador aprova.
- Dados iniciais (espaços, um administrador, alunos e eventos de exemplo) são carregados por um script de seed com o Admin SDK, ou manualmente no console.
- O plano Spark (gratuito) é suficiente para o volume do projeto. Cloud Functions, se usadas, exigem o plano Blaze, que também tem camada gratuita generosa.
- Notificações push (FCM) para vaga aberta e mudança de evento ficam como extra, se o prazo permitir.

**Alternativas consideradas.**
- **Manter o escopo de aparelho único com Room:** descartado pelo grupo.
- **Exportar e importar banco por arquivo:** não resolve concorrência entre usuários.

**Consequências.**
- Positivo: demonstração mais forte, com dados reais compartilhados e teste de concorrência acontecendo entre aparelhos.
- Positivo: as premissas que estavam em aberto foram confirmadas, então o projeto pode seguir sem bloqueios de definição.
- Negativo: escrita exige internet. A tela precisa mostrar erro claro quando o aparelho estiver offline.
- Negativo: a segurança depende das Security Rules bem escritas, e erros nelas expõem dados. Testes no emulador são obrigatórios antes da entrega.
- Negativo: depender de internet no dia da apresentação. Mitigação: ter o emulador configurado como plano B.

---

## Consequências gerais, pendências e histórico

**Riscos técnicos concentrados** em:
1. Transação de blocos de reservas (ADR-003), que deve ser prototipada primeiro.
2. Security Rules e checagem de permissões (ADR-004), que devem ser testadas no emulador.

**Pendências para o grupo:**
- Decidir se a promoção da fila (RF17) e a validação de sobreposição vão para Cloud Functions (plano Blaze) ou ficam no cliente com as transações descritas.
- Revisar e aprovar ADR-003, ADR-004 e ADR-005 para mudar o status para Aceita.
- Definir quem cria e mantém as Security Rules e o script de seed.

**Decisões adiadas:**
- Notificações push (FCM) para vaga aberta e mudança de evento.
- Custom claims de perfil, via Cloud Function.
- Cloud Function para validação de sobreposição no servidor.

**Histórico**

| Data | Mudança |
|---|---|
| 2026-11-11 | Versão inicial, com as seis decisões como Proposta, usando Room e escopo de aparelho único |
| 2026-11-25 | Backend trocado de Room para Firebase (Auth + Firestore). ADR-001 e ADR-006 aceitos. ADR-002 aceita (Kotlin e Compose confirmados). ADR-003, ADR-004 e ADR-005 reescritos para o Firestore |
