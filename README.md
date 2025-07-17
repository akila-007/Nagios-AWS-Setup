# Nagios-AWS-Setup
Step-by-step guide to set up Nagios Core on AWS EC2 and add a monitored EC2 host using NRPE

Nagios Core - AWS EC2 Setup and Host Monitoring

This guide and GitHub repository content explains how to install Nagios Core on an AWS EC2 instance and configure it to monitor another EC2 instance using NRPE.

🔍 What is Nagios Core?

Nagios Core is an open-source monitoring solution that allows you to track the availability and performance of systems, services, and network infrastructure. It works with plugins and agents to collect metrics and generate alerts.

Key Components:

Nagios Core: Central server that checks status

Plugins: Scripts used to check host or service states

NRPE (Nagios Remote Plugin Executor): Allows Nagios to execute plugins on remote Linux hosts

☁️ What You Will Do Practically

Launch 2 AWS EC2 Ubuntu servers (One for Nagios, One for Monitored Host)

Configure firewall (Security Group)

Install Nagios Core on one instance

Install NRPE on second instance

Add monitored host to Nagios config

🧰 Step-by-Step Setup

✅ Step 1: Create AWS EC2 Security Group

Open AWS EC2 Dashboard.

Create a new Security Group (or modify existing one)

Add inbound rules:

SSH (port 22) - from your IP

HTTP (port 80) - from anywhere (for web access)

TCP (port 5666) - from Nagios server IP (for NRPE)

✅ Step 2: Install and Configure Nagios Core (on EC2 Instance 1)

Update and Install Dependencies

sudo apt update && sudo apt upgrade -y
sudo apt install -y apache2 php libapache2-mod-php build-essential unzip \
    gcc make libgd-dev libssl-dev daemon wget libmcrypt-dev \
    libssl-dev bc gawk dc libwrap0-dev snmp libnet-snmp-perl gettext

Create Nagios User

sudo useradd nagios
sudo groupadd nagcmd
sudo usermod -a -G nagcmd nagios
sudo usermod -a -G nagcmd www-data

Download and Compile Nagios Core

cd /tmp
wget https://assets.nagios.com/downloads/nagioscore/releases/nagios-4.4.14.tar.gz
sudo tar -zxvf nagios-4.4.14.tar.gz
cd nagios-4.4.14
sudo ./configure --with-command-group=nagcmd
sudo make all
sudo make install
sudo make install-commandmode
sudo make install-init
sudo make install-config
sudo make install-webconf

Create Web UI User

sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin

Start Nagios and Apache

sudo a2enmod cgi
sudo systemctl start apache2
sudo systemctl enable apache2
sudo systemctl start nagios
sudo systemctl enable nagios

Now visit: http://<Nagios-Server-Public-IP>/nagios

✅ Step 3: Install NRPE and Plugins (on EC2 Instance 2)

Update and Install Dependencies

sudo apt update && sudo apt upgrade -y
sudo apt install -y autoconf gcc libmcrypt-dev make libssl-dev \
    snmp libnet-snmp-perl gettext

Download and Compile NRPE

cd /tmp
wget https://github.com/NagiosEnterprises/nrpe/releases/download/nrpe-4.1.0/nrpe-4.1.0.tar.gz
sudo tar -xzf nrpe-4.1.0.tar.gz
cd nrpe-4.1.0
sudo ./configure --enable-command-args
sudo make check_nrpe
sudo make install-plugin
sudo make install-daemon
sudo make install-daemon-config
sudo make install-init

Configure NRPE

sudo nano /usr/local/nagios/etc/nrpe.cfg

Find and modify:

allowed_hosts=127.0.0.1,<Nagios_Server_Private_IP>

Optional: Add command definitions:

command[check_disk]=/usr/lib/nagios/plugins/check_disk -w 20% -c 10% -p /
command[check_load]=/usr/lib/nagios/plugins/check_load -w 5.0,4.0,3.0 -c 10.0,6.0,4.0

Start NRPE

sudo systemctl start nrpe.service
sudo systemctl enable nrpe.service

✅ Step 4: Add Remote Host in Nagios

On your Nagios server, create new host config:

sudo nano /usr/local/nagios/etc/servers/server01.cfg

Example config:

define host {
  use             linux-server
  host_name       server01
  alias           Ubuntu Target Server
  address         54.159.156.31
  max_check_attempts 5
  check_period    24x7
  notification_interval 30
  notification_period  24x7
}

define service {
  use                 generic-service
  host_name           server01
  service_description Check NRPE Version
  check_command       check_nrpe
}

define service {
  use                 generic-service
  host_name           server01
  service_description Check Root Disk
  check_command       check_nrpe!check_disk
}

define service {
  use                 generic-service
  host_name           server01
  service_description Check Load
  check_command       check_nrpe!check_load
}

Edit Nagios config to include new hosts

sudo nano /usr/local/nagios/etc/nagios.cfg

Uncomment or add:

cfg_dir=/usr/local/nagios/etc/servers

Restart Nagios

sudo systemctl restart nagios

✅ Verify

Go to http://<Nagios-Server-IP>/nagios

Login with nagiosadmin

Click "Hosts" and "Services" to view the remote host status


