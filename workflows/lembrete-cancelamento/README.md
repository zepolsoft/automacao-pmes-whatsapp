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
   3. **Aguardar Resposta do Cliente** — Wait node (retomado via webhook), com limite de
      espera de 24h (ver seção "Timeout de espera" abaixo).
   4. **Cliente Respondeu ou Deu Timeout?** (IF) — diferencia resposta real de timeout:
      - **Sim (respondeu):** segue para **Classificar Resposta do Lembrete** — AI Agent
        (Claude) classifica a resposta em `confirmar` / `cancelar` / `remarcar` (e, no caso de
        remarcação, já extrai o novo dia/horário pedido) → **Confirmar, Cancelar ou
        Remarcar?** (Switch) — 4 caminhos:
        - **Confirmar:** envia mensagem final de confirmação.
        - **Cancelar:** deleta o evento no Calendar (pelo `event_id`) → confirma o cancelamento.
        - **Remarcar:** verifica se o novo horário está livre → se sim, **atualiza** o evento
          existente (não cria um novo) e atualiza a coluna `data` na planilha (casando pela
          coluna `event_id`) → confirma a remarcação; se não, pede outro horário.
        - **Não entendi (fallback):** pede para o cliente esclarecer a resposta.
      - **Não (timeout):** **Avisar Timeout do Lembrete no WhatsApp** — avisa o cliente que não
        houve resposta e que o agendamento foi mantido como está. Não passa pela
        classificação/confirmação/cancelamento/remarcação.
   5. Volta para o próximo agendamento do lote.

## Timeout de espera no lembrete

O node **Aguardar Resposta do Cliente** (Wait, `resume: "webhook"`) antes ficava esperando a
resposta do cliente indefinidamente — se o cliente nunca respondesse, a execução ficava presa
em "Waiting" para sempre.

- **Limit Wait Time:** ativado (`limitWaitTime: true`), com `limitType: "afterTimeInterval"`,
  `resumeAmount: 24`, `resumeUnit: "hours"` — a execução retoma automaticamente depois de 24h
  mesmo sem resposta do cliente.
- **Detecção de timeout:** o node IF **Cliente Respondeu ou Deu Timeout?**, logo depois do
  Wait, verifica se o texto da mensagem do cliente está presente no payload:
  `{{ $json.body?.messages?.[0]?.text?.body ?? $json.messages?.[0]?.text?.body ?? "" }}`
  com o operador **"not empty"**. Essa é a mesma expressão já usada em "Normalizar Resposta do
  Lembrete" para extrair o texto da resposta. Quando o Wait retoma por resposta real via
  webhook, esse campo vem preenchido com o texto do cliente. Quando retoma por timeout, o item
  que sai do Wait é o mesmo que entrou nele (a resposta do envio do lembrete no WhatsApp, que
  não tem `messages[0].text`), então a expressão resulta em string vazia — condição falsa.
- **Branch de timeout:** vai para o node **Avisar Timeout do Lembrete no WhatsApp**, que envia
  uma mensagem simples avisando que não houve resposta e que o agendamento continua como está,
  usando `$('Buscar Dados do Cliente na Planilha')` (nome, serviço, telefone) — essa referência
  continua acessível mesmo após o timeout, pois esse node já rodou antes do Wait no mesmo
  caminho de execução. Depois, o fluxo volta para **Processar Cada Agendamento** para continuar
  o loop com o próximo agendamento do lote (igual a todos os outros caminhos terminais deste
  workflow) — sem isso, o loop pararia e os agendamentos seguintes do dia não seriam
  processados.

## Histórico de correções

- **2026-09-22 — Lembretes/atualizações duplicadas na planilha (causa raiz):**
  - **Buscar Dados do Cliente na Planilha** (Google Sheets) estava sem `resource`/`operation`/
    filtro nenhum configurado (lia sem localizar a linha certa). Corrigido para
    `resource: "sheet"`, `operation: "read"`, com filtro por `event_id`
    (`lookupColumn: "event_id"`, `lookupValue: "={{ $json.id }}"`, usando o `id` do evento do
    Google Calendar em processamento no loop "Processar Cada Agendamento") e
    `options.returnFirstMatch: true`, garantindo 1 linha por evento.
  - **Atualizar Data na Planilha** (Google Sheets) estava com `columns.matchingColumns: []`
    (vazio) — sem coluna de correspondência, o Update do Google Sheets cria uma linha nova em
    vez de atualizar a existente, o que gerou linhas duplicadas por telefone a cada remarcação.
    Corrigido para `matchingColumns: ["event_id"]`. O node também reescrevia a data **antiga**
    (lida de `Buscar Dados do Cliente na Planilha`) em vez da nova data remarcada; corrigido
    para gravar `$('Classificar Resposta do Lembrete').item.json.output.novo_horario_inicio`
    na coluna `data`.
  - Esses dois nodes já haviam sido alinhados nas convenções de `agendamento.json` (`resource`
    explícito, filtro por `event_id`, `matchingColumns`) numa revisão anterior, mas a
    configuração foi resetada por edições feitas depois diretamente no editor do n8n — por isso
    a correção foi reaplicada e confirmada como persistente.
- **2026-09-22 — Ajustes de convenção pontuais que permanecem ativos:** as saídas do Switch
  "Confirmar, Cancelar ou Remarcar?" têm `renameOutput`/`outputKey` (`confirmar`/`cancelar`/
  `remarcar`) e as settings do workflow têm `timezone: "America/Sao_Paulo"` (usado pelas
  expressões `$today`/`$now`) e `callerPolicy: "workflowsFromSameOwner"`, alinhado com
  `agendamento.json`.

> **Nota:** o node "Buscar Agendamentos de Amanhã" e os nodes de evento do Google Calendar
> (`Cancelar Evento no Calendar`, `Atualizar Evento no Calendar`) não declaram `resource: "event"`
> explicitamente no momento — o n8n usa o resource padrão do node. Isso não é um bug funcional,
> mas difere da convenção explícita usada em `agendamento.json`; não foi reaplicado nesta rodada
> para não conflitar com edições feitas diretamente no editor do n8n.

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

## Credenciais e recursos conectados

Este workflow já está conectado a credenciais e recursos reais na instância n8n (WhatsApp
Business, Google Calendar, Google Sheets, Anthropic Claude). Os objetos de `credentials` (IDs de
credencial) são removidos do JSON exportado para este repo — nunca são commitados, conforme a
convenção do projeto. Os resource locators (planilha, aba, calendário, `phoneNumberId`), por não
serem credenciais, refletem a configuração real em uso:

- Planilha: **Clientes - Automação PMEs**, aba **Sheet1** — colunas `nome`, `telefone`,
  `servico`, `data`, `event_id`.
- Calendário: o calendário do Google associado à conta usada pelo negócio.

Ao clonar/importar este JSON numa outra instância, é preciso reconectar as credenciais e, se for
usar uma planilha/calendário diferentes, ajustar os resource locators dos nodes.
