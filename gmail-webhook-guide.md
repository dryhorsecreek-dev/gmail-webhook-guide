# Gmail Email Forwarding/Notifications to AI Assistant Bot

A comprehensive installation guide for setting up Gmail email forwarding to a custom AI assistant bot (such as Clawdbot) using the Gmail Watch API, Google Pub/Sub, Tailscale Funnel, and the gog CLI.

## 1. Overview and Architecture

This guide describes how to build a real-time email notification pipeline that delivers Gmail messages to your AI assistant bot. The architecture uses a series of connected components, each handling a specific part of the message flow.

### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  EMAIL SENT                                                                  │
└────────────────────────────────┬────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  GMAIL WATCH API                                                             │
│  (Monitors inbox for new messages)                                           │
└────────────────────────────────┬────────────────────────────────────────────┘
                                   │ Notification
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  GOOGLE PUB/SUB TOPIC                                                        │
│  (Receives Gmail push notifications)                                         │
└────────────────────────────────┬────────────────────────────────────────────┘
                                   │ Push subscription
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  TAILSCALE FUNNEL                                                            │
│  (Public HTTPS endpoint via Tailscale network)                              │
└────────────────────────────────┬────────────────────────────────────────────┘
                                   │ Reverse proxy
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  GOG CLI SERVE                                                               │
│  (Local HTTP bridge, receives Pub/Sub webhooks)                             │
└────────────────────────────────┬────────────────────────────────────────────┘
                                   │ HTTP request
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  BOT WEBHOOK ENDPOINT                                                        │
│  (AI assistant bot processes the email)                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Component Overview

The pipeline consists of four distinct stages. First, Gmail's Watch API monitors your inbox and publishes notifications to a Google Pub/Sub topic whenever new emails arrive. Second, a Pub/Sub push subscription delivers these notifications to a publicly accessible HTTPS endpoint. Third, Tailscale Funnel provides that public endpoint by exposing your local machine to the internet through Tailscale's secure network. Fourth, the gog CLI serves as a local HTTP bridge, receiving the Pub/Sub webhooks and forwarding them to your bot's internal webhook endpoint.

This architecture was chosen for its reliability, security, and relative simplicity. It avoids the complexity of setting up a traditional server with port forwarding, instead leveraging Tailscale's secure tunneling capabilities.

## 2. Prerequisites

Before beginning the installation, ensure you have all required accounts, tools, and permissions in place. This section outlines everything needed to successfully complete the setup.

### Required Accounts

You will need a Google account with access to Gmail, which will be the source of email notifications. You also need a Google Cloud Platform account to create a project, enable APIs, and configure Pub/Sub resources. Finally, a Tailscale account is required for the Funnel feature that exposes your local server to the internet.

### Required Tools and Software

Install the Google Cloud SDK (gcloud CLI) from Google's official documentation for your operating system. This tool enables you to interact with Google Cloud services from the command line. You will also need the gog CLI tool, which provides convenient commands for managing Gmail Watch and serving webhooks. Installation instructions are available in the gog repository. The Tailscale client must be installed on the machine that will run the gog serve process, and you must authenticate it with your Tailscale account.

### System Requirements

The machine running gog serve must be able to run continuously (or be restarted reliably) since the Gmail Watch subscription and bot webhook receiver need consistent uptime. It should have a stable internet connection and the ability to keep the Tailscale client running in the background. A Linux server or a Raspberry Pi works well for this purpose.

### Google Cloud Permissions

The Google Cloud project requires an owner or editor role to enable APIs and create service accounts. If you are working within an organization, you may need to request these permissions from your administrator. The Pub/Sub topic and subscription can be created once the necessary APIs are enabled.

## 3. Google Cloud Setup

This section walks through creating the Google Cloud infrastructure needed to receive Gmail notifications and deliver them via Pub/Sub.

### Create a Google Cloud Project

Begin by creating a new project in the Google Cloud Console. Navigate to the project selector dropdown in the top navigation bar and select "New Project." Give your project a descriptive name such as "email-bot-integration" and choose an appropriate organization if applicable. Once the project is created, switch to it using the gcloud CLI:

```bash
gcloud config set project [YOUR_PROJECT_ID]
```

### Enable Required APIs

Enable the Gmail API and Pub/Sub API for your project. The Gmail API allows your application to watch for changes to the mailbox, while Pub/Sub provides the messaging infrastructure for delivering notifications:

```bash
gcloud services enable gmail.googleapis.com
gcloud services enable pubsub.googleapis.com
```

### Create a Pub/Sub Topic

Create a dedicated Pub/Sub topic to receive Gmail notifications. Choose a topic name that reflects its purpose, such as "gmail-notifications" or "email-bot-topic":

```bash
gcloud pubsub topics create gmail-notifications
```

Note the topic name in the format `projects/[YOUR_PROJECT_ID]/topics/gmail-notifications` as you will need it for subsequent configuration steps.

### Configure the Gmail API Service Account

Google uses a special service account called "gmail-api-push@system.gserviceaccount.com" to deliver Gmail push notifications. You must grant this service account the ability to publish messages to your Pub/Sub topic. Run the following command to add the publisher role:

```bash
gcloud pubsub topics add-iam-policy-binding gmail-notifications \
    --member="serviceAccount:gmail-api-push@system.gserviceaccount.com" \
    --role="roles/pubsub.publisher"
```

This permission allows Gmail's infrastructure to send notifications to your topic whenever new emails arrive in the watched mailbox.

## 4. gog CLI Setup

The gog CLI simplifies Gmail Watch management and provides a built-in HTTP server for receiving webhooks. This section covers installation and initial configuration.

### Install gog CLI

Install gog using your system's package manager or by building from source. For most Linux systems, you can download a pre-built binary from the releases page. Make sure the binary is executable and available in your PATH:

```bash
# Example installation (verify latest method in gog documentation)
curl -L https://example.com/gog-install.sh | sh
```

### Configure Authentication

gog needs authorization to access your Gmail account. Initialize the authentication using the keyring backend, which securely stores your credentials:

```bash
gog auth keyring file
```

Then authorize your Gmail account:

```bash
gog auth add your-email@gmail.com
```

You'll be prompted to create a passphrase for the keyring and complete an OAuth flow in your browser.

The file-based backend stores credentials in an encrypted file within your home directory, which is more reliable for server environments where a graphical keyring daemon may not be running.

## 5. Gmail Watch Configuration

With gog configured, you can now start watching your Gmail inbox. The Watch API monitors for new messages and publishes notifications to your Pub/Sub topic.

### Start Gmail Watch

Begin watching your Gmail account, specifying the Pub/Sub topic you created earlier. Replace `[YOUR_PROJECT_ID]` with your actual Google Cloud project identifier:

```bash
gog gmail watch start \
    --account your-email@gmail.com \
    --topic projects/[YOUR_PROJECT_ID]/topics/gmail-notifications
``` The watch will remain active for seven days before expiring, which is a built-in limitation of the Gmail Watch API.

### Verify Watch Status

Confirm that the watch is active by checking its status:

```bash
gog gmail watch status --account your-email@gmail.com
```

You should see output indicating that the watch is running and pointing to your configured topic. If the watch is not active, re-run the start command and verify your authentication credentials.

## 6. Tailscale Funnel Setup

Tailscale Funnel exposes a local service to the internet through Tailscale's network, providing HTTPS encryption without requiring traditional port forwarding or a dedicated public IP address.

### Install and Authenticate Tailscale

Install the Tailscale client for your operating system and authenticate with your Tailscale account. On Linux, this typically involves adding the Tailscale repository and installing the package:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

You may also need to install `zstd` if prompted:

```bash
sudo apt-get install zstd -y
```

Start the Tailscale daemon and authenticate:

```bash
sudo tailscale up
```

This outputs a URL — open it in your browser to sign in with Google, GitHub, or Microsoft. Approve the device when prompted.

### Enable Tailscale Funnel

The Funnel feature allows incoming connections from the internet to reach your machine through the Tailscale network. Enable Funnel as a background proxy to the port where gog serve will listen (e.g., 8788):

```bash
sudo tailscale funnel --bg 8788
```

When Funnel is active, Tailscale assigns a public HTTPS URL to your machine. Verify the Funnel status:

```bash
tailscale funnel status
```

Record the public URL shown, as you will need it when configuring the Pub/Sub push subscription.

### Important: Path Handling

When configuring the Funnel, proxy the root path (/) without using the --set-path flag. The --set-path option strips the request path from incoming URLs, which breaks the Pub/Sub webhook delivery mechanism. The Pub/Sub service sends requests to a specific path such as /gmail-pubsub, and this path must be preserved through the entire pipeline.

## 7. Pub/Sub Push Subscription

Now create a push subscription that delivers Pub/Sub messages to your Tailscale Funnel endpoint. This subscription bridges Google's infrastructure to your local machine.

### Create the Push Subscription

Create a subscription configured for push delivery (not pull), pointing to your Tailscale Funnel URL with the /gmail-pubsub path appended:

```bash
gcloud pubsub subscriptions create gmail-webhook-sub \
    --topic=gmail-notifications \
    --push-endpoint=https://[YOUR_TAILSCALE_URL]/gmail-pubsub \
    --ack-deadline=60
```

Replace `[YOUR_TAILSCALE_URL]` with the actual URL from your Funnel status output. The ack-deadline of 60 seconds gives your bot adequate time to process each message before Pub/Sub considers it unacknowledged.

### Verify Subscription Configuration

Confirm that the subscription is configured for push mode:

```bash
gcloud pubsub subscriptions describe gmail-webhook-sub
```

Look for the "pushConfig" field containing your endpoint URL. If the subscription shows "pull" instead of "push," delete it and recreate it with the --push-endpoint parameter.

## 8. gog serve Configuration

The gog serve command runs an HTTP server that receives Pub/Sub webhooks and forwards them to your bot's webhook endpoint. This section covers the correct configuration.

### Start gog serve

Launch the gog serve process with the appropriate flags. Bind to localhost on a port such as 8080, specify the /gmail-pubsub path, provide your bot's webhook URL, and include the message body:

```bash
nohup gog gmail watch serve \
    --account your-email@gmail.com \
    --bind 127.0.0.1 \
    --port 8788 \
    --path /gmail-pubsub \
    --hook-url http://127.0.0.1:[BOT_PORT]/hooks/gmail \
    --hook-token [YOUR_BOT_WEBHOOK_TOKEN] \
    --include-body \
    --max-bytes 20000 > /tmp/gog-serve.log 2>&1 &
```

This configuration receives incoming Pub/Sub webhooks on port 8788 at the /gmail-pubsub path, then forwards them (with email body content up to 20KB) to your bot's webhook listener.

### Critical: Do Not Use --token Flag

A common mistake is adding the --token flag to the gog serve command. Pub/Sub push subscriptions do not send bearer tokens with their requests. Adding --token causes gog to expect authentication that never arrives, resulting in 401 Unauthorized errors. Omit the --token flag entirely when running gog serve for Pub/Sub webhooks.

## 9. Bot Webhook Configuration

Configure your AI assistant bot to accept incoming webhooks at the /hooks/gmail endpoint. The specific configuration depends on your bot's implementation.

### Webhook Endpoint Requirements

Ensure your bot listens on a local port (8088 in the example above) and exposes a POST endpoint at /hooks/gmail. The endpoint should accept JSON payloads containing Gmail notification data. Configure a separate authentication token for this webhook that is distinct from your gateway authentication token.

### Example Bot Configuration

In your bot's configuration file, add a webhook section similar to:

```yaml
webhooks:
  gmail:
    path: /hooks/gmail
    token: [UNIQUE_WEBHOOK_TOKEN]
    methods:
      - POST
```

Restart your bot after adding this configuration so the webhook endpoint becomes active.

## 10. Testing the Pipeline

Test each segment of the pipeline independently to ensure all components are working correctly before testing with real emails.

### Test 1: Local gog Endpoint

Send a test request directly to the gog serve endpoint to verify it is listening:

```bash
curl -v -X POST http://127.0.0.1:8788/gmail-pubsub \
    -H "Content-Type: application/json" \
    -d '{"message":{"data":"eyJ0ZXN0IjoidHJ1ZSJ9"}}'
```

If gog serve is running correctly, you should see an HTTP 200 response. Check logs with `cat /tmp/gog-serve.log`.

### Test 2: Through Tailscale Funnel

Send a request through your Tailscale Funnel URL to verify the public endpoint is reachable:

```bash
curl -v -X POST https://[YOUR_TAILSCALE_URL]/gmail-pubsub \
    -H "Content-Type: application/json" \
    -d '{"message":{"data":"eyJ0ZXN0IjoidHJ1ZSJ9"}}'
```

This tests the entire path from the public internet through Tailscale to your local gog serve.

### Test 3: Bot Webhook Directly

Test your bot's webhook endpoint directly (bypassing gog):

```bash
curl -X POST http://127.0.0.1:[BOT_PORT]/hooks/gmail \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer [YOUR_BOT_WEBHOOK_TOKEN]" \
    -d '{"test":"message"}'
```

Your bot should acknowledge this request and process the test payload.

### Test with Real Email

Send a real email to your Gmail account and observe the pipeline. Check gog serve logs for incoming requests, verify your bot receives the webhook, and confirm the email content is processed correctly. Real testing is essential because it validates the entire flow with authentic Gmail notifications.

## 11. Persistence and Automation

Proper persistence configuration ensures your notification pipeline survives system restarts and continues operating reliably.

### Persistent gog serve Process

The gog serve process will terminate when the terminal session ends. Use nohup or systemd to keep it running continuously. For a systemd service, create a unit file at /etc/systemd/system/gog-gmail-serve.service:

```ini
[Unit]
Description=gog Gmail Watch HTTP Server
After=network.target

[Service]
Type=simple
User=your-username
WorkingDirectory=/home/your-username
Environment=GOG_KEYRING_PASSWORD=your-keyring-passphrase
ExecStart=/usr/local/bin/gog gmail watch serve \
    --account your-email@gmail.com \
    --bind 127.0.0.1 \
    --port 8788 \
    --path /gmail-pubsub \
    --hook-url http://127.0.0.1:[BOT_PORT]/hooks/gmail \
    --hook-token [YOUR_BOT_WEBHOOK_TOKEN] \
    --include-body \
    --max-bytes 20000
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl enable gog-serve
sudo systemctl start gog-serve
```

### Tailscale Autostart

Configure Tailscale to start automatically on boot, ensuring your Funnel remains available:

```bash
sudo systemctl enable tailscaled
sudo systemctl start tailscaled
```

After rebooting, you must re-enable the Funnel:

```bash
sudo tailscale funnel --bg 8788
```

Consider adding this command to a startup script or systemd service.

### Automated Gmail Watch Renewal

Gmail Watch subscriptions expire after seven days. Create a cron job to renew the watch automatically:

```bash
# Edit crontab
crontab -e

# Add renewal job (runs every 6 days to ensure continuity)
0 9 * * 1 /usr/local/bin/gog gmail watch start --account your-email@gmail.com --topic projects/[YOUR_PROJECT_ID]/topics/gmail-notifications
```

## 12. Troubleshooting Common Issues

This section addresses the most frequent problems encountered when setting up the Gmail notification pipeline.

### 401 Unauthorized Errors

If you receive 401 errors in your gog serve logs, you likely added the --token flag. Pub/Sub push webhooks do not send bearer tokens. Remove --token from your gog serve command and restart the process.

### 404 Not Found Errors

404 errors indicate that the request path is being stripped or incorrectly routed. This commonly happens when using Tailscale Funnel with the --set-path flag, which removes the path from incoming requests. Reconfigure your Funnel to proxy the root path (/) without --set-path. Also verify that your Pub/Sub subscription push endpoint includes the full path /gmail-pubsub.

### Pub/Sub Subscription in Pull Mode

If Pub/Sub messages are not being delivered, check that your subscription is configured for push mode. Run `gcloud pubsub subscriptions describe gmail-webhook-sub` and verify the pushConfig section contains your endpoint URL. If it shows no push configuration, the subscription is in pull mode. Delete and recreate it with the --push-endpoint parameter.

### gog Keyring Corruption

On headless servers or systems without a running keyring daemon, gog authentication may fail. Switch to the file-based credential backend:

```bash
gog auth keyring file
gog auth add your-email@gmail.com
```

This uses an encrypted file stored in your home directory instead of relying on the system keyring service.

### gog Serve Process Dying

If gog serve stops unexpectedly, check the process logs for error messages. Ensure you are not using --token, as this causes crashes. Run gog serve with nohup or as a systemd service to prevent it from terminating when the terminal closes. Verify that no other process is using the same port.

## 13. Security Considerations

Protecting your notification pipeline requires attention to several security aspects.

### Separate Webhook Tokens

Use a dedicated authentication token for Gmail webhooks that is different from your main gateway or admin tokens. If a webhook token is compromised, the attacker gains access only to email notifications, not full system control. Store webhook tokens in your bot's configuration file, not in command-line arguments where they may appear in process listings.

### HTTPS Encryption

All traffic between Pub/Sub and your endpoint is encrypted because Tailscale Funnel provides HTTPS termination at its edge servers. The connection from Tailscale to your local machine travels through Tailscale's encrypted WireGuard tunnel, providing protection for internal traffic as well.

### IP Allowlisting

If your bot's infrastructure supports it, configure IP allowlisting to accept connections only from Tailscale IP addresses. This prevents unauthorized access to your webhook endpoint even if the URL is somehow exposed.

### Credential Protection

Never share API keys, service account credentials, or authentication tokens in chat messages, forums, or version control. Use environment variables or secure configuration files with restricted permissions (chmod 600) for storing sensitive values.

### Monitoring and Logging

Implement logging for incoming webhooks so you can detect unusual activity. Set up alerts for authentication failures or unexpected request patterns. Regular log review helps identify both legitimate issues and potential security incidents.

---

This guide provides a complete foundation for receiving real-time Gmail notifications in your AI assistant bot. The modular architecture allows you to modify individual components as needed while maintaining the overall pipeline integrity. Regular maintenance, particularly the Gmail Watch renewal cron job, ensures continuous operation without manual intervention.
