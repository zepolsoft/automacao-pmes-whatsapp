# Lembrete, Cancelamento e Remarcação

Workflow n8n: [`lembrete-cancelamento.json`](./lembrete-cancelamento.json) · [abrir na instância](https://n8n-n8n.wg1izd.easypanel.host/workflow/wkIfUOGhEom4rFow)

Todo dia às 8h, avisa cada cliente com horário marcado para o dia atual e trata a resposta
dele: confirmar, cancelar ou remarcar.

## Objetivo e o que este workflow NÃO faz

**Faz:** roda o ciclo diário do lembrete — dispara às 8h para todo agendamento do dia,
aguarda a resposta do cliente a ESSE lembrete específico, classifica em confirmar/cancelar/
remarcar, efetiva a ação (Calendar + planilha) e responde. Também roda, num trigger
independente às 22h, a rotina que marca atendimentos passados como "concluído".

**NÃO faz:**
- Não recebe nem processa mensagens espontâneas do cliente fora do contexto de um lembrete já
  enviado por este workflow — isso é o outro workflow,
  **[Agendamento via WhatsApp](../agendamento/README.md)**.
- Não cria agendamentos novos do zero (só remarca um agendamento existente para outro
  horário).
- Não interpreta livremente o que o cliente quer dizer fora das três opções (confirmar,
  cancelar, remarcar) — uma resposta fora desse escopo cai em `decisao = "indefinido"` e só
  pede esclarecimento, nunca tenta adivinhar uma intenção mais ampla como o outro workflow faz.

## Principais nodes e papel de cada um

| Node | Papel |
|---|---|
| Disparar Lembrete Diário às 8h | Schedule Trigger do ciclo principal |
| Buscar Agendamentos de Hoje (Planilha) / Filtrar Data de Hoje | Lê a planilha inteira e filtra só os agendamentos de hoje com status `agendado`/`remarcado` |
| Processar Cada Agendamento | Loop (Split in Batches) — processa um cliente por vez |
| Enviar Lembrete no WhatsApp | Envia o lembrete e pergunta confirmar/cancelar/remarcar |
| Aguardar Resposta do Cliente | Wait node (webhook), com timeout de 10 min |
| Cliente Respondeu ou Deu Timeout? | Diferencia resposta real de timeout |
| Reverificar Agendamento Antes do Timeout / Agendamento Ainda É o Mesmo? | Antes de avisar timeout, confere se o agendamento mudou por outro caminho enquanto esperava |
| Normalizar Resposta do Lembrete | Extrai texto da resposta e `message_id` do payload |
| Checar Mensagem Duplicada / Mensagem Já Processada? / Registrar Mensagem Processada | Deduplicação por `message_id` (ver "Regras de negócio" abaixo) |
| Checar Lock do Telefone / Telefone Ocupado? / Registrar Lock do Telefone | Lock de 30s por telefone, evita execuções paralelas do mesmo cliente |
| Classificar Resposta do Lembrete | AI Agent (Claude) — classifica confirmar/cancelar/remarcar/indefinido e escreve `confirmacao_texto` |
| Confirmar, Cancelar ou Remarcar? | Switch que roteia a decisão |
| Validar Horário de Funcionamento / Horário Dentro do Expediente? | Code determinístico: rejeita novo horário fora de 9h-18h (seg-sáb) ou já passado, na remarcação |
| Verificar Novo Horário Disponível / Novo Horário Disponível? | Checagem real no Google Calendar do novo horário pedido |
| Atualizar Evento no Calendar / Cancelar Evento no Calendar | Efetiva a ação no Calendar |
| Atualizar Data na Planilha / Atualizar Status na Planilha (Cancelar) | Grava o resultado na planilha "Clientes - Automação PMEs" |
| Enviar Confirmação Final / de Cancelamento / de Remarcação / Pedir Outro Horário / Avisar Horário Fora do Expediente / Pedir Esclarecimento / Avisar Timeout no WhatsApp | Respostas ao cliente (ver "Sincronização de tom" abaixo) |
| Marcar Atendimentos Concluídos às 22h (+ Buscar Todos / Filtrar Passados Pendentes / Marcar Como Concluído) | Segundo trigger, independente do ciclo do lembrete |

## Colunas da planilha "Clientes - Automação PMEs"

| coluna | lê? | escreve? | quando |
|---|---|---|---|
| `nome`, `telefone`, `servico` | lê (usados direto do item da planilha, sem lookup extra) | escreve `servico`/`preco`/`criado_em` de volta sem alterar | em toda remarcação/cancelamento/conclusão, reescritos com o valor que já estava (ver bug do range do Update Row, abaixo) |
| `data` | lê (para o lembrete de hoje e para checar se o agendamento mudou antes do timeout) | escreve | remarcação sobrescreve com a nova data |
| `event_id` | lê (chave de correspondência em toda escrita) | — | nunca escrito por este workflow (só lido) |
| `status` | lê (filtro do lembrete diário e da rotina das 22h) | escreve | `remarcado` na remarcação, `cancelado` no cancelamento, `concluido` na rotina das 22h |
| `preco`, `criado_em` | lê (para reescrever sem alterar) | escreve (reescrita, não recálculo) | em toda remarcação/cancelamento/conclusão |
| `atualizado_em` | — | escreve | em toda remarcação/cancelamento/conclusão |

## Regras de negócio implementadas

- **Horário de funcionamento** (segunda a sábado, 9h-18h) na remarcação — ver "Horário de
  funcionamento na remarcação" abaixo.
- **Validação de data no passado** — mesmo Code node acima, condição `jaPassou`.
- **Deduplicação de mensagens** por `message_id` (wamid) — ver "Robustez: deduplicação, lock
  por telefone e data no passado" abaixo.
- **Lock por telefone** (30s) — evita duas execuções paralelas do mesmo cliente, mesma seção.
- **Timeout de espera** (10 min) com reverificação antes de avisar — ver "Timeout de espera no
  lembrete" abaixo.

## Sincronização de tom com o workflow de agendamento

Este workflow e o **[Agendamento via WhatsApp](../agendamento/README.md)** atendem o mesmo
número de WhatsApp da barbearia, então, do ponto de vista do cliente, é uma conversa só — ele
não deve perceber que está "falando com duas IAs diferentes" dependendo de ter sido ele quem
escreveu primeiro ou o negócio quem mandou um lembrete. Por isso os dois workflows
compartilham deliberadamente:

- O mesmo campo de saída da IA, `confirmacao_texto`, com as mesmas instruções de **variedade e
  tom** no system message (não repetir a mesma estrutura/abertura de frase, usar o nome do
  cliente sem exagero, ser acolhedor em remarcação de última hora) — ver o system message de
  "Classificar Resposta do Lembrete" aqui e de "Interpretar Intenção do Cliente" no outro
  workflow.
- O mesmo texto-base (com variações sorteadas) para as situações que não passam pela IA:
  horário ocupado, fora do expediente, "não entendi".
- As mesmas regras de horário de funcionamento e validação de data no passado.

Apesar do tom compartilhado, os dois workflows têm **escopo e lógica totalmente
independentes** — não compartilham nenhum node, trigger, nem estado de execução:

- Este workflow trata especificamente o ciclo do lembrete diário: dispara às 8h, aguarda
  resposta a UM lembrete específico já enviado, e tem sua própria lógica de timeout — nunca
  reage a uma mensagem espontânea do cliente que não seja resposta a esse lembrete.
- O outro trata mensagens recebidas a qualquer momento, iniciadas pelo cliente.

Uma mudança de tom/estilo num dos dois (ex.: ajustar a seção VARIEDADE E TOM do prompt) deve,
na prática, ser replicada no outro para manter a experiência consistente — não existe hoje um
prompt compartilhado entre os dois workflows (cada AI Agent tem seu próprio system message),
então essa sincronização é manual e intencional, não automática.

## Fluxo

1. **Disparar Lembrete Diário às 8h** — Schedule Trigger.
2. **Buscar Agendamentos de Hoje (Planilha)** — Google Sheets, lê todas as linhas da planilha
   **"Clientes - Automação PMEs"** (aba "Sheet1") → **Filtrar Data de Hoje** (Filter) mantém só
   as linhas cuja coluna `data` cai no dia de hoje **e** cujo `status` é `"agendado"` ou
   `"remarcado"` (ver "Fonte de dados do lembrete" e "Colunas de status e preço" abaixo) —
   linhas `cancelado`, `concluido` ou `no_show` nunca geram lembrete.
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
        - **Cancelar:** deleta o evento no Calendar (pelo `event_id`) → **Atualizar Status na
          Planilha (Cancelar)** grava `status: "cancelado"` e `atualizado_em` na linha (não
          apaga a linha) → confirma o cancelamento.
        - **Remarcar:** **Validar Horário de Funcionamento** (Code) → **Horário Dentro do
          Expediente?** (IF) — mesma validação determinística (09h-18h, seg-sáb) do workflow
          "Agendamento via WhatsApp", ver "Horário de funcionamento na remarcação" abaixo.
          - **Fora do expediente:** avisa o cliente e pede outro horário, sem consultar o
            Calendar nem tocar na planilha.
          - **Dentro do expediente:** verifica se o novo horário está livre → se não, pede
            outro horário; se sim, **Propor Remarcação no WhatsApp** pergunta se pode
            confirmar a mudança e aguarda uma SEGUNDA resposta do cliente (ver "Confirmação
            explícita antes de remarcar" abaixo) — só então **atualiza** o evento existente
            (não cria um novo) e a coluna `data`, `status: "remarcado"` e `atualizado_em` na
            planilha (casando pela coluna `event_id`) → confirma a remarcação de verdade.
        - **Não entendi (fallback):** pede para o cliente esclarecer a resposta.
      - **Não (timeout):** **Avisar Timeout do Lembrete no WhatsApp** — avisa o cliente que não
        houve resposta e que o agendamento foi mantido como está. Não passa pela
        classificação/confirmação/cancelamento/remarcação.
   4. Volta para o próximo agendamento do lote.

Além do lembrete diário, este workflow tem um **segundo trigger independente**, sem nenhuma
relação com o fluxo acima (não compartilha nodes, só a mesma credencial/planilha):

5. **Marcar Atendimentos Concluídos às 22h** — Schedule Trigger, `America/Sao_Paulo` → **Buscar
   Todos os Agendamentos** (Google Sheets, lê a planilha inteira, sem filtro) → **Filtrar
   Agendamentos Passados Pendentes** (Filter: `data` já passou **e** `status` é `"agendado"` ou
   `"remarcado"`) → **Marcar Como Concluído** (Google Sheets `update`, casando por `event_id`,
   grava `status: "concluido"` e `atualizado_em`) — ver "Colunas de status e preço" abaixo.

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

## Confirmação explícita antes de remarcar (2026-09-24)

Antes desta mudança, assim que "Novo Horário Disponível?" confirmava que o novo horário estava
livre, o sistema já atualizava o evento no Calendar e a linha na planilha na mesma execução —
sem perguntar ao cliente antes. Agora existe uma segunda rodada de pergunta/resposta, separando
**proposta** (não escreve nada) de **execução** (só depois do "sim"):

1. **Propor Remarcação no WhatsApp** (renomeado do antigo node de confirmação final — o texto
   gerado por "Classificar Resposta do Lembrete" para `decisao = "remarcar"` agora é sempre uma
   pergunta, nunca uma confirmação definitiva) envia "posso confirmar a remarcação pra [novo
   horário]?" e não toca em Calendar/planilha.
2. **Aguardar Confirmação da Remarcação** (Wait, mesmo padrão do "Aguardar Resposta do
   Cliente": `resume: webhook`, timeout de 10 min) → **Cliente Confirmou a Remarcação ou Deu
   Timeout?** (IF, mesmo padrão de detecção de timeout do primeiro Wait).
   - **Timeout:** **Avisar Timeout da Confirmação no WhatsApp** — avisa que não houve resposta
     e que o horário **original** (não o proposto) foi mantido → volta pro loop.
   - **Respondeu:** **Normalizar Confirmação da Remarcação** (Set, extrai o texto da resposta)
     → **Classificar Confirmação da Remarcação** — um AI Agent **dedicado**, mais simples que
     o principal (tem seu próprio modelo Claude e parser só para essa pergunta de sim/não),
     recebe a pergunta que foi feita + a resposta do cliente e retorna `confirmado` (booleano)
     e um `confirmacao_texto` já adequado a cada caso.
     - `confirmado = false` (negou ou resposta ambígua): **Avisar Remarcação Não Confirmada no
       WhatsApp** — usa o `confirmacao_texto` do classificador, tranquilizando que o horário
       original continua valendo → volta pro loop, **sem tocar em Calendar/planilha**.
     - `confirmado = true`: só agora **Atualizar Evento no Calendar** → **Atualizar Data na
       Planilha** → **Enviar Confirmação Final da Remarcação no WhatsApp** (node novo, usa o
       `confirmacao_texto` do classificador dedicado, não mais o de "Classificar Resposta do
       Lembrete") → volta pro loop.

**Escopo deliberadamente reduzido em relação ao primeiro Wait:** esse segundo ciclo não tem
deduplicação por `message_id` nem lock por telefone, e não reverifica se o agendamento mudou
antes de avisar timeout (como o primeiro Wait faz via "Reverificar Agendamento Antes do
Timeout"). Decisão consciente para não dobrar a complexidade do workflow: atualizar o Calendar/
planilha é uma operação idempotente (repetir a mesma atualização não cria duplicata, diferente
de criar um recurso novo), então o risco de uma mensagem duplicada processada duas vezes aqui é
baixo — ao contrário do risco que a deduplicação do primeiro Wait existe para evitar.

## Colunas de status e preço

A planilha **"Clientes - Automação PMEs"** tem 4 colunas além das 5 originais (`nome`,
`telefone`, `servico`, `data`, `event_id`): `status`, `preco`, `criado_em`, `atualizado_em` —
detalhes de quando cada uma é escrita e por quê em `workflows/agendamento/README.md`, seção
"Colunas de status e preço na planilha" (é lá que `status: "agendado"` e `preco` são gravados
pela primeira vez, na criação do agendamento). Este workflow só **lê** `status`/`data` (no
filtro do lembrete diário) e **escreve** `status`/`atualizado_em` em três pontos:

- Remarcar (via resposta ao lembrete): `status: "remarcado"`.
- Cancelar (via resposta ao lembrete): `status: "cancelado"` — a linha não é mais apagada.
- Rotina das 22h: `status: "concluido"` para qualquer linha com `data` já passada e `status`
  ainda `"agendado"`/`"remarcado"` — assume que o atendimento aconteceu, a menos que já tenha
  sido marcado manualmente como `"cancelado"` ou `"no_show"` antes disso. `"no_show"` só é
  preenchido manualmente, direto na planilha; nenhum node grava esse valor automaticamente.

> **Bug corrigido (2026-09-24):** os três nodes acima também reescreviam `preco` e `criado_em`
> como vazio, mesmo sem estarem em `columns.value` — o node Update Row do Google Sheets escreve
> um range contíguo de células (da coluna mapeada mais à esquerda até a mais à direita: aqui,
> `status` a `atualizado_em`), e qualquer coluna nesse meio sem valor explícito vira vazio.
> Corrigido reenviando os valores atuais de `preco`/`criado_em` (já disponíveis no item — vieram
> da própria planilha via "Processar Cada Agendamento" ou, na rotina das 22h, do próprio `$json`
> lido) em vez de omiti-los. Mesmo bug e mesma correção no workflow "Agendamento via WhatsApp" —
> ver o README daquele workflow para o teste que validou a correção.

## Robustez: deduplicação, lock por telefone e data no passado (2026-09-24)

Mesma rodada de robustez aplicada em `workflows/agendamento/README.md` (seção "Robustez:
deduplicação, lock por telefone e data no passado"), replicada aqui com os mesmos nomes de node e
as mesmas duas Data Tables (`mensagens_processadas`, `locks_telefone` — compartilhadas entre os
dois workflows, mesmo `dataTableId`):

1. **Deduplicação de mensagens** — **Normalizar Resposta do Lembrete** passou a extrair
   `message_id` (`wamid`) do payload. **Checar Mensagem Duplicada** (`get`) →
   **Mensagem Já Processada?** (IF) → se já processada, **Ignorar Mensagem Duplicada** (NoOp,
   volta direto para **Processar Cada Agendamento** para não travar o loop); se não,
   **Registrar Mensagem Processada** (`insert`) → segue para o lock de telefone.
2. **Lock por telefone** — **Checar Lock do Telefone** (`get`) → **Telefone Ocupado?** (IF:
   `bloqueado_em` com menos de 30s?) → se ocupado, **Ignorar Mensagem (Telefone Ocupado)** (NoOp,
   volta para **Processar Cada Agendamento**); se não, **Registrar Lock do Telefone** (`upsert`)
   → segue para **Classificar Resposta do Lembrete**. Como este workflow tem um loop (Split in
   Batches), os dois nodes `NoOp` de "ignorar" precisam voltar para **Processar Cada
   Agendamento** — sem essa conexão de volta, o loop pararia no primeiro item duplicado/travado
   e os agendamentos seguintes do lote não seriam processados.
3. **Validação de data no passado** — **Validar Horário de Funcionamento** ganhou a mesma
   condição `jaPassou` do outro workflow, e o texto de **Avisar Horário Fora do Expediente no
   WhatsApp** foi ajustado para "Esse horário já passou ou está fora do nosso expediente [...]".

> **Bug crítico corrigido (2026-09-24, duas tentativas):** os nodes **Checar Mensagem
> Duplicada** e **Checar Lock do Telefone** precisam de `alwaysOutputData` (opção de execução
> do node, não um parâmetro) para que uma busca sem resultado — o caso normal, de mensagem nova
> e telefone sem lock ativo — ainda produza 1 item sintético vazio em vez de zero itens; sem
> isso, o IF logo depois nunca roda e o loop trava ali para todo cliente, sem processar
> confirmar/cancelar/remarcar. A primeira tentativa de correção (mesmo dia, mais cedo) usou uma
> operação de workflow que grava dentro de `parameters` — onde o n8n não lê essa opção — então
> ficou com uma cópia inofensiva e inútil do campo, sem nenhum efeito real; o bug continuou
> presente mesmo depois de "corrigido" e publicado. Descoberto ao investigar o mesmo bug
> reportado no workflow "Agendamento via WhatsApp" (ver o README daquele workflow para a
> investigação completa) e confirmado aqui pela mesma causa. Corrigido de vez usando a operação
> certa para configurações de node (grava na raiz do node, irmã de `parameters`), e confirmado
> com um teste isolado direto na Data Table real.

## Histórico de correções

- **2026-09-24 — Ciclo de vida do agendamento (status na planilha):** a planilha ganhou as
  colunas `status`, `preco`, `criado_em`, `atualizado_em` (ver "Colunas de status e preço"
  acima). Três mudanças neste workflow:
  1. **"Filtrar Data de Hoje"** ganhou uma segunda condição: só considera linhas com `status`
     `"agendado"` ou `"remarcado"`, para o lembrete diário nunca disparar para um agendamento já
     cancelado ou concluído.
  2. **Cancelamento via resposta ao lembrete** ganhou um node novo, **"Atualizar Status na
     Planilha (Cancelar)"**, entre "Cancelar Evento no Calendar" e a confirmação — esse ramo
     não tocava a planilha antes (só deletava o evento do Calendar), o que deixava a linha
     "presa" em `status: "agendado"` para sempre.
  3. Nova rotina independente, **"Marcar Atendimentos Concluídos às 22h"**, que marca como
     `"concluido"` qualquer linha com `data` já passada e `status` ainda pendente.
  - As 6 linhas que já existiam na planilha antes dessa mudança foram preenchidas uma única vez
    com `status: "agendado"` (workflow utilitário temporário, executado e arquivado depois),
    para não ficarem invisíveis para o filtro novo do lembrete diário.
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

## Wait node ↔ WhatsApp: ponte com o workflow de agendamento (2026-09-25)

> Isto ERA uma limitação conhecida do protótipo (texto original abaixo, mantido como
> histórico) — implementada em 2026-09-25. Ver seção "Ponte com o Wait node" mais abaixo para
> a implementação real.
>
> ~~O node **Aguardar Resposta do Cliente** usa `resume: webhook`: ao pausar, o n8n gera uma
> URL própria (`$execution.resumeUrl`) que precisa ser chamada para o workflow continuar. Como
> a resposta do cliente chega pelo webhook do WhatsApp (não por essa URL diretamente), falta
> ligar os dois pontos em produção... Isso não foi implementado neste protótipo para manter o
> foco na estrutura e lógica principal — fica como próximo passo antes de ir para produção.~~

## Padronização de formato de data/hora (2026-09-24)

`criado_em`/`atualizado_em` na planilha "Clientes - Automação PMEs" usavam `$now.toISO()`, que
inclui milissegundos (ex.: `2026-09-24T13:45:18.557-03:00`) — diferente do formato da coluna
`data`, que já vem sem milissegundos da IA (ex.: `2026-09-24T16:00:00-03:00`). Padronizado com
`$now.toISO({ suppressMilliseconds: true })` nos 3 nodes que escrevem `atualizado_em`
(**Atualizar Status na Planilha (Cancelar)**, **Atualizar Data na Planilha**, **Marcar Como
Concluído**). Vale só pra escritas novas — linhas antigas na planilha não foram alteradas.
Mesmo ajuste aplicado no workflow "Agendamento via WhatsApp" (`criado_em` em **Salvar Cliente na
Planilha**, `atualizado_em` em **Atualizar Linha na Planilha** e **Atualizar Linha na Planilha
(Cancelar)**).

## Identidade "Zap" (2026-09-24)

Mesma mudança de identidade/tom do workflow "Agendamento via WhatsApp": o assistente se chama
**Zap**. Aqui o ajuste é mais leve — **Classificar Resposta do Lembrete** é um classificador
fechado de 4 categorias (confirmar/cancelar/remarcar/indefinido), não um chat aberto, então só
a linha de abertura do persona foi adicionada ("Você é o Zap, assistente virtual de agendamento
de uma barbearia..."), por consistência de tom com o outro workflow. Nenhuma regra de
classificação mudou. Uma pergunta como "qual seu nome?" nesse contexto (resposta a um lembrete)
continua caindo em `decisao = "indefinido"`, tratada pelo node estático já existente **Pedir
Esclarecimento no WhatsApp** — fora do escopo desta mudança.

## Indicador de "digitando..." (2026-09-24)

Mesma mudança do workflow "Agendamento via WhatsApp": novo node **Ativar Indicador de
Digitação** (HTTP Request, POST direto na Graph API — o node nativo do WhatsApp não expõe
`markAsRead`/`typing_indicator`) logo depois de **Registrar Lock do Telefone**, antes de
**Classificar Resposta do Lembrete** (IA). Marca a resposta do cliente ao lembrete como lida e
ativa "digitando..." enquanto a IA classifica confirmar/cancelar/remarcar. Mesma credencial
`whatsAppApi` dos nodes de envio, `onError: continueRegularOutput` + `neverError: true` pra não
travar o fluxo se a chamada falhar.

Não aplicado à etapa de confirmação da remarcação (**Aguardar Confirmação da Remarcação** →
**Classificar Confirmação da Remarcação**), que não tem seu próprio dedup/lock (decisão
deliberada, ver "Robustez" abaixo) — fora do escopo desta mudança.

## Bug: indicador de digitando com message_id vazio (2026-09-25)

Mesmo bug do workflow "Agendamento via WhatsApp": `jsonBody` de **Ativar Indicador de
Digitação** usava `$json.message_id`, que resolvia contra a saída do node anterior imediato
(**Registrar Lock do Telefone**), sem esse campo. Corrigido referenciando
`$('Normalizar Resposta do Lembrete').item.json.message_id` explicitamente.

## Bug: reação com emoji tratada como resposta vazia (2026-09-25)

Cliente reagindo com emoji (não enviando mensagem nova) chega com `messages[0].type ===
"reaction"`, sem `text.body`. Os IFs **Cliente Respondeu ou Deu Timeout?** e **Cliente
Confirmou a Remarcação ou Deu Timeout?** tratavam isso como resposta vazia → timeout,
encerrando a espera cedo demais e enviando a mensagem de timeout de forma equivocada. Este
workflow não recebe webhook direto (só resume de Wait node), então não existe um "Filtrar
Apenas Mensagens" equivalente pra reaproveitar. Corrigido com dois novos IFs logo após cada
Wait resumir — **É uma Reação? (Lembrete)** e **É uma Reação? (Confirmação da Remarcação)**:
se for reação, volta pro mesmo Wait (rearma a espera, ignora completamente, sem processar nem
responder nada); se não for, segue o fluxo normal de respondeu/timeout de sempre.

## Ponte com o Wait node (2026-09-25)

Implementação da ponte descrita acima ("Wait node ↔ WhatsApp"). Como este workflow não tem
trigger de WhatsApp próprio (só Schedule Triggers), a resposta real do cliente chega sempre
pelo webhook do workflow "Agendamento via WhatsApp" — sem essa ponte, ela era processada lá
como mensagem avulsa, e o Wait daqui estourava por timeout mesmo com o cliente tendo
respondido.

**Do lado deste workflow** (a parte que grava a "ponte" pra ser encontrada):

- Nova Data Table **`esperas_lembrete`** (compartilhada com "Agendamento via WhatsApp"):
  colunas `telefone`, `resume_url`, `atualizado_em`.
- Novo node **Registrar Espera de Lembrete** logo antes de **Aguardar Resposta do Cliente**
  (upsert por telefone: `telefone`, `resume_url` = `{{ $execution.resumeUrl }}`,
  `atualizado_em` = `{{ $now.toISO() }}`) — e o mesmo padrão duplicado como **Registrar Espera
  de Lembrete (Confirmação da Remarcação)**, logo antes de **Aguardar Confirmação da
  Remarcação**. O gate de reação (**É uma Reação? (Lembrete)** / **(Confirmação da
  Remarcação)**) também passa por esse node de registro no caminho de volta pro Wait, pra
  manter `atualizado_em` sempre fresco a cada reentrada.
- **`$execution.resumeUrl` é estável durante toda a vida da execução** (confirmado com teste
  isolado) — não muda entre pausas repetidas do mesmo Wait, então um único registro por
  telefone por execução já cobre as reentradas causadas por reação.
- **Os dois Wait nodes precisaram de `httpMethod: "POST"` explícito** — sem isso, o padrão é
  GET, e uma chamada de resume com corpo JSON (necessário pra levar
  `body.messages[0].text.body` no formato que **Cliente Respondeu ou Deu Timeout?** já espera)
  é rejeitada com 404. Confirmado com um teste isolado real (workflow descartável, telefone
  fake): POST com corpo JSON resume a execução certa e o corpo cai em `$json.body` exatamente
  no formato esperado pelas expressões já existentes — nenhuma delas precisou mudar.

**Do lado do outro workflow** ("Agendamento via WhatsApp"): ver seção "Ponte com o Wait node do
workflow de Lembrete" no README daquele workflow — é lá que a mensagem recebida é checada
contra `esperas_lembrete` e encaminhada pro `resume_url`, se houver uma espera ativa.

Sem exclusão explícita da linha em `esperas_lembrete` quando a espera é resolvida — mesma
decisão de janela fixa sem "unlock" já usada em `locks_telefone`.

**Auditoria de risco (2026-09-25)**: janela de 13min vs. timeout de 10min dos Wait nodes,
timing do refresh, múltiplos agendamentos no mesmo telefone (testado empiricamente — dois
agendamentos em fila pro mesmo telefone resumem cada um com a resposta certa, sem contaminação
cruzada, graças ao `splitInBatches(batchSize:1)` sequencial de **Processar Cada Agendamento** +
`$execution.resumeUrl` fixo por execução), crescimento da tabela, e fuso horário — nenhum risco
real encontrado. Detalhes completos na seção "Auditoria de risco da janela de tempo" do README
do workflow "Agendamento via WhatsApp".

## Error Workflow centralizado (2026-09-24)

`settings.errorWorkflow` deste workflow, configurado direto na instância n8n, aponta para o
workflow **[Notificação de Erros](../notificacao-erros/README.md)**: qualquer erro que não
esteja coberto por `onError: continueRegularOutput` num node interrompe a execução normalmente,
mas também dispara aquele workflow, que avisa no WhatsApp com o nome do workflow, o node que
falhou e o resumo do erro.

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
