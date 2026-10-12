# Transfer to Production — Go-Live Guide

A step-by-step guide for moving **Reactive Shop** from local sandbox testing to a secure, public, production store.

> 📖 **Online Interactive Setup Guide**:  
> 👉 **[https://josemodi97.github.io/reactive-demo-docs/](https://josemodi97.github.io/reactive-demo-docs/)**  
> Browse full-resolution annotated screenshots and interactive section walkthroughs on GitHub Pages (or locally at [`setup-guide/index.html`](setup-guide/index.html)).

---

## 1. Overview & Architecture

Reactive Shop uses an isolated multi-environment architecture:

- **Isolated Credentials**: Stored in `integration-settings.json`, encrypted via AES with `SHOP_SECRETS_KEY`.
- **Zero-Downtime Switching**: Every payment provider has distinct **Sandbox** and **Production** credential tabs under **Admin → Integrations**.
- **No Interruption**: Switching an environment takes effect immediately without restarting the application server. In-flight sandbox sessions complete in sandbox, while all subsequent orders route to production.

---

## 2. Server, Domain & HTTPS Reverse Proxy

Payment webhooks and callbacks (especially Safaricom Daraja STK push and Pesapal IPN) strictly require a public **HTTPS** URL. Connections to `localhost` or unencrypted HTTP are rejected by gateway servers.

### 2.1 Set your Public URL
Set `SHOP_PUBLIC_URL` to your production domain without a trailing slash:
```env
SHOP_PUBLIC_URL=https://shop.example.com
```
All payment return URLs, IPNs, and email notification links automatically derive from this address.

### 2.2 Reverse Proxy Configuration (Nginx)
Run the application on port 8080 behind Nginx with Let's Encrypt SSL:

```nginx
server {
    listen 80;
    server_name shop.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name shop.example.com;

    ssl_certificate     /etc/letsencrypt/live/shop.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/shop.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;

        # Keep server-sent event (SSE) live streams open
        proxy_buffering off;
        proxy_read_timeout 3600s;
    }
}
```

### 2.3 Forwarded Headers
Set `SHOP_FORWARD_HEADERS=framework` in your environment. This instructs Netty to trust the `X-Forwarded-For` header set by Nginx, ensuring that built-in rate limiters track real client IPs instead of the reverse proxy's IP.

---

## 3. Production Database & Persistence

In-memory H2 is for testing only; all tables and data vanish when the process stops. Production requires a durable, relational database engine.

### 3.1 Supported Databases
- **MariaDB** (10.6+)
- **MySQL** (8.0+)
- **PostgreSQL** (14+)
- **Microsoft SQL Server** (2019+)
- **Oracle Database** (19c+)

### 3.2 Configuration
Supply both R2DBC and Flyway JDBC URLs via environment variables:
```env
SPRING_R2DBC_URL=r2dbc:mariadb://localhost:3306/reactive_shop
SPRING_R2DBC_USERNAME=shop_user
SPRING_R2DBC_PASSWORD=StrongDatabasePasswordHere

SPRING_FLYWAY_URL=jdbc:mariadb://localhost:3306/reactive_shop
SPRING_FLYWAY_USER=shop_user
SPRING_FLYWAY_PASSWORD=StrongDatabasePasswordHere
```

On first startup against a clean database, Flyway automatically creates the full schema, indexes, initial catalog, and settings.

### 3.3 Automated Backups
Schedule daily database dumps via cron:
```bash
# MariaDB / MySQL:
mysqldump -u shop_user -p'StrongDatabasePasswordHere' reactive_shop | gzip > /backups/db-$(date +\%F).sql.gz

# PostgreSQL:
pg_dump -U postgres -d reactive_shop -F c -f /backups/db-$(date +\%F).dump
```

---

## 4. Production Safety Checks & Secret Hardening

When started with `SHOP_PRODUCTION=true` (the default), `ProductionSafetyCheck` verifies essential security parameters at boot and **refuses to start** if unsafe development values remain.

| Setting | Production Requirement | Description |
|---|---|---|
| `SHOP_PRODUCTION` | `true` | Enables strict production validations and enforces IP rate limiting. |
| `JWT_SECRET` | 32+ characters | Random secret used to sign customer and admin auth tokens. |
| `ADMIN_PASSWORD` | 12+ characters | Must not be `admin123`. Rotate the default admin password. |
| `SHOP_SECRETS_KEY` | 32+ characters | AES key encrypting integration credentials. Back up `shop-install.properties`. |
| `SHOP_CORS_ALLOWED_ORIGINS` | Empty or your site origins | Disallows wildcard `*` or `localhost`. |

> **Critical**: Back up `shop-install.properties` off-server. Without `shop.secrets-key`, stored credentials in `integration-settings.json` cannot be decrypted after moving to a new server.

---

## 5. Gateway Transition: Sandbox to Live

Configure live credentials in **Admin → Integrations**. For each gateway you offer, follow these steps:

### 5.1 PayPal
1. In [developer.paypal.com](https://developer.paypal.com/dashboard/), switch the toggle to **Live**.
2. Open **Apps & Credentials** → **Create App** (type: *Merchant*). Copy the live **Client ID** and **Secret**.
3. In shop **Admin → Integrations → PayPal**, select **Edit settings for: Production**.
4. Paste the live Client ID and Secret.
5. Click **Save production**, then click **Test**.
6. Set **Environment in use** to **Production**, and toggle **Offered at checkout** to ON.
7. *Card Payments Tip*: Under your PayPal Business Account settings (*Website Payments*), ensure **PayPal Account Optional** is enabled so buyers without a PayPal account can pay with a card.

### 5.2 M-Pesa Express (Safaricom Daraja)
1. In [developer.safaricom.co.ke](https://developer.safaricom.co.ke/), open **Go Live**.
2. Submit your live Paybill or Till Shortcode and authorized Business Administrator details.
3. Under **My Apps → Production Apps**, copy your live **Consumer Key** and **Consumer Secret**. Safaricom provides your production **Passkey**.
4. In shop **Admin → Integrations → M-Pesa Express**, select **Edit settings for: Production**.
5. Paste your live Consumer Key, Consumer Secret, Business Shortcode, and Passkey.
6. Verify Callback URL is `https://shop.example.com/api/payment/callback`.
7. Click **Save production**, click **Test**, and set **Environment in use** to **Production**.

### 5.3 Pesapal
1. Log in to your live merchant portal at [pesapal.com](https://www.pesapal.com/).
2. Retrieve your live **Consumer Key** and **Consumer Secret**.
3. In shop **Admin → Integrations → Pesapal**, select **Edit settings for: Production**.
4. Change the API Base URL to: `https://pay.pesapal.com/v3`.
5. Enter the live Key and Secret. Ensure IPN URL is `https://shop.example.com/api/payments/pesapal/ipn`.
6. Click **Save production**, click **Test**, and set **Environment in use** to **Production**.

### 5.4 Transactional Email (SMTP)
1. Choose a transactional email service (e.g. Brevo, Amazon SES, SendGrid, or your web host).
2. Configure your sending domain with **SPF**, **DKIM**, and **DMARC** DNS records.
3. In shop **Admin → Integrations → Email (SMTP)**, enter host, port, credentials, and your verified From address.
4. Click **Test** to send a test email. Confirm it arrives in your inbox (not spam).
5. Toggle **Send emails** to ON.

### 5.5 Google Sign-In (Firebase)
1. In [Firebase Console](https://console.firebase.google.com/) → **Authentication → Settings → Authorized domains**, add `shop.example.com`.
2. In [Google Cloud Console](https://console.cloud.google.com/apis/credentials) → **OAuth 2.0 Client IDs**, add `https://shop.example.com` to **Authorized JavaScript origins** for One Tap.

---

## 6. Clearing Test Data: Restart Store Fresh

Before opening the store to public shoppers, purge any mock orders and reserved stock accumulated during sandbox testing:

1. Log in as admin and navigate to **Admin → Overview**.
2. Click **Restart store fresh** in the top-right toolbar.
3. Confirm the prompt.

**What this resets**:
- Purges all test orders and line items.
- Restores all reserved catalog stock back to original quantities.
- Clears test shopping carts (both guest and registered).
- Resets payment tracking logs.
- Resets invoice and order sequence counters.
- **Preserved intact**: Catalog items, categories, pricing, branding, and all saved integration credentials.

---

## 7. Real-Money Smoke Test & Refund

Perform one live end-to-end verification before announcing launch:

1. Add a low-cost item (or temporary $1 / 10 KES product) to the cart.
2. Complete checkout as a customer with a real credit card or M-Pesa phone number.
3. Verify that:
   - The payment processes without errors.
   - The customer is redirected to the confirmation screen.
   - Order status transitions immediately to `PAID`.
   - Confirmation email is received in the customer inbox.
   - The transaction appears in **Admin → Stock Sold**.
4. Immediately refund the transaction from the payment provider dashboard (PayPal or Pesapal).

---

## 8. Continuous Process Management (Systemd)

Create a Systemd service to keep Reactive Shop running 24/7 with automatic restart on crashes or reboots:

```ini
# /etc/systemd/system/reactive-shop.service
[Unit]
Description=Reactive Shop E-Commerce Platform
After=network.target

[Service]
Type=simple
User=shop
WorkingDirectory=/var/www/reactive-shop
ExecStart=/usr/bin/java -Xms512m -Xmx2g -jar reactive-shop.jar
Restart=always
RestartSec=10
EnvironmentFile=/var/www/reactive-shop/.env

# Security sandbox
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now reactive-shop
sudo systemctl status reactive-shop
```

---

## 9. Pre-Launch Checklist

- [ ] Public domain points to server over HTTPS with a valid SSL certificate.
- [ ] `SHOP_PUBLIC_URL` set to `https://yourdomain.com` (no trailing slash).
- [ ] Persistent database (MariaDB / MySQL / PostgreSQL) active with regular backups.
- [ ] `SHOP_PRODUCTION=true` enabled; strong `JWT_SECRET` and `ADMIN_PASSWORD`.
- [ ] `shop-install.properties` backed up off-server.
- [ ] Gateway credentials verified under **Admin → Integrations → Production** and toggled to **Production**.
- [ ] Transactional email DNS records (SPF, DKIM, DMARC) verified and test email received in inbox.
- [ ] Firebase and Google OAuth authorized domains updated with the production address.
- [ ] Test orders and carts purged via **Admin → Overview → Restart store fresh**.
- [ ] Real-money test transaction successfully executed and refunded.
- [ ] Systemd service or Docker daemon configured with auto-restart on boot.
