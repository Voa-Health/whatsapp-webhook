<div align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/Voa-Health/tiss/main/.github/images/logo-dark.svg">
    <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/Voa-Health/tiss/main/.github/images/logo-light.svg">
    <img alt="Voa" src="https://raw.githubusercontent.com/Voa-Health/tiss/main/.github/images/logo-light.svg" width="160">
  </picture>
</div>

<div align="center">
  <h3>Webhook mínimo da WhatsApp Cloud API, pronto para o Render.</h3>
</div>

<br>

Um único arquivo Node + Express que responde à verificação da Meta, confere a
assinatura dos eventos e, conforme a configuração, repassa o payload para outro
serviço ou responde com eco as mensagens de texto recebidas.

## Início rápido

```sh
npm install
VERIFY_TOKEN=um-valor-qualquer npm start
```

Requer Node 18+. Em desenvolvimento local, as variáveis também podem vir de um
arquivo `.env`, que o `.gitignore` já ignora.

```sh
curl "http://localhost:3000/?hub.mode=subscribe&hub.verify_token=um-valor-qualquer&hub.challenge=123"
```

A resposta é `123`, e o log mostra `WEBHOOK VERIFIED`.

## Como funciona

| Rota | Uso |
| --- | --- |
| `GET /` e `GET /webhooks/whatsapp` | Verificação da Meta com `hub.mode`, `hub.verify_token` e `hub.challenge`. |
| `POST /` e `POST /webhooks/whatsapp` | Recebe os eventos, registra o payload no log e responde `200`. |
| `GET /healthz` | Retorna `{"ok": true}`. |

A cada `POST`, o serviço:

1. **Confere** o `X-Hub-Signature-256` com `WHATSAPP_APP_SECRET`, quando
   definido; assinatura inválida retorna `403`.
2. **Registra** o payload completo no log do processo.
3. **Repassa** os bytes originais e o cabeçalho de assinatura para
   `HACS_WEBHOOK_FORWARD_URL`, quando definida, para que o destino possa
   verificar a assinatura de novo.
4. **Responde com eco** (`Echo: <texto>`) às mensagens de texto, se o eco
   estiver ligado.

O eco fica desligado por padrão quando há URL de repasse, para evitar respostas
duplicadas, e ligado quando não há. `WHATSAPP_ECHO_REPLY` força qualquer um dos
dois comportamentos.

> [!WARNING]
> O payload inteiro vai para o log, inclusive o conteúdo das mensagens. Trate
> os logs do serviço com o mesmo cuidado que os dados dos usuários.

## Configuração

| Variável | Uso |
| --- | --- |
| `VERIFY_TOKEN` | Obrigatória. Valor escolhido por você e repetido no painel da Meta; `WHATSAPP_VERIFY_TOKEN` é aceita como alternativa. |
| `WHATSAPP_APP_SECRET` | Recomendada. Segredo do app na Meta; liga a validação de assinatura. |
| `HACS_WEBHOOK_FORWARD_URL` | URL que recebe o repasse dos eventos. Sem ela, nada é repassado. |
| `WHATSAPP_ECHO_REPLY` | `0` desliga o eco; qualquer outro valor liga. |
| `WHATSAPP_ACCESS_TOKEN` | Token da Cloud API, necessário para o eco. |
| `WHATSAPP_PHONE_NUMBER_ID` | ID do número na Cloud API, necessário para o eco. |
| `WHATSAPP_API_VERSION` | Versão da Graph API; padrão `v22.0`. |
| `PORT` | Porta HTTP; padrão `3000`. O Render define automaticamente. |

Não há URL de repasse padrão, para que nenhum payload saia para um destino não
configurado.

## Deploy no Render

O [`render.yaml`](render.yaml) descreve o serviço para o fluxo de Blueprint,
com eco desligado e as demais variáveis preenchidas no painel. Para criar pela
interface:

1. Crie um **Web Service** a partir deste repositório.
2. Use `npm install` como build e `npm start` como start.
3. Defina as variáveis de [Configuração](#configuração).

No painel do app na Meta, em WhatsApp → Configuração do webhook:

1. Informe `https://<seu-servico>.onrender.com/` (ou `/webhooks/whatsapp`) como
   URL de callback.
2. Use o mesmo valor de `VERIFY_TOKEN` como token de verificação.
3. Assine o campo `messages`.

Se a verificação passar, o log do Render mostra `WEBHOOK VERIFIED`.

Ao repassar para outro serviço, configure nele o mesmo `WHATSAPP_APP_SECRET`
para validar a assinatura de ponta a ponta.
