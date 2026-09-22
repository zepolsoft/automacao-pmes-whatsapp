# Agendamento via WhatsApp

Workflow n8n: [`agendamento.json`](./agendamento.json) · [abrir na instância](https://n8n-n8n.wg1izd.easypanel.host/workflow/BIOdwZebPkUPyzRu)

Recebe a mensagem de um cliente no WhatsApp, usa IA para entender o que ele quer e agenda
automaticamente, se houver horário livre.

## Fluxo

1. **Receber Mensagem WhatsApp** — trigger do WhatsApp Business Cloud (Meta Cloud API).
2. **Normalizar Dados da Mensagem** — extrai telefone, nome e texto da mensagem do payload do webhook.
3. **Interpretar Intenção do Cliente** — AI Agent (Claude) que lê a mensagem e extrai intenção,
   serviço, data/horário (convertendo datas relativas como "amanhã" para data absoluta) e um
   texto da data por extenso em português, pronto para a resposta ao cliente.
4. **Tem Data Para Agendar?**
   - **Sim:** intenção é "agendar" e a IA extraiu `data_hora_inicio` → segue para verificar
     disponibilidade.
   - **Não:** dúvida, cancelamento, remarcação, ou "agendar" sem data extraída → responde no
     WhatsApp com o `confirmacao_texto` da IA (pedindo esclarecimento, por exemplo), sem tentar
     consultar o Calendar com uma data vazia.
5. **Verificar Disponibilidade** — Google Calendar, checa se o horário pedido está livre.
6. **Horário Disponível?**
   - **Sim:** Cria o evento no Calendar → salva nome, telefone, serviço, data e `event_id` na
     planilha do Google Sheets → confirma o agendamento no WhatsApp.
   - **Não:** responde no WhatsApp pedindo outro dia/horário.

## Credenciais (placeholder)

O workflow foi criado com credenciais fictícias — é preciso conectar as reais na instância n8n
antes de usar:

- `WhatsApp Business (Meta Cloud API)` — trigger e envio de mensagens.
- `Anthropic Claude` — modelo usado pelo AI Agent.
- `Google Calendar` — verificar disponibilidade e criar evento.
- `Google Sheets` — salvar o cliente agendado.

Também é preciso configurar, direto na instância:
- O `phoneNumberId` do WhatsApp Business (está com um placeholder nos nodes de envio).
- Qual calendário e qual planilha/aba usar (resource locators estão em branco, prontos para
  selecionar na lista assim que a credencial for conectada).
