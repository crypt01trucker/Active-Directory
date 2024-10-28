# Setting Up an On-Premises Active Directory Home Lab with Splunk

## Objective
This guide aims to create a on-premises environment consisting of an Active Directory, a domain-joined Windows 10 VM, a Splunk server for log collection, and Kali Linux VM to simulate a brute-force attack on the RDP session of the Windows 10 VM. While this guide uses **Hyper-V** for virtualization, you can also choose **VMware** or **VirtualBox** to complete this setup.

## Tools and Resources Required
- **Hyper-V** (or VMware/VirtualBox)
- **Splunk** (as the SIEM)
- **Kali Linux** (attacker)
- **Windows 10** (Victim machine)
- **Windows Server** (Active Directory)
- **Password Manager** (to securely store credentials for multiple accounts)

### 1. Download the Necessary ISO Files

Before we begin setting up the VMs, download the required ISO files and virtual hard disks and for example save them in your `C:\ISO` folder:

- [Windows Server 2022](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022)
- [Windows 10](https://www.microsoft.com/en-us/software-download/windows10)
- [Kali Linux](https://www.kali.org/get-kali/#kali-virtual-machines)
- [Ubuntu Server (optional for later setups)](https://ubuntu.com/download/server)

### 2. Installing Hyper-V

**Hyper-V** is a built-in hypervisor available on Windows that lets you create and manage VMs. If you don’t have it installed, follow these steps:

1. Press `Windows + R` to open the **Run** box.
2. Type `appwiz.cpl` and hit Enter to open the **Control Panel**.
3. On the left, click **Turn Windows features on or off**.
4. Scroll down and select **Hyper-V**.
5. Click **OK** and restart your computer if prompted.

### 3. Create Virtual Machines in Hyper-V

#### Windows 10 Victim VM

1. Open **Hyper-V Manager** from the Start menu.
2. Click **Action** > **New** > **Virtual Machine**.
3. Name the VM (e.g., "Windows10-Victim").
4. Choose **Generation 2** for the VM generation.
5. Assign at least 2048 MB (2 GB) of memory.
6. Leave the network connection as **Not Connected** for the initial setup.
7. Choose **Create a virtual hard disk** and assign at least 50 GB of storage.
8. Select **Install an operating system from a bootable image file (.iso)**.
9. Click **Browse** and select the **Windows 10** ISO file you downloaded earlier.
10. Click **Finish** to create the VM.

#### Windows Server 2022 VM (Active Directory)

1. Follow the same steps as creating the **Windows 10** VM.
2. Name the VM (e.g., "AD-Server").
3. Choose **Generation 2** for the VM generation.
4. Allocate at least **4096 MB (4 GB)** of memory for the VM.
5. Choose **Create a virtual hard disk** with at least **50 GB** of storage.
6. Select the **Windows Server 2022** ISO file.
7. During the installation, choose the second option: **Desktop Experience** (this includes the GUI).
8. On the first startup, set the **Administrator password**. Save it securely in your password manager to avoid losing access.

#### Kali Linux VM (Attacker)

1. Name the VM (e.g., "Kali-Attacker").
2. Choose **Generation 2** for the VM generation.
3. Allocate at least **2048 MB (2 GB)** of memory.
4. For the network connection, choose the **Default Switch** for the initial setup.
5. Instead of an ISO, **extract the Kali Linux virtual hard disk** from the download.
6. When creating the VM, choose the extracted **Kali virtual hard drive**.
7. Click **Finish** to complete the VM setup.
8. **Connect** to the VM and log in with the default credentials:
   - Username: `kali`
   - Password: `kali`
9. Open the terminal and update the system:
```bash
sudo apt-get update && apt-get upgrade -y
```

#### Ubuntu Server VM (Splunk Lab)

1. **Name the VM** (e.g., "Splunk").
2. Allocate at least **8192 MB (8 GB)** of memory.
3. Choose the **Default Switch** for networking for the initial setup.
4. Create a **100 GB virtual hard disk**.
5. Select the **Ubuntu Server ISO** for installation.
6. Click **Finish** to complete the VM setup.
7. **Connect** to the VM and follow the installation steps:
   - Choose **English**, then accept all default settings.
   - Set a **name**, **server name**, **username**, and **password**.
   - No need to enable SSH for this lab.
8. If prompted, **reboot**.
9. Once rebooted, update the server:
```bash
sudo apt-get update && apt-get upgrade -y
```
10. If you get a permission error, switch to root user:
```bash
sudo su
```
    After updates, exit the root user:
```bash
exit
```

## Creating a NAT Virtual Switch

We'll create a Virtual Switch and configure it with NAT (Network Address Translation) so that our VMs can access the internet. You can either create the virtual switch through Hyper-V Manager and continue the NAT configuration in Powershell or set it up entirely using PowerShell.

For this guide, I will be using the **PowerShell method**.

### Create the NAT Virtual Switch in PowerShell

Run the following command in **PowerShell (Admin mode)** to create the virtual network and assign it a name:

```powershell
New-NetNat -Name "NATNetwork" -InternalIPInterfaceAddressPrefix 192.168.200.0/24
```
This command creates a NAT virtual switch with the name `NATNetwork` and uses the IP range `192.168.200.0/24`.

### Find the Network Interface Index

After creating the virtual switch, we need to find the interface index to configure the NAT gateway:

```powershell
Get-NetAdapter
```

This will display all virtual switches. Look for the adapter name you just created and locate the **ifIndex** (Interface Index).

### Assign an IP Address to the Virtual Switch

Once you have the interface index, assign an IP address to your virtual switch, which will serve as the default gateway for your VMs. For example, if the **ifIndex** is `54`, the command would be:

```powershell
New-NetIPAddress -IPAddress 192.168.200.1 -PrefixLength 24 -InterfaceIndex 54
```

- `IPAddress`: This will be the default gateway (e.g., `192.168.200.1`).
- `PrefixLength`: This represents the subnet mask (`/24` or `255.255.255.0`).
- `InterfaceIndex`: The interface index from the `Get-NetAdapter` command.

### Finalize NAT Configuration

Run the following command to set up the NAT network:

```powershell
New-NetNat -Name "NATNetwork" -InternalIPInterfaceAddressPrefix 192.168.200.0/24
```

### Assign the NAT Virtual Switch to VMs in Hyper-V

1. Go back to **Hyper-V Manager** and assign the virtual network you just created (the **NAT Virtual Switch**) to each of your VMs.
2. Sign into each VM and configure a static **IPv4** address so that it's within the same range as the virtual switch (e.g., `192.168.200.x`).
3. This ensures that all VMs are on the same network, allowing them to communicate with each other and access the internet through the NAT gateway.

### Configure Kali Linux Static IP

1. Right-click on the network interface in the top-right corner of your screen.
2. Click on **Edit Connections**.
3. Choose **Wired Connection 1** and click on the gear icon.
4. Navigate to the **IPv4** tab.
5. In the **Method** section, select **Manual**.
6. Click on **Add** and input the IP address.
7. Set the network mask to **24**.
8. Input the default gateway you configured when setting up the virtual switch.
9. In the **DNS Server** field, enter Google's DNS IP: `8.8.8.8`.
10. Press **Save**.



### Configure Ubuntu Server Static IP
To configure a static IP address on Ubuntu server we need to configure the yaml file that is stored in the '/etc/nelplan' directory. First let's navigate to that directory.
```bash
cd /etc/netplan
```
This is where you'll find YAML files that control your network configuration. These files usually have a `.yaml` extension, and their names may vary depending on your installation.
To find the network configuration file you need to edit, use the `ls` command.

```bash
ls -la
```

- **ls**: Lists files and directories.
- **-l**: Shows detailed info (permissions, owner, size).
- **-a**: Includes hidden files (files starting with .).

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.200.20/24]
      nameservers: 
        addresses: [8.8.8.8]
      routes:
        - to: default
          via: 192.168.200.1
```
- **`eth0`**: This is the name of the network interface. Make sure to replace this with your actual interface name if it's different. You can find the interface name using the `ip addr` command.
- **`dhcp4: no`**: Disables DHCP, so the system won't try to automatically assign an IP address.
- **`addresses`**: This sets the static IP and subnet mask. In this example, the IP is `192.168.200.20` with a subnet of `/24` (which translates to `255.255.255.0`).
- **`nameservers`**: This sets the DNS server. In this example, we’re using Google’s DNS (`8.8.8.8`).
- **`routes`**: This defines the default gateway for the network (`192.168.200.1`).

> **Tip**: Be careful with indentation. YAML files require tabs, not spaces, for indentation.

## Installing Splunk on Ubuntu Server
1. Download Splunk package using wget:
   ```bash
   wget -O splunk.deb "https://download.splunk.com/products/splunk/releases/9.3.1/linux/splunk-9.3.1-0b8d769cb912-linux-2.6-amd64.deb"
   ```
   
2. Install Splunk:
   ```bash
   sudo dpkg -i splunk.deb
   ```
3. Navigate to Splunk Directory and List All Files.
4. ```bash
   cd /opt/splunk
   ls -l
   ```
   These commands will show you the user and group associated with the Splunk Files, confirming that Splunk only has the necessary privileges.

3. Start Splunk and enable boot-start:
   ```bash
   sudo /opt/splunk/bin/splunk start
   sudo /opt/splunk/bin/splunk enable boot-start -user splunk
   ```

## Setting Static IP Addresses for Windows Target VM and Windows Server
### 1. Open Network Connections
- Right-click on the **Windows icon** at the bottom left corner of your screen.
- Click on **Run**.
- Type `ncpa.cpl` and press **Enter**.

### 2. Configure Network Adapter
- Right-click on the **network adapter** and select **Properties**.
- Click on **Internet Protocol Version 4 (TCP/IPv4)** and then click on **Properties**.

### 3. Set Static IP Address for Windows Target VM
- Select **Use the following IP address**:
  - **IP address**: Set an IP address within the virtual switch range you created, e.g., `192.168.200.10`
  - **Subnet mask**: `255.255.255.0` (/24)
  - **Default gateway**: The gateway IP set when you created the virtual switch, e.g., `192.168.200.1`
- Set DNS server addresses:
  - **Preferred DNS server**: `192.168.200.100` (Windows Server IP, for domain joining)
  - **Alternate DNS server**: `8.8.8.8` (Google DNS)
- Click **OK** to save the settings.

### 4. Set Static IP Address for Windows Server VM
- Follow the same steps as above.
- Assign the following IP settings:
  - **IP address**: Set an IP address within the virtual switch range, e.g., `192.168.200.100`
  - **Subnet mask**: `255.255.255.0` (/24)
  - **Default gateway**: `192.168.200.1` (same as the target VM)
- Set DNS server addresses:
  - **Preferred DNS server**: `192.168.200.100` (The server's own IP)
- Click **OK** to save the settings.

### 5. Verify Configuration
- After setting the static IPs, open a command prompt and ping a website (e.g., `ping google.com`) to check if the internet connection is working.
- Ping from the Windows target VM to the Windows server to ensure they can communicate. Note that by default, ping may be disabled on the Windows target VM.

## Installing Sysmon and Splunk Forwarder on Windows 10 and Windows Server
### 1. Install Splunk Universal Forwarder
1. **Download Splunk Universal Forwarder**:
   - Go to [splunk.com](https://www.splunk.com) and sign in with the account you created earlier.
   - Navigate to **Products > Free Trials and Downloads**.
   - Scroll down to **Universal Forwarder** and click on **Get my free download**.
   - Choose the **Windows 64-bit** version.

2. **Run the Installer**:
   - Once the download is complete, run the installer.
   - Check the box to accept the license agreement and choose **On-Premises Splunk**.
   - Enter a username (e.g., `admin`) and click on **Generate random password**.
   - For **Deployment Server**, leave it blank as we don’t have one for this lab.
   - For **Receiving Indexer**, enter the IP address of your Ubuntu Splunk VM with the default port `9997`.
   - Click on **Install** to complete the setup.
  
### 2. Install Sysmon
1. **Download Sysmon**:
   - Go to the [Sysinternals website](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) and download Sysmon.

2. **Download Sysmon Config File**:
   - Get the Sysmon config XML file from [OLAF GitHub](https://github.com/olafhartong/sysmon-modular).
   - Scroll down and select `sysmonconfig.xml`.
   - Click on **Raw**, right-click, select **Save as**, and download it into your download folder.

3. **Extract and Install Sysmon**:
   - Extract the Sysmon zip file first.
   - Run PowerShell as an administrator and change to the download directory:
     ```powershell
     cd path\to\download\directory
     ```
   - Run the following command to install Sysmon with the config file:
     ```powershell
     .\Sysmon64.exe -i ..\sysmonconfig.xml
     ```
   - Note: The `..\sysmonconfig.xml` indicates that the config file is in the parent directory, not the same folder as the extracted `Sysmon64.exe` file.

### 3. Configure Splunk Forwarder
1. **Modify Inputs.conf File**:
   - Navigate to `C:\Program Files\SplunkUniversalForwarder\etc\system\default`.
   - Copy the `inputs.conf` file to the local directory: `C:\Program Files\SplunkUniversalForwarder\etc\system\local`.

2. **Edit Inputs.conf File**:
   - Open Notepad as an administrator.
   - Open the `inputs.conf` file in the local directory.
   - Erase everything and paste the following configuration:
     ```plaintext
     [WinEventLog://Application]
     index = endpoint
     disabled = false
     [WinEventLog://Security]
     index = endpoint
     disabled = false
     [WinEventLog://System]
     index = endpoint
     disabled = false
     [WinEventLog://Microsoft-Windows-Sysmon/Operational]
     index = endpoint
     disabled = false
     renderXml = true
     source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
     ```

3. **Save the File:**
   - Save the file and restart the Splunk Universal Forwarder service for the changes to take effect.

## Promoting Windows Server to Domain Controller
1. In Server Manager, add AD DS role.
   
2. Create a new forest with your desired domain name (e.g., soclab.local).

3. After installation, create Organizational Units (OUs) and users in Active Directory.

## Joining Windows 10 VM to Domain
1. Right-click on "This PC" > Properties > Change settings > Change... and enter your domain name.

2. Use domain admin credentials to join.

## Brute Force Attack Simulation with Kali Linux

### Enable RDP on Windows 10 VM
1. Go to Settings > Remote Desktop and enable it.

### Install Crowbar on Kali Linux
```bash
sudo apt-get install -y crowbar
```

### Prepare Password List for Brute Force Attack
1. Navigate to the wordlist directory:
   ```bash
   cd /usr/share/wordlists/
   ```
   
2. Copy rockyou.txt to your project directory:
   ```bash
   cp rockyou.txt ~/Desktop/ad-project/
   ```

3. Use only the first 50 passwords:
   ```bash
   head -n 50 rockyou.txt > password.txt
   ```

## Conclusion

This setup provides a comprehensive learning environment for understanding SIEM systems, Active Directory management, and security testing methodologies using real-world tools and scenarios.

> Ensure to take snapshots of your VMs periodically for backup purposes during this setup process!
