# Lembrete, Cancelamento e Remarcação

Workflow n8n: [`lembrete-cancelamento.json`](./lembrete-cancelamento.json) · [abrir na instância](https://n8n-n8n.wg1izd.easypanel.host/workflow/wkIfUOGhEom4rFow)

Todo dia às 9h, avisa cada cliente com horário marcado para o dia seguinte e trata a resposta
dele: confirmar, cancelar ou remarcar.

## Fluxo

1. **Disparar Lembrete Diário às 9h** — Schedule Trigger.
2. **Buscar Agendamentos de Amanhã** — Google Calendar, busca todos os eventos do dia seguinte.
3. **Processar Cada Agendamento** — loop (Split in Batches, um agendamento por vez). Para cada um:
   1. **Buscar Dados do Cliente na Planilha** — encontra a linha na planilha pelo `event_id`.
   2. **Enviar Lembrete no WhatsApp** — pergunta se confirma, cancela ou quer remarcar.
   3. **Aguardar Resposta do Cliente** — Wait node (retomado via webhook).
   4. **Classificar Resposta do Lembrete** — AI Agent (Claude) classifica a resposta em
      `confirmar` / `cancelar` / `remarcar` (e, no caso de remarcação, já extrai o novo
      dia/horário pedido).
   5. **Confirmar, Cancelar ou Remarcar?** (Switch) — 4 caminhos:
      - **Confirmar:** envia mensagem final de confirmação.
      - **Cancelar:** deleta o evento no Calendar (pelo `event_id`) → confirma o cancelamento.
      - **Remarcar:** verifica se o novo horário está livre → se sim, **atualiza** o evento
        existente (não cria um novo) e atualiza a coluna `data` na planilha (casando pela
        coluna `event_id`) → confirma a remarcação; se não, pede outro horário.
      - **Não entendi (fallback):** pede para o cliente esclarecer a resposta.
   6. Volta para o próximo agendamento do lote.

## Ajustes de configuração alinhados com `agendamento.json`

Este workflow foi revisado e alinhado às convenções de configuração de node usadas em
`workflows/agendamento/agendamento.json`:

- **Buscar Dados do Cliente na Planilha** (Google Sheets) — estava sem `resource`/`operation`
  definidos e sem filtro; agora usa `resource: "sheet"`, `operation: "read"`, com filtro por
  `event_id` (`lookupColumn`/`lookupValue`) e `options.returnFirstMatch: true`, no mesmo padrão
  das leituras usadas no workflow de agendamento.
- **Atualizar Data na Planilha** (Google Sheets) — estava sem `resource` e sem `columns`
  configurado (o que faria o node falhar em tempo de execução por falta de `matchingColumns`).
  Agora usa `resource: "sheet"`, `columns.mappingMode: "defineBelow"` com
  `matchingColumns: ["event_id"]` e o schema completo das 5 colunas da planilha
  (`nome`, `telefone`, `servico`, `data`, `event_id`), igual ao node "Atualizar Linha na
  Planilha" do workflow de agendamento. A coluna gravada também foi corrigida de `data_hora`
  para `data`, que é o nome real da coluna na planilha "Agendamentos" (definida em
  `agendamento.json`).
- **Buscar Agendamentos de Amanhã**, **Cancelar Evento no Calendar** e **Atualizar Evento no
  Calendar** (Google Calendar) — passaram a declarar `resource: "event"` explicitamente, como
  em todos os nodes de evento do Google Calendar em `agendamento.json`.
- **Confirmar, Cancelar ou Remarcar?** (Switch) — as saídas `confirmar`, `cancelar` e
  `remarcar` agora têm `renameOutput`/`outputKey` definidos, seguindo a convenção do projeto de
  nomear cada saída de Switch/If pelo caso que representa (mesmo padrão do Switch "Qual a
  Intenção do Cliente?" em `agendamento.json`).
- **Configurações do workflow** — adicionado `timezone: "America/Sao_Paulo"` e
  `callerPolicy: "workflowsFromSameOwner"`, alinhando com as settings do workflow de
  agendamento (o timezone é usado pelas expressões `$today`/`$now` deste workflow).
- **Nodes de envio no WhatsApp** — parâmetros alinhados ao formato usado em `agendamento.json`
  (`operation`, `phoneNumberId`, `recipientPhoneNumber`, `textBody`, `additionalFields`), sem os
  campos redundantes `resource`/`messageType` que estavam nos nodes deste workflow.

## Limitação conhecida do protótipo — Wait node

O node **Aguardar Resposta do Cliente** usa `resume: webhook`: ao pausar, o n8n gera uma URL
própria (`$execution.resumeUrl`) que precisa ser chamada para o workflow continuar. Como a
resposta do cliente chega pelo webhook do WhatsApp (não por essa URL diretamente), falta ligar
os dois pontos em produção. O caminho recomendado:

- Ao enviar o lembrete, salvar o `resumeUrl` (ou o `execution.id`) junto com o telefone do
  cliente (ex.: numa aba de "aguardando resposta" na planilha ou numa Data Table do n8n).
- Ter um workflow (pode reusar o trigger do WhatsApp do workflow de agendamento) que, ao
  receber uma mensagem, verifica se aquele telefone está "aguardando resposta de lembrete" e,
  se estiver, repassa o texto para o `resumeUrl` salvo.

Isso não foi implementado neste protótipo para manter o foco na estrutura e lógica principal —
fica como próximo passo antes de ir para produção.

## Credenciais (placeholder)

O workflow foi criado com credenciais fictícias — é preciso conectar as reais na instância n8n
antes de usar:

- `WhatsApp Business (Meta Cloud API)` — envio de mensagens.
- `Anthropic Claude` — modelo usado pelo AI Agent de classificação.
- `Google Calendar` — buscar, cancelar e atualizar eventos.
- `Google Sheets` — buscar e atualizar os dados do cliente.

Também é preciso configurar, direto na instância, o `phoneNumberId` do WhatsApp Business e
selecionar o calendário e a planilha/aba corretos nos resource locators de cada node.
