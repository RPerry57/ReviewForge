### The Logic: Why a GitHub App?

Before clicking through configuration screens, it is essential to understand *why* we are building a **GitHub App** instead of using a standard personal access token or an OAuth app.

* **Bot Identity & Permissions:** A GitHub App acts as its own distinct automated entity (a bot user). Instead of borrowing your personal account permissions, it has its own tightly scoped permissions (e.g., *only* read code and write PR comments), which is a crucial security best practice in enterprise software.
* **Installation Model:** Unlike OAuth apps where a user grants access to their entire account, GitHub Apps can be installed granularly on specific repositories or organizations. This mirrors how real-world enterprise tooling (like Datadog, SonarQube, or GitHub Actions) operates.
* **Webhook Architecture:** GitHub Apps natively subscribe to repository events at scale, using cryptographic signatures (HMAC) to ensure that payloads genuinely originate from GitHub and haven't been tampered with.

---

### Step 1: Register Your GitHub App

To let GitHub know your application exists and where to send events, you must register it in your account settings.

1. Go to your **GitHub Profile** -> **Settings** -> **Developer Settings** (at the very bottom of the left sidebar).
2. Click on **GitHub Apps** -> **New GitHub App**.
3. Fill out the core identification fields:
* **GitHub App name:** `ReviewForge-Local` (or any unique name).
* **Homepage URL:** `[https://github.com](https://github.com)` (placeholder for now).
* **Webhook URL:** For local development, GitHub cannot send HTTP requests directly to `localhost`. You need a tunneling tool like **ngrok** to expose your local port. Run `ngrok http 8000` in your terminal, copy the generated public HTTPS URL (e.g., `[https://xxxx.ngrok-free.app](https://xxxx.ngrok-free.app)`), and append `/webhooks/github` to it: `[https://xxxx.ngrok-free.app/webhooks/github](https://xxxx.ngrok-free.app/webhooks/github)`.
* **Webhook secret:** Generate a random secure string (e.g., using `openssl rand -hex 20` in your terminal) and save it. **This secret is used to sign webhook payloads cryptographically.**


4. Set the **Repository Permissions**:
* **Pull requests:** `Read & write` (needed to fetch diffs and post review comments).
* **Contents:** `Read` (needed to clone the repository code inside our Docker sandbox).


5. Set **Subscribe to events**:
* Check **Pull request** (this ensures GitHub fires a webhook whenever a PR is opened, synchronized, or edited).


6. Click **Create GitHub App**.

---

### Step 2: Generate Credentials

Once created, you need two pieces of credentials that your FastAPI backend will use to authenticate with GitHub:

1. **App ID:** Found at the top of your newly created GitHub App settings page.
2. **Private Key:** Scroll down to the "Private keys" section and click **Generate a private key**. This downloads a `.pem` file. Move this file into your project directory (and ensure it is added to your `.gitignore` so you never commit credentials to GitHub).

---

### Step 3: Expose Localhost via ngrok (The Logic)

When GitHub sends a webhook, it makes an outgoing POST request over the internet. Your development machine sits behind a router and a firewall, meaning GitHub has no route to `127.0.0.1:8000`.

**ngrok** solves this by opening a secure reverse proxy tunnel. It gives you a public URL (`*.ngrok-free.app`) that routes incoming internet traffic directly to your local machine's port 8000, allowing you to test webhooks live during development.