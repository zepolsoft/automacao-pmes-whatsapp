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
   frase genérica de "não entendi" — ver o bug corrigido logo abaixo.
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
   - **Sim:** Cria o evento no Calendar → salva nome, telefone, serviço, data e `event_id` na
     planilha do Google Sheets → confirma o agendamento no WhatsApp.
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
     - **Dentro do expediente:** **Verificar Disponibilidade para Remarcar** (mesma checagem
       de disponibilidade do fluxo de agendar) → **Novo Horário Disponível?**
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

9. **Buscar Agendamento para Cancelar** / **Selecionar Agendamento Mais Recente (Cancelar)** /
   **Encontrou Agendamento Para Cancelar?** — mesma lógica de busca do fluxo de remarcar.
   - **Não encontrou:** reaproveita o node que avisa que não há agendamento ativo.
   - **Encontrou:** **Cancelar Evento no Calendar** (`delete`, usando o `event_id`) →
     **Remover Linha da Planilha** (`delete` de linha, usando o `row_number` que o Google
     Sheets retorna automaticamente na leitura) → confirma o cancelamento no WhatsApp.

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
