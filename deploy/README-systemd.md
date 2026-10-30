# VPS deployment without Docker (systemd)

Prefer this over `deploy/README.md` (Docker) if you'd rather manage the
processes directly with systemd — same 24/7 reliability, one less layer.

## 1. Create a dedicated user and directory

```bash
sudo useradd --system --create-home --shell /bin/bash mailmind
sudo mkdir -p /opt/mailmind-ai
sudo chown mailmind:mailmind /opt/mailmind-ai
```

## 2. Clone and set up as that user

```bash
sudo -u mailmind -i
cd /opt/mailmind-ai
git clone https://github.com/ankitkumar/mailmind-ai.git .

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
nano .env   # set ENVIRONMENT=production, API_KEY, POSTGRES_PASSWORD/DATABASE_URL, Telegram, etc.

# Postgres (if not already installed):
sudo apt install postgresql
sudo -u postgres createuser mailmind
sudo -u postgres createdb -O mailmind mailmind
sudo -u postgres psql -c "ALTER USER mailmind PASSWORD 'yourpassword';"
# then set DATABASE_URL=postgresql://mailmind:yourpassword@localhost/mailmind in .env

alembic upgrade head

python -m app.gmail.auth   # one-time OAuth2 flow, needs credentials.json present
deactivate
exit   # back to your normal user
```

## 3. Dashboard setup

```bash
sudo -u mailmind -i
cd /opt/mailmind-ai/dashboard
npm install
npm run build
cp .env.local.example .env.local
nano .env.local   # BACKEND_API_URL=http://localhost:8000, BACKEND_API_KEY=<same as API_KEY above>
exit
```

## 4. Install the systemd units

```bash
sudo cp deploy/mailmind-backend.service /etc/systemd/system/
sudo cp deploy/mailmind-dashboard.service /etc/systemd/system/
sudo cp deploy/mailmind-prune-logs.service /etc/systemd/system/
sudo cp deploy/mailmind-prune-logs.timer /etc/systemd/system/

sudo systemctl daemon-reload
sudo systemctl enable --now mailmind-backend
sudo systemctl enable --now mailmind-dashboard
sudo systemctl enable --now mailmind-prune-logs.timer
```

## 5. Check status / logs

```bash
sudo systemctl status mailmind-backend
sudo journalctl -u mailmind-backend -f
sudo journalctl -u mailmind-dashboard -f
```

## 6. Nginx + HTTPS

Same nginx config and certbot steps as `deploy/README.md` §4, except point
`proxy_pass` at `http://127.0.0.1:3000` (dashboard) and
`http://127.0.0.1:8000` (backend) instead of Docker service names.

## 7. Updating

```bash
sudo -u mailmind -i
cd /opt/mailmind-ai
git pull
source .venv/bin/activate && pip install -r requirements.txt && alembic upgrade head && deactivate
cd dashboard && npm install && npm run build
exit

sudo systemctl restart mailmind-backend mailmind-dashboard
```
