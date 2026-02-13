# WhatsApp Channel Setup

This guide covers configuring the WhatsApp channel for IronClaw using the WhatsApp Cloud API.

## Overview

The WhatsApp channel lets you interact with IronClaw via WhatsApp messages. It supports:

- **Webhook mode** (only mode): WhatsApp Cloud API is webhook-only
- **Text messages**: Receive and respond to text messages
- **Business account**: Uses WhatsApp Business Platform

## Prerequisites

- IronClaw installed and configured (`ironclaw onboard`)
- A [Meta Developer](https://developers.facebook.com/) account
- A WhatsApp Business account linked to Meta

## Quick Start

### 1. Create a Meta App

1. Go to [Meta Developer Portal](https://developers.facebook.com/apps/)
2. Click **Create App** > **Business** > fill in details
3. On the app dashboard, click **Add Product** > **WhatsApp** > **Set Up**
4. Under **API Setup**, note your:
   - **Phone number ID** (the business phone that sends/receives)
   - **Temporary access token** (for testing; use a System User token for production)

### 2. Configure via Setup Wizard

```bash
ironclaw onboard
```

When prompted, enable the WhatsApp channel and paste your access token. The wizard will:

- Validate the token against the Graph API
- Generate a webhook verify token (or use your custom one)

### 3. Configure Webhook in Meta Developer Portal

1. In your Meta app, go to **WhatsApp** > **Configuration**
2. Set **Callback URL** to your publicly accessible endpoint:
   ```
   https://your-domain.com/webhook/whatsapp
   ```
3. Set **Verify Token** to the token from step 2
4. Subscribe to the **messages** webhook field

For local development, use a tunnel:

```bash
# ngrok
ngrok http 8080

# Cloudflare
cloudflared tunnel --url http://localhost:8080
```

## Manual Installation

If the channel isn't installed via the wizard:

```bash
# Build the WhatsApp channel (requires wasm32-wasip2 target)
rustup target add wasm32-wasip2
cargo install wasm-tools
./channels-src/whatsapp/build.sh

# Install
mkdir -p ~/.ironclaw/channels
cp channels-src/whatsapp/whatsapp.wasm channels-src/whatsapp/whatsapp.capabilities.json ~/.ironclaw/channels/
```

## Secrets

The channel expects these secrets:

| Secret | Required | Description |
|--------|----------|-------------|
| `whatsapp_access_token` | Yes | WhatsApp Cloud API access token from Meta Developer Portal |
| `whatsapp_verify_token` | Optional | Webhook verification token (auto-generated if not set) |

Configure via:

- **Setup wizard**: Saves to encrypted secrets store
- **Environment**: `WHATSAPP_ACCESS_TOKEN=your_token`

## Configuration

Edit `~/.ironclaw/channels/whatsapp.capabilities.json`:

| Option | Default | Description |
|--------|---------|-------------|
| `api_version` | `v18.0` | Graph API version |
| `reply_to_message` | `true` | Whether to include reply context in responses |

## How It Works

1. User sends a WhatsApp message to your business number
2. Meta sends a webhook POST to `/webhook/whatsapp`
3. IronClaw host validates the webhook secret
4. WASM channel parses the Cloud API payload and emits the message
5. Agent processes and responds
6. WASM channel sends the response via Graph API (`/{phone_number_id}/messages`)
7. The host injects the access token into the Authorization header

## Webhook Verification

When you first configure the webhook URL in Meta's developer portal, Meta sends a GET request with:

- `hub.mode=subscribe`
- `hub.challenge=<random string>`
- `hub.verify_token=<your configured token>`

The channel responds with the challenge value to complete verification.

## Security

- **Credential injection**: The WASM module never sees the raw access token. It uses `{WHATSAPP_ACCESS_TOKEN}` placeholders that the host replaces before sending HTTP requests.
- **Webhook validation**: The verify token is checked by the host before forwarding requests to the WASM module.
- **HTTP allowlist**: The channel can only make requests to `graph.facebook.com`.
- **Rate limiting**: 80 requests/minute, 1000 requests/hour.

## Supported Message Types

Currently supported:
- **Text messages**: Full send/receive support

Not yet supported (contributions welcome):
- Image, audio, video, document messages
- Location messages
- Contact card messages
- Interactive messages (buttons, lists)
- Template messages

## Troubleshooting

### Messages not delivered

- Verify your access token is valid: `curl "https://graph.facebook.com/v18.0/me?access_token=YOUR_TOKEN"`
- Check that the webhook URL is publicly accessible (Meta requires HTTPS)
- Ensure you subscribed to the **messages** webhook field

### Webhook verification fails

- Confirm the verify token matches what you configured in Meta's portal
- The callback URL must be HTTPS (Meta rejects HTTP)

### "Permission denied" errors

- Ensure your access token has the `whatsapp_business_messaging` permission
- For production, use a System User token instead of the temporary one

### Status updates causing loops

- The channel automatically filters status updates (delivered, read) and only processes actual messages
