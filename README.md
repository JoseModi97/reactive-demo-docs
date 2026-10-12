# Reactive Demo — Integrations & Production Setup Guide

Welcome to the documentation and integration setup guide for the **Reactive Demo / Reactive Shop** application.

📖 **Live Documentation & Interactive Guide:**  
👉 **[https://josemodi97.github.io/reactive-demo-docs/](https://josemodi97.github.io/reactive-demo-docs/)**

---

## 📚 Contents

- **[Interactive Setup Guide (GitHub Pages)](https://josemodi97.github.io/reactive-demo-docs/)**
  - **Prerequisites & Architecture**: Settings management, AES-256 encrypted credential store, sandbox vs. production toggling, public HTTPS webhooks.
  - **PayPal**: Sandbox merchant apps, Client ID & Secret configuration, sandbox buyer checkout flow, production cutover.
  - **M-Pesa (Safaricom Daraja)**: Developer portal setup, Consumer Key & Secret, Business Shortcode (Paybill/Till), Passkey, STK Push simulation, go-live requirements.
  - **Pesapal**: Account onboarding, 3.0 API Consumer Key & Secret, IPN webhook registration, test cards & mobile money.
  - **Google OAuth 2.0**: Cloud Console project, OAuth Consent Screen, Authorized Redirect URIs, Client ID & Secret.
  - **Transactional Email (Brevo SMTP)**: SMTP settings, master key generation, verified sender identities, template testing.

- **[Production Transfer & Go-Live Guide](PRODUCTION_TRANSFER.md)**
  - Detailed server hardening, SSL/HTTPS reverse proxy (Nginx / Caddy), firewall configuration, environment secrets management (`SHOP_SECRETS_KEY`), webhook verification, and monitoring checklist.

---

## 🚀 Running Offline Locally

If you clone this repository to view offline:
```bash
git clone https://github.com/JoseModi97/reactive-demo-docs.git
cd reactive-demo-docs
# Open directly in any browser
start index.html # On Windows
open index.html  # On macOS
xdg-open index.html # On Linux
```

All 19 high-resolution annotated screenshots are contained within `img/` for 100% offline self-contained browsing.
