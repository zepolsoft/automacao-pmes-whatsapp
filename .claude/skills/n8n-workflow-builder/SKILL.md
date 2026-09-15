---
name: n8n-workflow-builder
description: Como estruturar e criar workflows n8n via MCP neste projeto (automacao-pmes-whatsapp)
---

# n8n Workflow Builder

Como criar/editar workflows n8n via MCP para este projeto de automação de WhatsApp.

## Antes de escrever qualquer workflow

1. Chamar `get_workflow_sdk_reference` — nunca adivinhar sintaxe do SDK.
2. Chamar `get_workflow_best_practices` para cada técnica relevante (ex.: "scheduling",
   "chatbot", "triage") antes de decidir nodes/padrões.
3. Chamar `search_nodes` para descobrir os nodes necessários (WhatsApp, Google Calendar,
   Google Sheets, AI Agent, Switch, Wait, Cron/Schedule Trigger etc.).
4. Chamar `get_node_types` com todos os node IDs escolhidos (incluindo discriminators de
   resource/operation/mode) antes de escrever parâmetros — nunca adivinhar nomes de parâmetro.
5. Para valores de resource locator/load-options (ex.: seletor de planilha, calendário),
   usar `explore_node_resources` em vez de inventar IDs.

## Estrutura no repo

- Cada workflow criado via MCP vive em `workflows/<nome-do-workflow>/`.
- Depois de criar/editar o workflow na instância n8n, sempre exportar o JSON atualizado
  para essa pasta (ex.: `workflows/agendamento/agendamento.json`).
- Cada pasta de workflow tem um `README.md` curto explicando o que o workflow faz, o
  trigger, e as integrações usadas.

## Credenciais

- Usar credenciais fictícias/placeholder (ex.: `whatsapp-placeholder`,
  `google-calendar-placeholder`) ao criar os nodes — não criar nem referenciar credenciais
  reais neste protótipo.
- Nunca commitar JSON de workflow que contenha tokens, chaves de API ou segredos reais.

## Validação

- Depois de montar o workflow, rodar `validate_workflow` (e `validate_node_config` para
  nodes críticos) antes de considerar o workflow pronto.
