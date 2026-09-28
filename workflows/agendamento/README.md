# Agendamento via WhatsApp

Workflow n8n: [`agendamento.json`](./agendamento.json) · [abrir na instância](https://n8n-n8n.wg1izd.easypanel.host/workflow/BIOdwZebPkUPyzRu)

Recebe a mensagem de um cliente no WhatsApp, usa IA para entender o que ele quer e agenda
automaticamente, se houver horário livre.

## Objetivo e o que este workflow NÃO faz

**Faz:** processa qualquer mensagem recebida no WhatsApp da barbearia, a qualquer momento do
dia — interpreta a intenção (agendar, remarcar, cancelar, consultar, dúvida), verifica
disponibilidade real no Google Calendar, cria/atualiza/cancela o evento, grava o resultado na
planilha "Clientes - Automação PMEs" e responde o cliente na mesma conversa. Quando o cliente
pergunta sobre um agendamento que já tem (ex.: "que horas marquei hoje?"), a resposta vem de
uma consulta real na planilha, não de um palpite da IA — ver "Saída 'consultar'" no Fluxo
abaixo.

**NÃO faz:**
- Não dispara lembretes automáticos antes de um horário marcado — isso é o outro workflow,
  **[Lembrete, Cancelamento e Remarcação](../lembrete-cancelamento/README.md)**.
- Não processa a resposta do cliente a um lembrete diário (confirmar/cancelar/remarcar em cima
  de um lembrete já enviado) — mesmo que a mensagem chegue pelo mesmo número de WhatsApp, é o
  outro workflow que trata essa conversa específica (ver "Sincronização de tom" abaixo para
  como os dois convivem no mesmo número sem o cliente perceber a diferença).
- Não marca atendimentos como "concluído" depois que o horário passa — essa rotina roda só no
  outro workflow (schedule diário às 22h).

## Principais nodes e papel de cada um

| Node | Papel |
|---|---|
| Receber Mensagem WhatsApp | Trigger (webhook do WhatsApp Business Cloud) |
| Filtrar Apenas Mensagens | Descarta eventos de status (entrega/leitura), só deixa passar mensagens de texto reais |
| Normalizar Dados da Mensagem | Extrai telefone, nome, texto e `message_id` do payload |
| Checar Mensagem Duplicada / Mensagem Já Processada? / Registrar Mensagem Processada | Deduplicação por `message_id` (ver "Regras de negócio" abaixo) |
| Checar Lock do Telefone / Telefone Ocupado? / Registrar Lock do Telefone | Lock de 10s por telefone, evita execuções paralelas do mesmo cliente |
| Buscar Serviços e Preços / Formatar Lista de Serviços | Lê a planilha de serviços e monta o texto injetado no prompt da IA |
| Buscar Agendamentos Ativos do Cliente / Formatar Agendamentos Ativos | Lê as linhas do telefone na planilha e injeta no prompt a lista de agendamentos ativos e futuros, com o `event_id` de cada um — é daí que a IA escolhe o `agendamento_alvo` (ver "Qual agendamento cancelar ou remarcar") |
| Identificar Agendamento Escolhido (Remarcar) / (Cancelar) / Precisa Escolher Qual Agendamento? / Perguntar Qual Agendamento no WhatsApp | Decide qual agendamento alterar sem adivinhar; com 2+ ativos e referência ambígua, pergunta qual antes de agir |
| Buscar Eventos dos Próximos Dias / Calcular Horários Livres | Lê os eventos do Calendar dos próximos 14 dias e monta as janelas livres injetadas no prompt — é daí que a IA tira as sugestões quando o cliente pede "me sugere uma data" (ver "Sugestão de horários livres") |
| Interpretar Intenção do Cliente | AI Agent (Claude) — classifica intenção, extrai serviço/data/hora e escreve `confirmacao_texto`. Retry 1x e saída de erro se a resposta não bater com o schema |
| Avisar Cliente Sobre Falha da IA / Escalar Falha da IA para a Equipe | Saída de erro da IA: avisa o cliente que alguém da equipe vai responder e dispara o Error Workflow (ver "Falha de formato da IA") |
| Corrigir Falsa Confirmação em Dúvida | Rede de segurança: impede a IA de "confirmar" algo sem checar disponibilidade de verdade |
| Qual a Intenção do Cliente? | Switch que roteia para agendar / remarcar / cancelar / consultar / dúvida |
| Tem Dados Completos Para Agendar? / Tem Novo Horário Para Remarcar? | Gates determinísticos: nada é criado/atualizado em Calendar/Sheets até ter serviço + data + horário completos |
| Validar Horário de Funcionamento (+ Remarcar) | Code determinístico: rejeita horário fora de 9h-18h (seg-sáb) ou já passado |
| Verificar Disponibilidade / Listar Eventos no Novo Horário (Remarcar) + Verificar Disponibilidade para Remarcar | Checagem real no Google Calendar (a de remarcar exclui o próprio evento do cliente da lista de conflitos) |
| Filtrar Agendamento Ativo (Remarcar) / (Cancelar) | Mantém só linhas `agendado`/`remarcado` antes de escolher "o" agendamento do cliente — evita pegar uma linha já concluída/cancelada |
| Cliente Confirmou o Horário? (Agendar) / (Remarcar) | Gate: só cria/atualiza Calendar+Sheets quando `output.confirmado = true` — ver "Confirmação explícita antes de escrever" |
| Propor Horário no WhatsApp | Node compartilhado: pergunta "posso confirmar?" quando `confirmado = false`, sem tocar em Calendar/Sheets |
| Criar Evento no Calendar / Atualizar Evento no Calendar / Cancelar Evento no Calendar | Efetiva a ação no Calendar |
| Salvar Cliente na Planilha / Atualizar Linha na Planilha / Atualizar Linha na Planilha (Cancelar) | Grava o resultado na planilha "Clientes - Automação PMEs" |
| Confirmar Agendamento / Confirmar Remarcação / Confirmar Cancelamento / Responder Dúvida / Sugerir Outro Horário / Avisar Horário Fora do Expediente / Avisar Sem Agendamento Ativo no WhatsApp | Respostas ao cliente (ver "Sincronização de tom" abaixo sobre como são escritas) |
| Buscar Agendamentos do Cliente (Consultar) | Google Sheets, lê todas as linhas com o `telefone` do cliente — só leitura, não cria/altera/cancela nada |
| Formatar Resposta da Consulta | Code: filtra só `status` `agendado`/`remarcado`, ordena por data e monta a mensagem com os dados reais (nunca com dado inventado pela IA) |
| Responder Consulta no WhatsApp | Envia a mensagem montada por "Formatar Resposta da Consulta" |

## Colunas da planilha "Clientes - Automação PMEs"

| coluna | lê? | escreve? | quando |
|---|---|---|---|
| `nome`, `telefone`, `servico` | lê (remarcar/cancelar, para achar a linha e reaproveitar dados) | escreve (na criação) | criação grava os três; remarcar/cancelar reescrevem `servico`/`preco`/`criado_em` com o valor que já estava, sem alterar |
| `data` | lê (remarcar, para achar o agendamento mais recente) | escreve | criação grava a data escolhida; remarcação sobrescreve com a nova data |
| `event_id` | lê (remarcar/cancelar, para achar a linha e o evento no Calendar) | escreve | gravado na criação, nunca muda depois |
| `status` | lê (indiretamente, via `event_id`/`data` para achar a linha certa) | escreve | `agendado` na criação, `remarcado` na remarcação, `cancelado` no cancelamento |
| `preco` | lê (remarcar/cancelar, para reescrever sem alterar) | escreve | calculado 1x na criação (lookup na planilha de serviços); nunca recalculado depois |
| `criado_em` | lê (remarcar/cancelar, para reescrever sem alterar) | escreve | gravado 1x na criação, nunca muda depois |
| `atualizado_em` | — | escreve | em toda remarcação/cancelamento |
| `beneficiario` (coluna J, 2026-09-28) | lê (antes da IA, na lista de agendamentos ativos) | escreve (só na criação) | "Eu mesmo" por padrão; "Nome - vínculo" ou só o vínculo quando é para outra pessoa. Remarcar/cancelar não mexem nela (fica à direita de `atualizado_em`, fora do range dos updates) |

Na saída "consultar", **Buscar Agendamentos do Cliente (Consultar)** lê `telefone`, `servico`,
`data` e `status` de todas as linhas do cliente (sem filtro de status na própria busca — o
filtro por `agendado`/`remarcado` acontece depois, em "Formatar Resposta da Consulta") e não
escreve nada em nenhuma coluna.

Detalhes de cada bug já corrigido nessas colunas (inclusive o bug do range do Update Row) estão
na seção "Colunas de status e preço na planilha" mais abaixo.

## Regras de negócio implementadas

- **Horário de funcionamento** (segunda a sábado, 9h-18h) — ver seção "Horário de
  funcionamento" abaixo.
- **Validação de data no passado** — um horário dentro do expediente mas já passado hoje é
  tratado como inválido, mesma seção acima (`jaPassou`).
- **Deduplicação de mensagens** por `message_id` (wamid) — ver "Robustez: deduplicação, lock
  por telefone e data no passado" abaixo.
- **Lock por telefone** (10s) — evita duas execuções paralelas do mesmo cliente, mesma seção.
- **Nunca confirmar sem checar disponibilidade real** — ver seção dedicada abaixo.
- **Agendar vs. remarcar vs. cancelar vs. consultar** distinguidos por regras explícitas no
  prompt da IA — ver "REGRA CRÍTICA — AGENDAR VS REMARCAR" e "REGRA — CONSULTAR VS DÚVIDA VS
  AGENDAR" no system message do node "Interpretar Intenção do Cliente".
- **Consulta de agendamento com dados reais** (2026-09-24) — quando o cliente pergunta sobre um
  agendamento que já tem, a resposta vem de uma leitura real da planilha (ver "Saída
  'consultar'" no Fluxo abaixo), nunca de um palpite da IA.
- **Sugestão de horários livres** (2026-09-28) — quando o cliente pede pra IA sugerir um dia, ela
  oferece 2-3 opções tiradas da agenda real, em vez de insistir que ele escolha — ver seção
  dedicada abaixo.
- **Falha de formato da IA não derruba a conversa** (2026-09-28) — ver seção dedicada abaixo.
- **Nunca adivinhar qual agendamento cancelar/remarcar** (2026-09-28) — com 2+ agendamentos
  ativos e referência ambígua, lista e pergunta antes de agir; ver "Qual agendamento cancelar ou
  remarcar" abaixo.

## Sincronização de tom com o workflow de lembrete

Este workflow e o **[Lembrete, Cancelamento e Remarcação](../lembrete-cancelamento/README.md)**
atendem o mesmo número de WhatsApp da barbearia, então, do ponto de vista do cliente, é uma
conversa só — ele não deve perceber que está "falando com duas IAs diferentes" dependendo de
ter sido ele quem escreveu primeiro ou o negócio quem mandou um lembrete. Por isso os dois
workflows compartilham deliberadamente:

- O mesmo campo de saída da IA, `confirmacao_texto`, com as mesmas instruções de **variedade e
  tom** no system message (não repetir a mesma estrutura/abertura de frase, usar o nome do
  cliente sem exagero, ser acolhedor em recusa/atraso/remarcação de última hora) — ver o
  system message de "Interpretar Intenção do Cliente" aqui e de "Classificar Resposta do
  Lembrete" no outro workflow.
- O mesmo texto-base (com variações sorteadas) para as situações que não passam pela IA:
  horário ocupado, fora do expediente, "não entendi".
- As mesmas regras de horário de funcionamento e validação de data no passado.

Apesar do tom compartilhado, os dois workflows têm **escopo e lógica totalmente
independentes** — não compartilham nenhum node, trigger, nem estado de execução:

- Este workflow trata mensagens recebidas a qualquer momento, iniciadas pelo cliente (ou em
  resposta a uma mensagem anterior dele mesmo).
- O outro trata especificamente o ciclo do lembrete diário: dispara às 8h, aguarda resposta a
  UM lembrete específico já enviado, e tem sua própria lógica de timeout — nunca reage a uma
  mensagem espontânea do cliente que não seja resposta a esse lembrete.

Uma mudança de tom/estilo num dos dois (ex.: ajustar a seção VARIEDADE E TOM do prompt) deve,
na prática, ser replicada no outro para manter a experiência consistente — não existe hoje um
prompt compartilhado entre os dois workflows (cada AI Agent tem seu próprio system message),
então essa sincronização é manual e intencional, não automática.

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

9. **Tem Dados Completos Para Agendar?** (renomeado em 2026-09-24, ver "Bug corrigido" abaixo)
   - **Sim** (intenção = agendar, `data_hora_inicio` presente E `servico` diferente de "não
     especificado"): segue para validar o horário de funcionamento.
   - **Não** (falta data OU falta serviço): responde no WhatsApp com o `confirmacao_texto`
     da IA pedindo o que falta, sem tocar em Calendar ou planilha.
10. **Validar Horário de Funcionamento** (Code) → **Horário Dentro do Expediente?** (IF) —
   validação determinística, independente do prompt da IA (ver "Horário de funcionamento"
   abaixo).
   - **Sim:** segue para verificar disponibilidade.
   - **Não:** **Avisar Horário Fora do Expediente no WhatsApp** — avisa que a barbearia
     funciona das 9h às 18h (seg-sáb) e pede outro horário, sem consultar o Calendar.
11. **Verificar Disponibilidade** — Google Calendar, checa se o horário pedido está livre.
12. **Horário Disponível?**
   - **Sim:** **Cliente Confirmou o Horário? (Agendar)** (IF, checa `output.confirmado` — ver
     "Confirmação explícita antes de escrever" abaixo).
     - **Não** (`confirmado = false`, é uma proposta nova): **Propor Horário no WhatsApp** —
       envia o `confirmacao_texto` perguntando se pode confirmar, sem tocar em Calendar/Sheets.
     - **Sim** (`confirmado = true`, cliente acabou de responder "sim" à proposta): Cria o
       evento no Calendar → salva nome, telefone, serviço, data, `event_id`,
       `status: "agendado"`, `preco` (copiado de "Buscar Serviços e Preços" nesse momento — ver
       "Colunas de status e preço na planilha" abaixo) e `criado_em` na planilha do Google
       Sheets → confirma definitivamente o agendamento no WhatsApp.
   - **Não** (horário ocupado): responde no WhatsApp pedindo outro dia/horário.

   > **Bug corrigido (2026-09-22):** o node "Confirmar Agendamento no WhatsApp" tinha um texto
   > fixo ("Prontinho! Seu horário para {{ servico }} ficou confirmado para ... Até lá! 😊")
   > envolvendo o `confirmacao_texto` gerado pela IA — que já é uma frase completa e natural.
   > Isso duplicava a mensagem. O campo passou a usar apenas `{{ confirmacao_texto }}`, igual
   > aos outros nodes de confirmação. Os nodes "Sugerir Outro Horário no WhatsApp" e "Responder
   > Dúvida no WhatsApp" foram revisados e não tinham esse problema.

   > **Bug crítico corrigido (2026-09-24):** cliente pediu para reagendar um horário existente
   > ("Queria reagendar meu horário das 16h para às 11h") sem repetir o serviço na mesma
   > mensagem. A IA classificou como `intencao = "agendar"` (a REGRA CRÍTICA só permitia
   > `"remarcar"` com confirmação nesta conversa, e esse era um agendamento de uma conversa
   > anterior) com `servico = "não especificado"`, e o antigo **"Tem Data Para Agendar?"** só
   > checava `data_hora_inicio` — nunca `servico` — deixando passar: o sistema **criou um
   > evento novo no Calendar e uma linha nova na planilha** com `servico: "não especificado"`
   > às 11h, sem tocar no agendamento real das 16h. Ao informar o serviço depois, o cliente foi
   > informado que o horário "já estava ocupado" — na verdade em conflito com o próprio evento
   > fantasma recém-criado.
   >
   > Corrigido em duas camadas:
   > 1. **"Tem Data Para Agendar?"** renomeado para **"Tem Dados Completos Para Agendar?"** e
   >    ganhou uma 3ª condição: `servico` diferente de `"não especificado"`. Agora nada é
   >    criado nem atualizado até ter serviço + data + horário completos na mesma classificação.
   > 2. **REGRA CRÍTICA — AGENDAR VS REMARCAR** relaxada: antes só permitia `"remarcar"` com
   >    confirmação enviada NESTA conversa — o que forçava `"agendar"` sempre que um cliente
   >    recorrente pedia para reagendar algo marcado numa conversa anterior. Agora classifica
   >    `"remarcar"` sempre que o cliente claramente se refere a mudar um horário já marcado
   >    (palavras como "reagendar", "remarcar", "mudar/trocar meu horário"), mesmo sem
   >    confirmação nesta conversa — confiando na rede de segurança já existente
   >    (`"Encontrou Agendamento Para Remarcar?"` → `"Redirecionar para Fluxo de Agendar"`,
   >    ver "Remarcar sem agendamento existente" abaixo) para os casos em que a IA erra e não
   >    existe agendamento real: o sistema redireciona sozinho para o fluxo de agendar, sem
   >    travar o cliente.
   >
   > O registro incorreto criado por esse bug (evento do Calendar + linha da planilha) foi
   > removido manualmente antes da correção; o agendamento real das 16h foi conferido e
   > permaneceu intacto durante toda a investigação.

### Saída "remarcar"

9. **Buscar Agendamento para Remarcar** — Google Sheets (`read`, com filtro por `telefone`,
   retornando todas as linhas que baterem) → **Filtrar Agendamento Ativo (Remarcar)** (Filter,
   mantém só linhas com `status: "agendado"` ou `"remarcado"`, `alwaysOutputData: true` — ver
   "Bug corrigido" abaixo) → **Identificar Agendamento Escolhido (Remarcar)** (Code: usa o
   `agendamento_alvo` da IA, ou o único ativo; nunca adivinha — ver "Qual agendamento cancelar
   ou remarcar" abaixo) → **Encontrou Agendamento Para Remarcar?** (IF checando se `event_id`
   veio preenchido).
   - **Não encontrou:** **Precisa Escolher Qual Agendamento? (Remarcar)** (IF).
     - **Sim** (2+ ativos e nenhum identificado): **Perguntar Qual Agendamento no WhatsApp**
       lista os agendamentos e pergunta qual remarcar — nada é alterado.
     - **Não** (nenhum ativo): **Redirecionar para Fluxo de Agendar** (Code) → **Tem Dados
       Completos Para Agendar?**, reentrando no mesmo caminho do branch "agendar" (ver "Remarcar
       sem agendamento existente" abaixo) — não envia aviso de "não encontrei" nem para o fluxo.
   - **Encontrou:** **Tem Novo Horário Para Remarcar?** (IF checando `data_hora_inicio`
     notEmpty — ver "Bug corrigido" abaixo).
     - **Não** (cliente ainda não disse pra quando quer remarcar): reaproveita "Responder
       Dúvida no WhatsApp" perguntando o novo dia/horário, sem tocar em Calendar/planilha.
     - **Sim:** **Validar Horário de Funcionamento (Remarcar)** (Code) → **Horário Dentro
       do Expediente? (Remarcar)** (IF) — mesma validação determinística do fluxo de agendar,
       aplicada ao novo `data_hora_inicio`/`data_hora_fim` da IA (ver "Horário de funcionamento"
       abaixo).
       - **Fora do expediente:** reaproveita o node "Avisar Horário Fora do Expediente no
         WhatsApp" do fluxo de agendar, sem consultar o Calendar.
       - **Dentro do expediente:** **Listar Eventos no Novo Horário (Remarcar)** (Google
         Calendar, `resource: event`, `getAll`, lista os eventos que colidem com o novo
         horário) → **Verificar Disponibilidade para Remarcar** (Code, filtra da lista o
         evento cujo `id` é igual ao `event_id` já salvo do próprio cliente antes de decidir
         `available` — ver "Remarcar para horário próximo do atual" abaixo) → **Novo Horário
         Disponível?**
       - **Sim:** **Cliente Confirmou o Horário? (Remarcar)** (IF, checa `output.confirmado` —
         ver "Confirmação explícita antes de escrever" abaixo).
         - **Não** (`confirmado = false`): **Propor Horário no WhatsApp** pergunta se pode
           confirmar a mudança, sem tocar em Calendar/planilha.
         - **Sim** (`confirmado = true`): **Atualizar Evento no Calendar** (`update`, usando o
           `event_id` encontrado) → **Atualizar Linha na Planilha** (`update`, casando pela
           coluna `event_id`, atualizando `data`, `status: "remarcado"` e `atualizado_em` —
           `servico`/`preco`/`criado_em` são reescritos sem alterar, ver "Colunas de status e
           preço" abaixo) → confirma definitivamente a remarcação no WhatsApp.
       - **Não:** reaproveita o node "Sugerir Outro Horário no WhatsApp" do fluxo de agendar.

   > **Bug corrigido (2026-09-22):** o node "Confirmar Remarcação no WhatsApp" tinha um texto
   > fixo ("Prontinho! Sua remarcação ficou assim: ... Até lá! 😊") envolvendo o
   > `confirmacao_texto` gerado pela IA — que já é uma frase completa e natural. Isso duplicava
   > a mensagem. O campo passou a usar apenas `{{ confirmacao_texto }}`, igual ao node
   > "Responder Dúvida no WhatsApp".

   > **Bug crítico corrigido (2026-09-24):** cliente mandou só "Quero remarcar", sem dizer
   > pra quando. Sem nenhum gate, o item seguia direto para "Validar Horário de Funcionamento
   > (Remarcar)" com `data_hora_inicio` vazio — o Code calculava `dentro_do_expediente: false`
   > (data inválida) e caía em "Avisar Horário Fora do Expediente no WhatsApp", que **falhou**
   > (`"Bad request"`, capturado pelo Error Workflow): esse node específico tinha o
   > `phoneNumberId` com o valor placeholder do template em vez do real (único entre os 8 nodes
   > de WhatsApp com esse problema — não tinha relação com a expressão da mensagem, que já era
   > seguro contra campos vazios). Corrigidas as duas causas: 1) `phoneNumberId` do node
   > ajustado para o valor real; 2) novo node **"Tem Novo Horário Para Remarcar?"** inserido
   > entre "Encontrou Agendamento Para Remarcar?" e a validação de horário, garantindo que
   > "remarcar sem dizer quando" pare em "Responder Dúvida no WhatsApp" pedindo a data, sem
   > nunca chegar perto da validação de expediente ou do Calendar.

   > **Bug crítico corrigido (2026-09-24, mesmo dia):** testando o fix acima, "Quero
   > remarcar" fez o sistema selecionar como "o agendamento do cliente" uma linha **já
   > concluída** de um teste anterior, em vez do corte ativo das 16h — porque "Selecionar
   > Agendamento Mais Recente (Remarcar)" só pega a última linha da planilha por telefone,
   > sem olhar o `status`. Se o cliente tem mais de uma linha e a mais recente por ordem de
   > inserção não é a ativa, o sistema mexeria no evento/linha errados. Corrigido inserindo
   > **"Filtrar Agendamento Ativo (Remarcar)"** antes do Limit, mantendo só linhas
   > `agendado`/`remarcado`. Verificado com um teste isolado direto na planilha real antes de
   > aplicar: com o filtro, a linha selecionada para o telefone de teste passou a ser
   > corretamente a ativa, não a concluída.

### Saída "cancelar"

9. **Buscar Agendamento para Cancelar** — Google Sheets (`read`, filtro por `telefone`) →
   **Filtrar Agendamento Ativo (Cancelar)** (Filter, mesma lógica e mesmo bug corrigido do
   branch "remarcar" acima) → **Identificar Agendamento Escolhido (Cancelar)** (mesma lógica do
   branch "remarcar") → **Encontrou Agendamento Para Cancelar?**.
   - **Não encontrou:** **Precisa Escolher Qual Agendamento? (Cancelar)** (IF).
     - **Sim** (2+ ativos e nenhum identificado): **Perguntar Qual Agendamento no WhatsApp**
       lista os agendamentos e pergunta qual cancelar — nada é cancelado.
     - **Não** (nenhum ativo): reaproveita o node que avisa que não há agendamento ativo.
   - **Encontrou:** **Cancelar Evento no Calendar** (`delete`, usando o `event_id`) →
     **Atualizar Linha na Planilha (Cancelar)** (`update`, casando pela coluna `event_id`,
     grava `status: "cancelado"` e `atualizado_em` — **não apaga mais a linha**, ver "Colunas
     de status e preço na planilha" abaixo) → **Confirmar Cancelamento no WhatsApp**, com texto
     montado a partir da linha realmente cancelada (serviço + dia por extenso + horário), não
     da frase da IA.

### Saída "consultar" (2026-09-24)

9. **Buscar Agendamentos do Cliente (Consultar)** (Google Sheets, `read`, filtro por
   `telefone`, `alwaysOutputData: true`) → **Formatar Resposta da Consulta** (Code) → confirma
   no WhatsApp. Sem checagem de disponibilidade nem escrita — é só leitura.
   - O Code node filtra as linhas retornadas mantendo só `status: "agendado"` ou
     `"remarcado"` (ignora `cancelado`/`concluido`/`no_show`), ordena por `data` e monta a
     mensagem a partir dos dados reais — nunca do `confirmacao_texto` da IA, que para esta
     saída é só uma frase de transição descartada (ver "REGRA — CONSULTAR VS DÚVIDA VS
     AGENDAR" no system message de "Interpretar Intenção do Cliente").
   - **Nenhum agendamento ativo encontrado:** avisa educadamente e pergunta se quer marcar um
     horário.
   - **Um agendamento ativo:** confirma serviço e data/horário por extenso (ex.: "Tem sim! Seu
     Corte de adulto está marcado para quinta-feira, dia 24 de setembro, às 16h.").
   - **Mais de um agendamento ativo:** lista todos, um por linha.

### Saída "duvida" (fallback)

9. **Responder Dúvida no WhatsApp** — mesmo node reaproveitado pelo "Tem Dados Completos Para
   Agendar?" (agendar sem data ou sem serviço): responde com o `confirmacao_texto` gerado pela
   IA. Também é usado quando
   o cliente pergunta sobre serviços/preços — a IA responde com base na `lista_servicos`.

### Saídas "encaminhar" e "fora_do_escopo" (2026-09-28)

Ver [Escopo da conversa](#escopo-da-conversa-2026-09-28). Com essas duas saídas, o fallback
"duvida" do Switch passou do índice 4 para o 6.

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
> `"agendar"`. Esse item é conectado direto em **"Tem Dados Completos Para Agendar?"** — a mesma entrada
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

## Confirmação explícita antes de escrever (2026-09-24)

Antes desta mudança, assim que o fluxo confirmava serviço + data/hora completos e via que o
horário estava livre no Calendar, ele já criava o evento/atualizava a linha na mesma execução
— sem perguntar ao cliente antes. Isso já tinha causado bugs (ex.: o bug de duplicidade
documentado acima, onde um agendamento parcial foi criado com `servico: "não especificado"`).
Agora existe uma etapa de confirmação explícita, separando claramente **coleta/proposta** (não
escreve nada) de **execução** (só depois do "sim" do cliente):

1. A IA (`Interpretar Intenção do Cliente`) ganhou um novo campo no output, `confirmado`
   (booleano): `true` só quando a mensagem atual é uma resposta afirmativa a uma proposta de
   horário que o próprio assistente acabou de fazer nesta conversa (ex.: perguntou "posso
   confirmar?" e o cliente respondeu "sim"); `false` em qualquer outro caso, inclusive a
   primeira vez que um horário é proposto.
2. Depois de **Horário Disponível?** (agendar) ou **Novo Horário Disponível?** (remarcar)
   confirmarem que o horário está livre, um novo IF — **Cliente Confirmou o Horário?
   (Agendar)** / **(Remarcar)** — decide o que fazer:
   - `confirmado = false`: **Propor Horário no WhatsApp** (node compartilhado pelos dois
     fluxos) envia o `confirmacao_texto` da IA, que nessa etapa é sempre uma pergunta (ex.:
     "Perfeito! Corte amanhã às 10h está livre — posso confirmar?") — nada é criado ou
     atualizado em Calendar/Sheets.
   - `confirmado = true`: segue para **Criar Evento no Calendar** / **Atualizar Evento no
     Calendar** normalmente, como antes.
3. Quando o cliente responde "sim" (ou pede pra mudar algo), a mensagem seguinte passa de novo
   por todo o fluxo — a IA releia o histórico da conversa para recuperar serviço/data/horário
   que estavam sendo propostos, marca `confirmado = true` (ou `false` se o cliente pediu outra
   coisa) e o sistema reavalia horário de funcionamento e disponibilidade real de novo antes de
   decidir — não confia cegamente no que foi checado na proposta anterior, protegendo contra o
   horário ter sido ocupado por outra pessoa nesse meio-tempo.
4. Se o cliente responder negativamente ou pedir pra mudar algo (outro horário, outro
   serviço), a IA gera uma nova proposta com `confirmado = false`, reentrando no passo 2 — sem
   precisar de nenhuma lógica nova, é o mesmo fluxo de sempre se repetindo.

Isso vale simetricamente para **agendar** e **remarcar** — os dois passam pelo mesmo padrão de
gate, cada um com seu próprio IF de confirmação, mas compartilhando o node de proposta.

### Bug: lock de telefone bloqueando a confirmação (2026-09-24)

Primeiro teste ao vivo do fluxo acima: cliente pediu corte amanhã às 15h, recebeu a pergunta de
confirmação, respondeu "Pode!" — e nada aconteceu (sem evento novo no Calendar, sem linha na
planilha, sem mensagem final). Investigando as execuções (`search_workflow_executions` /
`get_workflow_execution`):

- A pergunta de confirmação ("E as 15h?") foi processada corretamente — disponibilidade
  checada, `confirmado: false`, mensagem de proposta enviada.
- A resposta "Pode!" chegou ~26s depois. Como cada mensagem reentra do zero pelo trigger
  (**Receber Mensagem WhatsApp** → dedup → lock), ela caiu de novo em **Checar Lock do
  Telefone** / **Telefone Ocupado?** — e como o lock da mensagem anterior ainda estava dentro da
  janela de 30s, **Telefone Ocupado?** deu `true` e a mensagem foi descartada em **Ignorar
  Mensagem (Telefone Ocupado)**, sem nunca chegar na IA, no Calendar ou na planilha.
- Confirmado lendo Calendar/planilha diretamente (workflow utilitário descartável): nenhum
  evento novo, nenhuma linha nova — o evento "16h" que pareceu ter sido criado era um evento de
  teste antigo (`corte de cabelo - José Teste`, criado em 2026-09-18, sem linha correspondente
  na planilha e com `timeZone: America/New_York` — anômalo, de um teste manual anterior), não
  algo gerado por este teste. Ou seja: não havia bug de fuso horário no node **Criar Evento no
  Calendar** (que não foi alterado por esta mudança — continua usando
  `data_hora_inicio`/`data_hora_fim` da IA, já em ISO com offset `-03:00`, direto em `start`/
  `end`); o horário "errado" observado era um evento antigo sendo confundido com o novo teste.

**Causa raiz:** a janela de 30s do lock foi dimensionada só para cobrir a duração de uma única
execução (seção "Robustez" abaixo). Ela não previa que o próprio fluxo passaria a exigir uma
segunda mensagem do cliente (a confirmação) pouco depois da primeira — e uma resposta rápida a
"posso confirmar?" cai naturalmente dentro de qualquer janela de trinta segundos.

**Correção:** janela reduzida de 30s para 10s em **Telefone Ocupado?** — ainda cobre a duração
real observada de uma execução (~3-11s) para o propósito original (evitar duas execuções
paralelas do mesmo número), mas não bloqueia mais uma resposta humana normal a uma pergunta de
confirmação. O fluxo de **remarcar** no workflow "Lembrete, Cancelamento e Remarcação" não sofre
desse problema: a segunda etapa de confirmação lá usa um node Wait nativo (resume por webhook),
que retoma direto no meio do fluxo sem passar de novo pelo dedup/lock.

### Bug: remarcar para um horário livre travava sem erro (2026-09-24)

Depois de corrigir o lock, novo teste: pedir remarcação pra um horário livre também não enviava a
pergunta de confirmação — nenhuma mensagem chegava, sem erro no Error Workflow. Execução (#668)
mostrou `lastNodeExecuted: "Listar Eventos no Novo Horário (Remarcar)"` — o node seguinte
(**Verificar Disponibilidade para Remarcar**) nunca chegou a rodar.

**Causa:** o branch de **agendar** checa disponibilidade com a operação nativa de
`availability` do node Google Calendar (`resource: calendar`), que sempre devolve exatamente 1
item com `available: true/false`, esteja o horário livre ou ocupado. O branch de **remarcar**
faz diferente: **Listar Eventos no Novo Horário (Remarcar)** lista eventos crus
(`resource: event`, `operation: getAll`) no intervalo pedido, e um Code node
(**Verificar Disponibilidade para Remarcar**) calcula `available` comparando os `id`s retornados
com o `event_id` do agendamento atual (pra não contar o próprio evento como conflito). Quando o
horário pedido está livre — o caso normal — a listagem retorna **zero itens**, e sem
`alwaysOutputData` nenhum item chega no Code node seguinte: a execução simplesmente para ali, sem
erro, sem rodar o IF **Novo Horário Disponível?** nem nada depois dele.

**Correção:**
1. `alwaysOutputData: true` em **Listar Eventos no Novo Horário (Remarcar)**, pra sempre passar
   pelo menos um item sintético adiante mesmo com zero eventos encontrados.
2. Ajuste em **Verificar Disponibilidade para Remarcar** pra não contar esse item sintético (sem
   `id`) como conflito: `filter(item => item.json.id && item.json.id !== eventoAtualId)` em vez
   de só `filter(item => item.json.id !== eventoAtualId)` — sem isso, o item sintético (`id`
   undefined) seria tratado como um conflito real e `available` sairia sempre `false`, mesmo com
   o horário livre.

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
     existe `bloqueado_em` com menos de 10 segundos?)
     - **Sim:** **Ignorar Mensagem (Telefone Ocupado)** (NoOp) — encerra sem responder.
     - **Não:** **Registrar Lock do Telefone** (`upsert`, grava `telefone` + `bloqueado_em: agora`)
       → segue para **Buscar Serviços e Preços**.
   - Janela fixa, sem node de "unlock" no final — decisão deliberada para manter o fluxo
     simples; suficiente para cobrir a duração normal de uma execução, mesmo sabendo que uma
     execução anormalmente lenta deixaria uma segunda mensagem passar. Reduzida de 30s para 10s
     em 2026-09-24 (ver "Bug: lock de telefone bloqueando confirmação" abaixo) — o valor original
     foi escolhido pensando só em cobrir a duração de UMA execução; ela nunca tinha em mente que o
     próprio fluxo passaria a exigir uma SEGUNDA mensagem do cliente (a confirmação) minutos ou
     segundos depois.
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
| `locks_telefone` | `telefone`, `bloqueado_em` | lock de 10s para evitar execuções paralelas do mesmo número |

> **Bug crítico corrigido (2026-09-24):** a partir do deploy desta rodada (23/09 20h56 UTC), o
> workflow parou de responder **qualquer** mensagem recebida, a qualquer hora do dia — não só
> fora do expediente. Causa: **Checar Mensagem Duplicada** e **Checar Lock do Telefone** (Data
> Table, `operation: get`) ficaram sem `alwaysOutputData` configurado corretamente. Quando a
> busca não encontra nenhuma linha — o caso normal, toda mensagem nova de um cliente — o node
> não produz nenhum item de saída, e a execução para ali mesmo, silenciosamente, antes de
> qualquer checagem de horário de funcionamento ou chamada à IA. O n8n registra essa execução
> como `"success"` (não lança erro), então o Error Workflow nunca disparava — não havia como o
> responsável perceber pela notificação de erro.
>
> Causa raiz do porquê `alwaysOutputData` nunca "pegou": `alwaysOutputData` é uma opção de
> execução do node (fica na raiz do node, irmã de `parameters`) — não é um parâmetro do node.
> Duas tentativas anteriores de configurá-la falharam por motivos diferentes: a primeira, ao
> criar esses nodes via uma operação de workflow que não aceita esse campo (foi descartado em
> silêncio); a segunda, ao tentar corrigir via uma operação que grava dentro de `parameters`
> (onde o n8n não lê essa opção — o node ficava com uma cópia inofensiva e inútil do campo).
> Corrigido usando a operação certa para configurações de node (não de parâmetro), que grava a
> opção no lugar certo. Confirmado com um teste isolado no node real (mesma Data Table, mesma
> configuração): a busca por uma mensagem inexistente passou a retornar 1 item sintético vazio,
> em vez de zero itens.

## Bug: remarcar só o horário apagava o serviço do evento (2026-09-24)

Ao remarcar mencionando só o novo horário (ex.: "Gostaria de remarcar para amanhã às 15h", sem
repetir o serviço), a planilha ficava correta (`servico` continuava "Corte de adulto") mas o
**título do evento no Google Calendar** virava "não especificado - José Zavaleta", perdendo o
serviço original.

**Causa:** **Atualizar Evento no Calendar** montava o `summary` direto a partir de
`output.servico` da IA — que sai `"não especificado"` quando o cliente não menciona o serviço
nessa mensagem (a IA não inventa um serviço que não foi dito). A planilha já não tinha esse
problema porque **Atualizar Linha na Planilha** sempre usava o `servico` da linha atual
(`Selecionar Agendamento Mais Recente (Remarcar)`) — mas incondicionalmente, o que também não
está certo: se o cliente pedisse pra trocar o serviço JUNTO com o horário, a planilha ignoraria
essa troca.

**Correção**, aplicada de forma simétrica nos dois nodes: usa o `servico` extraído da mensagem
atual só quando o cliente de fato mencionou um (diferente de `"não especificado"`/vazio);
caso contrário, mantém o `servico` já salvo na linha do agendamento.
`($json.output.servico === 'não especificado' || $json.output.servico === '') ? <servico atual
da linha> : $json.output.servico`. Isso cobre os dois cenários: remarcar só o horário preserva o
serviço original (Calendar e planilha), e remarcar pedindo também outro serviço agora atualiza
os dois lugares — antes só o Calendar mudava (de forma errada), a planilha nunca mudava.

## Padronização de formato de data/hora (2026-09-24)

`criado_em`/`atualizado_em` na planilha "Clientes - Automação PMEs" usavam `$now.toISO()`, que
inclui milissegundos (ex.: `2026-09-24T13:45:18.557-03:00`) — diferente do formato da coluna
`data`, que já vem sem milissegundos da IA (ex.: `2026-09-24T16:00:00-03:00`). Padronizado com
`$now.toISO({ suppressMilliseconds: true })` nos 3 nodes que escrevem essas colunas
(**Salvar Cliente na Planilha**, `criado_em`; **Atualizar Linha na Planilha** e **Atualizar
Linha na Planilha (Cancelar)**, `atualizado_em`). Vale só pra escritas novas — linhas antigas na
planilha não foram alteradas. Mesmo ajuste aplicado no workflow "Lembrete, Cancelamento e
Remarcação".

**Correção (2026-09-25):** esse fix não funcionava, por dois motivos:

1. `suppressMilliseconds: true` do Luxon só omite os milissegundos quando eles são `.000` —
   com `$now` real quase sempre saía com ms (ex.: `2026-09-24T22:00:48.681-03:00` na execução
   818 do workflow de lembrete). Trocado nos 6 pontos de escrita (3 aqui, 3 no lembrete) por
   `$now.toFormat("yyyy-MM-dd'T'HH:mm:ssZZ")`, que sempre gera `2026-09-25T10:27:19-03:00`
   (verificado na instância com timezone `America/Sao_Paulo`).
2. Neste workflow, o autosave da UI `bc9c9e6c` (09-24 16:58 UTC, uma aba do editor aberta com
   o estado antigo) sobrescreveu a versão MCP `b9284ee2` 6 minutos depois, voltando os 3 nodes
   pra `$now.toISO()` — e revertendo junto o fix de "preservar serviço original ao remarcar"
   (`summary` de **Atualizar Evento no Calendar** e `servico` de **Atualizar Linha na
   Planilha**). Os dois foram reaplicados. Antes de editar via MCP, feche abas do editor abertas
   nesse workflow.

## Identidade "Zap" (2026-09-24)

O assistente virtual da barbearia agora tem nome: **Zap**. Mudança só de identidade/tom no
system message de **Interpretar Intenção do Cliente** — nenhuma regra de negócio (intencao,
disponibilidade, confirmação, etc.) foi alterada:

- Linha de abertura do persona nomeia o assistente ("Você é o Zap, assistente virtual de
  agendamento de uma barbearia...").
- Nova regra de cenário: quando o cliente pergunta o nome/identidade do assistente (ex.:
  "qual seu nome?", "quem é você?", "você é uma pessoa de verdade?"), continua classificado
  como `intencao = "duvida"` — só muda o texto de resposta, sempre deixando claro que é uma
  automação, nunca fingindo ser uma pessoa real. Exemplo: "Sou o Zap, assistente virtual da
  barbearia! 😊 Posso te ajudar a agendar, remarcar ou cancelar um horário."
- Saudação inicial (duvida + primeira mensagem da conversa, sem histórico anterior) pode
  opcionalmente se apresentar como Zap — sem repetir o nome nas mensagens seguintes.

Testado com um workflow utilitário descartável (Manual Trigger → Set simulando a mensagem
"Qual seu nome?" → mesmo AI Agent/system message/parser do node real, apontando pro mesmo
credential Anthropic): retornou `intencao: "duvida"`, `confirmado: false`,
`confirmacao_texto: "Sou o Zap, assistente virtual da barbearia! 😊 Posso te ajudar a agendar,
remarcar ou cancelar um horário."` — confirmando o comportamento esperado sem precisar de uma
mensagem real via WhatsApp.

## Indicador de "digitando..." (2026-09-24)

Novo node **Ativar Indicador de Digitação** logo depois de **Registrar Lock do Telefone**,
antes de **Buscar Serviços e Preços**/IA — marca a mensagem recebida como lida e ativa
"digitando..." no WhatsApp do cliente enquanto o resto do fluxo processa (busca de serviços,
IA, Calendar, planilha).

O node nativo do WhatsApp Business Cloud (`n8n-nodes-base.whatsApp`) só tem operações de
`send`/`sendAndWait`/`sendTemplate`/mídia — não expõe `markAsRead`/`typing_indicator`. Por isso
é um **HTTP Request** chamando a Graph API diretamente:

```
POST https://graph.facebook.com/v21.0/{phone-number-id}/messages
{ "messaging_product": "whatsapp", "status": "read", "message_id": "<message_id>",
  "typing_indicator": { "type": "text" } }
```

- Autenticação via `authentication: predefinedCredentialType` + `nodeCredentialType:
  whatsAppApi`, reaproveitando a mesma credencial dos nodes de envio (token gerenciado pelo
  n8n, não hardcoded no node).
- `message_id` vem de `Normalizar Dados da Mensagem` (mesmo campo usado pela deduplicação).
- Não trava o fluxo se falhar: `onError: continueRegularOutput` no node + `neverError: true`
  na resposta — é só efeito visual, não crítico.
- O indicador some sozinho quando a mensagem de resposta final é enviada (ou depois de ~25s),
  sem precisar de node de "desligar".

## Bug: indicador de digitando com message_id vazio (2026-09-25)

Primeiro teste real do indicador de "digitando" (seção acima): o node rodou sem travar o
fluxo (como projetado), mas a chamada à Graph API retornava erro (`"The parameter to is
required"`, `code: 100`) — o indicador nunca aparecia de fato. Causa: `jsonBody` usava
`$json.message_id`, mas `$json` ali reflete a saída do node anterior imediato (**Registrar
Lock do Telefone**, um upsert na Data Table `locks_telefone`), que não tem campo
`message_id` — só `telefone`/`bloqueado_em`/`id`/etc. O corpo da requisição ia incompleto.
Corrigido referenciando `$('Normalizar Dados da Mensagem').item.json.message_id`
explicitamente, em vez de confiar em `$json` atravessar nodes intermediários sem esse campo.

## Bug: reação com emoji tratada como mensagem vazia (2026-09-25)

Cliente reagindo com emoji a uma mensagem anterior (recurso de reação nativo do WhatsApp, não
uma mensagem nova) chega com `messages[0].type === "reaction"`, sem `text.body`. O workflow
tratava isso como mensagem de texto vazia, gerando respostas repetidas de "não consegui ver
sua mensagem". Corrigido adicionando uma segunda condição em **Filtrar Apenas Mensagens**
(mesmo node que já filtra eventos de status/entrega/leitura): `messages[0].type !== "reaction"`.
Evento de reação é descartado no mesmo lugar, sem acionar IA nem enviar resposta alguma —
mesmo tratamento já dado a eventos de status.

## Ponte com o Wait node do workflow de Lembrete (2026-09-25)

Bug de roteamento entre os dois workflows: quando o cliente responde a um lembrete diário (o
workflow "Lembrete, Cancelamento e Remarcação" está com o node **Aguardar Resposta do Cliente**
— Wait, `resume: webhook` — pausado esperando aquele telefone), a resposta chega no webhook do
WhatsApp normalmente, e como só este workflow tem um trigger de WhatsApp real (o de lembrete só
tem Schedule Triggers), a mensagem era processada aqui como se fosse avulsa — resposta genérica
e fora de contexto, e o Wait do lembrete acabava estourando por timeout mesmo o cliente tendo
respondido. Essa ponte nunca tinha sido implementada (era uma limitação documentada no README
do lembrete: "Limitação conhecida do protótipo — Wait node").

**Implementação**, logo depois de **Registrar Lock do Telefone**, antes de qualquer
processamento normal:

1. **Verificar Espera de Lembrete** (Data Table `esperas_lembrete`, `get` por telefone,
   `alwaysOutputData: true`) — nova Data Table, compartilhada com o workflow de lembrete, que
   guarda `telefone` + `resume_url` (`$execution.resumeUrl` do lembrete) + `atualizado_em`.
2. **Há Espera de Lembrete Ativa?** (IF): `resume_url` existe e `atualizado_em` está dentro dos
   últimos 13 minutos (cobre o `resumeAmount` de 10min do Wait + margem de processamento) —
   mesmo padrão de `$json.campo ? (...) : false` já usado em `Telefone Ocupado?`.
   - **Não**: segue o fluxo normal (indicador de digitando, IA, etc.) — nada muda.
   - **Sim**: **Encaminhar Mensagem para Lembrete** (HTTP Request, POST pro `resume_url`,
     corpo = payload bruto de `Receber Mensagem WhatsApp`) — não roda mais nada aqui, nem IA
     nem indicador de digitando.
     - Sucesso: **Mensagem Encaminhada para Lembrete** (NoOp, termina aqui).
     - Falha (`onError: continueErrorOutput`, pra ter os dois outputs — sucesso e erro): cai de
       volta no fluxo normal em vez de perder a mensagem — cobre o caso de uma espera
       "fantasma" que já foi resolvida mas ainda não expirou pela janela de 13min.

**Duas descobertas confirmadas com testes isolados (workflows utilitários descartáveis, com
infraestrutura real — Data Table real, mecanismo de resume real, telefones fake pra não afetar
dados/clientes reais) antes de aplicar em produção:**
- `$execution.resumeUrl` é estável durante toda a vida da execução (não muda entre pausas
  repetidas do mesmo Wait, por exemplo pela reentrada por reação) — só precisa ser gravado uma
  vez por telefone por execução (e de novo a cada reentrada por reação, como refresh).
- O Wait node com `resume: webhook` **exige POST explícito** (`httpMethod: "POST"`) pra aceitar
  um corpo JSON — sem isso, o padrão é GET e a chamada de resume é rejeitada com 404. Os dois
  Wait nodes do workflow de lembrete não tinham esse campo definido; corrigido lá (ver o README
  daquele workflow).

Sem exclusão explícita da linha em `esperas_lembrete` quando a espera é resolvida — decisão
deliberada, mesmo padrão de janela fixa sem "unlock" já usado em `locks_telefone`: uma espera
antiga fica simplesmente fora da janela de 13min depois de resolvida, e mesmo se a chamada de
resume for tentada contra uma execução que já terminou, o fallback em caso de erro garante que
a mensagem não se perde.

### Auditoria de risco da janela de tempo (2026-09-25)

Antes de considerar essa ponte pronta pra produção, auditei node a node os pontos de risco de
uma abordagem baseada em janela de tempo (não em exclusão ativa). Resumo — nenhum risco real
encontrado, nenhuma mudança de código necessária, só confirmação:

1. **Janela (13min) vs. timeout do Wait (10min)**: os dois Wait nodes do lembrete usam
   `resumeAmount: 10, resumeUnit: minutes`. A janela de checagem (13min) é sempre MAIOR que o
   timeout do Wait — logo, nunca existe uma "zona morta" onde o Wait ainda está esperando mas a
   janela já expirou. No pior caso (tentar encaminhar pra uma espera que já estourou por
   timeout entre 10min e 13min atrás), a chamada de resume falha (404, execução não está mais
   esperando ali) e o `onError: continueErrorOutput` cai de volta pro fluxo normal — sem
   mensagem perdida, só uma tentativa extra de ~500ms.
2. **Refresh a cada reação**: confirmado por construção (não por convenção) — a única forma de
   reentrar em qualquer um dos dois Wait nodes é passando pelo respectivo node "Registrar
   Espera..." primeiro (tanto na entrada inicial quanto no loop-back de reação). Não existe
   caminho no grafo que resuma o Wait sem regravar `atualizado_em` antes.
3. **Múltiplos agendamentos no mesmo telefone**: `Processar Cada Agendamento` é um
   `splitInBatches` com `batchSize: 1` (padrão) — processamento estritamente sequencial, um
   item por vez, e uma pausa no Wait bloqueia a execução inteira até resolver (resposta ou
   timeout) antes do próximo item começar. Como `$execution.resumeUrl` é fixo pra toda a
   execução (não muda entre pausas), o segundo agendamento do mesmo telefone NUNCA escreve um
   `resume_url` diferente enquanto o primeiro ainda está em espera — ele simplesmente não
   começa a rodar até o primeiro terminar. **Testado empiricamente** com um workflow
   descartável simulando dois agendamentos em fila pro mesmo telefone fake: o primeiro resume
   corretamente com a resposta certa, o segundo (que só começa depois) também resume
   corretamente com a resposta certa, sem contaminação cruzada. Uma ambiguidade residual real,
   mas inerente ao domínio (não corrigível por implementação): se o cliente responder a um
   lembrete DEPOIS que ele já estourou por timeout E o segundo lembrete já foi enviado, a
   resposta tardia cai no segundo Wait — comportamento razoável (a conversa do primeiro já
   estava encerrada por timeout de qualquer forma), documentado aqui como limitação conhecida,
   não como bug.
4. **Linha órfã**: `esperas_lembrete` não é lida em nenhum outro lugar do sistema além dos 3
   nodes desta ponte. A escrita é sempre `upsert` por telefone (nunca `insert`), então a tabela
   nunca cresce sem limite — existe no máximo uma linha por telefone distinto que já recebeu um
   lembrete, reaproveitada pra sempre. Sem risco de volume/custo ao longo do tempo.
5. **Fuso horário**: `atualizado_em` é gravado com `$now.toISO()` nos dois workflows (mesmo
   padrão já usado e comprovado em produção por `bloqueado_em` em `locks_telefone`), e a
   comparação usa `DateTime.fromISO(...) > DateTime.now().minus(...)` — comparação de instante
   real via Luxon, não de string, então funciona corretamente independente do formato de exibição
   (UTC `Z` ou `-03:00`) do timestamp armazenado.

## Error Workflow centralizado (2026-09-24)

`settings.errorWorkflow` deste workflow, configurado direto na instância n8n, aponta para o
workflow **[Notificação de Erros](../notificacao-erros/README.md)**: qualquer erro que não
esteja coberto por `onError: continueRegularOutput` num node (ver seção acima sobre a rede de
segurança de `onError`) interrompe a execução normalmente, mas também dispara aquele workflow,
que avisa no WhatsApp com o nome do workflow, o node que falhou e o resumo do erro.

## Sugestão de horários livres (2026-09-28)

**Bug de produção (execução 1177):** quando o cliente pedia "me sugere uma data livre", a IA não
tinha como saber o que estava livre — só validava uma data que o cliente já tivesse escolhido.
Ela entrava em loop pedindo "uma data certinha", e numa das respostas chegou a inventar que
"qualquer dia de segunda a sábado costuma ter horários".

**Como funciona agora:** antes da IA, **Buscar Eventos dos Próximos Dias** lê os eventos do
Calendar (hoje + 14 dias) e **Calcular Horários Livres** (Code) monta uma linha por dia com as
janelas livres, já descontando:

- eventos ocupados, com o mesmo critério do free/busy usado em "Verificar Disponibilidade"
  (ignora cancelados e eventos marcados como "livre"; evento de dia inteiro bloqueia o dia);
- expediente (seg-sáb, 9h-18h) e domingos;
- antecedência mínima de 1h a partir de agora, arredondada pra cima em múltiplos de 30 min;
- janelas menores que o serviço mais curto da planilha.

A lista entra no system message na seção "HORÁRIOS LIVRES NA AGENDA", e a regra "CLIENTE PEDE
SUGESTÃO DE DIA/HORÁRIO" manda a IA oferecer 2-3 opções concretas, só dessa lista, com início +
duração do serviço cabendo na janela (60 min se o serviço ainda não foi dito). Quando o cliente
escolhe uma das opções que foram oferecidas ("a segunda", "quarta às 9h") e o serviço já é
conhecido, `confirmado = true`: o fluxo normal de agendar ainda roda "Validar Horário de
Funcionamento" e "Verificar Disponibilidade" antes de "Criar Evento", então uma sugestão errada da
IA nunca vira evento sem a checagem de sempre.

Se a leitura do Calendar falhar (`onError: continueRegularOutput`), a lista vira
"INDISPONÍVEL" e a IA pede o dia e o horário de preferência do cliente em vez de inventar
disponibilidade.

Limites: só enxerga 14 dias à frente (pra "tem algo em novembro?" a IA pede uma data específica)
e a lista não inclui a pausa de almoço — o expediente é o mesmo 9h-18h de "Validar Horário de
Funcionamento".

## Qual agendamento cancelar ou remarcar (2026-09-28)

**Bug de produção (execução 1246, 28/09 às 09:02):** o cliente tinha 4 agendamentos ativos (Corte
Baixo na Máquina hoje às 10h, Corte Feminino Com Lavagem hoje às 13h, Corte 3D sexta 02/10 às 11h,
Corte Social na Tesoura 10/10) e pediu "cancelar o corte de hoje para minha esposa". O sistema
cancelou o **Corte 3D de 02/10**, que nem era de hoje.

A escolha não foi da IA: ela só devolveu `intencao: "cancelar"`, `servico: "corte"`, sem data.
Quem escolheu foi o node Limit "Selecionar Agendamento Mais Recente (Cancelar)" (`keep:
lastItems`), que pegava **a última linha ativa da planilha**. Naquele momento, era o Corte 3D que
tinha acabado de ser remarcado. A IA nunca viu a lista de agendamentos, e mesmo assim escreveu
"Cancelei aqui o corte da sua esposa marcado pra hoje", mensagem que foi para o cliente. O mesmo
Limit existia no caminho de remarcar.

**Como funciona agora:**

1. **Buscar Agendamentos Ativos do Cliente** + **Formatar Agendamentos Ativos** (antes da IA)
   injetam no prompt a seção "AGENDAMENTOS ATIVOS DESTE CLIENTE": uma linha por agendamento
   ativo e futuro, no formato `[event_id] serviço — dia da semana, dd/MM (AAAA-MM-DD) às HH:mm`.
2. A regra "QUAL AGENDAMENTO CANCELAR OU REMARCAR" manda a IA preencher o campo novo
   `agendamento_alvo` (também adicionado ao schema do parser) com o `event_id`, **só** quando o
   serviço, a data e/ou o horário citados baterem com um único agendamento. Referências como "o
   de hoje" (havendo 2 hoje), "o corte", "o da minha esposa", "o mais cedo" ou "meu horário" são
   ambíguas: `agendamento_alvo = ""` e a IA pergunta qual, sem deduzir pelo tipo de serviço, pela
   ordem ou pelo que foi mexido antes na conversa.
3. **Identificar Agendamento Escolhido (Cancelar/Remarcar)** (Code, no lugar dos Limit) é a
   trava determinística:
   - usa o agendamento cujo `event_id` é o `agendamento_alvo`;
   - se não houver alvo válido e o cliente tiver só **um** agendamento ativo, usa esse;
   - com 2+ ativos e nenhum identificado (inclusive um id inventado pela IA), **não escolhe**:
     sai com `precisa_escolher: true` e a pergunta em `mensagem_escolha`, e **Perguntar Qual
     Agendamento no WhatsApp** envia essa pergunta. A mensagem é a pergunta da IA, que fica na
     memória e deixa o próximo "o das 13h" fazer sentido. Se o texto da IA parecer ação
     concluída ("cancelei", "remarcado"...) ou trouxer data numérica, o código usa em vez dela uma
     pergunta montada com os dados reais: "Você tem dois agendamentos hoje: Corte Baixo na Máquina
     às 10h e Corte Feminino Com Lavagem às 13h — qual deles você quer cancelar?".
   - Agendamento com horário já passado não entra na lista (nem no prompt).
4. **Confirmar Cancelamento no WhatsApp** deixou de usar a frase da IA: monta o texto com o
   serviço, o dia e o horário da linha que foi de fato cancelada.

### Consulta com vários agendamentos (2026-09-28)

"Formatar Resposta da Consulta" passou a listar só os agendamentos **futuros**. Antes, um
"agendado" de hoje cedo que já tinha passado ainda aparecia como marcado. Cada item também mostra
"(para X)" quando o agendamento não é do próprio cliente.

### Beneficiário (2026-09-28)

Coluna `beneficiario` (J) na planilha "Clientes - Automação PMEs": para quem é o atendimento.

- **Criação:** a IA preenche o campo novo `beneficiario` do schema. O padrão é "Eu mesmo". Se o
  cliente diz que é para outra pessoa, vai "Nome - vínculo" quando ele dá o nome ("Ana - esposa")
  ou só o vínculo quando não dá ("Esposa", "Filho", "Pai"). "Salvar Cliente na Planilha" grava o
  valor, com "Eu mesmo" se vier vazio. "Criar Evento no Calendar" acrescenta "Para: …" na
  descrição do evento quando não é o próprio cliente. A proposta e a confirmação mencionam a
  pessoa ("…pra Ana amanhã às 10h").
- **Desambiguação:** a lista de agendamentos ativos no prompt mostra `— para: <beneficiário>`, e a
  regra "QUAL AGENDAMENTO CANCELAR OU REMARCAR" casa por beneficiário + serviço + data + horário.
  Se só um agendamento é do beneficiário citado, age direto. Se sobrar mais de um (ex.: dois
  "Filho"), pergunta listando só os que continuam possíveis. A pergunta do código e a confirmação
  de cancelamento mostram "(para X)" quando não é o próprio cliente.
- **Linhas antigas (vazias):** aparecem como "não informado" e não casam com nenhum beneficiário.
  Se só um agendamento tem o beneficiário citado, é esse. Se nenhum tem e existem linhas "não
  informado", a IA pergunta, sem deduzir pelo tipo de serviço. Esse é o caso de todas as linhas
  que já existiam em 28/09; elas não foram preenchidas retroativamente, porque não dá para saber
  de quem eram.
- A coluna foi criada escrevendo só a célula J1, depois de confirmar que o cabeçalho tinha
  exatamente as 9 colunas esperadas. Os updates dos dois workflows mapeiam até `atualizado_em`
  (coluna I), então não apagam J (ver o bug de range do Update Row mais abaixo).

Os nodes "Selecionar Agendamento Mais Recente (...)" foram renomeados para "Identificar
Agendamento Escolhido (...)". As menções ao nome antigo nas seções de bugs anteriores descrevem o
comportamento da época.

## Falha de formato da IA (2026-09-28)

**Bug de produção (execução 1177):** o "Parser Estruturado de Agendamento" rejeitou a resposta com
`Model output doesn't fit required format` e a execução inteira morreu sem responder o cliente. O
conteúdo da resposta estava correto; o problema foi o formato — o modelo devolveu o campo `output`
como uma **string** contendo o JSON (`{"output":"{\"intencao\":...}"}`) em vez de um objeto.

Três camadas de proteção agora:

1. **Prompt** — regra explícita em REGRAS GERAIS: `output` é um objeto, nunca uma string com JSON.
2. **Retry** — "Interpretar Intenção do Cliente" tem `retryOnFail` (2 tentativas, 1s entre elas).
   Como o agente não salva nada na memória quando falha, a segunda tentativa começa limpa. O
   indicador de "digitando..." continua ativo, então o cliente não percebe a nova tentativa.
3. **Saída de erro** — se as 2 tentativas falharem (`onError: continueErrorOutput`), o item sai
   pela segunda saída do agente com `{ error }`: **Avisar Cliente Sobre Falha da IA** manda uma
   mensagem dizendo que alguém da equipe vai responder, e **Escalar Falha da IA para a Equipe**
   (Stop and Error) encerra a execução com erro. Isso dispara o Error Workflow, que avisa o
   responsável no WhatsApp com o nome do cliente, o telefone, a mensagem original e o motivo.

`retryOnFail` e `continueErrorOutput` funcionam juntos no AI Agent v3: foi testado num workflow
isolado com um schema impossível de satisfazer. Com retry, o parser rodou 2 vezes; sem retry,
rodou 1 vez; nos dois casos o item saiu pela saída de erro com status `success`.

## Escopo da conversa (2026-09-28)

Três categorias além do fluxo de agendamento, decididas pelo "Interpretar Intenção do Cliente"
(regra "O QUE VOCÊ NÃO SABE E O QUE NÃO É DA BARBEARIA" do prompt):

| Categoria | `intencao` | O que acontece |
|---|---|---|
| Serviços, preços da lista, horário de funcionamento, saudações, **endereço e formas de pagamento** (bloco "INFORMAÇÕES DA BARBEARIA" do prompt, desde 28/09) | `duvida` | A IA responde direto com os dados do prompt, sem acrescentar nada (ex.: rota, parcelamento) |
| Pergunta legítima sobre a barbearia que os dados não cobrem (serviço/produto fora da planilha, estacionamento, parcelamento…) | `encaminhar` | Cliente recebe texto fixo ("…já repassei sua dúvida pro responsável…"). O responsável recebe no WhatsApp "📩 Dúvida de cliente sobre o negócio (não é erro do sistema)", com nome, telefone, mensagem e link `wa.me` |
| Assunto sem relação com a barbearia, ou tentativa de mudar as instruções (jailbreak/prompt injection, inclusive disfarçada de pergunta sobre o negócio) | `fora_do_escopo` | Texto fixo de recusa; nada é encaminhado |

- **Textos fixos no node, não gerados pela IA.** "Recusar Assunto Fora do Escopo no WhatsApp",
  "Avisar Cliente Sobre Dúvida Encaminhada no WhatsApp" e "Avisar Dúvida Já Encaminhada no
  WhatsApp" enviam um `textBody` literal. Mesmo que uma injeção manipule o `confirmacao_texto`, a
  mensagem enviada não muda: o que a IA ainda decide é só a categoria. O prompt manda a IA
  preencher `confirmacao_texto` com o mesmo texto, para o histórico da memória ficar coerente com
  o que foi enviado.
- **Anti-spam (1 encaminhamento por telefone a cada 30 min).** Segue o padrão do
  `locks_telefone`, numa Data Table própria `encaminhamentos_duvida` (`telefone`,
  `encaminhado_em`). A tabela é separada porque o lock é reescrito a cada mensagem.
  - Fluxo: "Checar Encaminhamento Recente" (get, `alwaysOutputData`) → "Já Encaminhou Nos Últimos
    30 Min?".
  - Se sim: "Avisar Dúvida Já Encaminhada no WhatsApp" responde que a dúvida já foi repassada e
    pede para aguardar o contato; nada é reenviado.
  - Se não: "Registrar Encaminhamento de Dúvida" (upsert) → "Encaminhar Dúvida ao Responsável no
    WhatsApp" → "Avisar Cliente Sobre Dúvida Encaminhada no WhatsApp".
- **Aviso ao responsável.** Mesmo número de envio e mesmo destino (5511975049937) do node do Error
  Workflow "Notificação de Erros". Não chama aquele workflow porque ele só tem Error Trigger, que
  não recebe dados de outro workflow. O node tem `onError: continueRegularOutput`: se o aviso falhar
  (ex.: fora da janela de 24h da Meta), o cliente ainda recebe a resposta.

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
