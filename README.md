# PCH Sweepstakes Auto-Entry System

> Automated sweepstakes entry submission for Publishers Clearing House (PCH) using Selenium WebDriver in a containerized Selenium Grid environment.

![Architecture Diagram](docs/architecture_diagram.png)

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Project Structure](#project-structure)
- [Script Reference](#script-reference)
- [Troubleshooting](#troubleshooting)
- [Anti-Bot Evasion](#anti-bot-evasion)
- [Scaling](#scaling)
- [Monitoring & Debugging](#monitoring--debugging)
- [License](#license)

---

## Overview

This project automates the process of entering PCH sweepstakes competitions. It uses:

- **Selenium WebDriver** for browser automation
- **Selenium Grid 4** running in **Podman containers** for distributed browser execution
- **Python 3.12+** with a dedicated virtual environment
- **systemd services** for container orchestration and auto-restart
- **Cron jobs** for scheduled daily execution

The system supports multiple user accounts, handles advertisement videos, manages login/logout sessions, and implements anti-bot detection evasion techniques.

---

## Architecture

```
+------------------------------------------------------------------+
|                         PCH.com Website                          |
|  Sweepstakes Pages | Advertisements | Anti-Bot Detection         |
+------------------------------------------------------------------+
                              ^
                              | HTTPS
+-----------------------------+------------------------------------+
|                      Linux Host (Rocky/Ubuntu)                   |
|                                                                  |
|  +------------------+    +------------------+                  |
|  | Cron Scheduler   |    | Python Scripts   |                  |
|  | 0 4,11 * * *   |--->| pchprizes2.py    |                  |
|  | pchprizes2.py N|    | _pch_psp.py      |                  |
|  +------------------+    +--------+---------+                  |
|                                   |                            |
|  +------------------+            | HTTP 4444                  |
|  | Python venv      |<-----------+                            |
|  | selenium 4.35+  |                                         |
|  +------------------+                                         |
|                                                                  |
|  +------------------+    +------------------+                  |
|  | Podman Engine    |    | Systemd Services |                  |
|  | Rootless         |<---| selenium-hub     |                  |
|  | SELinux compat |    | selenium-node1..N|                  |
|  +------------------+    +------------------+                  |
|                                                                  |
|  +----------------------------------------------------------+  |
|  |              Selenium Grid (Podman Containers)             |  |
|  |  +------------------+    +------------------+            |  |
|  |  | Selenium Hub     |    | Node 1 (Chrome)  |            |  |
|  |  | Port 4444        |<-->| Port 5900/7900   |            |  |
|  |  | Event Bus        |    | shm-size: 1g     |            |  |
|  |  +--------+---------+    +------------------+            |  |
|  |           |                    ...                        |  |
|  |           +----------------> Node N (Chrome/Firefox)     |  |
|  +----------------------------------------------------------+  |
|                                                                  |
|  +------------------+    +------------------+                  |
|  | Browser Binaries |    | Log Files        |                  |
|  | Chrome 142+      |    | ~/pchprizes2.log |                  |
|  | ChromeDriver     |    +------------------+                  |
|  | Firefox/Gecko    |                                         |
|  +------------------+                                         |
+------------------------------------------------------------------+
```

### Key Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Container Runtime** | Podman | Rootless, daemonless container engine |
| **Grid Orchestrator** | Selenium Grid 4 | Distributes browser sessions across nodes |
| **Browser Nodes** | Chrome/Firefox containers | Execute actual browser automation |
| **Script Engine** | Python 3.12 + Selenium | Entry submission logic |
| **Scheduler** | Cron | Daily automated runs |
| **Service Manager** | systemd | Container lifecycle & auto-restart |

---

## Prerequisites

### Hardware
- Linux host (tested on Rocky Linux 9, Ubuntu 22.04+)
- Minimum 4GB RAM (8GB+ recommended for multiple nodes)
- Internet connectivity for PCH.com access

### Software
```bash
# Rocky Linux 9 / RHEL 9
sudo dnf install -y podman systemd python3.12 java-21-openjdk

# Ubuntu 22.04+
sudo apt update
sudo apt install -y podman systemd python3.12 openjdk-21-jdk
```

### Network
- Wi-Fi or Ethernet connection
- Port 4444 available on localhost for Selenium Hub

---

## Installation

### 1. Install Podman

```bash
# Rocky Linux 9
sudo dnf install -y podman
sudo systemctl enable podman
sudo systemctl start podman

# Ubuntu
sudo apt install -y podman
```

### 2. Configure Container Registries

For systems without default registries (e.g., Amazon Linux):

```bash
sudo mkdir -p /etc/containers/

# Create registries.conf
sudo tee /etc/containers/registries.conf << 'EOF'
unqualified-search-registries = ["registry.access.redhat.com", "registry.redhat.io", "docker.io"]
short-name-mode = "permissive"
EOF

# Create policy.json
sudo tee /etc/containers/policy.json << 'EOF'
{
    "default": [{"type": "insecureAcceptAnything"}],
    "transports": {
        "docker": {
            "registry.access.redhat.com": [
                {"type": "signedBy", "keyType": "GPGKeys", "keyPaths": ["/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release"]}
            ],
            "registry.redhat.io": [
                {"type": "signedBy", "keyType": "GPGKeys", "keyPaths": ["/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release"]}
            ]
        },
        "docker-daemon": {"": [{"type": "insecureAcceptAnything"}]}
    }
}
EOF
```

### 3. Pull Selenium Images

```bash
sudo podman pull selenium/hub:latest
sudo podman pull selenium/node-chrome:latest
sudo podman pull selenium/node-firefox:latest
```

### 4. Create Container Network

```bash
sudo podman network create selenium-grid
```

### 5. Create Systemd Service Files

#### Selenium Hub Service

```bash
sudo tee /etc/systemd/system/selenium-hub.service << 'EOF'
[Unit]
Description=Selenium Grid Hub (Podman)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=%t/containers

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70
ExecStart=/usr/bin/podman run \
        --cidfile=%t/%n.ctr-id \
        --cgroups=no-conmon \
        --rm \
        --sdnotify=conmon \
        -d \
        --replace \
        --name selenium-hub \
        -p 4444:4444 \
        --network selenium-grid \
        selenium/hub:latest
ExecStop=/usr/bin/podman stop \
        --ignore -t 10 \
        --cidfile=%t/%n.ctr-id
ExecStopPost=/usr/bin/podman rm \
        -f \
        --ignore -t 10 \
        --cidfile=%t/%n.ctr-id
Type=notify
NotifyAccess=all

[Install]
WantedBy=default.target
EOF
```

#### Selenium Node Service (Chrome)

```bash
sudo tee /etc/systemd/system/selenium-node1.service << 'EOF'
[Unit]
Description=Selenium Grid Node 1 - Chrome (Podman)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=%t/containers

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70
ExecStart=/usr/bin/podman run \
        --cidfile=%t/%n.ctr-id \
        --cgroups=no-conmon \
        --rm \
        --sdnotify=conmon \
        -d \
        --replace \
        --name selenium-node1 \
        -e SE_EVENT_BUS_HOST=selenium-hub \
        -e SE_EVENT_BUS_PUBLISH_PORT=4442 \
        -e SE_EVENT_BUS_SUBSCRIBE_PORT=4443 \
        --network selenium-grid \
        --shm-size=1g \
        selenium/node-chrome:latest
ExecStop=/usr/bin/podman stop \
        --ignore -t 10 \
        --cidfile=%t/%n.ctr-id
ExecStopPost=/usr/bin/podman rm \
        -f \
        --ignore -t 10 \
        --cidfile=%t/%n.ctr-id
MemoryLimit=1536M
MemorySoftLimit=1024M
Type=notify
NotifyAccess=all

[Install]
WantedBy=default.target
EOF
```

> **Note:** To add more nodes, copy `selenium-node1.service` to `selenium-node2.service`, etc., and update `--name selenium-node2`.

### 6. Enable and Start Services

```bash
sudo systemctl daemon-reload
sudo systemctl enable selenium-hub selenium-node1
sudo systemctl start selenium-hub selenium-node1
```

### 7. Verify Grid is Running

```bash
# Check containers
sudo podman ps -a

# Access Grid Console
# Open browser to: http://localhost:4444
# Or via CLI:
curl http://localhost:4444/wd/hub/status
```

### 8. Set Up Python Environment

```bash
# Install Python 3.12+ if not available
sudo dnf install -y python3.12  # Rocky Linux
# or
sudo apt install -y python3.12  # Ubuntu

# Create virtual environment
python3.12 -m venv ~/selenium_env
source ~/selenium_env/bin/activate

# Upgrade pip and install dependencies
pip install --upgrade pip
pip install selenium==4.35.0 requests bs4 setuptools     2captcha-python undetected-chromedriver     undetected-geckodriver webdriver-manager
```

### 9. Install Browser Binaries

#### Chrome for Testing + ChromeDriver

```bash
# Download matching versions from Chrome for Testing
# Check https://googlechromelabs.github.io/chrome-for-testing/ for latest

CHROME_VERSION="142.0.7444.59"

# Chrome for Testing
wget https://storage.googleapis.com/chrome-for-testing-public/${CHROME_VERSION}/linux64/chrome-linux64.zip
sudo unzip chrome-linux64.zip -d /opt/
echo "export PATH=\$PATH:/opt/chrome-linux64" >> ~/.bash_profile
source ~/.bash_profile

# ChromeDriver
wget https://storage.googleapis.com/chrome-for-testing-public/${CHROME_VERSION}/linux64/chromedriver-linux64.zip
unzip chromedriver-linux64.zip
sudo cp chromedriver-linux64/chromedriver /usr/bin/chromedriver
sudo chown root:root /usr/bin/chromedriver
sudo chmod +x /usr/bin/chromedriver
```

#### Firefox + geckodriver

```bash
wget -O ~/FirefoxSetup.tar.bz2 "https://download.mozilla.org/?product=firefox-latest&os=linux64"
sudo tar xf ~/FirefoxSetup.tar.bz2 -C /opt/
echo "export PATH=/opt/firefox:\$PATH" >> ~/.bash_profile
source ~/.bash_profile

wget https://github.com/mozilla/geckodriver/releases/download/v0.36.0/geckodriver-v0.36.0-linux64.tar.gz
tar xf geckodriver-v0.36.0-linux64.tar.gz
sudo mv geckodriver /usr/bin/
sudo chown root:root /usr/bin/geckodriver
```

### 10. Clone and Configure Scripts

```bash
git clone <your-repo-url> ~/pch-sweepstakes
cd ~/pch-sweepstakes

# Make scripts executable
chmod +x pchprizes2.py
chmod +x _pch_psp.py
```

---

## Configuration

### User Credentials

Edit `pchprizes2.py` to configure your PCH user accounts:

```python
if userNum == 1:
    email = "your-email-1@example.com"
    password = "your-secure-password"
    # ...
```

> **Security Note:** Store credentials securely. Consider using environment variables or a secrets manager instead of hardcoding.

### Prize URLs

Update the `currentPrizes` array in `pchprizes2.py` with active sweepstakes URLs:

```python
currentPrizes = [
    "https://www.pch.com/sweepstakes/all-pch-sweeps/10k-backyard-sweepstakes",
    "https://www.pch.com/sweepstakes/all-pch-sweeps/10k-unleash-the-loot",
    # ... add/remove as needed
]
```

> **Important:** Regularly check and update this list. Expired or removed prizes can cause the script to crash.

### Cron Schedule

Add to your crontab for automated daily runs:

```bash
crontab -e
```

```
# Run at 4:00 AM and 11:00 AM daily for users 1-8
0 4,11 * * * /home/$(whoami)/selenium_env/bin/python3 /home/$(whoami)/pch-sweepstakes/pchprizes2.py 1 >> /home/$(whoami)/pchprizes2.log 2>&1
0 4,11 * * * /home/$(whoami)/selenium_env/bin/python3 /home/$(whoami)/pch-sweepstakes/pchprizes2.py 2 >> /home/$(whoami)/pchprizes2.log 2>&1
# ... repeat for users 3-8
```

For X11 display (if running with GUI):

```
0 4,11 * * * DISPLAY=:1 XAUTHORITY=/home/$(whoami)/.Xauthority /home/$(whoami)/selenium_env/bin/python3 /home/$(whoami)/pch-sweepstakes/pchprizes2.py 1
```

---

## Usage

### Manual Execution

```bash
# Activate virtual environment
source ~/selenium_env/bin/activate

# Run for a specific user (1-8)
python3 ~/pch-sweepstakes/pchprizes2.py 1

# Or make executable and run directly
chmod +x ~/pch-sweepstakes/pchprizes2.py
~/pch-sweepstakes/pchprizes2.py 1
```

### Script Arguments

| Argument | Description |
|----------|-------------|
| `1-8` | User number to process (maps to credentials in script) |

### Execution Flow

1. Script connects to Selenium Hub at `http://localhost:4444/wd/hub`
2. Hub assigns an available Chrome node
3. Script logs into PCH with user credentials
4. Navigates to each prize URL in the array
5. Clicks "Enter" button, handles advertisements
6. Submits entry form
7. Logs out and moves to next user
8. Closes browser session

---

## Project Structure

```
pch-sweepstakes/
├── README.md                    # This file
├── docs/
│   └── architecture_diagram.png # System architecture diagram
├── pchprizes2.py               # Main entry script
├── _pch_psp.py                 # Shared utility functions
├── requirements.txt            # Python dependencies
├── systemd/
│   ├── selenium-hub.service    # Hub systemd service
│   └── selenium-node.service   # Node systemd service template
└── .gitignore
```

### File Descriptions

#### `pchprizes2.py`
Main entry point. Handles:
- Command-line argument parsing (user number)
- Prize URL array management
- User credential mapping
- Entry submission loop
- Logging

#### `_pch_psp.py`
Shared utility library containing:

| Function | Purpose |
|----------|---------|
| `initializeBrowser(URL)` | Creates headless Chrome instance with anti-bot options |
| `pchLogin(driver, email, password)` | Authenticates with PCH |
| `pchLogOut(driver)` | Signs out of PCH session |
| `closeAdvert(driver)` | Handles and closes advertisement videos/popups |
| `closeAllBrowserWindows(driver)` | Closes all tabs and quits driver |
| `closeOtherBrowserWindows(driver)` | Closes secondary ad tabs |
| `scrollPageDown(driver, howMuch)` | Scrolls page by percentage |
| `switchToiFrame(driver, iFrameTitle)` | Switches context to iframe |
| `getActiveElement(active_element)` | Returns CSS selector of active element |
| `getAllChildElements(active_element)` | Lists all child elements |
| `writeMessageToFile(fileName, message)` | Appends timestamped log entries |
| `closeBrowserThenQuit(driver)` | Graceful shutdown with error handling |

---

## Script Reference

### Browser Initialization Options

The script uses the following Chrome options for anti-bot evasion:

```python
options = webdriver.ChromeOptions()
options.add_argument("--no-sandbox")
options.add_argument("--headless=new")
options.add_argument("--window-size=1920,1080")
options.add_argument("--disable-extensions")
options.add_argument("--proxy-server='direct://'")
options.add_argument("--proxy-bypass-list=*")
options.add_argument("--start-maximized")
options.add_argument('--disable-gpu')
options.add_argument('--ignore-certificate-errors')
options.add_argument('--force-device-scale-factor=1')
options.add_argument('--high-dpi-support=1')
options.add_argument("--incognito")
options.add_argument('--disable-blink-features=AutomationControlled')
options.add_experimental_option('useAutomationExtension', False)
options.add_experimental_option("excludeSwitches", ["enable-automation"])
```

### Remote WebDriver Connection

```python
driver = webdriver.Remote(
    command_executor='http://localhost:4444/wd/hub',
    options=chrome_options
)
```

---

## Anti-Bot Evasion

PCH.com implements anti-bot detection. The following techniques are used to bypass detection:

### 1. Automation Flag Removal
```python
options.add_argument('--disable-blink-features=AutomationControlled')
options.add_experimental_option('useAutomationExtension', False)
options.add_experimental_option("excludeSwitches", ["enable-automation"])
```

### 2. Custom User-Agent
```python
custom_user_agent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
options.add_argument(f"--user-agent={custom_user_agent}")
```

### 3. Incognito Mode
Prevents session data persistence between runs:
```python
options.add_argument("--incognito")
```

### 4. Human-Like Behavior
- Randomized `time.sleep()` delays between actions
- Page scrolling to trigger lazy-loaded elements
- Muted audio to prevent detection:
```python
chrome_options.add_argument("--mute-audio")
```

### 5. CAPTCHA Solving (Optional)
2captcha extension integration for handling CAPTCHA challenges:
```bash
# Create volume for extension
sudo podman volume create 2captcha
sudo podman volume mount 2captcha
sudo cp /path/to/2captcha.crx /var/lib/containers/storage/volumes/2captcha/_data/
```

---

## Scaling

### Horizontal Scaling (Add More Nodes)

```bash
# Copy and modify service file
sudo cp /etc/systemd/system/selenium-node1.service /etc/systemd/system/selenium-node2.service

# Edit: change --name selenium-node2
sudo sed -i 's/selenium-node1/selenium-node2/g' /etc/systemd/system/selenium-node2.service

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable selenium-node2
sudo systemctl start selenium-node2
```

### Vertical Scaling (More Sessions Per Node)

Add to node service `[Service]` section:
```ini
Environment=SE_NODE_MAX_SESSIONS=2
Environment=SE_NODE_OVERRIDE_MAX_SESSIONS=true
```

> **Warning:** Only increase if host has sufficient CPU/RAM. Default is 1 session per node.

---

## Monitoring & Debugging

### Selenium Grid Console
```
http://localhost:4444/ui
```

### Live Session Viewing (VNC)
```bash
# VNC Viewer
vncviewer localhost:5900
# Password: secret

# Or use NoVNC in browser:
# Click camera icon in Grid Console
```

### Container Logs
```bash
# Hub logs
sudo podman logs selenium-hub

# Node logs
sudo podman logs selenium-node1
```

### Session Management
```bash
# List active sessions
curl http://localhost:4444/wd/hub/sessions

# If a session hangs, restart the node:
sudo systemctl restart selenium-node1

# Or delete via Grid UI:
# Sessions > Click "i" under Capabilities > DELETE
```

### Script Logs
```bash
# View execution log
tail -f ~/pchprizes2.log
```

---

## Troubleshooting

### Issue: "Permission Denied" from PCH.com
**Solution:** Ensure anti-bot flags are set:
```python
options.add_argument('--disable-blink-features=AutomationControlled')
options.add_experimental_option('useAutomationExtension', False)
options.add_experimental_option("excludeSwitches", ["enable-automation"])
```

### Issue: "user-data-dir in use" error
**Solution:** Use `--incognito` mode or specify unique `--user-data-dir` per session.

### Issue: Chrome ignores window-size in headless
**Solution:** Use `driver.set_window_size()` after `driver.get()` or add:
```python
options.add_argument('--force-device-scale-factor=1')
options.add_argument('--high-dpi-support=1')
```

### Issue: Script crashes after ~40 minutes
**Solution:** The script is designed to process users in batches. Run with different user numbers:
```bash
# Terminal 1
python3 pchprizes2.py 1
# Terminal 2
python3 pchprizes2.py 2
# etc.
```

### Issue: Session not clearing after script abort
**Solution:**
```bash
# Via Grid UI: Sessions > DELETE
# Or restart node:
sudo systemctl restart selenium-node1
```

### Issue: Wayland vs X11 (for GUI display)
**Rocky Linux 9 and below:**
```bash
sudo sed -i 's/#WaylandEnable=false/WaylandEnable=false/' /etc/gdm/custom.conf
sudo systemctl restart gdm
```

**Ubuntu:**
```bash
sudo apt install -y xubuntu-desktop
```

### Issue: Wi-Fi not detected
```bash
sudo nmcli radio wifi on
sudo nmcli device set <interface> managed yes
sudo systemctl restart NetworkManager
```

---

## Using undetected-chromedriver (uc)

For sites with advanced bot detection:

```bash
# Install uc-compatible Chrome version
cd /tmp
wget https://storage.googleapis.com/chrome-for-testing-public/143.0.7499.192/linux64/chromedriver-linux64.zip
unzip chromedriver-linux64.zip
sudo cp chromedriver-linux64/chromedriver /usr/bin/chromedriver143
wget https://storage.googleapis.com/chrome-for-testing-public/143.0.7499.192/linux64/chrome-linux64.zip
sudo unzip chrome-linux64.zip -d /opt/chrome-linux64-143
```

```python
import undetected_chromedriver as uc

options = uc.ChromeOptions()
options.binary_location = '/opt/chrome-linux64-143/chrome'
options.headless = False
driver = uc.Chrome(use_subprocess=True, options=options)
```

---

## Custom Node Images with Extensions

To create a node with pre-installed Chrome extensions:

```bash
# 1. Create container with extension volume
sudo podman create --name selenium-node-custom \
    -e SE_EVENT_BUS_HOST=selenium-hub \
    -e SE_EVENT_BUS_PUBLISH_PORT=4442 \
    -e SE_EVENT_BUS_SUBSCRIBE_PORT=4443 \
    --network selenium-grid \
    --shm-size=1g \
    -v 2captcha:/2captcha \
    selenium/node-chrome:latest

sudo podman start selenium-node-custom

# 2. Install extension via VNC/NoVNC
# 3. Backup profile
sudo podman exec selenium-node-custom bash -c "tar -czf /tmp/profile.tar.gz /home/seluser/.config/google-chrome"
sudo podman cp selenium-node-custom:/tmp/profile.tar.gz .

# 4. Commit to new image
sudo podman commit selenium-node-custom my-custom-selenium-node

# 5. Update service file to use new image
```

---

## License

This project is for educational purposes only. Use at your own risk. Respect PCH.com's Terms of Service.

---

## Acknowledgments

- [Selenium](https://www.selenium.dev/) - Browser automation framework
- [Podman](https://podman.io/) - Container engine
- [Chrome for Testing](https://googlechromelabs.github.io/chrome-for-testing/) - Official Chrome builds for automation
