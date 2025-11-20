# ENVIRONMENT SETUP SUMMARY

Overview
This document outlines the required tools, version, configurations, and other environment variables needed to run the project’s data pipeline locally. Once set up, this should allow the team to fully replicate the pipeline whether it be running ingestion scripts or executing dbt models used during development

Required Software
First, ensure the following tools are installed/set up:
* WSL (Ubuntu): Linux environment for Windows 
* Python: ingestion scripts
* Git: version control and project repository
* Docker: containerized services
* DBeaver: SQL database GUI
* Metabase: data visualization/dashboards
* uv: Python env & package manager

WSL (Ubuntu)
For setting up a Linux environment on Windows machines and running other tools (Ex. Docker)
* Install here: learn.microsoft.com/en-us/windows/wsl/install
* Quick install by running this command in Windows Powershell as Admin:

wsl --install

* Once rebooted, open Ubuntu from Start Menu and set a username/password
* Update base system by running this command in WSL (Ubuntu):


sudo apt update && sudo apt upgrade -y

Python
For running ingestion scripts and data extraction
* If not yet installed, go to python.org/downloads
* Version: 3.9 or above (project tested on: 3.9)
* Installing through Linux (Ubuntu/Debian)


sudo apt update
sudo apt install -y python3 python3-venv python3-pip
python3 --version

* Installing through Homebrew (macOS)


brew install python
python3 --version

* Verify installation by running the following command on shell:


python --version

* Recommendation: Create a virtual environment (venv or uv) on your machine. Doing so ensures:
   * Consistent package versions
   * Clean isolation from system Python installed
   * Smooth execution of the DLT ingestion scripts
Git
For cloning project repository
* If not yet installed, go to git-scm.com/install
* Version: 2.42 or above (project tested on 2.43.0)
* Install through Linux (Ubuntu/Debian) then verify


sudo apt install git -y
git --version

* Install through Homebrew (macOS) then verify


brew install git
git --version

* Configure Git (name, email, default branch, line endings)


  git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
git config --global core.autocrlf input   # macOS/Linux
# On Windows (PowerShell): git config --global core.autocrlf true

* Clone the repository


git clone https://github.com/luciten/ftw-de-capstone-project.git
cd ftw-de-capstone-project

Docker
For running containerized services such as ClickHouse (DB) and Metabase
* There are several options for installing
* First is installing Docker Desktop from here: docs.docker.com/desktop/setup/install/windows-install/
* For WSL (Ubuntu), go to Docker Desktop > Settings > Resources > WSL integration, Ubuntu distro to be toggled ON
* Verify connection:


docker run hello-world

* If not installing Docker Desktop, next option is running Native Docker Engine inside WSL
* Enable systemd in WSL:


sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF

* Then in Windows Powershell:


wsl --shutdown

* Reopen WSL (Ubuntu), and install Docker Engine


# prerequisites
sudo apt install -y ca-certificates curl gnupg


# keyring
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc


# repo
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo ${UBUNTU_CODENAME:-$VERSION_CODENAME}) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

* Enable and use Docker without sudo 


sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# pick up new group without reopening the terminal:
newgrp docker
docker run hello-world
(For permission denied error on /var/run/docker.sock, try opening a new terminal or running: newgrp docker. If still not running, check: systemctl status docker)


* For macOS, install Docker here: docs.docker.com/desktop/setup/install/mac-install/
* Verify installation:


docker run hello-world

DBeaver
For querying and exploring/managing database
* Windows: install from dbeaver.io/download/
* macOS: install from dbeaver.io/download/ or through Homebrew:


brew install --cask dbeaver-community

uv
For setting up Python environment and installing required packages quickly and cleanly


* Installing through Linux/Homebrew (universal one-liner)

curl -LsSf https://astral.sh/uv/install.sh | sh

   * Verify installation:


uv --version

   * If the command was not found, restart your terminal or add ~/.local/bin to your PATH. To add, run the following commands and verify installation after:


   * Linux


echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

   * Homebrew


echo 'export PATH="/opt/homebrew/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

Project and Environment Setup


   1. Clone the project repository from Git


git clone https://github.com/luciten/ftw-de-capstone-project
cd ftw-de-capstone-project

   2. Create and activate a virtual environment using uv
Doing so ensures: consistent package versions, clean isolation from system Python installed, smooth execution of the DLT ingestion scripts
Linux/Homebrew


uv venv
source .venv/bin/activate 

Windows


uv venv
source .venv\Scripts\activate

   3. Install dependencies/packages to be used by Python


uv pip install -r requirements.txt

   4. When done with running Python scripts, exit uv:


deactivate



Other Utilities
dbt/Clickhouse
After data ingestion/extraction, dbt was used for data transformation and quality checks


dbt was run using Docker, which was used with the following setup:

profiles.yml
default:
  outputs:
    local:
      type: clickhouse
      schema: default
      host: clickhouse
      port: 9000
      user: default
      password: ""
      secure: false
    
    remote:
      type: clickhouse
      schema: mart_grp4
      host: 54.87.106.52
      port: 8123
      user: ftw_grp4
      password: Pika@025!FTW


      threads: 4
      secure: false
  
  target: "{{ env_var('DBT_TARGET', 'remote') }}"

   * dbt uses two possible profiles: (target can be set in .env)
   * Local: connect to ClickHouse running in Docker
   * Remote: connects to shared remote ClickHouser server


dbt_project.yml

name: 'capstone_project'
version: '1.0.0'
config-version: 2


profile: 'default'


model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]


target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"
log-path: "logs"


models:
  capstone_project:
    clean:
      +schema: clean_grp4
      +materialized: table
    mart:
      +schema: mart_grp4
      +materialized: view

.env


# ClickHouse container info
CLICKHOUSE_CONTAINER=clickhouse
CLICKHOUSE_HTTP_PORT=8123
CLICKHOUSE_TCP_PORT=9000


# ClickHouse credentials (needed by dbt & DLT), replace if applicable
REMOTE_CH_HOST=54.87.106.52             
REMOTE_CH_USER=ftw_grp4          
REMOTE_CH_PASS=Pika@025!FTW      


# DLT
DLT_CONTAINER=dlt

# dbt
DBT_CONTAINER=dbt
DBT_TARGET=remote                 # match profiles.yml target (local or remote)

# Metabase
METABASE_CONTAINER=metabase
METABASE_PORT=3001

ClickHouse connection (set in config files above and connected via Dbeaver)
   * host: 54.87.106.52                        
   * port: 8123
   * database/schema: default                        
   * username: ftw_grp4
   * password: Pika@025!FTW
  
Troubleshooting
Data not loading
   * Check: Python ingestion scripts → ClickHouse raw tables
   * Tool: DBeaver to inspect raw layer
Transformations failing
   * Check: dbt logs → staging/mart table creation
   * Tool: dbt test, dbt run --debug
Dashboards showing wrong data
   * Check: ClickHouse mart tables → Metabase queries
   * Tool: DBeaver to validate mart layer data
Performance problem
   * Check: ClickHouse query logs → table partitioning → indexes
   * Tool: ClickHouse system tables, EXPLAIN queries
