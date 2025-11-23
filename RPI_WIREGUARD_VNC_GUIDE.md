# Raspberry Pi Remote Access Guide: WireGuard + VNC

This guide provides a complete walkthrough for setting up a Raspberry Pi running Ubuntu 24.04 for secure remote access from a Windows machine. We will configure a static IP address, install WireGuard for a secure VPN tunnel, and use x11vnc to access the Raspberry Pi's graphical desktop.

---

### Step 1: Set a Static IP Address for your Raspberry Pi

For any server, it's crucial to have a predictable IP address. A static IP ensures that your port forwarding rules and other network configurations will always work, as the Pi's local address won't change every time it reconnects to your network.

**Using the Ubuntu Desktop GUI:**

1.  Navigate to **Settings** > **Wi-Fi**.
2.  Click the **gear icon** next to the network you are connected to.
3.  Go to the **IPv4** tab and select **Manual**.
4.  Enter the following details:
    *   **Address:** `192.168.1.27` (This will be your Pi's new, permanent local IP. You can choose a different one, but make sure it's outside your router's DHCP range).
    *   **Netmask:** `255.255.255.0` (This is standard for most home networks).
    *   **Gateway:** `192.168.1.1` (This is the IP address of your router. It might be different, like `192.168.0.1` or `10.0.0.1`).
    *   **DNS:** `8.8.8.8` and `1.1.1.1` (These are public DNS servers from Google and Cloudflare).
5.  Click **Apply**.
6.  Turn your Wi-Fi off and on again to apply the new settings.
7.  Verify the new IP address by opening a terminal and running:
    ```bash
    hostname -I
    ```
    The output should show `192.168.1.27`.

---

### Step 2: Install WireGuard

WireGuard is a modern, fast, and secure VPN. We will use it to create a secure "tunnel" from your Windows PC to your Raspberry Pi, no matter where you are.

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
    This command creates two files: `privatekey` (your server's secret) and `publickey` (the one you'll share).

2.  You will need to view these keys for the configuration files. You can use the `cat` command:
    ```bash
    cat privatekey
    cat publickey
    ```

---

### Step 4: Create the WireGuard Server Configuration

This file defines the WireGuard VPN server settings on your Raspberry Pi.

1.  Create and edit the configuration file using nano:
    ```bash
    sudo nano /etc/wireguard/wg0.conf
    ```

2.  Paste the following configuration into the file. **You must replace `<PASTE YOUR SERVER PRIVATE KEY HERE>` with the actual content of your `privatekey` file.**

    ```ini
    [Interface]
    # This is the virtual IP address for your Raspberry Pi on the VPN network
    Address = 10.8.0.1/24
    # This is the port WireGuard will listen on for incoming connections
    ListenPort = 51820
    # Paste the contents of your 'privatekey' file here
    PrivateKey = <PASTE YOUR SERVER PRIVATE KEY HERE>
    # These commands enable IP forwarding, allowing VPN clients to access the internet through the Pi
    PostUp = echo 1 > /proc/sys/net/ipv4/ip_forward
    PostDown = echo 0 > /proc/sys/net/ipv4/ip_forward

    [Peer]
    # This section is for your Windows client
    # You will need to generate a key pair on your Windows machine and paste its public key here
    PublicKey = <PASTE YOUR CLIENT (Windows) PUBLIC KEY HERE>
    # This assigns a virtual IP to your Windows client within the VPN
    AllowedIPs = 10.8.0.2/32
    ```

---

### Step 5: Enable the WireGuard Service

These commands will start the WireGuard server and configure it to launch automatically every time your Raspberry Pi boots up.

```bash
# Enable the service to start on boot
sudo systemctl enable wg-quick@wg0

# Start the service immediately
sudo systemctl start wg-quick@wg0

# Check the status of your WireGuard interface
sudo wg
```

---

### Step 6: Configure the Windows WireGuard Client

On your Windows PC, you'll need the official WireGuard client.

1.  Open the WireGuard application and create a new, empty tunnel.
2.  Use the following configuration. You will need to generate a new key pair within the Windows client and use its public key in the server configuration (`wg0.conf`) from Step 4.

    ```ini
    [Interface]
    # This is the virtual IP for your Windows machine on the VPN
    Address = 10.8.0.2/24
    # The client automatically generates and fills in its private key
    PrivateKey = <THIS WILL BE FILLED IN BY THE WINDOWS CLIENT>
    DNS = 1.1.1.1

    [Peer]
    # This is the public key from your Raspberry Pi (the 'publickey' file)
    PublicKey = <PASTE YOUR SERVER (Pi) PUBLIC KEY HERE>
    # This tells the client to route all traffic for the VPN's subnet through the tunnel
    AllowedIPs = 10.8.0.0/24
    # This is your home's public IP address and the port you are forwarding
    Endpoint = <YOUR HOME'S PUBLIC IP ADDRESS>:51820
    # This helps keep the connection alive through firewalls
    PersistentKeepalive = 25
    ```

---

### Step 7: Configure Port Forwarding on Your Router

This step is critical. It tells your home router to send incoming WireGuard traffic to your Raspberry Pi.

1.  Log in to your router's administration page.
2.  Find the **Port Forwarding** section (often under "Advanced" or "Firewall" settings).
3.  Create a new rule with the following settings:
    *   **Protocol:** UDP
    *   **External Port:** 51820
    *   **Internal IP Address:** `192.168.1.27` (the static IP of your Pi)
    *   **Internal Port:** 51820
4.  Save and apply the rule.

---

### Step 8: Install x11vnc for Remote Desktop

x11vnc is a VNC server that lets you view the *actual* desktop of your Raspberry Pi, not a virtual one.

1.  Install the package:
    ```bash
    sudo apt install x11vnc
    ```
2.  Set a password for VNC connections. You will be prompted to enter and confirm it.
    ```bash
    x11vnc -storepasswd
    ```

---

### Step 9: Create a Service to Run x11vnc Automatically

To ensure the VNC server starts on boot, we'll create a system service for it.

1.  First, find your username:
    ```bash
    whoami
    ```
    Remember this for the next step.

2.  Create the service file:
    ```bash
    sudo nano /etc/systemd/system/x11vnc.service
    ```

3.  Paste the following text. **IMPORTANT: Replace both instances of `lomadlidar` with your own username.**

    ```ini
    [Unit]
    Description=Start x11vnc at startup
    After=display-manager.service

    [Service]
    Type=simple
    User=lomadlidar
    ExecStart=/usr/bin/x11vnc -display :0 -auth guess -forever -loop -noxdamage -repeat -rfbauth /home/lomadlidar/.vnc/passwd -rfbport 5900 -shared

    [Install]
    WantedBy=multi-user.target
    ```

4.  Enable and start the service:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable x11vnc.service
    sudo systemctl start x11vnc.service
    ```

---

### Step 10: Testing and Connecting

1.  **Test the VPN:** On your Windows PC, activate the WireGuard tunnel. Open a Command Prompt and ping the Raspberry Pi's VPN address:
    ```
    ping 10.8.0.1
    ```
    If you get replies, your VPN is working!

2.  **Connect to VNC:** Using a VNC client like TigerVNC or RealVNC Viewer, connect to the Raspberry Pi using its **VPN IP address** and the correct port:
    *   **Address:** `10.8.0.1:5900`

You should now see your Raspberry Pi's desktop.

---

### Appendix: Useful Restart Commands

If you make changes to your configurations, you can use these commands to restart the services.

```bash
# Restart WireGuard
sudo wg-quick down wg0
sudo wg-quick up wg0

# Restart the VNC Server
sudo systemctl restart x11vnc.service
```
