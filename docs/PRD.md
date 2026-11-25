# PRD — CampusHub

Autor: @Gaspa · Atualizado em 2026-11-25

## 1. Visão geral

O CampusHub é um app Android (Kotlin, Jetpack Compose) que reúne em um só lugar a reserva de espaços do campus e a divulgação de eventos, mostrando para cada espaço se ele está livre, reservado ou ocupado por um evento.

**Problema.** Reservas de sala e eventos costumam correr por canais separados (planilhas, mensagens, pedidos informais). O resultado são choques de horário entre eventos e reservas, salas ociosas e pouca visibilidade do que acontece no campus.

**Proposta.** Um calendário único por espaço, com regras claras de uso, três perfis de acesso (aluno, organizador, administrador) e um painel de relatórios para a administração.

**Contexto do projeto.** Trabalho de semestre de desenvolvimento Android, feito em grupo. Os dados ficam em um servidor Firebase (Firebase Auth para login e Cloud Firestore para os dados), compartilhados entre todos os aparelhos. O peso técnico está nas regras de negócio, na consistência das reservas (sem sobreposição em concorrência), e no controle de perfis, verificado no app e nas regras de segurança do servidor.

## 2. Objetivos e não-objetivos

O objetivo central é que nenhum espaço tenha duas reservas ou eventos aprovados ao mesmo tempo, com cada perfil vendo apenas o que lhe cabe, em dados compartilhados entre aparelhos.

### Objetivos

1. Impedir reservas e eventos conflitantes no mesmo espaço e horário, mesmo com dois usuários tentando ao mesmo tempo.
2. Mostrar o estado atual de cada espaço (livre, reservado, evento em andamento, bloqueado) calculado pelos horários.
3. Dar à administração um relatório de uso: espaços mais usados, horários de pico e taxa de faltas em reservas.
4. Entregar no semestre um fluxo completo e demonstrável em dois aparelhos: login, reserva, criação de evento, aprovação e relatório.

### Como saberemos que deu certo

- Em teste que dispara duas tentativas paralelas de reservar o mesmo horário, só uma é criada.
- Cada perfil acessa apenas as telas e ações do seu papel, e a camada de domínio e as regras de segurança recusam ações sem permissão, mesmo quando chamadas fora da tela.
- Um aparelho cria uma reserva e outro aparelho a vê em tempo real.

### Não-objetivos desta versão

- Câmera, QR code, leitura de documentos ou reconhecimento de imagem.
- Sensores de ocupação ou integração com hardware do campus.
- Pagamentos, chat entre usuários e integração com o sistema acadêmico da instituição.
- Versão web ou iOS.
- Servidor próprio (API). O backend é o Firebase.

## 3. Perfis de usuário

O app tem três perfis, e cada um enxerga telas e ações diferentes.

| Perfil | Quem é | O que pode fazer |
|---|---|---|
| Aluno | Usuário padrão, cadastrado com e-mail | Ver espaços e seu estado, reservar e cancelar, inscrever-se em eventos, confirmar presença na reserva |
| Organizador | Aluno ou entidade promovido pelo administrador | Tudo do aluno, mais criar e editar os próprios eventos e ver inscritos e lista de espera |
| Administrador | Equipe responsável pelos espaços | Cadastrar e bloquear espaços, aprovar ou rejeitar eventos, promover organizadores e consultar relatórios |

### Jornadas principais

1. **Aluno:** abre a lista de espaços, filtra por tipo e horário, escolhe um espaço livre, reserva e recebe a confirmação. No horário, toca em "Confirmar presença".
2. **Organizador:** cria um evento (espaço, data, horário, vagas), aguarda a aprovação e depois acompanha inscritos e lista de espera.
3. **Administrador:** recebe o pedido de evento, vê se há conflito, aprova ou rejeita e consulta o relatório de uso.

## 4. Requisitos funcionais

São 22 requisitos em seis módulos. "Must" entra no MVP e "Should" entra se o prazo permitir.

| ID | Módulo | Requisito | Prioridade |
|---|---|---|---|
| RF01 | Autenticação | Cadastro e login com e-mail e senha (Firebase Auth) | Must |
| RF02 | Autenticação | Redefinição de senha por e-mail (fluxo do Firebase Auth) | Should |
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

Os valores abaixo são o padrão proposto e ficam em configuração (arquivo `AppConfig` no código, ou documento `config/regras` no Firestore), para o grupo ajustar sem mexer na lógica.

| ID | Regra | Valor padrão | Onde é aplicada |
|---|---|---|---|
| RN01 | Início e fim de qualquer reserva, evento ou bloqueio em múltiplos de 30 minutos | 30 min | Domínio |
| RN02 | A reserva ou o evento deve caber no horário de funcionamento do espaço | definido por espaço | Domínio |
| RN03 | Duração de uma reserva de aluno | mínimo 30 min, máximo 2 h | Domínio |
| RN04 | Reservas ativas (futuras e não canceladas) por aluno na mesma semana, de segunda a domingo | 2 | Transação (contador semanal) |
| RN05 | Antecedência para reservar | mínimo 1 h, máximo 14 dias | Domínio e Security Rules |
| RN06 | O aluno cancela a própria reserva até certo tempo antes do início; depois disso só o administrador cancela | 1 h antes | Domínio e Security Rules (`request.time`) |
| RN07 | Confirmação de presença: o aluno confirma na janela ao redor do início; sem confirmação a reserva vira "não compareceu" | de 15 min antes a 15 min depois | Domínio e Security Rules (`request.time`) |
| RN08 | Reserva, evento (pendente ou aprovado) e bloqueio ocupam o espaço e não podem se sobrepor; rejeição ou cancelamento libera o horário | sem sobreposição | Transação com blocos de 30 min |
| RN09 | O evento exige antecedência mínima para dar tempo de aprovação | 3 dias | Domínio e Security Rules |
| RN10 | Vagas de um evento não passam da capacidade do espaço; evento aprovado não muda de espaço nem de horário, só é cancelado e recriado | capacidade do espaço | Transação (contador de inscritos) |
| RN11 | Um aluno se inscreve uma vez por evento; com o evento lotado, entra na lista de espera por ordem de chegada e é promovido quando abrir vaga | 1 inscrição por evento | ID do documento (`uid`) e transação |
| RN12 | Ao bloquear um espaço, o sistema lista as reservas afetadas e o administrador confirma o cancelamento delas, com motivo | confirmação obrigatória | Domínio |
| RN13 | Taxa de faltas = reservas "não compareceu" divididas pelas reservas já encerradas no período, sem contar as canceladas | por período | Domínio (leitura) |

## 6. Requisitos não funcionais

| Categoria | Requisito |
|---|---|
| Plataforma | Android 8.0 (API 26) ou superior, em Kotlin com Jetpack Compose, e backend Firebase (Auth e Cloud Firestore) |
| Desempenho | Lista de espaços e seus estados aparece em até 2 s; a tela atualiza sozinha quando os dados mudam (listeners do Firestore em Flow) |
| Segurança | Senhas gerenciadas pelo Firebase Auth (hash com salt); permissões por perfil verificadas no domínio e nas Security Rules; usuário não altera o próprio perfil; o app não guarda senha |
| Usabilidade | Material 3, modo claro e escuro e mensagens de erro claras, como "horário já ocupado" |
| Confiabilidade | Leitura funciona sem internet (cache offline do Firestore). Reserva, evento e inscrição exigem conexão e mostram erro claro quando offline |
| Manutenibilidade | MVVM, pacotes por funcionalidade e regras de negócio em classes puras, sem dependência do Android nem do Firebase |
| Testes | Testes unitários das regras RN01 a RN13; testes de repositórios e de Security Rules com o Firebase Emulator Suite; teste de concorrência com duas tentativas paralelas no mesmo horário |
| Privacidade | Guardar apenas nome e e-mail do usuário, além do perfil e dos dados de uso (reservas e inscrições) |

## 7. Modelo de dados

Firestore com coleções de nível superior e subcoleções. Um espaço tem muitas reservas, eventos e bloqueios, e um evento tem muitas inscrições. A regra de não sobreposição usa blocos de 30 min com ID determinístico (ADR-003).

| Caminho | Campos principais | Observação |
|---|---|---|
| `users/{uid}` | nome, email, role, criadoEm | `role`: aluno, organizador ou administrador. O `uid` vem do Firebase Auth. O usuário não altera `role` |
| `users/{uid}/weeks/{yyyyWW}` | reservasAtivas | Contador para RN04, atualizado na transação de reserva |
| `spaces/{spaceId}` | nome, tipo, capacidade, abre, fecha, ativo | `abre` e `fecha` definem o horário de funcionamento |
| `spaces/{spaceId}/slots/{yyyyMMddHHmm}` | tipo (reserva, evento ou bloqueio), origemId | Um documento por bloco de 30 min ocupado. Usado só para a regra de sobreposição |
| `reservations/{id}` | spaceId, userId, inicio, fim, status, presencaEm, criadoEm | `status`: ativa ou cancelada. "compareceu" e "não compareceu" derivam de `presencaEm` e da janela |
| `events/{id}` | spaceId, organizerId, titulo, descricao, inicio, fim, vagas, inscritosCount, status, motivoRejeicao | `status`: pendente, aprovado, rejeitado ou cancelado |
| `events/{id}/registrations/{uid}` | status, criadoEm | `status`: inscrito, espera ou cancelado. O ID é o `uid`, o que garante uma inscrição por evento. A ordem da espera vem de `criadoEm` |
| `blocks/{id}` | spaceId, inicio, fim, motivo, criadoPor | Bloqueio de manutenção feito pelo administrador |
| `config/regras` | valores das regras RN03, RN04, RN05, RN06, RN07, RN09 | Opcional: permite ajuste sem alterar o código |

## 8. Mapa de telas

| Perfil | Telas |
|---|---|
| Todos | Login e cadastro · Lista de espaços com estado · Detalhe do espaço com a grade do dia · Lista de eventos · Detalhe do evento · Perfil |
| Aluno | Nova reserva · Minhas reservas (cancelar e confirmar presença) · Minhas inscrições |
| Organizador | Criar e editar evento · Meus eventos (inscritos e lista de espera) |
| Administrador | Gerenciar espaços · Bloquear espaço · Pedidos de evento (aprovar e rejeitar) · Usuários e promoção · Relatórios |

## 9. Escopo do MVP e cronograma

### MVP (obrigatório)

- Login e três perfis (RF01, RF03, RF21)
- Espaços e bloqueios (RF04 a RF06)
- Reservas com todas as regras da seção 5 (RF07 a RF12)
- Eventos com aprovação, inscrição e lista de espera (RF13 a RF17)
- Estado do espaço (RF19) e relatório do administrador (RF20)
- Security Rules com testes no emulador

### Se sobrar tempo

- Gestão do evento pelo organizador (RF18) e cancelamento pelo administrador (RF22)
- Notificações push de vaga aberta e de mudança de evento (FCM)
- Calendário semanal visual por espaço
- Exportar o relatório em PDF ou CSV
- Cloud Functions para validação de sobreposição no servidor (ADR-003)

O cronograma abaixo assume 8 semanas e um grupo que trabalha em paralelo. Ajuste ao prazo real da disciplina.

| Fase | Semanas | Entregável |
|---|---|---|
| Fundação e login | 1 | Projeto Android com Compose, Hilt e Firebase configurado; login e cadastro (RF01, RF03) |
| Espaços e reservas | 2 a 4 | Protótipo da transação de blocos (semana 2), depois reservas com as regras RN01 a RN06 (RF07 a RF11) |
| Eventos | 3 a 5 | Criação, aprovação, inscrição e lista de espera (RF13 a RF17) |
| Ocupação e presença | 4 a 5 | Estado do espaço (RF19) e confirmação de presença (RF12) |
| Painel do administrador | 5 a 6 | Espaços, bloqueios, promoção e relatório (RF04, RF06, RF20, RF21) |
| Testes e regras de negócio | 3 a 7 | Testes unitários, de repositório, de Security Rules e de concorrência |
| Entrega e demonstração | 8 | Dados de exemplo, demonstração em dois aparelhos e apresentação |

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
| CA08 | Aluno tenta aprovar evento ou mudar o próprio perfil, pelo app ou direto no Firestore | A operação é recusada com erro de permissão, pelo domínio e pelas Security Rules (ADR-004) |
| CA09 | Espaço com reserva em andamento, evento aprovado ou bloqueio | A lista mostra "reservado até HH:MM", "evento em andamento" ou "bloqueado" (RF19) |
| CA10 | Administrador abre o relatório do mês | Vê espaços mais usados, horários de pico e taxa de faltas coerentes com os dados de teste (RF20) |

### Plano de testes

- **Testes unitários:** cada regra de negócio (RN01 a RN13) e o cálculo do estado do espaço, em classes puras, sem Firebase.
- **Testes de permissão:** casos de uso por perfil, cobrindo CA08 no domínio.
- **Testes de Security Rules:** com o Firebase Emulator Suite e a biblioteca de testes de regras, cobrindo CA08 no servidor e as janelas de RN06 e RN07.
- **Testes de repositório:** com o emulador do Firestore, cobrindo a transação de blocos, o contador de RN04 e a lista de espera.
- **Teste de concorrência:** duas coroutines tentando o mesmo horário no emulador, cobrindo CA01.

## 11. Riscos e mitigações

Probabilidade e impacto são estimativas iniciais para o grupo revisar.

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| A transação de blocos de reserva é mais complexa que o previsto | Média | Alto | Prototipar na semana 2, antes de construir as telas |
| Security Rules com erro expõem dados ou bloqueiam o app | Média | Alto | Escrever as regras junto com os testes no emulador, e testar cada perfil em cada caso |
| Alguma ação esquecer a checagem de permissão por perfil | Média | Alto | Centralizar a checagem em uma classe e testar cada ação com cada perfil |
| Escrita falha sem internet e o usuário perde a ação | Média | Médio | Mensagem clara de erro offline e nenhuma gravação parcial, pois as transações são atômicas |
| Dados de blocos ficam fora de sincronia com reservas e eventos | Média | Alto | Toda escrita passa pelo Repository em uma única transação, com testes de consistência |
| Internet ou Firebase indisponível no dia da apresentação | Média | Alto | Ter o emulador configurado como plano B e um vídeo gravado da demonstração |
| O escopo cresce com notificações, calendário visual e outros extras | Alta | Médio | Congelar o MVP e só abrir extras depois da semana 6 |
| Divisão desigual do trabalho no grupo | Média | Médio | Dividir por módulos (login e perfis, reservas, eventos, painel) e revisar o código uns dos outros |
| A demonstração parecer simples | Baixa | Médio | Usar dois aparelhos ao vivo, mostrar os três perfis e rodar o teste de concorrência |
| Cliente modificado grava blocos direto, contornando a regra de sobreposição | Baixa | Médio | Limitação aceita (ADR-003); evolução para Cloud Function se o tempo permitir |
