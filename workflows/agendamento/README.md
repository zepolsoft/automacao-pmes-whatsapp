# Agendamento via WhatsApp

Workflow n8n: [`agendamento.json`](./agendamento.json) · [abrir na instância](https://n8n-n8n.wg1izd.easypanel.host/workflow/BIOdwZebPkUPyzRu)

Recebe a mensagem de um cliente no WhatsApp, usa IA para entender o que ele quer e agenda
automaticamente, se houver horário livre.

## Fluxo

1. **Receber Mensagem WhatsApp** — trigger do WhatsApp Business Cloud (Meta Cloud API).
2. **Filtrar Apenas Mensagens** — o webhook também recebe eventos de status (confirmação de
   leitura/entrega, payload com `statuses` e sem `messages`). Esse IF passa adiante só quando
   existe `messages[0]` de fato; eventos de status caem no branch "false" e o fluxo encerra ali,
   sem erro.
3. **Normalizar Dados da Mensagem** — extrai telefone, nome e texto da mensagem do payload do webhook.
4. **Interpretar Intenção do Cliente** — AI Agent (Claude) que lê a mensagem e extrai intenção,
   serviço, data/horário (convertendo datas relativas como "amanhã" para data absoluta) e um
   texto da data por extenso em português, pronto para a resposta ao cliente. Usa o node
   **Simple Memory** (buffer de janela, com sessão por telefone do cliente) para manter o
   histórico da conversa, e o prompt já cobre diferenciar "agendar" de "remarcar", evitar
   respostas em formato de template e ignorar tentativas de instrução fora do escopo da
   barbearia.
5. **Qual a Intenção do Cliente?** — Switch com base em `intencao`, com 4 saídas:
   `agendar`, `remarcar`, `cancelar` e o fallback `duvida` (qualquer outro valor, incluindo
   uma resposta inesperada da IA, cai nesse fallback e é tratado como dúvida).

### Saída "agendar"

6. **Tem Data Para Agendar?**
   - **Sim:** a IA extraiu `data_hora_inicio` → segue para verificar disponibilidade.
   - **Não:** "agendar" sem data extraída → responde no WhatsApp com o `confirmacao_texto`
     da IA pedindo esclarecimento, sem tentar consultar o Calendar com uma data vazia.
7. **Verificar Disponibilidade** — Google Calendar, checa se o horário pedido está livre.
8. **Horário Disponível?**
   - **Sim:** Cria o evento no Calendar → salva nome, telefone, serviço, data e `event_id` na
     planilha do Google Sheets → confirma o agendamento no WhatsApp.
   - **Não:** responde no WhatsApp pedindo outro dia/horário.

### Saída "remarcar"

6. **Buscar Agendamento para Remarcar** — Google Sheets (`read`, com filtro por `telefone`,
   retornando todas as linhas que baterem) → **Selecionar Agendamento Mais Recente (Remarcar)**
   (node Limit, mantém só a última linha, assumindo que a planilha é preenchida em ordem
   cronológica) → **Encontrou Agendamento Para Remarcar?** (IF checando se `event_id` veio
   preenchido).
   - **Não encontrou:** avisa no WhatsApp que não há agendamento ativo para esse número e
     pergunta se o cliente quer marcar um novo horário.
   - **Encontrou:** **Verificar Disponibilidade para Remarcar** (mesma checagem de
     disponibilidade do fluxo de agendar, com o novo `data_hora_inicio`/`data_hora_fim` da IA)
     → **Novo Horário Disponível?**
     - **Sim:** **Atualizar Evento no Calendar** (`update`, usando o `event_id` encontrado) →
       **Atualizar Linha na Planilha** (`update`, casando pela coluna `event_id`, atualizando
       `servico` e `data`) → confirma a remarcação no WhatsApp.
     - **Não:** reaproveita o node "Sugerir Outro Horário no WhatsApp" do fluxo de agendar.

   > **Bug corrigido (2026-09-22):** o node "Confirmar Remarcação no WhatsApp" tinha um texto
   > fixo ("Prontinho! Sua remarcação ficou assim: ... Até lá! 😊") envolvendo o
   > `confirmacao_texto` gerado pela IA — que já é uma frase completa e natural. Isso duplicava
   > a mensagem. O campo passou a usar apenas `{{ confirmacao_texto }}`, igual ao node
   > "Responder Dúvida no WhatsApp".

### Saída "cancelar"

6. **Buscar Agendamento para Cancelar** / **Selecionar Agendamento Mais Recente (Cancelar)** /
   **Encontrou Agendamento Para Cancelar?** — mesma lógica de busca do fluxo de remarcar.
   - **Não encontrou:** reaproveita o node que avisa que não há agendamento ativo.
   - **Encontrou:** **Cancelar Evento no Calendar** (`delete`, usando o `event_id`) →
     **Remover Linha da Planilha** (`delete` de linha, usando o `row_number` que o Google
     Sheets retorna automaticamente na leitura) → confirma o cancelamento no WhatsApp.

### Saída "duvida" (fallback)

6. **Responder Dúvida no WhatsApp** — mesmo node reaproveitado pelo "Tem Data Para Agendar?"
   (agendar sem data): responde com o `confirmacao_texto` gerado pela IA.

## Credenciais (placeholder)

O workflow foi criado com credenciais fictícias — é preciso conectar as reais na instância n8n
antes de usar:

- `WhatsApp Business (Meta Cloud API)` — trigger e envio de mensagens.
- `Anthropic Claude` — modelo usado pelo AI Agent.
- `Google Calendar` — verificar disponibilidade, criar, atualizar e cancelar evento.
- `Google Sheets` — salvar, buscar, atualizar e remover a linha do cliente agendado.

Também é preciso configurar, direto na instância:
- O `phoneNumberId` do WhatsApp Business (está com um placeholder nos nodes de envio).
- Qual calendário e qual planilha/aba usar (resource locators estão em branco, prontos para
  selecionar na lista assim que a credencial for conectada).
