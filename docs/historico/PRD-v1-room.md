# PRD — CampusHub (versão 1, histórica)

> **Esta versão está superada.** Ela descreve o escopo com banco local em Room e um único aparelho, de 2026-11-11. A versão vigente está em `docs/PRD.md`. Este arquivo é mantido apenas como registro da evolução do produto.

Autor: @Gaspa · 2026-11-11

## 1. Visão geral

O CampusHub é um app Android (Kotlin) que reúne em um só lugar a reserva de espaços do campus e a divulgação de eventos, mostrando para cada espaço se ele está livre, reservado ou ocupado por um evento.

**Problema.** Reservas de sala e eventos costumam correr por canais separados (planilhas, mensagens, pedidos informais). O resultado são choques de horário entre eventos e reservas, salas ociosas e pouca visibilidade do que acontece no campus.

**Proposta.** Um calendário único por espaço, com regras claras de uso, três perfis de acesso (aluno, organizador, administrador) e um painel de relatórios para a administração.

**Contexto do projeto.** Trabalho de semestre de desenvolvimento Android, feito em grupo. Os dados ficam em um banco local com Room (SQLite), em um único aparelho, e os três perfis são demonstrados trocando de usuário. Nesta versão não há câmera, QR code, sensores, IA nem servidor: o peso técnico está nas regras de negócio, no banco de dados e no controle de perfis.

## 2. Objetivos e não-objetivos

O objetivo central é que nenhum espaço tenha duas reservas ou eventos aprovados ao mesmo tempo, com cada perfil vendo apenas o que lhe cabe.

**Objetivos**

1. Impedir reservas e eventos conflitantes no mesmo espaço e horário.
2. Mostrar o estado atual de cada espaço (livre, reservado, evento em andamento, bloqueado) calculado só pelos horários.
3. Dar à administração um relatório de uso: espaços mais usados, horários de pico e taxa de faltas em reservas.
4. Entregar no semestre um fluxo completo e demonstrável: login, reserva, criação de evento, aprovação e relatório.

**Como saberemos que deu certo**

- Em teste que dispara duas tentativas paralelas de reservar o mesmo horário, só uma é criada.
- Cada perfil acessa apenas as telas e ações do seu papel, e a camada de domínio recusa ações sem permissão mesmo quando chamadas fora da tela.
- Todas as regras de negócio (seção 5) têm ao menos um teste unitário.

**Não-objetivos desta versão**

- Câmera, QR code, leitura de documentos ou reconhecimento de imagem.
- Sensores de ocupação ou integração com hardware do campus.
- Dados compartilhados entre aparelhos: servidor, Firebase ou sincronização.
- Pagamentos, chat entre usuários e integração com o sistema acadêmico da instituição.
- Versão web ou iOS.

## 3. Perfis de usuário

O app tem três perfis, e cada um enxerga telas e ações diferentes.

| Perfil | Quem é | O que pode fazer |
|---|---|---|
| Aluno | Usuário padrão, cadastrado com e-mail | Ver espaços e seu estado, reservar e cancelar, inscrever-se em eventos, confirmar presença na reserva |
| Organizador | Aluno ou entidade promovido pelo administrador | Tudo do aluno, mais criar e editar os próprios eventos e ver inscritos e lista de espera |
| Administrador | Equipe responsável pelos espaços | Cadastrar e bloquear espaços, aprovar ou rejeitar eventos, promover organizadores e consultar relatórios |

**Jornadas principais**

1. **Aluno:** abre a lista de espaços, filtra por tipo e horário, escolhe um espaço livre, reserva e recebe a confirmação. No horário, toca em "Confirmar presença".
2. **Organizador:** cria um evento (espaço, data, horário, vagas), aguarda a aprovação e depois acompanha inscritos e lista de espera.
3. **Administrador:** recebe o pedido de evento, vê se há conflito, aprova ou rejeita e consulta o relatório de uso.

## 4. Requisitos funcionais

São 22 requisitos em seis módulos. "Must" entra no MVP e "Should" entra se o prazo permitir.

| ID | Módulo | Requisito | Prioridade |
|---|---|---|---|
| RF01 | Autenticação | Cadastro e login com e-mail e senha | Must |
| RF02 | Autenticação | Redefinição de senha por um administrador | Should |
| RF03 | Autenticação | Todo usuário tem um perfil; novos cadastros entram como aluno | Must |
| RF04 | Espaços | Administrador cadastra, edita e desativa espaços (nome, tipo, capacidade, horário de funcionamento) | Must |
| RF05 | Espaços | Lista de espaços com filtro por tipo e busca por nome | Must |
| RF06 | Espaços | Administrador bloqueia um espaço por período, com motivo | Must |
| RF07 | Reservas | Aluno consulta os horários disponíveis de um espaço em um dia | Must |
| RF08 | Reservas | Aluno cria reserva escolhendo espaço, dia, início e fim | Must |
| RF09 | Reservas | Sistema recusa reserva que se sobreponha a outra reserva, evento ou bloqueio | Must |
| RF10 | Reservas | Sistema aplica os limites de uso da seção 5 | Must |
| RF11 | Reservas | Aluno lista e cancela as próprias reservas futuras | Must |
| RF12 | Reservas | Aluno confirma presença na janela de confirmação; sem confirmação a reserva vira "não compareceu" | Must |
| RF13 | Eventos | Organizador cria evento (título, descrição, espaço, data, horário, vagas) como pedido pendente | Must |
| RF14 | Eventos | Administrador aprova ou rejeita o evento, com motivo na rejeição | Must |
| RF15 | Eventos | Evento aprovado ocupa o espaço no horário e aparece na lista de eventos | Must |
| RF16 | Eventos | Aluno se inscreve e cancela a inscrição; se lotado, entra na lista de espera | Must |
| RF17 | Eventos | Ao abrir uma vaga, o primeiro da lista de espera é promovido automaticamente | Must |
| RF18 | Eventos | Organizador vê inscritos e lista de espera e pode cancelar o próprio evento | Should |
| RF19 | Ocupação | Cada espaço mostra "livre agora", "reservado até HH:MM", "evento em andamento" ou "bloqueado" | Must |
| RF20 | Painel admin | Relatório por período: espaços mais usados, horários de pico e taxa de faltas | Must |
| RF21 | Painel admin | Administrador promove um usuário a organizador | Must |
| RF22 | Painel admin | Administrador cancela reserva ou evento quando necessário, com motivo | Should |

## 5. Regras de negócio

Os valores abaixo são o padrão proposto e ficam em configuração, para o grupo ajustar sem mexer no código das regras.

| ID | Regra | Valor padrão |
|---|---|---|
| RN01 | Início e fim de qualquer reserva, evento ou bloqueio em múltiplos de 30 minutos | 30 min |
| RN02 | A reserva ou o evento deve caber no horário de funcionamento do espaço | definido por espaço |
| RN03 | Duração de uma reserva de aluno | mínimo 30 min, máximo 2 h |
| RN04 | Reservas ativas (futuras e não canceladas) por aluno na mesma semana, de segunda a domingo | 2 |
| RN05 | Antecedência para reservar | mínimo 1 h, máximo 14 dias |
| RN06 | O aluno cancela a própria reserva até certo tempo antes do início; depois disso só o administrador cancela | 1 h antes |
| RN07 | Confirmação de presença: o aluno confirma na janela ao redor do início; sem confirmação a reserva vira "não compareceu" | de 15 min antes a 15 min depois |
| RN08 | Reserva, evento (pendente ou aprovado) e bloqueio ocupam o espaço e não podem se sobrepor; rejeição ou cancelamento libera o horário | sem sobreposição |
| RN09 | O evento exige antecedência mínima para dar tempo de aprovação | 3 dias |
| RN10 | Vagas de um evento não passam da capacidade do espaço; evento aprovado não muda de espaço nem de horário, só é cancelado e recriado | capacidade do espaço |
| RN11 | Um aluno se inscreve uma vez por evento; com o evento lotado, entra na lista de espera por ordem de chegada e é promovido quando abrir vaga | 1 inscrição por evento |
| RN12 | Ao bloquear um espaço, o sistema lista as reservas afetadas e o administrador confirma o cancelamento delas, com motivo | confirmação obrigatória |
| RN13 | Taxa de faltas = reservas "não compareceu" divididas pelas reservas já encerradas no período, sem contar as canceladas | por período |

## 6. Requisitos não funcionais

| Categoria | Requisito |
|---|---|
| Plataforma | Android 8.0 (API 26) ou superior, em Kotlin com Jetpack Compose e banco local Room (SQLite) |
| Desempenho | Lista de espaços e seus estados aparece em até 2 s; a tela atualiza sozinha quando os dados mudam (Flow) |
| Segurança | Senha guardada só como hash com salt; permissões por perfil verificadas na camada de domínio; usuário não altera o próprio perfil |
| Usabilidade | Material 3, modo claro e escuro e mensagens de erro claras, como "horário já ocupado" |
| Confiabilidade | Funciona sem internet; as migrações do Room preservam os dados entre versões do app |
| Manutenibilidade | MVVM, pacotes por funcionalidade e regras de negócio em classes puras, sem dependência do Android |
| Testes | Testes unitários das regras RN01 a RN13, testes dos DAOs com banco em memória e um teste de concorrência com duas tentativas paralelas no mesmo horário |
| Privacidade | Guardar apenas nome e e-mail do usuário, no próprio aparelho |

## 7. Modelo de dados

São seis tabelas no Room. Um espaço tem muitas reservas, eventos e bloqueios, e um evento tem muitas inscrições. A regra de não sobreposição vem de uma consulta dentro de uma transação, que olha reservas, eventos e bloqueios do mesmo espaço (ADR-003).

| Tabela | Campos principais | Observação |
|---|---|---|
| users | id, nome, email, senhaHash, salt, role, criadoEm | role: aluno, organizador ou administrador; e-mail único |
| spaces | id, nome, tipo, capacidade, abre, fecha, ativo | abre e fecha definem o horário de funcionamento |
| reservations | id, spaceId, userId, inicio, fim, status, presencaEm, criadoEm | status: ativa ou cancelada; "compareceu" e "não compareceu" derivam de presencaEm |
| events | id, spaceId, organizerId, titulo, descricao, inicio, fim, vagas, status, motivoRejeicao | status: pendente, aprovado, rejeitado ou cancelado |
| registrations | id, eventId, userId, status, criadoEm | status: inscrito, espera ou cancelado; a ordem da espera vem de criadoEm |
| blocks | id, spaceId, inicio, fim, motivo, criadoPor | bloqueio de manutenção feito pelo administrador |

## 8. Mapa de telas

| Perfil | Telas |
|---|---|
| Todos | Login e cadastro · Lista de espaços com estado · Detalhe do espaço com a grade do dia · Lista de eventos · Detalhe do evento · Perfil |
| Aluno | Nova reserva · Minhas reservas (cancelar e confirmar presença) · Minhas inscrições |
| Organizador | Criar e editar evento · Meus eventos (inscritos e lista de espera) |
| Administrador | Gerenciar espaços · Bloquear espaço · Pedidos de evento (aprovar e rejeitar) · Usuários e promoção · Relatórios |

## 9. Escopo do MVP e cronograma

**MVP (obrigatório)**

- Login e três perfis (RF01, RF03, RF21)
- Espaços e bloqueios (RF04 a RF06)
- Reservas com todas as regras da seção 5 (RF07 a RF12)
- Eventos com aprovação, inscrição e lista de espera (RF13 a RF17)
- Estado do espaço (RF19) e relatório do administrador (RF20)

**Se sobrar tempo**

- Recuperação de senha (RF02), gestão do evento pelo organizador (RF18) e cancelamento pelo administrador (RF22)
- Notificações push de vaga aberta e de mudança de evento
- Calendário semanal visual por espaço
- Exportar o relatório em PDF ou CSV

O cronograma assume 8 semanas e um grupo que trabalha em paralelo. Ajuste ao prazo real da disciplina.

| Fase | Semanas | Entregável |
|---|---|---|
| Fundação e login | 1 | Projeto, login e cadastro |
| Espaços e reservas | 2 a 4 | Espaços, bloqueios e reservas com as regras |
| Eventos | 3 a 5 | Criação, aprovação, inscrição e lista de espera |
| Ocupação e presença | 4 a 5 | Estado do espaço e confirmação de presença |
| Painel do administrador | 5 a 6 | Gestão de espaços, promoção e relatórios |
| Testes e regras de negócio | 3 a 7 | Testes unitários, de DAO e de concorrência |
| Entrega e demonstração | 8 | Demonstração e apresentação |

O MVP fica pronto ao fim da semana 6, com duas semanas para testes e entrega.

## 10. Critérios de aceite e testes

O MVP está aceito quando os dez cenários abaixo passam na demonstração.

| ID | Cenário | Resultado esperado |
|---|---|---|
| CA01 | Duas tentativas de reservar o mesmo horário, em paralelo no teste | Só uma reserva é criada e a outra recebe "horário já ocupado" (RF09, RN08) |
| CA02 | Aluno com duas reservas na semana tenta uma terceira | A reserva é recusada com a mensagem do limite (RN04) |
| CA03 | Reserva fora do horário de funcionamento do espaço | A reserva é recusada (RN02) |
| CA04 | Aluno cancela com mais de 1 h de antecedência | A reserva é cancelada e o horário volta a ficar livre (RN06) |
| CA05 | Aluno confirma presença, ou deixa passar a janela | Com confirmação a reserva conta como compareceu; sem ela, como não compareceu e entra na taxa de faltas (RN07, RN13) |
| CA06 | Organizador cria evento em horário livre e o administrador rejeita | O evento pendente ocupa o horário e, ao ser rejeitado, o libera (RN08) |
| CA07 | Evento lotado recebe nova inscrição e depois um inscrito cancela | O novo aluno entra na lista de espera e o primeiro da fila é promovido (RN11) |
| CA08 | Aluno tenta aprovar evento ou mudar o próprio perfil pelos casos de uso | A operação é recusada com erro de permissão (ADR-004) |
| CA09 | Espaço com reserva em andamento, evento aprovado ou bloqueio | A lista mostra "reservado até HH:MM", "evento em andamento" ou "bloqueado" (RF19) |
| CA10 | Administrador abre o relatório do mês | Vê espaços mais usados, horários de pico e taxa de faltas coerentes com os dados de teste (RF20) |

**Plano de testes**

- Testes unitários para cada regra de negócio e para o cálculo do estado do espaço.
- Testes de permissão por perfil nos casos de uso, cobrindo CA08.
- Testes dos DAOs com banco em memória e um teste com duas tentativas paralelas no mesmo horário, cobrindo CA01.

## 11. Riscos e mitigações

Probabilidade e impacto são estimativas iniciais para o grupo revisar.

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| A consulta de sobreposição e a transação de reservas são mais complexas que o previsto | Média | Alto | Prototipar na semana 2, antes de construir as telas |
| Alguma ação esquecer a checagem de permissão por perfil | Média | Alto | Centralizar a checagem em uma classe e testar cada ação com cada perfil |
| O escopo cresce com notificações, calendário visual e outros extras | Alta | Médio | Congelar o MVP e só abrir extras depois da semana 6 |
| O professor esperar dados compartilhados entre aparelhos ou um servidor | A confirmar | Alto | Perguntar nas primeiras semanas; o ADR-006 descreve a evolução para Firebase ou API própria |
| Divisão desigual do trabalho no grupo | Média | Médio | Dividir por módulos (login e perfis, reservas, eventos, painel) e revisar o código uns dos outros |
| A demonstração parecer simples por ter um aparelho só | Média | Médio | Trocar de usuário ao vivo para mostrar os três perfis, usar dados de exemplo e demonstrar o teste de concorrência |
