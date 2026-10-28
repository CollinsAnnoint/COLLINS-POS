# Deploying Collins Unguku Restaurant POS

The system is a single Node.js server with **zero runtime dependencies**, so it
runs anywhere Node 18+ runs. It stores everything (menu, orders, photos,
settings, encrypted payment credentials) as files, which means the host must
provide a **persistent disk** — that is what `DATA_DIR` is for.

| File | Purpose |
|---|---|
| `server.js` + `index.html` + `assets/` | The app (ships in the Docker image) |
| `pos-data.json` | Seed menu + settings, copied to the disk on first boot |
| `Dockerfile` | Production image (Node 20, no dependencies to install) |
| `railway.json` / `render.yaml` | One-click configs for Railway / Render |
| `DATA_DIR` | Folder on the persistent disk (e.g. `/data`). Holds the live database, uploaded photos, owner PIN file, and encrypted payment secrets |

---

## Recommended path: Railway (~$5/month, free HTTPS address)

Why: always-on (no sleeping), free `https://…up.railway.app` address with
HTTPS included, persistent volumes, and redeploys on every `git push`.
The public HTTPS address is also exactly what M-Pesa Daraja callbacks need.

1. **Put the code on GitHub** (once):
   ```bash
   git init && git add -A && git commit -m "Collins Unguku POS v1"
   # create an empty private repo on github.com, then:
   git remote add origin https://github.com/<you>/collins-unguku-pos.git
   git push -u origin main   # or master, matching your default branch
   ```
   Secrets (`.pos-owner-auth.json`, `.payment-*`) are already in
   `.gitignore` — they will never be pushed.
2. **Create the Railway project:** railway.app → New Project → Deploy from
   GitHub → pick the repo. Railway detects the `Dockerfile` and builds it.
3. **Add the disk:** click the service → Volumes → Add Volume, mount path
   **`/data`**. (Without this, menu changes and orders vanish on redeploy.)
4. **Add the variable:** service → Variables → add `DATA_DIR` = `/data`.
   Railway sets `PORT` itself; the server picks it up automatically.
5. **Get the address:** service → Settings → Networking → Generate Domain.
   You get something like `https://collins-unguku-pos-production.up.railway.app`.
6. **First boot — save the owner PIN:** open Deploy Logs and find
   `New owner PIN (save privately; …): 123456`. **Copy it now** — it is shown
   once. (Lost it? Delete the volume's `.pos-owner-auth.json` via a
   one-off shell, or wipe and re-add the volume — wiping also erases orders.)
7. **Finish setup in the browser:** open your address → Manage → enter the PIN
   → create the owner account. Then run one test Cash sale and print a receipt.
8. **Updates later:** `git push` — Railway rebuilds and redeploys; the `/data`
   disk is untouched, so nothing is lost.

## Alternative: Render

**Two ways — pick one:**

**A. Free, no card (testing/demo only): manual web service.** Render's
Blueprint flow always demands a card on file, but the plain web-service
flow does not for the Free plan:
1. Dashboard → **New → Web Service** (NOT Blueprint) → select your repo.
2. Runtime: **Docker** (Dockerfile auto-detected). Instance type: **Free**.
3. No environment variables needed (`PORT` is set by Render; leave
   `DATA_DIR` unset — data is temporary on the free plan).
4. Create → wait ~5 min → copy the owner PIN from the logs → test.
5. Limits: sleeps after 15 min idle (~1 min to wake); **no disk, so orders,
   menu edits, uploads, and the owner PIN reset whenever it restarts**.
   Fine for trying it out — not for the real till.

**B. Production: Blueprint + Starter + disk (~$7.25/mo, card required).**
Needs a card on file (a free M-Pesa GlobalPay virtual Visa works:
M-Pesa App or `*334#` → Lipa na M-Pesa → M-Pesa GlobalPay).

## Alternative: own VPS (Hetzner / DigitalOcean, ~€4.50+/mo)

For whoever manages your server. Needs Docker + a domain or public IP:

```bash
git clone https://github.com/<you>/collins-unguku-pos.git && cd collins-unguku-pos
docker build -t collins-pos .
docker run -d --restart unless-stopped --name collins-pos \
  -p 127.0.0.1:8000:8000 -v collins-pos-data:/data collins-pos
docker logs collins-pos   # copy the one-time owner PIN
```

Put Caddy or Nginx in front for HTTPS (required for M-Pesa callbacks):
`your-domain.tld → reverse proxy → 127.0.0.1:8000`.

## Alternative: restaurant PC (offline till, no live M-Pesa)

If internet at the shop is unreliable and you can live without live mobile
payments for now, the till can run on the shop computer itself:

1. Install Node.js 20 LTS from nodejs.org.
2. Copy `server.js`, `index.html`, `assets/`, `pos-data.json` to a folder
   (no `DATA_DIR` needed — it stores data next to itself, as today).
3. Start it with `node server.js` (double-click a `start-pos.bat` containing
   that line on Windows). Open `http://localhost:8000`.
4. Other devices on the shop Wi-Fi can open `http://<PC-IP>:8000`.
5. Limitation: no public HTTPS address, so Daraja STK callbacks and Pesapal
   IPN cannot reach it — use Cash / Room charge until the cloud move.

## Backups (do this monthly, and before any big change)

- **Easy:** Manage → sign in → download/copy `pos-data.json` from the disk.
  On Railway: service → Volumes → … → download, or run a one-off shell and
  `cat /data/pos-data.json`. Also back up `/data/assets/uploads/`.
- Keep one dated copy off-site (email it to yourself or a USB stick).

## Going live with M-Pesa (when credentials arrive)

1. Get Daraja **production** credentials from developer.safaricom.co.ke
   (Consumer Key, Consumer Secret, passkey, shortcode). Keep them private —
   enter them only in the POS screen, never in chat or email.
2. In the POS: Manage → owner PIN → Payments → environment **sandbox** first
   → enter credentials → Test → run a KES 1 STK push to your own phone.
3. Set the **callback base URL** to your public address
   (e.g. `https://…up.railway.app`) — no path, HTTPS.
4. When sandbox passes, switch to **live**, test with KES 1 again, then open up.
5. Pesapal: same flow, then press **Register IPN** so card notifications arrive.

## Costs at a glance

| Option | Approx. cost | Sleeps? | M-Pesa-ready? |
|---|---|---|---|
| Railway + volume | ~$5/mo usage | No | Yes |
| Render Starter + disk | ~$7.25/mo | No | Yes |
| Small VPS | ~€4.50–6/mo | No | Yes (with HTTPS proxy) |
| Shop PC | Free | No | No (cash/manual only) |

## Troubleshooting

| Symptom | Fix |
|---|---|
| Menu/orders reset after redeploy | The disk is missing or not mounted at `/data`; check Volumes + `DATA_DIR` |
| `PORT` / address-in-use errors | Only one server per machine/port; check for a second `node server.js` |
| Lost owner PIN | Remove `.pos-owner-auth.json` from the disk and restart (a new PIN prints once) |
| M-Pesa stuck on "pending" | Callback base URL must be the exact public HTTPS origin; check deploy logs for Daraja errors |
