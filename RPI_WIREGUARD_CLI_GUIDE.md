# Raspberry Pi CLI Remote Access Guide: WireGuard + SSH

This guide provides a complete walkthrough for setting up a Raspberry Pi running Ubuntu 24.04 for secure **command-line** remote access from a Windows machine. We will configure a static IP address and then install WireGuard to create a secure VPN tunnel, allowing you to SSH into your Raspberry Pi from anywhere.

---

### Step 1: Set a Static IP Address for your Raspberry Pi

For any server, it's crucial to have a predictable IP address. A static IP ensures that your port forwarding rules will always work, as the Pi's local address won't change every time it reconnects to your network.

**A) CLI Method (Netplan - Recommended for Servers)**

Ubuntu uses Netplan for network configuration. You can set a static IP by editing a YAML file.

1.  Find your network configuration file. It's usually in `/etc/netplan/` and might be named `50-cloud-init.yaml` or similar.
    ```bash
    ls /etc/netplan/
    ```
2.  Edit the file with `nano`. (Replace the filename with yours).
    ```bash
    sudo nano /etc/netplan/50-cloud-init.yaml
    ```
3.  Modify the file to set the static IP. Change `dhcp4: true` to `dhcp4: no` and add the `addresses`, `gateway4`, and `nameservers` sections. Your file should look something like this (ensure indentation is correct):

    ```yaml
    network:
        ethernets:
            eth0:
                dhcp4: no
                addresses: [192.168.1.27/24]
                gateway4: 192.168.1.1
                nameservers:
                    addresses: [8.8.8.8, 1.1.1.1]
        version: 2
    ```
    *Note: If you are using Wi-Fi, the section might be `wifis:` and the interface name (like `wlan0`) will be different.*

4.  Apply the new configuration:
    ```bash
    sudo netplan apply
    ```

**B) GUI Method (If you have a desktop environment installed)**

1.  Navigate to **Settings** > **Wi-Fi**.
2.  Click the **gear icon** next to your connected network.
3.  Go to the **IPv4** tab and select **Manual**.
4.  Enter the details as described in the previous guide (IP: `192.168.1.27`, Gateway: `192.168.1.1`, etc.).
5.  Click **Apply** and toggle your Wi-Fi off and on.

---

### Step 2: Install WireGuard

WireGuard is a modern, fast, and secure VPN. We will use it to create a secure "tunnel" from your Windows PC to your Raspberry Pi.

```bash
sudo apt update
sudo apt install wireguard
```

---

### Step 3: Generate Server Keys

WireGuard uses a pair of cryptographic keys for security: a private key that stays on the server and a public key that you share with clients.

1.  Generate the keys in your terminal:
    ```bash
    wg genkey | tee privatekey | wg pubkey > publickey
    ```
2.  You will need to view these keys to copy and paste them. Use the `cat` command:
    ```bash
    cat privatekey
    cat publickey
    ```

---

### Step 4: Create the WireGuard Server Configuration

This file defines the WireGuard VPN server settings on your Raspberry Pi.

1.  Create and edit the configuration file:
    ```bash
    sudo nano /etc/wireguard/wg0.conf
    ```
2.  Paste the following configuration, replacing the placeholder text with your actual keys.

    ```ini
    [Interface]
    Address = 10.8.0.1/24
    ListenPort = 51820
    PrivateKey = <PASTE YOUR SERVER (Pi) PRIVATE KEY HERE>
    PostUp = echo 1 > /proc/sys/net/ipv4/ip_forward
    PostDown = echo 0 > /proc/sys/net/ipv4/ip_forward

    [Peer]
    PublicKey = <PASTE YOUR CLIENT (Windows) PUBLIC KEY HERE>
    AllowedIPs = 10.8.0.2/32
    ```

---

### Step 5: Enable the WireGuard Service

These commands will start the WireGuard server and configure it to launch automatically every time your Raspberry Pi boots up.

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo wg
```
The last command, `sudo wg`, will show you the current status of the WireGuard interface.

---

### Step 6: Configure the Windows WireGuard Client

On your Windows PC, open the WireGuard client and create a new tunnel with the following configuration. The client will generate its own private key and you will need to paste its public key into the `wg0.conf` file on the Pi.

```ini
[Interface]
Address = 10.8.0.2/24
PrivateKey = <THIS WILL BE FILLED IN BY THE WINDOWS CLIENT>
DNS = 1.1.1.1

[Peer]
PublicKey = <PASTE YOUR SERVER (Pi) PUBLIC KEY HERE>
AllowedIPs = 10.8.0.0/24
Endpoint = <YOUR HOME'S PUBLIC IP ADDRESS>:51820
PersistentKeepalive = 25
```

---

### Step 7: Configure Port Forwarding on Your Router

This crucial step tells your home router to send incoming WireGuard traffic to your Raspberry Pi.

1.  Log in to your router's administration page.
2.  Find the **Port Forwarding** section.
3.  Create a new rule:
    *   **Protocol:** UDP
    *   **External Port:** 51820
    *   **Internal IP Address:** `192.168.1.27` (the static IP of your Pi)
    *   **Internal Port:** 51820
4.  Save and apply the rule.

---

### Step 8: Testing and Connecting via SSH

Now we will test the connection and log in via SSH.

1.  **Test the VPN:** On your Windows PC, activate the WireGuard tunnel. Open a Command Prompt and `ping` the Raspberry Pi's VPN address:
    ```
    ping 10.8.0.1
    ```
    If you get replies, your VPN tunnel is working!

2.  **Connect with SSH:** Use an SSH client (like Windows Terminal or PuTTY) to connect to the Pi through the secure tunnel. Find your Pi's username by running `whoami` on the Pi itself.

    ```
    ssh your_username@10.8.0.1
    ```
    *(Replace `your_username` with your actual username on the Pi).*

You should now be logged into your Raspberry Pi's command line securely.

---

### Appendix: Useful Restart Commands

If you make changes to your WireGuard configuration, you can use these commands to restart the service.

```bash
sudo wg-quick down wg0
sudo wg-quick up wg0
```
