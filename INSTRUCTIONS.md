# 💖 Manvish - System & Operations Manual

This document is your complete operational guide for the **Manvish** web apps and the **24/7 Automated WhatsApp Romantic Scheduler** running on Oracle Cloud.

---

## 📌 Quick Reference Card

| Item | Details |
| :--- | :--- |
| **Live Website** | [https://manvish.netlify.app](https://manvish.netlify.app) |
| **GitHub Repository** | `git@github.com:krvish21/manvish.git` (branch `main`) |
| **Oracle VM IP** | `144.24.127.21` |
| **VM Username** | `ubuntu` |
| **SSH Private Key** | `~/Downloads/ssh-key-2026-11-16(1).key` |
| **PM2 Process Name**| `manvish-bot` |
| **Scheduled Time**  | **08:00 AM IST** daily |
| **VM Project Path** | `/home/ubuntu/manvish` |

---

## 🔌 1. Connecting to the Oracle Cloud VM

From your local machine terminal:

```bash
ssh -i "~/Downloads/ssh-key-2026-11-16(1).key" ubuntu@144.24.127.21
```

> **Note:** If you ever move the `.key` file or get a permissions warning, make sure permissions are set to read-only for your user:
> ```bash
> chmod 400 ~/Downloads/ssh-key-2026-11-16\(1\).key
> ```

---

## 🚀 2. How to Deploy Changes

### A. Deploying Scheduler / Bot Changes (Local to VM)
*(Modifying `generator.js`, `scheduler.js`, WhatsApp message templates, or `.env`)*

A dedicated one-command deployer is set up. From your local machine:

```bash
cd /home/vishwajeet/dev/slop/manvish/whatsapp-scheduler
npm run deploy
```

**What it does automatically:**
1. Syncs your local code to the VM using `rsync` over SSH.
2. Keeps your WhatsApp session credentials intact (`auth_info_baileys/` is untouched).
3. Restarts `manvish-bot` with PM2 and prints the status.

---

### B. Deploying Website Changes (`index.html`, `together.html`, `apart.html`)
The frontend is hosted directly on **Netlify**, not on the VM. To update the site:

```bash
cd /home/vishwajeet/dev/slop/manvish
git add .
git commit -m "Update site layout / styles"
git push origin main
```
Netlify automatically detects the push and deploys the live update within 15–20 seconds.

---

### C. Modifying Directly on the VM
If you SSH into the VM and make changes there:

```bash
# 1. Pull changes if you committed to GitHub
cd ~/manvish
git pull

# 2. Restart the PM2 process to apply new code
pm2 restart manvish-bot

# 3. Save the process state
pm2 save
```

---

## ⚙️ 3. Managing the Bot with PM2

When connected to the VM (`ubuntu@144.24.127.21`):

| Action | Command |
| :--- | :--- |
| **Check bot status** | `pm2 status` |
| **View live logs** | `pm2 logs manvish-bot` |
| **View last 50 log lines** | `pm2 logs manvish-bot --lines 50 --nostream` |
| **Restart the bot** | `pm2 restart manvish-bot` |
| **Stop the bot** | `pm2 stop manvish-bot` |
| **Start the bot** | `pm2 start ~/manvish/whatsapp-scheduler/scheduler.js --name "manvish-bot"` |
| **Send now (Default recipient)** | `cd ~/manvish/whatsapp-scheduler && npm run send-now` |
| **Test send to a specific number** | `cd ~/manvish/whatsapp-scheduler && node scheduler.js --now --to 91XXXXXXXXXX` |
| **Preview letter without sending** | `cd ~/manvish/whatsapp-scheduler && npm run test-letter` |

---

## 🔄 4. How the Automated Pipeline Works

Every morning at **08:00 AM IST**:
1. **Cron Trigger:** `scheduler.js` activates inside the background PM2 daemon on Oracle Cloud.
2. **AI Webpage Generation:** Calls Google Gemini API (`gemini-3.5-flash-lite`) via `generator.js`. Gemini composes a personalized, responsive romantic letter and writes it to `letters/YYYY-MM-DD.html`.
3. **Auto-Push to GitHub:** The VM automatically commits the new HTML file and pushes it to `origin main` using the VM's SSH key.
4. **Netlify Build:** Netlify detects the commit and serves the new page at:
   `https://manvish.netlify.app/letters/YYYY-MM-DD.html`
5. **Personal WhatsApp Message:** Baileys connects to WhatsApp as your personal number and sends a good morning message with the direct link to the new letter.

---

## 🔐 5. Configuration & Environment Variables (`.env`)

Located on the VM at: `~/manvish/whatsapp-scheduler/.env`

```env
# Google Gemini API Key
GEMINI_API_KEY=your_gemini_api_key

# Recipient WhatsApp Number (Country code + number, NO '+' or spaces)
RECIPIENT_PHONE_NUMBER=91XXXXXXXXXX

# Optional test phone number for manual tests (e.g. your own number)
TEST_RECIPIENT_NUMBER=91YYYYYYYYYY

# Daily Send Time in 24-hour HH:MM format (Asia/Kolkata timezone)
SEND_TIME=08:00

# Sender and Recipient Names
SENDER_NAME=Vishu
RECIPIENT_NAME=Sabaa

# Anniversary Origin Date & Time
START_DATE=2024-07-03T07:58:00+05:30

# Netlify Base URL
SITE_BASE_URL=https://manvish.netlify.app

# Auto commit and push to Git for Netlify deployment
AUTO_GIT_PUSH=true
```

To edit these values on the VM:
```bash
nano ~/manvish/whatsapp-scheduler/.env
pm2 restart manvish-bot --update-env
```

---

## 🛠️ 6. Troubleshooting & Recovery

### WhatsApp Disconnected / Needs Re-Linking
If WhatsApp Web sessions are revoked or session keys expire:
1. SSH into the VM:
   ```bash
   ssh -i "~/Downloads/ssh-key-2026-11-16(1).key" ubuntu@144.24.127.21
   ```
2. Stop PM2 temporarily:
   ```bash
   pm2 stop manvish-bot
   ```
3. Run the login pairing command:
   ```bash
   cd ~/manvish/whatsapp-scheduler
   npm run login
   ```
4. Scan the QR code displayed in your terminal from your phone (**WhatsApp > Linked Devices > Link a Device**).
5. Once it prints `✅ WhatsApp connected successfully!`, restart PM2:
   ```bash
   pm2 start manvish-bot
   pm2 save
   ```

### VM Reboots
PM2 is configured as a systemd service (`pm2-ubuntu.service`). If Oracle Cloud ever restarts the VM, PM2 and `manvish-bot` will **automatically launch on boot** without any manual intervention.
