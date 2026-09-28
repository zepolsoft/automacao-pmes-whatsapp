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

---

## Registro de execuções

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
