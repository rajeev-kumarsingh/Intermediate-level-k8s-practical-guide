
# 🚀 Prometheus Installation on Ubuntu (Manual Method)

This guide helps you install Prometheus on an Ubuntu system step-by-step.

---

## 🧱 Step 1: Update Your System

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 👤 Step 2: Create a Prometheus User

```bash
sudo useradd --no-create-home --shell /bin/false --system prometheus
```

---

## 📁 Step 3: Create Required Directories

```bash
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
```

Change ownership:

```bash
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

---

## 📦 Step 4: Download Prometheus

```bash
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz
tar -xvf prometheus-2.52.0.linux-amd64.tar.gz
cd prometheus-2.52.0.linux-amd64
```

---

## 🛠 Step 5: Move Binaries

```bash
sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

---

## ⚙️ Step 6: Move Config and Consoles

```bash
sudo cp -r consoles /etc/prometheus
sudo cp -r console_libraries /etc/prometheus
sudo cp prometheus.yml /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus
```

---

## 🔧 Step 7: Create a systemd Service

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Paste the following:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

---

## ▶️ Step 8: Start and Enable Prometheus

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

Check status:

```bash
sudo systemctl status prometheus
```

---

## 🌐 Step 9: Access Web UI

Open your browser and visit:

```
http://localhost:9090
```

---

## ✅ Done!

You now have Prometheus installed and running on Ubuntu.
