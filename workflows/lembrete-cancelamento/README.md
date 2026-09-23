# Lembrete, Cancelamento e Remarcação

Workflow n8n: [`lembrete-cancelamento.json`](./lembrete-cancelamento.json) · [abrir na instância](https://n8n-n8n.wg1izd.easypanel.host/workflow/wkIfUOGhEom4rFow)

Todo dia às 8h, avisa cada cliente com horário marcado para o dia atual e trata a resposta
dele: confirmar, cancelar ou remarcar.

## Fluxo

1. **Disparar Lembrete Diário às 8h** — Schedule Trigger.
2. **Buscar Agendamentos de Hoje (Planilha)** — Google Sheets, lê todas as linhas da planilha
   **"Clientes - Automação PMEs"** (aba "Sheet1") → **Filtrar Data de Hoje** (Filter) mantém só
   as linhas cuja coluna `data` cai no dia de hoje (ver "Fonte de dados do lembrete" abaixo).
3. **Processar Cada Agendamento** — loop (Split in Batches, um agendamento por vez). Para cada um:
   1. **Enviar Lembrete no WhatsApp** — avisa o horário do agendamento (formatado a partir da
      coluna `data`, ver "Histórico de correções" abaixo) e pergunta se confirma, cancela ou
      quer remarcar. Usa os dados (`nome`, `telefone`, `servico`, `data`, `event_id`) direto do
      item da planilha que chegou nesta iteração do loop — não há mais um lookup separado.
   2. **Aguardar Resposta do Cliente** — Wait node (retomado via webhook), com limite de
      espera de 24h (ver seção "Timeout de espera" abaixo).
   3. **Cliente Respondeu ou Deu Timeout?** (IF) — diferencia resposta real de timeout:
      - **Sim (respondeu):** segue para **Classificar Resposta do Lembrete** — AI Agent
        (Claude) classifica a resposta em `confirmar` / `cancelar` / `remarcar` (e, no caso de
        remarcação, já extrai o novo dia/horário pedido). Uma resposta que é só um emoji ou um
        agradecimento curto ("ok", "blz", "obrigado", "👍") é tratada como confirmação
        implícita — `confirmar` — em vez de cair em `decisao = "indefinido"` (ver "Histórico de
        correções" abaixo) → **Confirmar, Cancelar ou
        Remarcar?** (Switch) — 4 caminhos:
        - **Confirmar:** envia mensagem final de confirmação.
        - **Cancelar:** deleta o evento no Calendar (pelo `event_id`) → confirma o cancelamento.
        - **Remarcar:** **Validar Horário de Funcionamento** (Code) → **Horário Dentro do
          Expediente?** (IF) — mesma validação determinística (09h-18h, seg-sáb) do workflow
          "Agendamento via WhatsApp", ver "Horário de funcionamento na remarcação" abaixo.
          - **Fora do expediente:** avisa o cliente e pede outro horário, sem consultar o
            Calendar nem tocar na planilha.
          - **Dentro do expediente:** verifica se o novo horário está livre → se sim,
            **atualiza** o evento existente (não cria um novo) e atualiza a coluna `data` na
            planilha (casando pela coluna `event_id`) → confirma a remarcação; se não, pede
            outro horário.
        - **Não entendi (fallback):** pede para o cliente esclarecer a resposta.
      - **Não (timeout):** **Avisar Timeout do Lembrete no WhatsApp** — avisa o cliente que não
        houve resposta e que o agendamento foi mantido como está. Não passa pela
        classificação/confirmação/cancelamento/remarcação.
   4. Volta para o próximo agendamento do lote.

## Timeout de espera no lembrete

O node **Aguardar Resposta do Cliente** (Wait, `resume: "webhook"`) antes ficava esperando a
resposta do cliente indefinidamente — se o cliente nunca respondesse, a execução ficava presa
em "Waiting" para sempre.

- **Limit Wait Time:** ativado (`limitWaitTime: true`), com `limitType: "afterTimeInterval"`,
  `resumeAmount: 10`, `resumeUnit: "minutes"` — a execução retoma automaticamente depois de 10
  minutos mesmo sem resposta do cliente (valor de produção; ver "Bug: mensagem de timeout com
  dados desatualizados" abaixo — chegou a ficar em 5 minutos por um teste manual esquecido).
- **Detecção de timeout:** o node IF **Cliente Respondeu ou Deu Timeout?**, logo depois do
  Wait, verifica se o texto da mensagem do cliente está presente no payload:
  `{{ $json.body?.messages?.[0]?.text?.body ?? $json.messages?.[0]?.text?.body ?? "" }}`
  com o operador **"not empty"**. Essa é a mesma expressão já usada em "Normalizar Resposta do
  Lembrete" para extrair o texto da resposta. Quando o Wait retoma por resposta real via
  webhook, esse campo vem preenchido com o texto do cliente. Quando retoma por timeout, o item
  que sai do Wait é o mesmo que entrou nele (a resposta do envio do lembrete no WhatsApp, que
  não tem `messages[0].text`), então a expressão resulta em string vazia — condição falsa.
- **Branch de timeout:** antes de avisar o cliente, passa por **Reverificar Agendamento Antes
  do Timeout** → **Agendamento Ainda É o Mesmo?** (ver "Bug: mensagem de timeout com dados
  desatualizados" abaixo) — só quando o agendamento não mudou é que vai para **Avisar Timeout
  do Lembrete no WhatsApp**, que envia uma mensagem simples avisando que não houve resposta e
  que o agendamento continua como está, usando `$('Processar Cada Agendamento')` (nome,
  serviço, telefone) — essa referência continua acessível mesmo após o timeout, pois esse node
  já rodou antes do Wait no mesmo caminho de execução. Depois, o fluxo volta para **Processar
  Cada Agendamento** para continuar o loop com o próximo agendamento do lote (igual a todos os
  outros caminhos terminais deste
  workflow) — sem isso, o loop pararia e os agendamentos seguintes do dia não seriam
  processados.

## Fonte de dados do lembrete

O node de busca do dia (**"Buscar Agendamentos de Hoje (Planilha)"**) lê a planilha **"Clientes
- Automação PMEs"** (aba "Sheet1"), não o Google Calendar — ver "Histórico de correções" abaixo
para o motivo. Cada linha já traz `nome`, `telefone`, `servico`, `data` e `event_id` prontos, o
que elimina o lookup que existia antes.

O Google Sheets não tem como filtrar por "só a parte da data" de uma coluna de datetime
diretamente na leitura (o filtro nativo do node só faz igualdade exata contra o valor bruto da
célula) — por isso a leitura traz **todas** as linhas, e um node **Filter** logo depois
(**"Filtrar Data de Hoje"**) mantém só as linhas de hoje, comparando as duas datas já reduzidas
ao formato `yyyy-MM-dd` em `America/Sao_Paulo`:

```
Esquerda: {{ DateTime.fromISO($json.data).setZone('America/Sao_Paulo').toFormat('yyyy-MM-dd') }}
Direita:  {{ $today.toFormat('yyyy-MM-dd') }}
Operador: string equals
```

`$today` já vem no fuso do workflow (`America/Sao_Paulo`, definido nas settings), então não
precisa de `setZone` do lado direito. Isso ignora completamente o horário — um agendamento às
23h59 ou à 00h01 do mesmo dia é tratado igual.

A partir daqui, o item que entra em **"Processar Cada Agendamento"** já É o dado do cliente — os
nodes seguintes (envio do lembrete, confirmação, cancelamento, atualização de evento/planilha
na remarcação) leem tudo via `$json` (no primeiro node do loop) ou `$('Processar Cada
Agendamento').item.json` (nos nodes mais adiante, depois do Wait), sem nenhum lookup adicional.

## Horário de funcionamento na remarcação

A barbearia funciona de segunda a sábado, das 09h às 18h — mesma regra do workflow
"Agendamento via WhatsApp" (ver `workflows/agendamento/README.md`, seção "Horário de
funcionamento"). Quando o cliente responde ao lembrete pedindo para remarcar, o
`novo_horario_inicio`/`novo_horario_fim` extraído pela IA (**Classificar Resposta do
Lembrete**) passa por **Validar Horário de Funcionamento** (Code) → **Horário Dentro do
Expediente?** (IF), inseridos entre "Confirmar, Cancelar ou Remarcar?" (saída remarcar) e
"Verificar Novo Horário Disponível". O Code node calcula, em `America/Sao_Paulo` (Luxon):

- `novo_horario_inicio` tem hora entre 09:00 (inclusive) e 18:00 (exclusive);
- `novo_horario_fim` não ultrapassa 18:00;
- o dia da semana de `novo_horario_inicio` não é domingo (`weekday !== 7`).

Se qualquer condição falhar, o fluxo não chega a "Verificar Novo Horário Disponível" nem a
"Atualizar Evento no Calendar"/"Atualizar Data na Planilha" — vai direto para **Avisar Horário
Fora do Expediente no WhatsApp** (reaproveita o mesmo texto usado em "Agendamento via
WhatsApp"), e volta para **Processar Cada Agendamento** para continuar o loop. Antes dessa
mudança, o ramo de remarcação só checava disponibilidade no Calendar — um cliente podia pedir
(e a IA aceitar) um horário fora do expediente, e o evento seria atualizado normalmente desde
que a Calendar API confirmasse "livre" naquele horário.

## Histórico de correções

- **2026-09-24 — Faltava validação de horário de funcionamento na remarcação:** o ramo de
  remarcação (cliente responde ao lembrete pedindo outro horário) só checava disponibilidade
  no Google Calendar antes de atualizar o evento — não existia nenhuma checagem de horário de
  funcionamento (09h-18h, seg-sáb), diferente do workflow "Agendamento via WhatsApp", que já
  tinha essa validação nos fluxos de agendar e remarcar desde 2026-09-23. Um cliente podia
  pedir (e a IA aceitar) remarcar para um horário fora do expediente, e o evento seria
  atualizado normalmente contanto que a Calendar API confirmasse o horário como livre.
  Adicionados **Validar Horário de Funcionamento** (Code) → **Horário Dentro do Expediente?**
  (IF), replicando a mesma lógica e o mesmo texto de aviso do outro workflow — ver "Horário de
  funcionamento na remarcação" acima.
- **2026-09-24 — Emoji/agradecimento curto tratado como resposta não reconhecida:** cliente
  respondendo ao lembrete com só um emoji (👍, 🙏) ou um agradecimento curto ("ok", "blz",
  "obrigado") — normalmente reconhecendo o lembrete, sem pedir nada — caía em
  `decisao = "indefinido"` e recebia "Desculpa, não entendi sua resposta. Você quer confirmar,
  cancelar ou remarcar o horário?", soando robótico para algo que já era, na prática, uma
  confirmação. Nova regra no System Message de **Classificar Resposta do Lembrete**: esse tipo
  de resposta agora é tratado como confirmação implícita (`decisao = "confirmar"`), já que o
  cliente só está reconhecendo a mensagem, sem pedir cancelamento ou remarcação —
  `decisao = "indefinido"` fica reservada para respostas realmente ambíguas. Mesma regra
  aplicada no mesmo dia ao node "Interpretar Intenção do Cliente" do workflow "Agendamento via
  WhatsApp", que tinha o problema análogo.
- **2026-09-24 — Bug: mensagem de timeout com dados desatualizados ("execução fantasma"):**
  cliente recebeu o lembrete às 10h36 (corte hoje às 16h), negociou remarcar em sequência
  rápida (16h30 → ocupado, 17h → confirmado com sucesso pelo workflow "Agendamento via
  WhatsApp") e, minutos depois, recebeu de volta "Não recebi resposta sobre o lembrete do seu
  horário de corte hoje **às 16h**. Vou manter seu agendamento como está" — mencionando o
  horário antigo, já superado pela remarcação.
  - **Investigação:** não havia nenhuma execução em `"waiting"` (nem `"running"`/`"new"`) ativa
    em nenhum dos dois workflows no momento da investigação — a execução responsável (nº 443)
    já tinha completado. Ela havia sido disparada manualmente às 10h36 (`13:36:48Z`) durante
    testes deste mesmo dia, encontrou o cliente na planilha, enviou o lembrete e ficou esperando
    no node **Aguardar Resposta do Cliente**. O **"Aguardar Resposta do Cliente"** estava com
    `resumeAmount: 5` **minutos** (não os 10 minutos de produção) — sobra de um teste anterior
    que não tinha sido revertido. A execução expirou às 10h41 (`13:41:50Z`), exatamente na
    janela em que o cliente negociava o novo horário por WhatsApp.
  - **Causa raiz:** a resposta real do cliente ("17h", "17:30") não chega ao webhook desse Wait
    node — ela chega pelo webhook do WhatsApp, que é o trigger do workflow **"Agendamento via
    WhatsApp"** (ver "Limitação conhecida do protótipo — Wait node" abaixo; essa ponte nunca foi
    implementada). Por isso a remarcação foi processada corretamente pelo outro workflow
    (mudando `data` na planilha), mas esta execução, presa no Wait, **não tinha como saber
    disso** e, ao expirar, enviou a mensagem de timeout com o horário antigo que tinha capturado
    no momento do envio do lembrete — uma colisão de tempo entre os dois workflows, não uma
    duplicidade de execuções.
  - **Correções:**
    1. `resumeAmount` do **Aguardar Resposta do Cliente** corrigido de volta para `10` minutos.
    2. Nenhuma execução presa em `"waiting"` foi encontrada para cancelar.
    3. **Proteção estrutural:** antes de enviar a mensagem de timeout, dois novos nodes —
       **Reverificar Agendamento Antes do Timeout** (Google Sheets, relê a linha pelo
       `event_id`) → **Agendamento Ainda É o Mesmo?** (IF, compara a `data` atual da planilha
       com a `data` que esta execução capturou ao enviar o lembrete) — detectam se o
       agendamento mudou (remarcado) ou sumiu (cancelado) por **qualquer caminho**, inclusive
       pelo workflow de Agendamento, enquanto esta execução esperava. Se mudou, a mensagem de
       timeout é suprimida em vez de avisar o cliente com dado velho. Optei por essa checagem
       na planilha (fonte única da verdade) em vez de um controle "só a execução mais recente
       vale" por telefone: a remarcação do incidente real aconteceu num workflow totalmente
       separado, que nunca saberia de um controle desse tipo mantido só neste workflow.
  - **Limitação que continua não resolvida:** mesmo com essas correções, o Wait node **nunca
    recebe respostas reais** em produção (a ponte do "Limitação conhecida" abaixo segue
    pendente) — ou seja, toda execução deste fluxo vai, mais cedo ou mais tarde, cair no branch
    de timeout, mesmo quando o cliente responde normalmente pelo WhatsApp (a resposta só é
    processada pelo outro workflow). A proteção acima evita a mensagem ficar **desatualizada**,
    mas não faz o Wait node efetivamente escutar a resposta do cliente — isso continua sendo o
    próximo passo antes de produção, descrito em "Limitação conhecida do protótipo" abaixo.

- **2026-09-24 — Fonte de dados trocada de Calendar para a planilha (bug: loop travava e
  pulava clientes reais):** o Google Calendar usado como fonte (**"Buscar Agendamentos de
  Hoje"**) é a agenda pessoal, que mistura compromissos sem cliente correspondente na planilha
  (reuniões, entrevistas) com os agendamentos reais da barbearia. Quando o loop **"Processar
  Cada Agendamento"** chegava num desses eventos pessoais, o lookup seguinte (**"Buscar Dados
  do Cliente na Planilha"**, por `event_id`) não encontrava nada e os nodes seguintes, que
  dependiam de campos como `nome`/`telefone`, falhavam — por padrão isso para a execução
  inteira, então nenhum cliente real agendado depois daquele item no lote recebia o lembrete.
  Correção em duas partes:
  - **Fonte trocada para a planilha:** **"Buscar Agendamentos de Hoje"** (Calendar, `getAll`)
    substituído por **"Buscar Agendamentos de Hoje (Planilha)"** (Google Sheets, lê todas as
    linhas de "Clientes - Automação PMEs") → **"Filtrar Data de Hoje"** (Filter, mantém só as
    linhas de hoje — ver "Fonte de dados do lembrete" acima). Como a planilha só tem
    agendamentos reais, esse tipo de item "órfão" deixa de existir. O lookup **"Buscar Dados do
    Cliente na Planilha"** foi removido (o próprio item do loop já tem tudo); todos os nodes
    que referenciavam `$('Buscar Dados do Cliente na Planilha')` passaram a referenciar
    `$('Processar Cada Agendamento')`.
  - **Rede de segurança:** mesmo com a causa raiz eliminada, os nodes de envio/escrita dentro
    do loop (WhatsApp, Calendar, Sheets — 12 nodes) ganharam `onError: continueRegularOutput`,
    para que uma falha pontual em qualquer item (ex.: número de telefone inválido, API fora do
    ar) não pare o processamento dos clientes seguintes do dia.
  - **Testado manualmente** (execução de teste, `execute_workflow` em modo manual): a planilha
    tinha 6 linhas naquele momento (datas de 22, 24 ×2, 25 e 28/09, e uma de 23/09), e o filtro
    manteve corretamente só a linha de **23/09 às 16h (José Zavaleta, corte)** — o lembrete foi
    enviado com sucesso só para esse número.
- **2026-09-24 — Lembrete chegava depois da abertura e avisava sobre o dia errado:**
  - **Horário de disparo:** o Schedule Trigger rodava às 9h — exatamente quando a barbearia
    abre — então o lembrete chegava tarde demais para o cliente decidir com antecedência.
    Renomeado de **"Disparar Lembrete Diário às 9h"** para **"Disparar Lembrete Diário às 8h"**
    e `triggerAtHour` ajustado de `9` para `8` (mesmo fuso `America/Sao_Paulo`, já configurado
    nas settings do workflow).
  - **Dia errado buscado:** o node de busca no Calendar (`getAll`) usava
    `timeMin: {{ $today.plus(1, 'days') }}` / `timeMax: {{ $today.plus(2, 'days') }}` — ou seja,
    buscava os agendamentos de **amanhã**, não os de hoje, e a mensagem dizia "amanhã às
    14h" para um agendamento que era, na verdade, do dia seguinte à execução. Renomeado de
    **"Buscar Agendamentos de Amanhã"** para **"Buscar Agendamentos de Hoje"**, com
    `timeMin: {{ $today }}` / `timeMax: {{ $today.plus(1, 'days') }}` (00:00 de hoje até 00:00
    de amanhã, cobrindo o dia inteiro). As mensagens de **Enviar Lembrete no WhatsApp** e
    **Avisar Timeout do Lembrete no WhatsApp** trocaram "amanhã" por "hoje".
  - **Loop processando só 1 cliente:** investigado na execução de produção mais recente (nº
    381) — o node **Processar Cada Agendamento** (Split In Batches) já processava corretamente
    todos os eventos retornados pelo Calendar naquele dia (2 eventos → 2 iterações → 2
    mensagens de WhatsApp enviadas, uma por evento), e **Buscar Dados do Cliente na Planilha**
    / **Enviar Lembrete no WhatsApp** já usam o item corrente (`$json`) da iteração, não um
    item fixo. Não havia bug no loop: no dia da execução investigada só existia uma cliente
    (Veridiana) com dois agendamentos diferentes, por isso pareceu "1 lembrete" quando na
    verdade foram 2 mensagens distintas, uma para cada horário dela.
- **2026-09-24 — Lembrete e aviso de timeout sem o horário do agendamento:** as mensagens de
  **Enviar Lembrete no WhatsApp** e **Avisar Timeout do Lembrete no WhatsApp** mencionavam o
  serviço e "amanhã", mas nunca o horário — o cliente recebia "Passando para lembrar do seu
  horário de corte amanhã" sem saber a que horas era. Corrigido formatando a coluna `data` da
  planilha (ISO 8601, ex. `2026-09-24T14:00:00-03:00`) com Luxon, em `America/Sao_Paulo`, no
  formato natural em português (`14h` quando os minutos são zero, `14h30` quando não são):
  ```
  {{ $json.data ? (dt => dt.minute === 0 ? `${dt.toFormat('H')}h` : `${dt.toFormat('H')}h${dt.toFormat('mm')}`)(DateTime.fromISO($json.data).setZone('America/Sao_Paulo')) : '' }}
  ```
  Em **Enviar Lembrete no WhatsApp** a expressão lê `$json.data` (item corrente, saído direto de
  "Buscar Dados do Cliente na Planilha"); em **Avisar Timeout do Lembrete no WhatsApp** ela lê
  `$('Buscar Dados do Cliente na Planilha').item.json.data`, já que esse node está depois do
  Wait/timeout e precisa da referência nomeada ao node de origem. Os outros nodes de envio do
  workflow (confirmação final, cancelamento, remarcação, pedir outro horário, pedir
  esclarecimento) não precisaram do mesmo ajuste: a remarcação já usa
  `resposta_sugerida` (texto por extenso gerado pela IA com a nova data/horário) e os demais não
  fazem referência a um horário específico.
- **2026-09-22 — Lembretes/atualizações duplicadas na planilha (causa raiz):**
  - **Buscar Dados do Cliente na Planilha** (Google Sheets) estava sem `resource`/`operation`/
    filtro nenhum configurado (lia sem localizar a linha certa). Corrigido para
    `resource: "sheet"`, `operation: "read"`, com filtro por `event_id`
    (`lookupColumn: "event_id"`, `lookupValue: "={{ $json.id }}"`, usando o `id` do evento do
    Google Calendar em processamento no loop "Processar Cada Agendamento") e
    `options.returnFirstMatch: true`, garantindo 1 linha por evento.
  - **Atualizar Data na Planilha** (Google Sheets) estava com `columns.matchingColumns: []`
    (vazio) — sem coluna de correspondência, o Update do Google Sheets cria uma linha nova em
    vez de atualizar a existente, o que gerou linhas duplicadas por telefone a cada remarcação.
    Corrigido para `matchingColumns: ["event_id"]`. O node também reescrevia a data **antiga**
    (lida de `Buscar Dados do Cliente na Planilha`) em vez da nova data remarcada; corrigido
    para gravar `$('Classificar Resposta do Lembrete').item.json.output.novo_horario_inicio`
    na coluna `data`.
  - Esses dois nodes já haviam sido alinhados nas convenções de `agendamento.json` (`resource`
    explícito, filtro por `event_id`, `matchingColumns`) numa revisão anterior, mas a
    configuração foi resetada por edições feitas depois diretamente no editor do n8n — por isso
    a correção foi reaplicada e confirmada como persistente.
- **2026-09-22 — Ajustes de convenção pontuais que permanecem ativos:** as saídas do Switch
  "Confirmar, Cancelar ou Remarcar?" têm `renameOutput`/`outputKey` (`confirmar`/`cancelar`/
  `remarcar`) e as settings do workflow têm `timezone: "America/Sao_Paulo"` (usado pelas
  expressões `$today`/`$now`) e `callerPolicy: "workflowsFromSameOwner"`, alinhado com
  `agendamento.json`.

> **Nota:** o node "Buscar Agendamentos de Amanhã" e os nodes de evento do Google Calendar
> (`Cancelar Evento no Calendar`, `Atualizar Evento no Calendar`) não declaram `resource: "event"`
> explicitamente no momento — o n8n usa o resource padrão do node. Isso não é um bug funcional,
> mas difere da convenção explícita usada em `agendamento.json`; não foi reaplicado nesta rodada
> para não conflitar com edições feitas diretamente no editor do n8n.

## Limitação conhecida do protótipo — Wait node

O node **Aguardar Resposta do Cliente** usa `resume: webhook`: ao pausar, o n8n gera uma URL
própria (`$execution.resumeUrl`) que precisa ser chamada para o workflow continuar. Como a
resposta do cliente chega pelo webhook do WhatsApp (não por essa URL diretamente), falta ligar
os dois pontos em produção. O caminho recomendado:

- Ao enviar o lembrete, salvar o `resumeUrl` (ou o `execution.id`) junto com o telefone do
  cliente (ex.: numa aba de "aguardando resposta" na planilha ou numa Data Table do n8n).
- Ter um workflow (pode reusar o trigger do WhatsApp do workflow de agendamento) que, ao
  receber uma mensagem, verifica se aquele telefone está "aguardando resposta de lembrete" e,
  se estiver, repassa o texto para o `resumeUrl` salvo.

Isso não foi implementado neste protótipo para manter o foco na estrutura e lógica principal —
fica como próximo passo antes de ir para produção.

## Credenciais e recursos conectados

Este workflow já está conectado a credenciais e recursos reais na instância n8n (WhatsApp
Business, Google Calendar, Google Sheets, Anthropic Claude). Os objetos de `credentials` (IDs de
credencial) são removidos do JSON exportado para este repo — nunca são commitados, conforme a
convenção do projeto. Os resource locators (planilha, aba, calendário, `phoneNumberId`), por não
serem credenciais, refletem a configuração real em uso:

- Planilha: **Clientes - Automação PMEs**, aba **Sheet1** — colunas `nome`, `telefone`,
  `servico`, `data`, `event_id`.
- Calendário: o calendário do Google associado à conta usada pelo negócio.

Ao clonar/importar este JSON numa outra instância, é preciso reconectar as credenciais e, se for
usar uma planilha/calendário diferentes, ajustar os resource locators dos nodes.
