# Deploying OpenWA on Render

This guide walks you through deploying **OpenWA (WhatsApp API Gateway)** to [Render](https://render.com).

---

## ⚠️ Key Considerations Before Deploying

### 1. Memory & WhatsApp Engine Selection
- **Render Free Tier (512 MB RAM)**:
  - You **must** set `ENGINE_TYPE=baileys`.
  - Do **not** use `whatsapp-web.js` on the free tier — Puppeteer/Chromium requires 400MB–800MB+ RAM and will crash with Out Of Memory (OOM) errors.
  - Baileys connects via pure WebSocket and uses only ~30–80 MB RAM per session.
- **Render Starter Plan ($7/mo, 1 GB RAM) or Standard ($25/mo, 2 GB RAM)**:
  - Can run `baileys` (multiple sessions) or `whatsapp-web.js` (1–2 sessions).

### 2. Persistent Storage & Session Persistence
- **Render Free Tier**:
  - Does **not** have persistent disks.
  - Free services go to sleep after 15 minutes of inactivity.
  - When the service restarts or wakes up, the ephemeral filesystem (`/app/data`) is reset. This means WhatsApp session tokens are lost, and you must re-scan the QR code.
- **Render Starter Plan ($7/mo)**:
  - Supports attaching a **Persistent Disk** mounted at `/app/data`.
  - All WhatsApp credentials (`/app/data/sessions` / `/app/data/baileys`) and SQLite databases (`/app/data/*.sqlite`) are preserved permanently across restarts and new deploys.

---

## Deployment Methods

### Method 1: Automated 1-Click Deploy via Render Blueprint (Recommended)

OpenWA includes a pre-configured `render.yaml` Blueprint file in the repository root.

1. **Push your OpenWA repository to GitHub or GitLab**:
   ```bash
   git init
   git add .
   git commit -m "feat: initial commit with Render blueprint"
   git remote add origin https://github.com/<your-username>/<your-repo-name>.git
   git branch -M main
   git push -u origin main
   ```

2. **Deploy on Render**:
   - Go to the [Render Dashboard](https://dashboard.render.com).
   - Click **New +** in the top right and select **Blueprint**.
   - Connect your GitHub repository.
   - Render will detect `render.yaml` and configure:
     - Docker runtime using `Dockerfile`.
     - Health check path `/api/health/ready`.
     - Persistent Disk `openwa-data` mounted at `/app/data` (5 GB).
     - Environment variables (auto-generated `API_MASTER_KEY`, `ENGINE_TYPE=baileys`, etc.).
   - Click **Apply**.

> **Note on Free Tier with Blueprint**:
> If you want to use the Free Plan, edit `render.yaml` before pushing:
> - Change `plan: starter` to `plan: free`.
> - Delete or comment out the entire `disk:` section (`name`, `mountPath`, `sizeGB`).

---

### Method 2: Manual Web Service Setup via Render Dashboard

If you prefer to configure the service manually in the Render UI:

1. **Create Web Service**:
   - Go to [Render Dashboard](https://dashboard.render.com) > **New +** > **Web Service**.
   - Choose **Build and deploy from a Git repository**.
   - Select your OpenWA repository.

2. **Configure Service Settings**:
   - **Name**: `openwa` (or any name you like)
   - **Region**: Choose the region closest to you (e.g. Oregon, Frankfurt, Singapore)
   - **Language / Runtime**: `Docker`
   - **Dockerfile Path**: `./Dockerfile` (default)
   - **Instance Type**:
     - **Starter** ($7/month, recommended) or **Free**

3. **Configure Persistent Disk** *(Starter Plan or higher)*:
   - Scroll down to **Disks** and click **Add Disk**.
   - **Name**: `openwa-data`
   - **Mount Path**: `/app/data`
   - **Size**: `5 GB` (or `1 GB`)

4. **Add Environment Variables**:
   Click **Advanced** > **Add Environment Variable**:

   | Variable | Value | Description |
   | :--- | :--- | :--- |
   | `NODE_ENV` | `production` | Production mode |
   | `PORT` | `2785` | Application port |
   | `ENGINE_TYPE` | `baileys` | Low memory engine (~50 MB RAM) |
   | `AUTO_START_SESSIONS` | `true` | Reconnect sessions automatically on boot |
   | `DATABASE_TYPE` | `sqlite` | Embedded database |
   | `DATABASE_NAME` | `./data/openwa.sqlite` | SQLite database path |
   | `MAIN_DATABASE_NAME` | `./data/main.sqlite` | Auth & audit database path |
   | `SESSION_DATA_PATH` | `./data/sessions` | WWebJS session storage |
   | `BAILEYS_AUTH_DIR` | `./data/baileys` | Baileys authentication keys |
   | `STORAGE_TYPE` | `local` | Media storage |
   | `STORAGE_LOCAL_PATH` | `./data/media` | Media storage directory |
   | `SERVE_DASHBOARD` | `true` | Serves web dashboard on root `/` |
   | `ENABLE_SWAGGER` | `true` | Serves API docs on `/api/docs` |
   | `API_MASTER_KEY` | *(Set a 32+ char random string)* | Admin API Key (e.g. `owa_k1_1234567890abcdef1234567890abcdef`) |

5. **Set Health Check Path**:
   - In **Health Check Path**, enter: `/api/health/ready`

6. **Deploy**:
   - Click **Create Web Service**.
   - Render will build the Docker image and start OpenWA.

---

## 🚀 Post-Deployment: Accessing OpenWA

### 1. View Logs & Obtain Your Admin API Key
Once deployment completes, open the **Logs** tab in Render. You will see a banner like this:

```
  🟢 Welcome to OpenWA - WhatsApp API Gateway

  📊 Dashboard: https://openwa.onrender.com
  📚 API Docs:  https://openwa.onrender.com/api/docs

  🔑 API Key:
     owa_k1_...
```

If you set an explicit `API_MASTER_KEY` in environment variables, that is your admin key.

### 2. Open the Dashboard & Connect WhatsApp
1. Open your Render service URL in the browser (e.g. `https://your-openwa.onrender.com`).
2. Log in using your **API Key**.
3. Navigate to **Sessions** > **Create Session**:
   - Choose a session name (e.g. `default`).
   - Select engine `baileys` (or `whatsapp-web.js`).
   - Click **Start**.
4. Scan the QR code with WhatsApp on your phone (**Linked Devices** > **Link a Device**).
5. Your WhatsApp instance is now connected and ready to send and receive messages!
