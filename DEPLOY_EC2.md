
# Deploy Agentic Voice AI on AWS EC2 (HTTPS)

This guide deploys Agentic Voice AI on an **Ubuntu 24.04 LTS AWS EC2 instance** using Node.js, PM2, Nginx, and Let's Encrypt.

Replace `yourdomain.com`, `YOUR_ELASTIC_IP`, and `/path/to/key.pem` with your values.

## 1. EC2 Setup

1. Launch an EC2 instance with **Ubuntu 24.04 LTS**.
2. Choose **t3.small or larger**, depending on build-time memory requirements.
3. Configure security group inbound rules:
   - **22 (SSH):** Your IP address only.
   - **80 (HTTP):** Public access.
   - **443 (HTTPS):** Public access.
4. Allocate and associate an **Elastic IP**.
5. Configure your domain's DNS A record to point to the Elastic IP.

Do not expose application port `3000` through the EC2 security group.

## 2. SSH and Install Dependencies

Connect to the EC2 instance:

```bash
ssh -i /path/to/key.pem ubuntu@YOUR_ELASTIC_IP
```

Install system packages and Node.js 24 LTS:

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y curl nginx certbot python3-certbot-nginx git

curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs

sudo npm install -g pm2

node -v
npm -v
pm2 -v
```

## 3. Clone and Configure the Application

Clone the repository:

```bash
cd ~

git clone https://github.com/SurajPatelM/agentic-voice-ai.git

cd agentic-voice-ai

npm ci
```

If the repository does not contain a compatible `package-lock.json`, use `npm install` instead.

### Configure environment variables

```bash
cp .env.example .env.local
nano .env.local
chmod 600 .env.local
```

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Claude LLM API |
| `VAPI_API_KEY` | Vapi authentication |
| `VAPI_PHONE_NUMBER_ID` | Voice phone number |
| `VAPI_ASSISTANT_ID` | Voice assistant |
| `RESEND_API_KEY` | Email notifications |
| `RESEND_FROM_EMAIL` | Verified sender email |
| `TWILIO_ACCOUNT_SID` | SMS account |
| `TWILIO_AUTH_TOKEN` | SMS authentication |
| `TWILIO_PHONE_NUMBER` | SMS sender |
| `NEXT_PUBLIC_APP_URL` | `https://yourdomain.com` |

Never commit `.env.local` or API credentials.

### Build the application

```bash
npm run build
```

### Test locally

Start the application bound to localhost:

```bash
npm run start -- --hostname 127.0.0.1 --port 3000
```

In another SSH session:

```bash
curl -I http://127.0.0.1:3000/
```

Stop the test process using `Ctrl+C`.

## 4. Configure PM2

Start the application using PM2:

```bash
pm2 start npm \
  --name agentic-voice-ai \
  -- start -- --hostname 127.0.0.1 --port 3000

pm2 save
pm2 startup
```

Run the additional command printed by `pm2 startup` to enable automatic startup after reboot.

Verify application status:

```bash
pm2 status
pm2 logs agentic-voice-ai
```

The application runs on `127.0.0.1:3000` behind Nginx.

## 5. Configure Nginx Reverse Proxy

Create the Nginx configuration:

```bash
sudo nano /etc/nginx/sites-available/agentic-voice-ai
```

Add:

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout 120s;
    }
}
```

Enable the site:

```bash
sudo ln -sf \
  /etc/nginx/sites-available/agentic-voice-ai \
  /etc/nginx/sites-enabled/agentic-voice-ai

sudo rm -f /etc/nginx/sites-enabled/default

sudo nginx -t
sudo systemctl reload nginx
```

## 6. Configure HTTPS with Let's Encrypt

Generate and install the TLS certificate:

```bash
sudo certbot --nginx -d yourdomain.com
```

Follow the prompts to configure HTTPS.

Check certificate renewal:

```bash
sudo certbot renew --dry-run
```

Use `-d www.yourdomain.com` as an additional argument only if that subdomain is configured in DNS and Nginx.

## 7. Verify Deployment

Run from your laptop:

```bash
curl -fsS -o /dev/null -w "%{http_code}\n" \
  https://yourdomain.com/

curl -fsS -o /dev/null -w "%{http_code}\n" \
  https://yourdomain.com/dashboard
```

Expected HTTP status for publicly accessible pages: `200`.

On the server, optionally run the repository's smoke tests:

```bash
bash scripts/smoke-ec2.sh
```

Review that script before executing it to ensure it uses the correct domain and endpoints.

## 8. Configure Vapi Voice Integration

Set the Vapi assistant Server URL to:

```text
https://yourdomain.com/api/vapi/webhook
```

Configure:

- Assistant ID and phone number
- Server webhook URL
- Tool function names and schemas
- Synchronous tool execution (`async: false`)
- Inbound and outbound voice handling

Ensure tool definitions match:

- `getClaudeTools()`
- `getVapiModelFunctions()`

Verify that webhook responses follow the required Vapi tool-results contract.

See [VAPI_SETUP.md](VAPI_SETUP.md) for detailed instructions.

Before enabling public voice traffic, configure appropriate webhook authentication and validate incoming requests.

## 9. Update the Application

After pushing changes to GitHub:

```bash
cd ~/agentic-voice-ai

git pull

npm ci

npm run build

pm2 restart agentic-voice-ai --update-env

pm2 save
```

Note: Restarting PM2 clears the application's in-memory sessions and appointment records.

For durable persistence, migrate to a database before supporting real users.

## 10. Troubleshooting

### 502 Bad Gateway

Check whether the application is running:

```bash
pm2 status
pm2 logs agentic-voice-ai
```

Test the local application:

```bash
curl -I http://127.0.0.1:3000/
```

### Nginx Configuration Issues

```bash
sudo nginx -t

sudo systemctl status nginx

sudo tail -n 100 /var/log/nginx/error.log
```

### Environment Variables Not Loaded

Verify that `.env.local` exists in the application root:

```bash
cd ~/agentic-voice-ai
ls -la .env.local
```

Next.js loads `.env.local` when the application starts.

Restart PM2 after changing configuration.

### Email Delivery Failures

Check:

- Resend API key
- Verified sending domain
- `RESEND_FROM_EMAIL`
- Application logs

### Voice Integration Failures

Check:

- Vapi assistant and phone configuration
- Public HTTPS webhook URL
- Incoming webhook payload structure
- Tool names and parameter schemas
- API credentials and webhook authentication

### Application Runs Out of Memory During Build

Next.js builds can require more memory than a small EC2 instance provides.

Use a larger instance or build artifacts in CI and deploy them to EC2.

## Important Limitations

This deployment is intended for demonstration and development.

- Appointment and session data are stored in memory.
- Process restarts clear application state.
- Provider availability and prescription status are simulated.
- The application is not established as HIPAA-compliant.
- Real patient information must not be processed without appropriate security, privacy, and compliance controls.

A publicly accessible HTTPS deployment does not automatically make the application production-ready for healthcare use.
