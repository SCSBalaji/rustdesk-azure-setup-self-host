# RustDesk Server Setup on Azure VM

A comprehensive guide for setting up a self-hosted RustDesk server on an Azure Virtual Machine to enable secure remote desktop connections between Windows 10 Hyper-V VMs and Windows 11 host machines.

## Table of Contents

- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Step 1: Set Up Azure VM](#step-1-set-up-azure-vm)
  - [Step 2: Connect to the Azure VM](#step-2-connect-to-the-azure-vm)
  - [Step 3: Set Up RustDesk Server](#step-3-set-up-rustdesk-server-hbbs--hbbr)
  - [Step 4: Allow Firewall Rules](#step-4-allow-firewall-rules)
  - [Step 5: Get the Server Key](#step-5-get-the-server-key)
- [Configuration](#configuration)
  - [Step 6: Configure RustDesk Client](#step-6-configure-rustdesk-client)
- [Usage](#usage)
  - [Step 7: Test the Connection](#step-7-test-the-connection)
- [Troubleshooting & FAQs](#troubleshooting--faqs)
  - [Step 8: Troubleshooting](#step-8-troubleshooting)
- [Contributing](#contributing)
- [License](#license)
- [Additional Resources](#additional-resources)

## Features

- **Self-hosted Remote Desktop Solution**: Complete control over your remote desktop infrastructure
- **Secure Connection**: Uses your own relay server for enhanced privacy and security
- **Cross-platform Support**: Works between Windows 10 and Windows 11 machines
- **Docker Containerization**: Easy deployment and management using Docker and Docker Compose
- **Low Resource Requirements**: Can run on a small Azure VM (B1s with 1 vCPU, 1GB RAM)

## Prerequisites

- Azure account with permission to create resources
- SSH client installed on your local machine
- Basic knowledge of Linux commands
- Docker and Docker Compose
- RustDesk client software ([Download from RustDesk website](https://rustdesk.com/))

## Installation

### Step 1: Set Up Azure VM

1. **Log in to Azure Portal**
   - Go to the [Azure Portal](https://portal.azure.com/)
   - Navigate to **Virtual Machines** → Click **Create** → **Azure Virtual Machine**

2. **Configure the VM**
   - **Image**: Select Ubuntu 24.04 LTS
   - **Size**: Choose B1s (1 vCPU, 1GB RAM, 30GB SSD)
   - **Authentication Type**: Select SSH (recommended)
   - **Username**: Set a username (e.g., `azureuser`)

3. **Create the VM**
   - Click **Review + Create** → **Create VM**
   - Once the VM is created, go to the VM's **Overview** page and copy the **Public IP**

4. **Open Ports in Azure Network Security Group (NSG)**
   - Go to Azure Portal → Your VM → Networking
   - Click "Add inbound port rule"
   - Add these three rules:
     - TCP 21115-21119 (for RustDesk communication)
     - UDP 21116-21119 (for relay server)
     - TCP 80 (for web panel, optional)
   - Click Save after adding all rules

### Step 2: Connect to the Azure VM

1. **Open PowerShell on your Windows 11 machine and SSH into the Azure VM**:
```bash
ssh -i "PATH" azureuser@<YOUR_AZURE_VM_PUBLIC_IP>
```
Replace `PATH` with the location of the SSH private key location on local machine.
Replace `<YOUR_AZURE_VM_PUBLIC_IP>` with the actual IP.

2. **Update the system**:
```bash
sudo apt update && sudo apt upgrade -y
```

3. **Install Docker and Docker Compose**:
```bash
sudo apt install -y docker.io docker-compose curl
```

4. **Enable and start Docker**:
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### Step 3: Set Up RustDesk Server (HbBS & HbBr)

1. **Create a directory for RustDesk**:
```bash
mkdir -p ~/rustdesk-server && cd ~/rustdesk-server
```

2. **Create a `docker-compose.yml` file**:
```bash
nano docker-compose.yml
```

3. **Paste the following configuration**:
```yaml
version: '3.3'
services:
  hbbs:
    image: rustdesk/rustdesk-server:latest
    container_name: hbbs
    restart: unless-stopped
    network_mode: "host"
    command: >
      /usr/bin/hbbs -r <YOUR_AZURE_VM_PUBLIC_IP>:21117
    volumes:
      - ./data:/root

  hbbr:
    image: rustdesk/rustdesk-server:latest
    container_name: hbbr
    restart: unless-stopped
    network_mode: "host"
    command: >
      /usr/bin/hbbr
    volumes:
      - ./data:/root
```
Replace `<YOUR_AZURE_VM_PUBLIC_IP>` with your Azure VM's public IP.

4. **Save the file**:
Press `CTRL+X`, then `Y`, then `Enter`.

5. **Start RustDesk containers**:
```bash
sudo docker-compose up -d
```

6. **Verify the containers are running**:
```bash
sudo docker ps
```
You should see two containers: `hbbs` and `hbbr`.

7. **Verify Open Ports**
On your Azure VM, run:
```bash
sudo netstat -tulnp | grep LISTEN
```

If netstat is not found, install the required package:
```bash
sudo apt update && sudo apt install -y net-tools
```

Re-run the command:
```bash
sudo netstat -tulnp | grep LISTEN
```

### Step 4: Allow Firewall Rules

1. **Open required ports**:
```bash
sudo ufw allow 21115:21119/tcp
sudo ufw allow 21116/udp
sudo ufw enable
```

2. **Verify the firewall status**:
```bash
sudo ufw status
```
Ensure the ports `21115-21119/tcp` and `21116/udp` are allowed.

### Step 5: Get the Server Key

1. **Check the logs for the server key**:
```bash
sudo docker logs hbbs --tail 50
```

2. **Look for a line like**:
```
[INFO] Key: "VALUE"
```
Example: `[INFO] Key: 5opIdjZJsxPbXcs7fmUKMzAkoPsh8Pbcz5yBq1r9XRM=`

This is the server key you'll use in the RustDesk client.

## Configuration

### Step 6: Configure RustDesk Client

1. **Install RustDesk on your Windows 10 Hyper-V VM**:
   - Download the RustDesk client from [https://rustdesk.com/](https://rustdesk.com/)
   - Install it on the Windows 10 VM

2. **Configure RustDesk Client**:
   - Open RustDesk on the Windows 10 VM
   - Go to **Settings** → **Network**
   - Under **ID/Relay Server**, enter:
     - **ID Server**: `<YOUR_AZURE_VM_PUBLIC_IP>`
     - **Relay Server**: `<YOUR_AZURE_VM_PUBLIC_IP>`
     - **Key**: Paste the server key from Step 5
   - Click **OK**

## Usage

### Step 7: Test the Connection

1. **Install RustDesk on your Windows 11 host machine**:
   - Download and install RustDesk from [https://rustdesk.com/](https://rustdesk.com/)

2. **Connect to the Windows 10 VM**:
   - Open RustDesk on your Windows 11 machine
   - Enter the ID of the Windows 10 VM (shown in RustDesk)
   - Click **Connect**
   - Accept the connection on the Windows 10 VM

## Troubleshooting & FAQs

### Step 8: Troubleshooting

#### Check logs
If the connection fails, check the logs:
```bash
sudo docker logs hbbs
sudo docker logs hbbr
```

#### Restart containers
Restart the RustDesk containers if needed:
```bash
sudo docker-compose down
sudo docker-compose up -d
```

#### Verify ports
Ensure the ports are open and listening:
```bash
sudo netstat -tulnp | grep LISTEN
```

#### Common Issues

1. **Connection Timeout**
   - Check that all required ports are open in both Azure NSG and UFW
   - Verify that the server key is correctly entered in the client

2. **Resources Limitations**
   - If the VM struggles due to low resources, consider upgrading to a higher-tier VM (e.g., B2s with 2GB RAM)

3. **Docker Container Issues**
   - Verify Docker is running with `sudo systemctl status docker`
   - Check container logs for specific error messages

## Contributing

Contributions to improve this guide are welcome. Please follow these steps:

1. Fork this repository
2. Create a new branch (`git checkout -b feature/improvement`)
3. Make your changes
4. Commit your changes (`git commit -m 'Add some improvement'`)
5. Push to the branch (`git push origin feature/improvement`)
6. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Additional Resources

- [RustDesk Official Website](https://rustdesk.com/)
- [RustDesk GitHub Repository](https://github.com/rustdesk/rustdesk)
- [RustDesk Server Documentation](https://github.com/rustdesk/rustdesk-server)
- [Docker Documentation](https://docs.docker.com/)
- [Azure Virtual Machines Documentation](https://docs.microsoft.com/en-us/azure/virtual-machines/)
- [Azure Network Security Groups Documentation](https://docs.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
