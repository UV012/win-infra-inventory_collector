# Windows Infrastructure Inventory Management with Ansible + MySQL + Grafana

## 📋 Repository Overview
This repository provides an **automated infrastructure asset management solution** that collects hardware and software facts from Windows servers via Ansible, stores the data in MySQL, and visualizes it through Grafana dashboards. Perfect for IT operations teams needing real-time visibility into their Windows infrastructure without manual spreadsheets or expensive CMDB solutions.

## 🎯 Business Value
| Challenge	                                 |  Solution                                |
| ------------------------------------------ | -----------------------------------------|
| ❌ Manual inventory tracking               |	✅ Fully automated fact gathering       |
| ❌ Outdated asset documentation	           | ✅ Real-time data on schedule             |
| ❌ Disparate tools for monitoring/CMDB	   | ✅ Unified stack (Ansible → MySQL → Grafana)   |
| ❌ No historical tracking	                 | ✅ Database enables trend analysis           |
| ❌ Windows server visibility gaps	         | ✅ Standardized cross-platform collection    |

## 🏛️ Architecture
```
┌─────────────────┐      WinRM       ┌─────────────────┐
│   Ubuntu VM     │ ◄──────────────► │   Windows VM    │
│ (Control Node)  │                  │  (Target Node)  │
│   • Ansible     │                  │  • WinRM        │
│   • Python3     │                  │  • Admin access │
└────────┬────────┘                  └─────────────────┘
         │
         │ MySQL INSERT
         │ (facts: CPU, RAM, OS, IP)
         ▼
┌─────────────────┐      Read-Only    ┌─────────────────┐
│     MySQL       │ ◄─────────────── │     Grafana     │
│  (Ubuntu VM)    │                  │   (Ubuntu VM)   │
│ • asset_db      │                  │ • Table panel   │
│ • windows_assets│                  │ • Live queries  │
└─────────────────┘                  └─────────────────┘
```

## 🔄 Data Flow:
1. Ansible connects to Windows hosts via WinRM (basic authentication)
2. Gathers facts like hostname, OS, version, architecture, CPU cores, RAM (MB), IPv4 address
3. Transforms data into structured format
4. Inserts records into MySQL asset_db.windows_assets table
5. Grafana queries MySQL and displays in real-time dashboard

## 📂 Repository Structure
```
.
├── inventory.ini               # Windows hosts + connection vars
├── gather_facts.yml           # Main playbook: collect → insert
└── README.md                  # You are here
```

## ⚙️ Prerequisites
### Control Node (Ubuntu)
* Ansible 2.9+
* Python 3.6+
* MySQL client libraries
* `community.mysql` collection
```
# Update package index
sudo apt update

# Install Python 3.6+ and pip
sudo apt install -y python3 python3-pip

# Install Ansible (latest stable from apt)
sudo apt install -y ansible

# Verify Ansible version (should be >= 2.9)
ansible --version

# Install MySQL client libraries
sudo apt install -y default-libmysqlclient-dev python3-dev

# Install community.mysql collection via Ansible Galaxy
ansible-galaxy collection install community.mysql
```
### Windows Target Nodes
* **WinRM enabled and configured**
```
winrm quickconfig -force
Set-Item WSMan:\localhost\Service\Auth\Basic $true
Set-Item WSMan:\localhost\Service\AllowUnencrypted $true
```
* **Firewall rule for port 5985/5986**
```
Enable-NetFirewallRule -Name "WINRM-HTTP-In-TCP"
```
* **User in Administrators group**
```
# Add your user to Administrators group
net localgroup Administrators <YourUsername> /add
```

### Database
* MySQL 8.0
* Database: `asset_db`
* Table: `windows_assets`

## 🛠️ Configuration
### 1. Inventory File (inventory.ini)
```
[windows]
winvm   ansible_host=192.X.Y.Z
winserver ansible_host=192.X.Y.Z

[windows:vars]
ansible_user=Admin
ansible_password=YourSecurePassword
ansible_connection=winrm
ansible_winrm_transport=basic
ansible_winrm_server_cert_validation=ignore
```

### 2. Database Setup
Login:
```
sudo mysql
```

Create DB + user:
```
CREATE DATABASE asset_db;

CREATE USER 'user'@'localhost' IDENTIFIED BY 'YourPassword';
GRANT ALL PRIVILEGES ON asset_db.* TO 'user'@'localhost';
FLUSH PRIVILEGES;
```

Create table:
```
USE asset_db;

CREATE TABLE windows_assets (
    id INT AUTO_INCREMENT PRIMARY KEY,
    hostname VARCHAR(100) NOT NULL,
    os_name VARCHAR(255),
    os_version VARCHAR(50),
    architecture VARCHAR(50),
    cpu_cores INT,
    total_ram_mb INT,
    ipv4_address VARCHAR(15),
    collected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_hostname (hostname)
);
```

### 3. Database Credentials (in playbook)
Update `gather_facts.yml` with your MySQL credentials:
```
vars:
    db_name: asset_db
    db_user: your_user
    db_password: your_secure_password
```

## 🚀 Usage
### Run the full pipeline:
```
ansible-playbook -i inventory.ini gather_facts.yml
```

### Schedule with Cron (daily collection):
```
0 2 * * * /usr/bin/ansible-playbook -i /path/inventory.ini /path/gather_facts.yml
```

## 📊 Grafana Visualization
### Dashboard Features:
* **Windows Assets Table:** Hostname, OS, CPU, RAM, IP, last collection time
* **Optional:** Add panels for OS version distribution, RAM/CPU heatmaps

### Import Steps:
1. Login to Grafana (default: http://localhost:3000)
2. **Configuration → Data Sources → Add MySQL**
* Host: `localhost:3306`
* Database: `asset_db`
* User: user
* Password: YourPassword
* Add Dashboard
* Add panel
* Add visualization

**Sample Query for Table Panel:**
```
SELECT
  hostname,
  os_name,
  os_version,
  architecture,
  cpu_cores,
  total_ram_mb,
  ipv4_address,
  collected_at
FROM windows_assets
ORDER BY collected_at DESC;
```

## 🧪 Testing & Validation
| Component	| Test Command	| Expected |
| ---------- | --------------| ------------ |
| WinRM	| `ansible winvm -m win_ping -i inventory.ini`	| `"pong"` |
| MySQL connection	| `mysql -u user -p -e "SELECT 1"`	| `1` |
| Playbook check	| `ansible-playbook gather_facts.yml --check`	| No changes |

## 📈 Future Enhancements
* Add Linux support (dual inventory)
* Include disk capacity facts
* Historical trend dashboards in Grafana
* Slack/Teams alerts for new servers
* ServiceNow CMDB integration
* Dockerized Ansible control node

## 📊 Evidence of Expertise
### This project demonstrates:

| Skill	| Evidence |
| --------- | -------- |
| Infrastructure as Code |	Full Windows server inventory automated via Ansible |
| DevOps Integration	| Ansible → MySQL → Grafana pipeline |
| Cross-platform	| Linux control node managing Windows targets |
| Database design	| Normalized schema, efficient queries |
| Visualization	| Grafana dashboards for operations teams |
| Problem solving |	Solved the "spreadsheet CMDB" problem |

## 📄 License
MIT
