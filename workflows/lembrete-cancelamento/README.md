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
        existente (não cria um novo) e atualiza a data na planilha → confirma a remarcação;
        se não, pede outro horário.
      - **Não entendi (fallback):** pede para o cliente esclarecer a resposta.
   6. Volta para o próximo agendamento do lote.

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
