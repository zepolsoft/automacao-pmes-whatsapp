# Notificação de Erros

Workflow n8n: [`notificacao-erros.json`](./notificacao-erros.json) · [abrir na instância](https://n8n-n8n.wg1izd.easypanel.host/workflow/ZxJfBbFmiD5Hqp5o)

Error Workflow centralizado: avisa no WhatsApp sempre que um node de qualquer um dos outros
workflows falha sem ter `onError: continueRegularOutput` configurado (ou seja, uma falha que
pararia a execução por completo).

## Fluxo

1. **Receber Erro de Workflow** — Error Trigger. Não é chamado diretamente; o n8n dispara este
   workflow automaticamente quando um workflow que aponta para ele em `settings.errorWorkflow`
   tem uma execução interrompida por erro.
2. **Formatar Resumo do Erro** (Code) — lê `workflow.name`, `execution.lastNodeExecuted` e
   `execution.error.message` do payload do Error Trigger e monta uma mensagem curta:
   ```
   ⚠️ Erro no workflow "<nome>"
   Node: <node que falhou>
   Erro: <mensagem do erro>
   <link da execução, se disponível>
   ```
3. **Notificar Erro no WhatsApp** — envia a mensagem formatada para o número
   `5511975049937` (WhatsApp do responsável pelo negócio).

## Como está conectado aos outros workflows

`settings.errorWorkflow` está configurado, na instância n8n, para apontar para este workflow
(`ZxJfBbFmiD5Hqp5o`) em:

- **Agendamento via WhatsApp** (`BIOdwZebPkUPyzRu`)
- **Lembrete, Cancelamento e Remarcação** (`wkIfUOGhEom4rFow`)

Essa configuração vive nas `settings` de cada workflow na instância n8n (não em algo visível
diretamente na lista de nodes) — os arquivos `workflows/agendamento/agendamento.json` e
`workflows/lembrete-cancelamento/lembrete-cancelamento.json` neste repo não incluem `settings`
no JSON exportado, então essa ligação não aparece nesses arquivos; confira direto na instância
se precisar confirmar.

## Por que só cobre falhas "não tratadas"

A maioria dos nodes de envio/escrita dos outros dois workflows (WhatsApp, Calendar, Sheets) já
tem `onError: continueRegularOutput` — uma falha pontual (ex.: número inválido, API fora do ar)
não para a execução, só segue adiante sem aquele efeito colateral. Esse Error Workflow só
dispara para o outro caso: um erro que *não* está coberto por `onError` e que de fato interrompe
a execução inteira (ex.: erro de configuração, campo obrigatório faltando, credencial inválida).
Ou seja, ele funciona como uma rede de segurança para falhas inesperadas, não como o mecanismo
principal de tolerância a falhas pontuais — que já é tratado node a node.

## Credenciais e recursos conectados

Este workflow já está conectado a credenciais e recursos reais na instância n8n. O objeto de
`credentials` (ID da credencial) é removido do JSON exportado para este repo — nunca é
commitado, conforme a convenção do projeto. `phoneNumberId` e `recipientPhoneNumber`, por não
serem credenciais, refletem a configuração real em uso (mesma convenção já adotada em
`workflows/lembrete-cancelamento/README.md` para o calendário/planilha reais daquele workflow).

Ao clonar/importar este JSON numa outra instância, é preciso reconectar a credencial
`WhatsApp Business (Meta Cloud API)` e, se for notificar outro número, trocar
`recipientPhoneNumber` no node **Notificar Erro no WhatsApp**.
