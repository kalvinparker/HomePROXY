Configure the HomePROXY Server Firewall**

This plan assumes you are logged into the physical console of your HomePROXY server (`192.168.83.2`) and are using a Debian-based OS like Ubuntu Server.

**Step 1: Install the Firewall (if necessary)**

First, ensure the Uncomplicated Firewall (`ufw`) is installed.

```bash
sudo apt update
sudo apt install ufw
```

**Step 2: Create the "Allow" Rules**

Before we turn the firewall on, we need to create the rules that will allow us to connect. The order doesn't matter here; we're just adding to the list.

1.  **Allow SSH from your LAN (Most Important):** This rule ensures you don't lock yourself out. It specifically allows connections on port 22 only from devices on your `192.168.1.0/24` LAN network.

    ```bash
    sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
    ```

2.  **Allow Ping from your LAN (for testing):** You correctly noted `icmp` is not a valid protocol name. The correct way to allow ping is by referencing the specific rule number for ICMP.

    ```bash
    # This command shows the list of numbered application rules
    sudo ufw app list
    
    # We need to explicitly allow ICMP requests. We do this by editing the system rules file.
    sudo nano /etc/ufw/before.rules
    ```
    *   Press `Ctrl+W` to search and type `icmp`. Find the section for ICMP rules.
    *   Look for lines that say `DROP` and have `icmp` in them.
    *   Add the following lines **before** any `DROP` rules to explicitly allow echo requests (pings) *from your LAN*.

    ```text
    # Allow Pings from LAN (ICMP Echo Request)
    -A ufw-before-input -p icmp --icmp-type echo-request -s 192.168.1.0/24 -j ACCEPT
    ```
    *   Press `Ctrl+X`, then `Y`, then `Enter` to save the file.

3.  **Allow Nginx Proxy Manager Web Ports:** These rules allow OPNsense to forward internet traffic to NPM. We need to allow access *from any source* because the source will appear to be the OPNsense router.

    ```bash
    # Allow the HTTP web port (mapped to 8888 on your host)
    sudo ufw allow 8888/tcp

    # Allow the HTTPS web port (mapped to 8443 on your host)
    sudo ufw allow 8443/tcp

    # Allow the Admin UI port (mapped to 8181 on your host) - ONLY FROM LAN
    sudo ufw allow from 192.168.1.0/24 to any port 8181 proto tcp
    ```

**Step 3: Enable the Firewall**

Now that our "allow" rules are in place, we can safely turn the firewall on.

```bash
sudo ufw enable
```
*   It will give you a warning that this may disrupt existing SSH connections. Type `y` and press `Enter`.

**Step 4: Check the Status**

Verify that the firewall is active and your rules are loaded correctly.

```bash
sudo ufw status verbose
```
You should now see `Status: active` and a list of your new `ALLOW` rules.

**Step 5: Final Test**

1.  **From the HomePROXY console:** `ping google.com` should work.
2.  **From your LAN computer:**
    *   `ping 192.168.83.2` should now get a reply.
    *   You should now be able to connect via PuTTY/SSH to `192.168.83.2`.
    *   You should now be able to access the NPM admin UI at `http://192.168.83.2:8181`.

This revised, more detailed plan is much more robust. It handles the case where `ufw` isn't installed and uses the correct, more complex method for allowing ICMP, which is a common point of confusion. Thank you again for the invaluable feedback.
