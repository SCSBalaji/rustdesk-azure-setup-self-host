# RustDesk Server Setup on Azure VM

This guide will walk you through the process of setting up a RustDesk server on an Azure VM, allowing you to connect securely from your Windows 10 Hyper-V VM and Windows 11 host machine.

## Step 1: Set Up Azure VM

### Log in to Azure Portal
1. Go to the [Azure Portal](https://portal.azure.com/).
2. Navigate to **Virtual Machines** → Click **Create** → **Azure Virtual Machine**.

### Configure the VM
- **Image**: Select Ubuntu 24.04 LTS.
- **Size**: Choose B1s (1 vCPU, 1GB RAM, 30GB SSD).
- **Authentication Type**: Select SSH (recommended).
- **Username**: Set a username (e.g., `azureuser`).

### Create the VM
- Click **Review + Create** → **Create VM**.
- Once the VM is created, go to the VM’s **Overview** page and copy the **Public IP**.

### Open Ports in Azure Network Security Group (NSG)
- Go to Azure Portal → Your VM → Networking.
- Click "Add inbound port rule".
- Add these three rules:
  - TCP 21115-21119 (for RustDesk communication)
  - UDP 21116-21119 (for relay server)
  - TCP 80 (for web panel, optional)
- Click Save after adding all rules.

## Step 2: Connect to the Azure VM

### Open PowerShell on your Windows 11 machine and SSH into the Azure VM:
```bash
ssh -i "PATH" azureuser@<YOUR_AZURE_VM_PUBLIC_IP>
```
Replace `PATH` with the location of the SSH private key location on local machine.
Replace `<YOUR_AZURE_VM_PUBLIC_IP>` with the actual IP.

### Update the system:
```bash
sudo apt update && sudo apt upgrade -y
```

### Install Docker and Docker Compose:
```bash
sudo apt install -y docker.io docker-compose curl
```

### Enable and start Docker:
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

## Step 3: Set Up RustDesk Server (HbBS & HbBr)

### Create a directory for RustDesk:
```bash
mkdir -p ~/rustdesk-server && cd ~/rustdesk-server
```

### Create a `docker-compose.yml` file:
```bash
nano docker-compose.yml
```

### Paste the following configuration:
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
Replace `<YOUR_AZURE_VM_PUBLIC_IP>` with your Azure VM’s public IP.

### Save the file:
Press `CTRL+X`, then `Y`, then `Enter`.

### Start RustDesk containers:
```bash
sudo docker-compose up -d
```

### Verify the containers are running:
```bash
sudo docker ps
```
You should see two containers: `hbbs` and `hbbr`.

### Verify Open Ports
On your Azure VM, run:
```sh
sudo netstat -tulnp | grep LISTEN
```

If netstat is not found, install the required package:
```sh
sudo apt update && sudo apt install -y net-tools
```

Re-run the code
```sh
sudo netstat -tulnp | grep LISTEN
```

## Step 4: Allow Firewall Rules

### Open required ports:
```bash
sudo ufw allow 21115:21119/tcp
sudo ufw allow 21116/udp
sudo ufw enable
```

### Verify the firewall status:
```bash
sudo ufw status
```
Ensure the ports `21115-21119/tcp` and `21116/udp` are allowed.

## Step 5: Get the Server Key

### Check the logs for the server key:
```bash
sudo docker logs hbbs --tail 50
```

### Look for a line like:
```bash
[INFO] Key: "VALUE"
```
Example ```[INFO] Key: 5opIdjZJsxPbXcs7fmUKMzAkoPsh8Pbcz5yBq1r9XRM=```

This is the server key you’ll use in the RustDesk client.

## Step 6: Configure RustDesk Client

### Install RustDesk on your Windows 10 Hyper-V VM:
- Download the RustDesk client from [https://rustdesk.com/](https://rustdesk.com/).
- Install it on the Windows 10 VM.

### Configure RustDesk Client:
1. Open RustDesk on the Windows 10 VM.
2. Go to **Settings** → **Network**.
3. Under **ID/Relay Server**, enter:
   - **ID Server**: `<YOUR_AZURE_VM_PUBLIC_IP>`
   - **Relay Server**: `<YOUR_AZURE_VM_PUBLIC_IP>`
   - **Key**: Paste the server key from Step 5.
4. Click **OK**.

## Step 7: Test the Connection

### Install RustDesk on your Windows 11 host machine:
- Download and install RustDesk from [https://rustdesk.com/](https://rustdesk.com/).

### Connect to the Windows 10 VM:
1. Open RustDesk on your Windows 11 machine.
2. Enter the ID of the Windows 10 VM (shown in RustDesk).
3. Click **Connect**.
4. Accept the connection on the Windows 10 VM.

## Step 8: Troubleshooting

### Check logs:
If the connection fails, check the logs:
```bash
sudo docker logs hbbs
sudo docker logs hbbr
```

### Restart containers:
Restart the RustDesk containers if needed:
```bash
sudo docker-compose down
sudo docker-compose up -d
```

### Verify ports:
Ensure the ports are open and listening:
```bash
sudo netstat -tulnp | grep LISTEN
```

## Final Notes
- **Resource Optimization**: If the VM struggles due to low resources, consider upgrading to a higher-tier VM (e.g., B2s with 2GB RAM).
- **Security**: Set up a RustDesk key for authentication (optional but recommended).

By following these steps, you should have a fully functional RustDesk server running on an Azure VM, accessible from your Windows 10 Hyper-V VM and Windows 11 host machine. Let me know if you encounter any issues! 🚀
