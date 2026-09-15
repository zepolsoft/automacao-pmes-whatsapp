# Arquitetura do protótipo

## Visão geral

```
Cliente (WhatsApp)
      │
      ▼
[Workflow: agendamento]  ──► Google Calendar (verifica/cria evento)
      │                  ──► Google Sheets (salva nome, telefone, serviço, data, event_id)
      ▼
Resposta no WhatsApp

[Workflow: lembrete-cancelamento]  (roda todo dia às 9h)
      │
      ├─► Google Calendar: agendamentos de amanhã
      ├─► Google Sheets: dados do cliente (por event_id)
      ├─► WhatsApp: lembrete + aguarda resposta
      ├─► IA: classifica confirmar / cancelar / remarcar
      └─► Google Calendar + Google Sheets: aplica a decisão
```

A planilha do Google Sheets é o elo entre os dois workflows: o `event_id` do Google Calendar
criado no `agendamento` é a chave usada pelo `lembrete-cancelamento` para encontrar o cliente,
cancelar ou atualizar o evento certo.

## Decisões de design

- **Data/hora por extenso em português** é gerada pela própria IA (Claude) junto com a data em
  ISO 8601, em vez de formatada depois em uma expressão n8n — evita lidar com formatação de
  Luxon em português dentro de expressões e mantém a regra "sempre confirmar por extenso" da
  skill `whatsapp-message-style` centralizada no prompt.
- **Duração padrão de 1 hora** por serviço, assumida pela IA quando o cliente não especifica.
- **Estrutura de dados combinada** entre Calendar e Sheets: o Calendar é a fonte de verdade do
  horário; a planilha guarda os dados do cliente e o vínculo (`event_id`) com o evento.
- **Limitação conhecida**: o fluxo de espera de resposta no lembrete usa o Wait node do n8n em
  modo `resume: webhook`, que gera uma URL própria por execução. Ligar essa URL à resposta real
  do cliente no WhatsApp requer um passo adicional (ver `workflows/lembrete-cancelamento/README.md`),
  não implementado neste protótipo.

## Próximos passos sugeridos

1. Conectar as credenciais reais (WhatsApp, Google Calendar, Google Sheets, Anthropic) na
   instância n8n.
2. Resolver a limitação do Wait node descrita acima.
3. Testar o fluxo ponta a ponta com um número de WhatsApp de teste (sandbox da Meta).
