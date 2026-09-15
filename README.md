# Automação WhatsApp para PMEs (n8n + IA)

Protótipo de automação de agendamento via WhatsApp para pequenos negócios locais
(barbearia, clínica, dentista, academia), usando n8n + WhatsApp Business API (Meta Cloud API)
+ um agente de IA (Claude) para interpretar as mensagens dos clientes.

## O que tem aqui

- `workflows/agendamento/` — workflow que recebe mensagens no WhatsApp, extrai a intenção do
  cliente com IA, verifica disponibilidade no Google Calendar, cria o evento e confirma por
  WhatsApp.
- `workflows/lembrete-cancelamento/` — workflow que roda todo dia, envia lembrete dos
  agendamentos do dia seguinte e trata confirmação, cancelamento ou remarcação.
- `prompts/` — prompts usados pelos AI Agent nodes dos workflows.
- `docs/` — documentação e notas de projeto.
- `.claude/skills/` — skills do Claude Code usadas para manter consistência ao criar/editar
  workflows neste projeto.

## Instância n8n

Os workflows rodam em: https://n8n-n8n.wg1izd.easypanel.host

## Status

Protótipo em construção. Os workflows usam credenciais fictícias/placeholder para WhatsApp,
Google Calendar e Google Sheets — as credenciais reais serão conectadas depois.
