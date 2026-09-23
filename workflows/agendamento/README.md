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
4. **Buscar Serviços e Preços** — Google Sheets, lê todas as linhas da planilha **"Serviços -
   Barbearia"** (aba "Serviços"), com as colunas `servico`, `preco`, `duracao_minutos`.
5. **Formatar Lista de Serviços** — Code node, transforma as linhas em uma lista de texto (uma
   por linha, `- <serviço>: R$ <preco> (<duracao_minutos> min)`) num único campo
   `lista_servicos`, pronta para entrar no prompt da IA.
6. **Interpretar Intenção do Cliente** — AI Agent (Claude) que lê a mensagem e extrai intenção,
   serviço, data/horário (convertendo datas relativas como "amanhã" para data absoluta) e um
   texto da data por extenso em português, pronto para a resposta ao cliente. Usa o node
   **Simple Memory** (buffer de janela, com sessão por telefone do cliente) para manter o
   histórico da conversa, e o prompt já cobre diferenciar "agendar" de "remarcar", evitar
   respostas em formato de template e ignorar tentativas de instrução fora do escopo da
   barbearia. O prompt recebe a `lista_servicos` formatada na seção "SERVIÇOS DISPONÍVEIS E
   PREÇOS" e usa a duração real de cada serviço (em vez de uma duração fixa) para calcular
   `data_hora_fim` — ver detalhes na seção "Planilha de serviços e preços" abaixo. O prompt
   também instrui a IA a não gerar `data_hora_inicio`/`data_hora_fim` fora do horário de
   funcionamento (09h–18h, seg-sáb) — ver "Horário de funcionamento" abaixo, e a nunca
   classificar como `"duvida"` uma mensagem que pede um horário específico, nem gerar
   linguagem de confirmação fora dos casos `"agendar"`/`"remarcar"` — ver "Nunca confirmar
   sem checar disponibilidade" abaixo. Reações curtas do cliente (emoji isolado, "ok", "blz",
   "obrigado") também são tratadas com uma resposta natural de agradecimento, nunca com a
   frase genérica de "não entendi" — ver o bug corrigido logo abaixo. Avisos de atraso ("vou
   atrasar uns 10 minutos") também são tratados como `"duvida"`, sem mexer no Calendar/planilha
   — ver o bug corrigido logo abaixo.
7. **Corrigir Falsa Confirmação em Dúvida** (Code) — rede de segurança estrutural, independente
   do prompt: roda entre "Interpretar Intenção do Cliente" e o Switch de intenção (ver "Nunca
   confirmar sem checar disponibilidade" abaixo).

   > **Bug corrigido (2026-09-22):** com a memória de conversa (Simple Memory), depois de um
   > agendamento já CONFIRMADO com sucesso, uma mensagem solta do cliente (ex.: "Ok", "Beleza",
   > "Obrigado") podia ser reinterpretada pela IA como retomada de uma tentativa de agendamento
   > anterior que tinha ficado em aberto antes da confirmação (ex.: um horário recusado por
   > estar ocupado) — reabrindo checagens de disponibilidade antigas e confundindo o cliente.
   > Nova regra em REGRAS GERAIS: se já existe uma confirmação bem-sucedida na conversa e a
   > mensagem não tem pedido claro de ação, a IA trata como `intencao = "duvida"` e responde só
   > com um agradecimento curto, sem mencionar agendamentos antigos.

   > **Bug corrigido (2026-09-23):** durante a negociação do primeiro horário da conversa (sem
   > nenhum agendamento confirmado ainda), depois de duas ou três rejeições seguidas (horário
   > fora do expediente, depois horário ocupado), a IA chegou a classificar uma nova tentativa
   > do cliente como `intencao = "remarcar"` em vez de `"agendar"`. A regra de diferenciação
   > "agendar" vs "remarcar" foi reescrita de forma mais explícita (`remarcar` só conta quando
   > já existe uma mensagem de confirmação final enviada nesta conversa) e promovida para uma
   > seção de destaque, **REGRA CRÍTICA — AGENDAR VS REMARCAR**, logo no início de
   > "## O QUE VOCÊ DEVE IDENTIFICAR" — antes mesmo do item 1. A regra de horário de
   > funcionamento (item 4) também foi ajustada: antes ela forçava `intencao = "agendar"`
   > sempre que o horário pedido caía fora do expediente, o que podia sobrescrever um
   > `"remarcar"` legítimo; agora ela preserva a intencao já determinada pela regra crítica.

   > **Bug corrigido (2026-09-24):** cliente reagindo com só um emoji (👍, 🙏, ❤️) ou um
   > agradecimento curto ("ok", "blz", "obrigado") — normalmente uma reação a algo já
   > combinado, não um pedido novo — recebia de volta "Não entendi bem sua mensagem. Você
   > gostaria de marcar um horário ou tirar alguma dúvida sobre nossos serviços?", soando
   > robótico. A causa provável: a regra de "fora de escopo" em REGRAS GERAIS (que sugere
   > exatamente essa frase de convite) estava sendo aplicada a reações curtas, por não terem
   > relação direta com agendamento. Nova regra na seção 5 (Regras por cenário): mensagens
   > desse tipo continuam `intencao = "duvida"`, mas geram uma resposta curta e natural de
   > agradecimento/confirmação (ex.: "Por nada! Até lá 😊"), nunca a frase de "não entendi" nem
   > a pergunta genérica — essa fica reservada só para mensagens realmente ambíguas ou fora de
   > contexto. A regra de "fora de escopo" ganhou uma nota explícita deixando claro que reações
   > curtas não contam como fora de escopo.

   > **Melhoria (2026-09-24):** mensagens avisando atraso (ex.: "vou atrasar uns 10 minutos",
   > "chego um pouco depois", "atrasei um pouco") não são pedido de remarcação nem de
   > cancelamento — são só um aviso pontual sobre o mesmo horário já agendado. Nova regra na
   > seção 5: esse tipo de mensagem continua `intencao = "duvida"` (sem alterar
   > `data_hora_inicio`/`data_hora_fim`, sem gerar dados de remarcação), com uma
   > confirmacao_texto breve e tranquilizadora (ex.: "Tranquilo, José! Te esperamos por aqui
   > 😊"). Nunca aciona criação, atualização ou cancelamento de evento — o cliente segue com o
   > mesmo horário, só avisou que vai chegar mais tarde.
8. **Qual a Intenção do Cliente?** — Switch com base em `intencao`, com 4 saídas:
   `agendar`, `remarcar`, `cancelar` e o fallback `duvida` (qualquer outro valor, incluindo
   uma resposta inesperada da IA, cai nesse fallback e é tratado como dúvida).

### Saída "agendar"

9. **Tem Data Para Agendar?**
   - **Sim:** a IA extraiu `data_hora_inicio` → segue para validar o horário de funcionamento.
   - **Não:** "agendar" sem data extraída → responde no WhatsApp com o `confirmacao_texto`
     da IA pedindo esclarecimento, sem tentar consultar o Calendar com uma data vazia.
10. **Validar Horário de Funcionamento** (Code) → **Horário Dentro do Expediente?** (IF) —
   validação determinística, independente do prompt da IA (ver "Horário de funcionamento"
   abaixo).
   - **Sim:** segue para verificar disponibilidade.
   - **Não:** **Avisar Horário Fora do Expediente no WhatsApp** — avisa que a barbearia
     funciona das 9h às 18h (seg-sáb) e pede outro horário, sem consultar o Calendar.
11. **Verificar Disponibilidade** — Google Calendar, checa se o horário pedido está livre.
12. **Horário Disponível?**
   - **Sim:** Cria o evento no Calendar → salva nome, telefone, serviço, data, `event_id`,
     `status: "agendado"`, `preco` (copiado de "Buscar Serviços e Preços" nesse momento — ver
     "Colunas de status e preço na planilha" abaixo) e `criado_em` na planilha do Google Sheets
     → confirma o agendamento no WhatsApp.
   - **Não:** responde no WhatsApp pedindo outro dia/horário.

   > **Bug corrigido (2026-09-22):** o node "Confirmar Agendamento no WhatsApp" tinha um texto
   > fixo ("Prontinho! Seu horário para {{ servico }} ficou confirmado para ... Até lá! 😊")
   > envolvendo o `confirmacao_texto` gerado pela IA — que já é uma frase completa e natural.
   > Isso duplicava a mensagem. O campo passou a usar apenas `{{ confirmacao_texto }}`, igual
   > aos outros nodes de confirmação. Os nodes "Sugerir Outro Horário no WhatsApp" e "Responder
   > Dúvida no WhatsApp" foram revisados e não tinham esse problema.

### Saída "remarcar"

9. **Buscar Agendamento para Remarcar** — Google Sheets (`read`, com filtro por `telefone`,
   retornando todas as linhas que baterem) → **Selecionar Agendamento Mais Recente (Remarcar)**
   (node Limit, mantém só a última linha, assumindo que a planilha é preenchida em ordem
   cronológica) → **Encontrou Agendamento Para Remarcar?** (IF checando se `event_id` veio
   preenchido).
   - **Não encontrou:** **Redirecionar para Fluxo de Agendar** (Code) → **Tem Data Para
     Agendar?**, reentrando no mesmo caminho do branch "agendar" (ver "Remarcar sem
     agendamento existente" abaixo) — não envia mais aviso de "não encontrei" nem para o fluxo.
   - **Encontrou:** **Validar Horário de Funcionamento (Remarcar)** (Code) → **Horário Dentro
     do Expediente? (Remarcar)** (IF) — mesma validação determinística do fluxo de agendar,
     aplicada ao novo `data_hora_inicio`/`data_hora_fim` da IA (ver "Horário de funcionamento"
     abaixo).
     - **Fora do expediente:** reaproveita o node "Avisar Horário Fora do Expediente no
       WhatsApp" do fluxo de agendar, sem consultar o Calendar.
     - **Dentro do expediente:** **Listar Eventos no Novo Horário (Remarcar)** (Google Calendar,
       `resource: event`, `getAll`, lista os eventos que colidem com o novo horário) →
       **Verificar Disponibilidade para Remarcar** (Code, filtra da lista o evento cujo `id`
       é igual ao `event_id` já salvo do próprio cliente antes de decidir `available` — ver
       "Remarcar para horário próximo do atual" abaixo) → **Novo Horário Disponível?**
     - **Sim:** **Atualizar Evento no Calendar** (`update`, usando o `event_id` encontrado) →
       **Atualizar Linha na Planilha** (`update`, casando pela coluna `event_id`, atualizando
       `servico`, `data`, `status: "remarcado"`, `preco` recalculado e `atualizado_em`) →
       confirma a remarcação no WhatsApp.
     - **Não:** reaproveita o node "Sugerir Outro Horário no WhatsApp" do fluxo de agendar.

   > **Bug corrigido (2026-09-22):** o node "Confirmar Remarcação no WhatsApp" tinha um texto
   > fixo ("Prontinho! Sua remarcação ficou assim: ... Até lá! 😊") envolvendo o
   > `confirmacao_texto` gerado pela IA — que já é uma frase completa e natural. Isso duplicava
   > a mensagem. O campo passou a usar apenas `{{ confirmacao_texto }}`, igual ao node
   > "Responder Dúvida no WhatsApp".

### Saída "cancelar"

9. **Buscar Agendamento para Cancelar** / **Selecionar Agendamento Mais Recente (Cancelar)** /
   **Encontrou Agendamento Para Cancelar?** — mesma lógica de busca do fluxo de remarcar.
   - **Não encontrou:** reaproveita o node que avisa que não há agendamento ativo.
   - **Encontrou:** **Cancelar Evento no Calendar** (`delete`, usando o `event_id`) →
     **Atualizar Linha na Planilha (Cancelar)** (`update`, casando pela coluna `event_id`,
     grava `status: "cancelado"` e `atualizado_em` — **não apaga mais a linha**, ver "Colunas
     de status e preço na planilha" abaixo) → confirma o cancelamento no WhatsApp.

### Saída "duvida" (fallback)

9. **Responder Dúvida no WhatsApp** — mesmo node reaproveitado pelo "Tem Data Para Agendar?"
   (agendar sem data): responde com o `confirmacao_texto` gerado pela IA. Também é usado quando
   o cliente pergunta sobre serviços/preços — a IA responde com base na `lista_servicos`.

## Horário de funcionamento

A barbearia funciona de segunda a sábado, das 09h às 18h. O workflow bloqueia agendamentos
fora desse horário em duas camadas independentes:

1. **Prompt da IA** (node **Interpretar Intenção do Cliente**, seção 4 do system message): se
   o cliente pedir um horário fora do intervalo, ou um horário cujo término ultrapasse as 18h,
   a IA não gera `data_hora_inicio`/`data_hora_fim` — deixa os dois campos vazios e responde
   pedindo outro horário dentro do expediente (domingo: avisa que a barbearia não abre).
2. **Validação determinística** (não depende da IA acertar): nodes **Validar Horário de
   Funcionamento** (Code) → **Horário Dentro do Expediente?** (IF), inseridos depois de "Tem
   Data Para Agendar?" e antes de "Verificar Disponibilidade" — e sua contraparte **Validar
   Horário de Funcionamento (Remarcar)** → **Horário Dentro do Expediente? (Remarcar)**,
   inseridos depois de "Encontrou Agendamento Para Remarcar?" e antes de "Verificar
   Disponibilidade para Remarcar". O Code node calcula, em `America/Sao_Paulo`:
   - `data_hora_inicio` tem hora entre 09:00 (inclusive) e 18:00 (exclusive);
   - `data_hora_fim` não ultrapassa 18:00;
   - o dia da semana de `data_hora_inicio` não é domingo (`weekday !== 7`, padrão Luxon);

   e grava o resultado em `dentro_do_expediente` (boolean), que o IF checa. Se falhar
   qualquer condição, o fluxo não chega a "Verificar Disponibilidade"/"Verificar
   Disponibilidade para Remarcar" nem cria/atualiza evento ou linha na planilha — vai direto
   para **Avisar Horário Fora do Expediente no WhatsApp** (node compartilhado pelos dois
   fluxos), mesmo que a IA tenha gerado um horário inválido por algum motivo.

## Remarcar para horário próximo do atual

> **Bug corrigido (2026-09-24):** um cliente pedindo para remarcar de um horário para outro
> próximo (ex.: de 15h para 15h30) era recusado — "Esse horário já está ocupado" — mesmo o
> horário estando livre de verdade. A causa: **"Verificar Disponibilidade para Remarcar"**
> usava a operação `resource: calendar, operation: availability` (freebusy simples) do node
> Google Calendar, que não tem como excluir um evento específico da checagem. Ao consultar o
> novo horário, o próprio evento antigo do cliente (que ainda não tinha sido atualizado/movido)
> aparecia como conflito consigo mesmo.
>
> Correção: a checagem de disponibilidade na remarcação passou a ser feita em dois nodes:
> 1. **Listar Eventos no Novo Horário (Remarcar)** — Google Calendar, `resource: event`,
>    `operation: getAll`, `returnAll: true`, mesmos `timeMin`/`timeMax` de antes (o novo
>    horário pedido). Lista todos os eventos que colidem com esse intervalo, sem decidir nada.
> 2. **Verificar Disponibilidade para Remarcar** (agora um Code node, reaproveitando o nome
>    original) — filtra da lista qualquer evento cujo `id` seja igual ao `event_id` já salvo
>    na planilha para esse cliente (`$('Selecionar Agendamento Mais Recente (Remarcar)').item.
>    json.event_id`) antes de decidir: `available = true` só se sobrar zero conflitos depois de
>    remover o próprio evento do cliente da lista.
>
> ```js
> const eventoAtualId = $('Selecionar Agendamento Mais Recente (Remarcar)').item.json.event_id;
> const conflitos = $input.all().filter(item => item.json.id !== eventoAtualId);
> return [{ json: { available: conflitos.length === 0 } }];
> ```
>
> O resto do fluxo não mudou e já estava correto: se `available`, **Atualizar Evento no
> Calendar** usa `operation: update` (não cria um evento novo) com o `event_id` existente, e
> **Atualizar Linha na Planilha** casa pela coluna `event_id` para atualizar a `data` — nunca
> cria uma linha nova na planilha.

## Remarcar sem agendamento existente

> **Bug corrigido (2026-09-23, recorrente):** mesmo depois de reforçar duas vezes a regra de
> diferenciação "agendar" vs "remarcar" no prompt, a IA continuava, em alguns casos, classificando
> como `intencao = "remarcar"` uma proposta de horário sem nenhum agendamento confirmado antes
> (tipicamente depois de uma ou duas rejeições por horário ocupado/fora do expediente). Como não
> havia agendamento nenhum pra remarcar, "Encontrou Agendamento Para Remarcar?" caía no "não
> encontrado" e o fluxo mandava "Não encontrei nenhum agendamento ativo pra esse número. Quer
> marcar um horário novo?" e parava — só que a resposta do cliente a essa pergunta (repetir o
> mesmo horário) também era classificada como `"remarcar"`, travando o cliente num loop sem
> nunca conseguir marcar.
>
> Em vez de depender do prompt acertar (o que já tinha falhado duas vezes), a correção é
> **estrutural**: a saída "não encontrado" de **"Encontrou Agendamento Para Remarcar?"** foi
> reconectada do node "Avisar Sem Agendamento Ativo no WhatsApp" (que agora só é usado pelo
> branch "cancelar") para o novo node **"Redirecionar para Fluxo de Agendar"** (Code), que
> reconstrói o item no formato `{ output: { ...dados já extraídos pela IA nesta execução,
> intencao: "agendar" } }` — reaproveitando `servico`, `nome_cliente`, `data_hora_inicio`,
> `data_hora_fim` e `confirmacao_texto` que a IA já tinha extraído, só forçando `intencao` para
> `"agendar"`. Esse item é conectado direto em **"Tem Data Para Agendar?"** — a mesma entrada
> usada pelo branch "agendar" normal — e a partir daí segue o caminho de agendar sem nenhuma
> mudança: valida horário de funcionamento, verifica disponibilidade de verdade no Calendar e,
> se livre, cria o evento, salva na planilha e confirma; se ocupado, reaproveita "Sugerir Outro
> Horário no WhatsApp" pedindo outro horário — sem travar e sem exigir uma nova mensagem do
> cliente. Os nodes "Criar Evento no Calendar", "Salvar Cliente na Planilha" e "Confirmar
> Agendamento no WhatsApp" puderam ser reaproveitados sem nenhuma alteração porque todos já
> leem os dados via referência explícita ao node `$('Interpretar Intenção do Cliente')`, não
> pelo item corrente — então funcionam igual não importa por qual caminho o item chegou até eles.
>
> Isso funciona como rede de segurança permanente: mesmo que a IA volte a classificar
> erroneamente como "remarcar" no futuro, o cliente nunca mais fica preso em loop — o pior caso
> passa a ser "o sistema trata como agendamento novo", que é exatamente o resultado correto.

## Nunca confirmar sem checar disponibilidade

> **Bug corrigido (2026-09-23, crítico):** o cliente perguntou "Tem horário amanhã às 14h?" e o
> agente respondeu confirmando o horário — sem nunca checar o Google Calendar nem criar
> evento algum. Investigando a execução (nº 377): o node **Interpretar Intenção do Cliente**
> classificou a mensagem como `intencao = "duvida"`, com `data_hora_inicio`/`data_hora_fim`
> vazios e `confirmacao_texto = "Sim, José! Seu corte já está confirmado para amanhã às 14h.
> Até lá! 💈"`. O Switch "Qual a Intenção do Cliente?" caiu no fallback `duvida` e mandou essa
> mensagem direto pelo node **Responder Dúvida no WhatsApp** — que nunca passa por "Verificar
> Disponibilidade" nem "Criar Evento no Calendar". O horário perguntado já estava ocupado por
> outra cliente na agenda real; o sistema "confirmou" um agendamento que nunca existiu.

Correção em duas camadas:

1. **Prompt da IA** (node **Interpretar Intenção do Cliente**, seção REGRAS GERAIS): nova regra
   em destaque — qualquer mensagem que mencione uma data/horário específico junto de um pedido
   ou pergunta de disponibilidade (mesmo fraseada como pergunta: "tem horário amanhã às 14h?",
   "dá pra marcar sexta às 10h?") deve ser classificada como `"agendar"` (ou `"remarcar"`, se já
   houver confirmação anterior na conversa) — nunca `"duvida"`. `"duvida"` fica reservada para
   perguntas sem data/horário específico. A IA também é proibida de gerar `confirmacao_texto`
   com linguagem de confirmação ("confirmado", "marcado", "reservado", "agendado") fora dos
   casos `"agendar"`/`"remarcar"`.
2. **Proteção estrutural** (não depende da IA seguir a regra): node **Corrigir Falsa
   Confirmação em Dúvida** (Code), inserido logo depois de "Interpretar Intenção do Cliente" e
   antes do Switch "Qual a Intenção do Cliente?" — ou seja, toda saída da IA passa por ele antes
   de ser roteada. Ele verifica, com uma regex case-insensitive
   (`/confirmad[oa]|marcad[oa]|reservad[oa]|agendad[oa]/i`), se `intencao === "duvida"` e o
   `confirmacao_texto` contém linguagem de confirmação:
   - Se a IA **extraiu** `data_hora_inicio` (mesmo classificando errado como `"duvida"`):
     sobrescreve `intencao` para `"agendar"`, redirecionando o item para o fluxo real de
     agendamento — que passa por "Validar Horário de Funcionamento" e "Verificar
     Disponibilidade" de verdade antes de confirmar qualquer coisa.
   - Se **não há** `data_hora_inicio` (como no caso do bug — a IA não tinha extraído nenhuma
     data): substitui o `confirmacao_texto` por um pedido seguro de esclarecimento ("Pode me
     confirmar o dia e o horário exatos..."), sem deixar uma falsa confirmação sair pelo
     WhatsApp.

   Assim, nenhuma mensagem de confirmação chega ao cliente sem que o fluxo tenha efetivamente
   passado (ou vá passar) por uma checagem real de disponibilidade no Calendar.

## Planilha de serviços e preços

Nova planilha: **[Serviços - Barbearia](https://docs.google.com/spreadsheets/d/1fOA2tNHYHJPMlk4SKZfBJlm5E0BXWiDppYPfSiPI0sc/edit)**
(aba "Serviços"), colunas `servico` | `preco` | `duracao_minutos`. Criada com 4 linhas de
exemplo (valores fictícios, para editar com os preços/durações reais):

| servico | preco | duracao_minutos |
|---|---|---|
| Corte | 40 | 35 |
| Barba | 30 | 25 |
| Corte e barba | 65 | 55 |
| Sobrancelha | 15 | 10 |

O fluxo lê essa planilha a cada mensagem recebida (node **Buscar Serviços e Preços**), formata
as linhas num texto único (node **Formatar Lista de Serviços**, ex.:
`- Corte: R$ 40 (35 min)`) e injeta esse texto na seção **"SERVIÇOS DISPONÍVEIS E PREÇOS"** do
prompt de **Interpretar Intenção do Cliente**. A partir disso, o prompt foi ajustado para:

- Calcular `data_hora_fim` usando a **duração real do serviço identificado** (lookup na lista),
  somando durações quando o cliente combina dois serviços (ex.: "corte e barba") — caindo para
  1 hora padrão apenas se o serviço não estiver listado.
- Responder perguntas de `duvida` sobre serviços/preços com base na lista, sem inventar valores
  que não estejam nela.

## Colunas de status e preço na planilha (2026-09-24)

A planilha **"Clientes - Automação PMEs"** (Sheet1) ganhou 4 colunas novas, à direita das 5
originais (`nome`, `telefone`, `servico`, `data`, `event_id`):

| coluna | valores | quem escreve |
|---|---|---|
| `status` | `agendado`, `remarcado`, `cancelado`, `concluido`, `no_show` | os fluxos abaixo; `no_show` é só manual, direto na planilha |
| `preco` | valor copiado da planilha "Serviços - Barbearia" | ao criar/remarcar |
| `criado_em` | timestamp ISO (`America/Sao_Paulo`) | só na criação |
| `atualizado_em` | timestamp ISO | em qualquer escrita posterior (remarcar/cancelar) |

- **Criação** ("Salvar Cliente na Planilha"): `status: "agendado"`, `preco` calculado por
  ```
  {{ $('Buscar Serviços e Preços').all().find(i => i.json.servico === $('Interpretar Intenção
  do Cliente').item.json.output.servico)?.json.preco ?? '' }}
  ```
  (busca exata do `servico` escolhido pela IA na lista já lida no início da conversa — um
  valor copiado nesse momento, não uma fórmula/lookup dinâmico que mudaria depois se o preço do
  serviço mudasse na outra planilha) e `criado_em: {{ $now.toISO() }}`.
- **Remarcação** ("Atualizar Linha na Planilha"): grava `data`, `status: "remarcado"` e
  `atualizado_em`. **`servico`, `preco` e `criado_em` NÃO mudam** — são reescritos com o valor
  que já estava na linha (ver "Bug corrigido" abaixo), não recalculados; remarcar só muda o
  horário.
- **Cancelamento** ("Atualizar Linha na Planilha (Cancelar)", antes "Remover Linha da
  Planilha"): **mudou de `delete` para `update`** — grava `status: "cancelado"` e
  `atualizado_em`, mas **não apaga mais a linha**. `preco` e `criado_em` também são
  reescritos com o valor que já estava (mesmo motivo abaixo). A remarcação/cancelamento no
  fluxo do workflow "Lembrete, Cancelamento e Remarcação" segue a mesma convenção (ver o
  README daquele workflow).

As 6 linhas que já existiam na planilha antes dessa mudança foram preenchidas uma única vez
com `status: "agendado"` (via um workflow utilitário temporário, executado e arquivado depois),
para não ficarem com `status` vazio e passarem despercebidas pelos filtros novos.

> **Bug corrigido (2026-09-24):** logo depois de criar essas colunas, uma remarcação de teste
> zerou `preco` e `criado_em` na linha atualizada. Causa: o node Update Row do Google Sheets
> escreve um **range contíguo de células**, da coluna mapeada mais à esquerda até a mais à
> direita (nesta planilha: `servico` a `atualizado_em`) — qualquer coluna **dentro** desse
> range que não esteja em `columns.value` é sobrescrita com vazio, mesmo sem estar listada.
> Como `preco`/`criado_em` ficam entre `status` e `atualizado_em`, toda remarcação ou
> cancelamento os zerava, mesmo sem eu pedir para alterá-los — e a primeira versão da
> remarcação também recalculava `preco` via lookup, que ficava vazio sempre que o cliente não
> repetia o serviço ao remarcar. Corrigido reenviando os valores atuais de `servico`/`preco`/
> `criado_em` (lidos em "Selecionar Agendamento Mais Recente", que já tinha a linha completa
> desde antes do update) em vez de omiti-los ou recalculá-los — um "no-op" que preserva o valor
> sem depender de nenhum comportamento implícito do node. Testado com uma linha de teste
> (criada, atualizada e removida por um workflow utilitário): `preco`/`criado_em`
> permaneceram intactos, só `data`/`status`/`atualizado_em` mudaram.

## Robustez: deduplicação, lock por telefone e data no passado (2026-09-24)

Rodada de robustez pensada para não crescer a complexidade do fluxo principal — os nodes novos
vivem num desvio curto logo depois de **Normalizar Dados da Mensagem**, antes de qualquer
chamada à IA, planilha ou Calendar.

1. **Deduplicação de mensagens** — o WhatsApp Business Cloud pode reenviar o mesmo webhook (retry
   de rede, reentrega) para a mesma mensagem. **Normalizar Dados da Mensagem** passou a extrair
   `message_id` (`wamid`) do payload. Dois nodes novos, usando uma Data Table
   (`mensagens_processadas`):
   - **Checar Mensagem Duplicada** (`get`, filtro por `message_id`) → **Mensagem Já Processada?**
     (IF: resultado veio com `message_id`?)
     - **Sim:** **Ignorar Mensagem Duplicada** (NoOp) — encerra sem reenviar nenhuma resposta.
     - **Não:** **Registrar Mensagem Processada** (`insert`, grava `message_id` +
       `processado_em`) → segue o fluxo normal.
2. **Lock por telefone** — evita duas execuções paralelas do mesmo cliente (duas mensagens
   seguidas bem rápidas) escrevendo ao mesmo tempo na planilha/Calendar. Logo depois de registrar
   a mensagem como processada, usando uma segunda Data Table (`locks_telefone`):
   - **Checar Lock do Telefone** (`get`, filtro por `telefone`) → **Telefone Ocupado?** (IF:
     existe `bloqueado_em` com menos de 30 segundos?)
     - **Sim:** **Ignorar Mensagem (Telefone Ocupado)** (NoOp) — encerra sem responder.
     - **Não:** **Registrar Lock do Telefone** (`upsert`, grava `telefone` + `bloqueado_em: agora`)
       → segue para **Buscar Serviços e Preços**.
   - Janela de 30s fixa, sem node de "unlock" no final — decisão deliberada para manter o fluxo
     simples; suficiente para cobrir a duração normal de uma execução, mesmo sabendo que uma
     execução anormalmente lenta (>30s) deixaria uma segunda mensagem passar.
3. **Validação de data no passado** — os nodes **Validar Horário de Funcionamento** e **Validar
   Horário de Funcionamento (Remarcar)** ganharam uma condição `jaPassou` (compara o horário
   pedido com `DateTime.now().setZone('America/Sao_Paulo')`). Um horário dentro do expediente
   (9h-18h) mas que já passou (ex.: cliente pede "hoje às 10h" às 15h) agora cai no mesmo branch
   de "fora do expediente" — o texto de **Avisar Horário Fora do Expediente no WhatsApp** foi
   ajustado para cobrir os dois casos: "Esse horário já passou ou está fora do nosso expediente
   [...]".

Duas Data Tables novas no n8n, compartilhadas entre este workflow e o "Lembrete, Cancelamento e
Remarcação" (mesmo `dataTableId` nos dois):

| tabela | colunas | uso |
|---|---|---|
| `mensagens_processadas` | `message_id`, `processado_em` | deduplicação de webhooks reentregues |
| `locks_telefone` | `telefone`, `bloqueado_em` | lock de 30s para evitar execuções paralelas do mesmo número |

## Credenciais (placeholder)

O workflow foi criado com credenciais fictícias — é preciso conectar as reais na instância n8n
antes de usar:

- `WhatsApp Business (Meta Cloud API)` — trigger e envio de mensagens.
- `Anthropic Claude` — modelo usado pelo AI Agent.
- `Google Calendar` — verificar disponibilidade, criar, atualizar e cancelar evento.
- `Google Sheets` — salvar, buscar, atualizar e remover a linha do cliente agendado; também ler
  a planilha "Serviços - Barbearia" (serviços, preços e durações).

Também é preciso configurar, direto na instância:
- O `phoneNumberId` do WhatsApp Business (está com um placeholder nos nodes de envio).
- Qual calendário e qual planilha/aba usar (resource locators estão em branco, prontos para
  selecionar na lista assim que a credencial for conectada).
