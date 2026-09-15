# automacao-pmes-whatsapp

Protótipo de automação n8n + WhatsApp para pequenos negócios locais
(barbearia, clínica, dentista, academia): agendamento automático via IA,
lembretes e cancelamento/remarcação de horários.

## Instância n8n

- URL: https://n8n-n8n.wg1izd.easypanel.host
- Todos os workflows são criados/editados via MCP do n8n conectado a essa instância.

## Convenções

- Nomes de node em português (ex.: "Receber Mensagem WhatsApp", "Verificar Disponibilidade").
- Depois de criar ou editar um workflow via MCP, sempre exportar o JSON resultante para
  `workflows/<nome-do-workflow>/` neste repo.
- Nunca commitar credenciais reais. Usar placeholders/fictícias nos JSONs; credenciais reais
  ficam apenas na instância n8n, nunca no repo (ver `.gitignore`).

## Skills disponíveis

- `n8n-workflow-builder` — como estruturar e criar workflows via MCP neste projeto.
- `whatsapp-message-style` — tom e formato das mensagens enviadas ao cliente no WhatsApp.
- `n8n-node-conventions` — nomenclatura e organização dos nodes dentro dos workflows.
