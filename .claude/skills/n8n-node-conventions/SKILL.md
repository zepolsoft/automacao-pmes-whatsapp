---
name: n8n-node-conventions
description: Nomenclatura e organização dos nodes dentro dos workflows n8n deste projeto
---

# Convenções de nodes n8n

## Nomenclatura

- Nome do node sempre em português, descrevendo a ação (verbo + objeto), não o tipo técnico.
  - Bom: "Receber Mensagem WhatsApp", "Verificar Disponibilidade", "Criar Evento no Calendar",
    "Salvar Cliente na Planilha", "Responder Cliente"
  - Evitar: "Webhook1", "Google Calendar", "HTTP Request"
- Nodes de decisão (`If`/`Switch`) nomeados pela pergunta que respondem:
  "Horário Disponível?", "Cliente Confirmou, Cancelou ou Remarcou?"
- Cada saída de um `Switch`/`If` deve ter um nome de branch claro correspondente ao caso
  (ex.: "Confirmar", "Cancelar", "Remarcar").

## Organização

- Fluxo principal em linha reta da esquerda para a direita; ramificações (If/Switch) para
  baixo a partir do ponto de decisão.
- Um node = uma responsabilidade. Evitar node de Code fazendo múltiplas transformações não
  relacionadas — preferir Set/Edit Fields nomeados quando possível.
- AI Agent nodes: nomear pelo que o agente decide (ex.: "Interpretar Intenção do Cliente",
  "Classificar Resposta do Lembrete"), com o prompt do sistema em `prompts/` referenciado no
  node.

## Sticky notes

- Usar sticky notes para marcar as grandes seções do workflow (ex.: "1. Trigger", "2. IA",
  "3. Calendar", "4. Sheets", "5. Resposta") quando o workflow tiver mais de ~8 nodes.
