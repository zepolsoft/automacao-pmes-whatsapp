# Roteiro de testes — Agendamento, Lembrete e Notificação de Erros

Checklist para rodar **antes de publicar qualquer mudança** nos workflows. Nasceu dos bugs da
semana de 22–26/09/2026: duração do evento errada (1h fixa), resposta de lembrete descartada
(dedup/lock duplicado entre os dois workflows) e evento "fantasma" no Calendar (linha da
planilha apagada → remarcação virou agendamento novo).

Os cenários estão em dois grupos, porque o tipo de teste muda o que ele consegue provar:

- **Grupo A — Simulação** (`test_workflow` com pin data): lógica pura — interpretação da IA,
  sinônimos, cálculo de duração, roteamento por disponibilidade/expediente. Rápido, sem efeito
  colateral, mas **não pega** bugs que só existem no estado real do Calendar/planilha/Data Tables.
- **Grupo B — Integração real**: tudo que depende de efeito colateral de verdade, concorrência ou
  do relógio real. Os bugs de evento fantasma e da condição de corrida só apareceram assim.

---

## Como rodar o Grupo A (simulação)

Regras que valem para todo cenário:

1. **Fixe (pin) todo node com efeito externo**: WhatsApp (envio), Google Calendar, Google Sheets,
   Data Tables (`Checar/Registrar Mensagem`, `Checar/Registrar Lock`, `Verificar/Registrar Espera`)
   e HTTP (`Ativar Indicador de Digitação`, `Encaminhar Mensagem para Lembrete`).
   ⚠️ Node com credencial que **não** estiver fixado roda de verdade — um envio de WhatsApp sem pin
   manda mensagem real.
2. **Não fixe os agentes de IA** (`Interpretar Intenção do Cliente`, `Classificar Resposta do
   Lembrete`, `Classificar Confirmação da Remarcação`) quando o objetivo é testar interpretação — o
   modelo Claude roda de verdade.
3. Use telefones fictícios `55119000001xx`, um por cenário. A memória da IA é por telefone: cenários
   de várias mensagens usam o **mesmo** número, em execuções **sequenciais**.
4. Serviços fixados = cópia da planilha Serviços (hoje: Corte 3D 40 min, Corte Baixo na Máquina
   25, Corte Social na Tesoura 35, Corte Feminino Com Lavagem 60, Corte Básico Feminino 30).
5. Lembrete: a linha fixada em `Buscar Agendamentos de Hoje` precisa ter `data` = **hoje**
   (senão `Filtrar Data de Hoje` descarta), e horários de remarcação "hoje" precisam estar no futuro
   e dentro do expediente no momento da execução.
6. Conferir o resultado com `get_workflow_execution` (includeData) nos nodes: saída da IA,
   `Validar Horário de Funcionamento` (`data_hora_fim`/`novo_horario_fim`, `duracao_minutos`,
   `duracao_fonte`) e qual node terminal executou.

## Como rodar o Grupo B (integração)

**Ainda não existe ambiente isolado.** Para montar (a decidir):

- Cópia **desativada** dos dois workflows, com o trigger do WhatsApp trocado por um Webhook (para
  injetar payloads reais do WhatsApp via `execute_workflow`), apontando para uma **planilha de
  teste**, um **calendário de teste** e **Data Tables de teste** (dedup, lock, esperas).
- Envio de WhatsApp da cópia para um número de teste (ou desabilitado), nunca para clientes.
- Cenários que dependem do WhatsApp real (reenvio da Meta, mídia real) exigem uma pessoa mandando
  mensagem de um celular para o número de teste.

Até esse ambiente existir, o Grupo B fica **não executado** — não marcar como aprovado.

---

## Grupo A — Simulação

| # | Cenário | Esperado |
|---|---|---|
| 1 | Cliente novo, mensagem completa (serviço + data + hora) | IA extrai serviço/data; fim = início + duração da planilha; propõe e pede confirmação (`Propor Horário`); após "sim", `Criar Evento` |
| 2 | Mensagem vaga ("quero cortar o cabelo") | Pergunta serviço e data/hora; nada criado |
| 3 | Serviço por gíria/sinônimo ("baixo", "degradê") | Mapeia para o nome oficial quando inequívoco; se ambíguo, pergunta — nunca inventa serviço |
| 4 | Horário já ocupado (`Verificar Disponibilidade` → `available:false`) | `Sugerir Outro Horário`; `Criar Evento` não executa |
| 5 | Data no passado / fora do expediente / término depois das 18h | Avisa e pede outro horário; nada criado |
| 6 | Cliente muda de ideia antes de confirmar | Nova proposta com o dado novo; nada criado |
| 7 | Mensagem incompleta, completada em mensagens separadas | Na 2ª mensagem junta os dados e propõe |
| 8 | Cliente com agendamento ativo tenta marcar outro | Avisa que já existe agendamento e pergunta se quer remarcar ou marcar outro (comportamento a confirmar) |
| 10 | Serviço mais curto e mais longo | `data_hora_fim` = início + `duracao_minutos` (25 e 60 min), `duracao_fonte: planilha` |
| 11 | Confirmação com variações ("sim", "blz", "pode", "👍") | Todas reconhecidas como confirmação → `Criar Evento` |
| 12 | Áudio / imagem / figurinha em vez de texto | Não quebra; pede para escrever; nada criado |
| 13 | Lembrete: confirma presença | `Enviar Confirmação Final` |
| 14 | Lembrete: cancela | `Cancelar Evento` + `Atualizar Status (Cancelar)` |
| 15 | Lembrete: remarca para outro horário do mesmo dia | Propõe; após "sim", `Atualizar Evento` com fim = início + duração |
| 16 | Lembrete: remarca para outro dia | Idem 15, com a data nova |
| 18 | Lembrete: remarca para horário ocupado | `Pedir Outro Horário`; `Atualizar Evento` não executa |
| 19 | Lembrete: cliente não responde (timeout) | `Reverificar Agendamento` → `Avisar Timeout`; execução termina com sucesso |
| 24 | Situações esperadas (ocupado, fora do expediente) não disparam erro | Execução termina com `status: success` (o Error Workflow só dispara em falha) |
| 25a | Fuso horário perto da virada do dia (lógica) | Validação usa o dia/hora de São Paulo mesmo com servidor em UTC |
| 27 | **Cliente pede pra IA sugerir uma data livre** ("me sugere uma data livre", "qual dia você tem vago?") — fixe `Buscar Eventos dos Próximos Dias` com alguns horários ocupados | Oferece 2-3 opções concretas (dia + hora, por extenso), **todas** dentro das janelas de `Calcular Horários Livres.horarios_livres`, nenhuma em horário ocupado/domingo/passado; pergunta o serviço se faltar; nunca "não consigo escolher por você" nem pede só "uma data certinha"; `Responder Dúvida`, nada criado |
| 27b | Sugestão com preferência ("de tarde", "essa semana", serviço já informado) | Opções respeitam a preferência e cabem inteiras na janela (início + duração do serviço) |
| 27c | Cliente escolhe uma das opções sugeridas ("quarta às 9h, corte 3D") — mesmo telefone do 27, execução seguinte | `confirmado = true`, início = opção escolhida, fim = início + duração da planilha → `Verificar Disponibilidade` → `Criar Evento` |
| 27d | Data inválida seguida de pedido de sugestão ("quero dia 30/02/2027" → "veja uma data anterior a esse dia") — a conversa da execução 1177 | Na 2ª mensagem oferece opções da lista em vez de insistir numa data específica |
| 27e | Calendar fora do ar ao sugerir (fixe `Buscar Eventos dos Próximos Dias` com `[{ "error": "..." }]`) | `horarios_livres` = "INDISPONÍVEL"; a IA pede dia/horário de preferência e **não** inventa disponibilidade |
| 28 | Resposta da IA fora do schema (`Model output doesn't fit required format`) — não dá pra forçar no workflow real; testar num workflow isolado com o mesmo agente + um schema impossível | Parser roda 2x (retry); na 2ª falha: `Avisar Cliente Sobre Falha da IA` → `Escalar Falha da IA para a Equipe` com nome, telefone, mensagem e motivo na mensagem de erro; nada fica sem resposta pro cliente |
| 30 | **Cliente com 2+ agendamentos ativos pede pra cancelar com referência ambígua** — "cancelar o corte de hoje para minha esposa" com 2 agendamentos hoje (a mensagem da execução 1246). Fixe a **mesma** lista de linhas em `Buscar Agendamentos Ativos do Cliente` e em `Buscar Agendamento para Cancelar/Remarcar`, com horários no futuro no momento do teste | `agendamento_alvo = ""`, `Identificar Agendamento Escolhido (Cancelar)` → `precisa_escolher: true` → `Perguntar Qual Agendamento no WhatsApp` listando os de hoje; **`Cancelar Evento no Calendar` não executa**; texto nunca diz "cancelei" |
| 30b | Mesmo cenário com "o da minha esposa" (sem dia), "o mais cedo", "o corte", "meu horário" | Pergunta qual — não deduz pelo tipo de serviço, pela ordem nem pelo mais recente; nada cancelado |
| 30c | Mesmo cenário, pedido de **remarcar** ambíguo ("quero remarcar o de hoje") | `Perguntar Qual Agendamento no WhatsApp`; `Atualizar Evento no Calendar` não executa |
| 30d | Resposta à pergunta (mesmo telefone do 30/30c, execução seguinte): "o feminino" / "o das 15h, pra sexta às 16h" | `agendamento_alvo` = o escolhido; cancelar → `Cancelar Evento` com esse `event_id`; remarcar → novo horário validado e `Propor Horário` (só atualiza depois do "sim") |
| 30e | Referência inequívoca ("cancela o Corte 3D de sexta") com 2+ ativos | Age direto no agendamento certo, sem perguntar |
| 30f | Cliente com **um único** agendamento ativo: "cancelar meu horário" | Usa esse, sem perguntar à toa |
| 30g | Pergunta de escolha com data numérica na frase da IA ("dia 02/10") | `mensagem_escolha` troca pela pergunta montada pelo código, com dia por extenso |
| 31a | **Beneficiário resolve sozinho** — 2+ ativos, só um com `beneficiario` "Esposa" (e só um "Filho"); "cancela o corte da minha esposa" / "quero remarcar o do meu filho pra sexta às 15h". Fixe a coluna `beneficiario` nas linhas de `Buscar Agendamentos Ativos do Cliente` e `Buscar Agendamento para Cancelar/Remarcar` | Age direto no agendamento certo, **sem perguntar**: `Cancelar Evento` com o `event_id` da esposa / remarcação do filho proposta (`Propor Horário`) |
| 31b | **Ambiguidade real mesmo com o campo** — dois agendamentos "Filho"; "cancela o do meu filho" / "preciso remarcar o do meu filho" | `agendamento_alvo = ""` → `Perguntar Qual Agendamento no WhatsApp` listando **só os dois do filho**; nada cancelado/atualizado |
| 31c | Um "Esposa" + uma linha antiga sem beneficiário no mesmo dia; "cancela o da minha esposa" | Age no "Esposa" (a linha antiga não casa com nenhum beneficiário) |
| 31d | Todas as linhas antigas, sem beneficiário (dados reais de 28/09); "cancela o da minha esposa" | Pergunta qual — não deduz pelo tipo de serviço (ex.: "Corte Feminino") |
| 31e | Agendar para outra pessoa: "Corte Feminino Com Lavagem pra minha esposa Ana amanhã às 10h" → "Sim" | `beneficiario: "Ana - esposa"` na proposta e no "sim"; proposta menciona a Ana; `Salvar Cliente` grava `beneficiario` (Grupo B: conferir na planilha real) |

## Grupo B — Integração real

| # | Cenário | Esperado |
|---|---|---|
| 9 | Duas mensagens quase simultâneas do mesmo número | Só uma execução processa (lock); a outra é ignorada sem efeito colateral |
| 17 | Remarca duas vezes seguidas | Calendar fica com **1** evento, no horário final; planilha com 1 linha |
| 20 | Responde ao lembrete depois de já ter cancelado por outro canal | Não reativa nem remarca um agendamento cancelado |
| 21 | Resposta ao lembrete chega enquanto o Agendamento processa outra mensagem do mesmo cliente | Resposta não é descartada (regressão da condição de corrida de 26/09) |
| 22 | Linha da planilha apagada manualmente e cliente tenta remarcar | Não cria evento duplicado (depende do fallback pelo Calendar — **não implementado**) |
| 23 | Erro real (ex.: credencial inválida) | Notificação de erro chega no WhatsApp |
| 25b | Mensagem enviada entre 23h e 1h ("amanhã às 10h") | IA resolve "amanhã" pelo relógio real de São Paulo |
| 26 | Webhook do WhatsApp reenviando a mesma mensagem | Dedup por `message_id`; nenhum agendamento duplicado |
| 31f | Agendamento real para outra pessoa, depois remarcado e cancelado | Planilha grava `beneficiario` na coluna J na criação; remarcar/cancelar **não apagam** a coluna J |
| 29 | Falha real de formato da IA em produção | Cliente recebe o aviso de que a equipe vai responder e a notificação do Error Workflow chega no WhatsApp do responsável com telefone + mensagem do cliente |

---

## Registro de execuções

### 2026-09-28 (tarde, 2) — coluna `beneficiario` como critério de desambiguação

Versão testada: Agendamento `cce566bd` (testada como rascunho; publicada em 28/09 depois desta
rodada, substituindo `78e5f496`). Coluna `beneficiario` já criada em J1 na planilha real (linhas existentes ficaram
vazias). Execuções às ~11h50 de São Paulo.

| # | Resultado | Evidência (execução) |
|---|---|---|
| 31a | ✅ "Cancela o corte da minha esposa" (só um "Esposa") → cancelou direto `evtFemininoHoje`; "Quero remarcar o do meu filho pra sexta às 15h" (só um "Filho") → alvo `evtSocialSabado`, 02/10 15:00–15:35, `Propor Horário` | 1262, 1263 |
| 31b | ✅ "Cancela o do meu filho" e "Preciso remarcar o do meu filho" (dois "Filho") → "Seu filho tem dois horários marcados: o Corte Baixo na Máquina hoje às 15h e o Corte Social na Tesoura no sábado, dia 10 de outubro, às 9h. Qual deles…?"; nada alterado | 1264, 1269 |
| 31c | ✅ "Esposa" + linha antiga → cancelou o "Esposa" (com a regra final; na 1ª versão do prompt a regra mandava perguntar e a IA não seguiu — ajustada para o comportamento atual) | 1265, 1268 |
| 31d | ✅ Todas as linhas antigas → "Não consegui identificar qual é o da sua esposa… Qual deles é o dela?"; nada cancelado | 1267 |
| 31e | ✅ Proposta "Corte Feminino Com Lavagem pra Ana amanhã às 10h — posso confirmar?" com `beneficiario: "Ana - esposa"`; no "Sim", `confirmado: true` e `beneficiario` mantido → `Criar Evento` → `Salvar Cliente` (fixado; valor gravado não verificado na planilha real) | 1266, 1270 |

Verificação de estado real (somente leitura, workflow temporário arquivado): evento
`oi7l48sa9r4o86qmtaurtb0nbo` (Corte 3D 02/10 11h) segue `status: "cancelled"`, sem alteração
desde 12:02:35 UTC. **Não restaurado.**

### 2026-09-28 (tarde) — não adivinhar qual agendamento cancelar/remarcar (execução 1246)

Versão testada: Agendamento `78e5f496` (testada como rascunho; publicada em 28/09 depois desta
rodada, substituindo `a17c64a1`). Planilha fixada: Corte Baixo na Máquina hoje 15h, Corte Feminino Com Lavagem hoje
17h, Corte 3D sex 02/10 11h, Corte Social na Tesoura sáb 10/10 9h (+ 1 linha `concluido`).
Execuções às ~11h20–11h30 de São Paulo.

| # | Resultado | Evidência (execução) |
|---|---|---|
| 30 | ✅ "Também queria cancelar o corte de hoje para minha esposa" → "Vi que hoje tem dois horários marcados: o Corte Baixo na Máquina às 15h e o Corte Feminino Com Lavagem às 17h. Qual dos dois você quer cancelar?"; `Cancelar Evento` não executou | 1251 |
| 30b | ✅ "Cancela o mais cedo por favor" e "Preciso cancelar o da minha esposa" → listou os 4 e perguntou; nada cancelado | 1254, 1255, 1259 |
| 30c | ✅ "Quero remarcar o de hoje" → listou os 2 de hoje e perguntou qual remarcar | 1253 |
| 30d | ✅ "O feminino" → cancelou `evtFemininoHoje` (não o mais recente); "O das 15h, pra sexta às 16h" → alvo Corte Baixo, 02/10 16:00–16:25, `Propor Horário` | 1252, 1257 |
| 30e | ✅ "Cancela o Corte 3D de sexta" → cancelou direto `evt3DSexta` | 1256 |
| 30f | ✅ Um único ativo + "Preciso cancelar meu horário" → cancelou esse sem perguntar | 1258 |
| 30g | ✅ Frase da IA com "dia 02/10" → trocada pela pergunta do código ("…Corte 3D na sexta-feira, dia 2 de outubro, às 11h…") | 1259 |

Também testados localmente (Luxon, servidor em UTC) o `Identificar Agendamento Escolhido` com os
dados reais do incidente de 09:02 (resultado: pergunta, não cancela), id inventado pela IA,
agendamento já passado, nenhuma linha, e o texto de `Confirmar Cancelamento no WhatsApp` (node
fixado no teste, então o texto não é renderizado pelo n8n; validado fora).

Estado real verificado antes da correção (somente leitura, workflow temporário arquivado): o
evento `oi7l48sa9r4o86qmtaurtb0nbo` (Corte 3D 02/10 11h) está com `status: "cancelled"` no
Calendar desde 12:02:35 UTC; a linha da planilha ficou `cancelado`. **Não restaurado**, aguardando
confirmação com o cliente.

### 2026-09-28 — sugestão de horários livres + fallback de formato da IA (execução 1177)

Versão testada: Agendamento `41ab8bb8` (testada como rascunho; publicada em 28/09 depois desta
rodada, substituindo `64c85662`). Agenda fixada: hoje (seg 28/09) 09:30–12:00 ocupado, ter 29/09 dia inteiro ocupado,
qua 30/09 14:00–18:00 ocupado. Execuções rodadas às ~08h05 de São Paulo.

| # | Resultado | Evidência (execução) |
|---|---|---|
| 27 | ✅ "Me sugere uma data livre" → hoje às 12h, qua 30/09 às 9h, qui 01/10 às 14h; pediu o serviço. Nenhuma opção nos horários ocupados | 1212 |
| 27b | ✅ "corte baixo na máquina, qual dia você tem vago de tarde?" → hoje 14h, qua 13h (termina 13h25, antes do bloqueio das 14h), qui 15h | 1214 |
| 27c | ✅ "Quarta às 9h, corte 3D" → 30/09 09:00–09:40, `confirmado: true`, `Criar Evento` + `Confirmar Agendamento` | 1213 |
| 27d | ✅ "Quero dia 30/02/2027" → avisou que a data não existe; "Veja para min uma data anterior a esse dia" → ofereceu 3 opções da lista | 1217, 1218 |
| 27e | ✅ Calendar com erro → "não estou conseguindo consultar a agenda… me diz qual dia e horário você prefere"; nada inventado | 1215 |
| 28 | ✅ Workflow isolado `[TESTE] Saída de erro do parser da IA` (arquivado): parser 2x, saída de erro, `Stop and Error` com "A IA não conseguiu interpretar a mensagem de Cliente Teste (5511900000135)… Motivo: Model output doesn't fit required format" | 1200–1202, 1219 |
| 1 (regressão) | ✅ "Corte 3D quinta às 15h" → 01/10 15:00–15:40, `Propor Horário` (fluxo de data específica inalterado) | 1216 |

`Calcular Horários Livres` também foi testado localmente com Luxon e servidor em UTC: eventos
sobrepostos, evento passando das 18h, dia inteiro, cancelado, marcado como "livre", antecedência
de 1h (14:10 → primeira janela 15:30), fim do expediente e item vazio do `alwaysOutputData`.

Não coberto nesta rodada: 29 (depende de uma falha real em produção) e o Grupo B.

### 2026-09-26 — após correções de duração e da resposta de lembrete

Versões: Agendamento `64c85662`, Lembrete `4821dfad`.

**Grupo A: 18 de 19 aprovados.**

| # | Resultado | Evidência (execução) |
|---|---|---|
| 1 | ✅ Corte 3D 10:00–10:40, propôs; "sim" → `Criar Evento` | 1077, 1091 |
| 2 | ✅ Perguntou serviço e data/hora | 1078 |
| 3 | ✅ "baixo" → Corte Baixo na Máquina 11:00–11:25; "degradê" → perguntou máquina ou tesoura | 1079, 1080 |
| 4 | ✅ `Sugerir Outro Horário`, nada criado | 1081 |
| 5 | ✅ 17h30 com serviço de 60 min recusado (terminaria 18h30); "hoje às 10h" recusado (já passou) | 1082, 1083 |
| 6 | ✅ Trocou para terça 14:00–14:40, nada criado | 1084, 1092 |
| 7 | ✅ Serviço na 1ª mensagem, data na 2ª → propôs 15:00–15:35 | 1085, 1093 |
| 8 | ❌ **Falhou** — o caminho de agendar não consulta a planilha; um cliente com agendamento ativo consegue marcar outro sem aviso | 1091 (`Buscar Agendamento` não executa) |
| 10 | ✅ 25 min (11:00–11:25) e 60 min (14:00–15:00) | 1079, 1086 |
| 11 | ✅ "sim", "blz", "pode", "👍" → `Criar Evento` | 1091, 1094, 1095, 1096 |
| 12 | ✅ Áudio: pediu para escrever, nada criado. Imagem/figurinha seguem o mesmo caminho (só `reaction` é filtrada; texto vazio vai para a IA) — não executados separadamente | 1090 |
| 13 | ✅ `Enviar Confirmação Final` | 1072 |
| 14 | ✅ `Cancelar Evento` + planilha | 1073 |
| 15 | ✅ 17:35–18:00; IA propôs até 18:35 (1h), o cálculo pela planilha corrigiu e manteve dentro do expediente | 1071 |
| 16 | ✅ Segunda 10:00–10:25 → `Atualizar Evento` | 1074 |
| 18 | ✅ `Pedir Outro Horário`, nada atualizado | 1075 |
| 19 | ✅ Timeout → `Avisar Timeout` | 1076 |
| 24 | ✅ Todas as execuções de 4/5/18 terminaram `success` | 1075, 1081–1083 |
| 25a | ✅ 7/7 casos perto da meia-noite (servidor em UTC), rodando o código publicado de `Validar Horário de Funcionamento` localmente | — |

**Grupo B: 0 de 8 executados** — não há ambiente isolado. O 22 depende de um fallback que não
existe ainda.

Observações da rodada:
- As IAs ainda geram término com 1h quando o prompt manda "assumir 1 hora"; o código sobrescreve
  com a duração da planilha. Inofensivo, mas o texto dos prompts está desatualizado.
- Na mensagem vaga (#2) a IA listou só 3 dos 5 serviços (omitiu os femininos).
