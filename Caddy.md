### **Revised Guide: Caddy as the All-in-One Solution (v2.0)**

Here is the rewritten, battle-tested guide that incorporates all our lessons learned.

### **Caddy as the All-in-One Solution**

In this new model, Caddy will handle the roles that previously required multiple separate components:
1.  **Dynamic DNS Client:** Caddy will automatically keep your DDNS provider updated with your public IP address.
2.  **Automatic HTTPS (Certificate Renewal):** Caddy has built-in, fully automatic HTTPS. It will request and renew trusted Let's Encrypt certificates for all your sites. No manual ACME configuration is needed.
3.  **Reverse Proxy:** Caddy will securely route traffic to all your internal services.

**The Traffic Flow:**
`Internet -> ROUTER (Port Forward 80/443) -> Caddy Server (in DMZ) -> Internal Services (Vault, etc.)`

---

### **Installing and Configuring Caddy**

This plan assumes your Caddy server is on a dedicated machine in the DMZ at `192.168.83.2`.

**Step 1: Disable Conflicting Services in OPNsense**

To prevent conflicts, ensure no other service is trying to manage your DDNS.
1.  In OPNsense, go to **Services > Dynamic DNS > Settings**.
2.  **Disable** or **delete** any DDNS clients you had configured (e.g., for DuckDNS or No-IP).

**Step 2: Install Docker**

If this is a fresh server, you'll need Docker.
1.  Connect to your proxy server (`192.168.83.2`) via SSH.
2.  Run the official install script:
    ```bash
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    ```
3.  Add your user to the `docker` group to avoid using `sudo` for every command. **You must log out and log back in for this to take effect.**
    ```bash
    sudo usermod -aG docker $USER
    ```

**Step 3: Create the Caddy Project Structure**

We will create a clean structure for all of Caddy's files.

1.  Create a project directory and navigate into it:
    ```bash
    mkdir ~/caddy-project
    cd ~/caddy-project
    ```

2.  Create an empty `Caddyfile` where we will define our proxy rules:
    ```bash
    touch Caddyfile
    ```

**Step 4: Create the `Dockerfile` for a Custom Caddy Image**

This is the robust method to build Caddy with the DDNS plugin.

1.  Create a file named `Dockerfile`:
    ```bash
    nano Dockerfile
    ```

2.  Paste the following content into the file. This is a multi-stage build that creates a clean, small final image.
    ```dockerfile
    ARG CADDY_VERSION=2

    FROM caddy:${CADDY_VERSION}-builder AS builder

    RUN xcaddy build \
        --with github.com/mholt/caddy-dynamicdns

    FROM caddy:${CADDY_VERSION}

    COPY --from=builder /usr/bin/caddy /usr/bin/caddy
    ```

3.  Save and exit (`Ctrl+X`, `Y`, `Enter`).

**Step 5: Create the `docker-compose.yml` File**

This file will now use our `Dockerfile` to build and run the Caddy container.

1.  Create the Docker Compose file:
    ```bash
    nano docker-compose.yml
    ```

2.  Paste the following configuration. **Remember to replace `your_duckdns_token_here` with your actual token.**
    ```yaml
    services:
      caddy:
        build: .
        container_name: caddy
        restart: unless-stopped
        ports:
          - "80:80"
          - "443:443"
          - "443:443/udp"
        volumes:
          - ./Caddyfile:/etc/caddy/Caddyfile
          - ./data:/data
          - ./config:/config
        environment:
          - DUCKDNS_TOKEN=d5d32851-0eee-4581-bc3e-a09e2c3c14f9
    ```

3.  Save and exit.

**Step 6: Build and Launch Caddy**

This is now a single, reliable command.

1.  Make sure you are in the `~/caddy-project` directory.
2.  Run the following command:
    ```bash
    docker compose up --build -d
    ```
    *   `--build`: This tells Docker Compose to build the custom image from your `Dockerfile`.
    *   `-d`: Runs the container in the background.

Docker will now build your custom image and start the container. You can verify it's running with `docker ps`.

**Step 7: Configure your `Caddyfile`**

This is where you define everything Caddy does.

1.  Open the `Caddyfile` for editing:
    ```bash
    nano Caddyfile
    ```

2.  Paste in the following configuration. **Replace all placeholders** (like `your-duckdns-domain.duckdns.org`, `vault.your-noip-domain.ddns.net`, IPs, and ports) with your actual values.

    ```caddy
    #
    # Global Options Block
    #
    {
        # Configure Dynamic DNS to update your DuckDNS domain
        dynamic_dns {
            provider duckdns {$DUCKDNS_TOKEN}
            domains {
                your-duckdns-domain.duckdns.org
            }
        }

        # Email for Let's Encrypt account (important for recovery)
        email your-email@example.com
    }

    #
    # Site Definitions (Reverse Proxy Rules)
    # Caddy will AUTOMATICALLY get SSL certificates for these domains.
    #

    # Proxy Host for Vaultwarden
    # Assumes your No-IP domain is a CNAME pointing to your DuckDNS domain.
    vault.your-noip-domain.ddns.net {
        reverse_proxy 192.168.1.6:8080
    }

    # Proxy Host for Portainer
    portainer.your-noip-domain.ddns.net {
        # Portainer's web UI runs on port 9000 by default
        reverse_proxy 192.168.83.2:9000
    }

    # Add more hosts as needed...
    ```

3.  Save and exit.

4.  **Reload Caddy** to apply the new `Caddyfile` configuration. This command is instant and causes no downtime.
    ```bash
    docker compose exec caddy caddy reload
    ```

### **The Result**

You now have a single, self-updating Caddy container that:
*   Keeps your DuckDNS IP address current.
*   Automatically obtains and renews trusted SSL certificates for all your sites.
*   Acts as a secure reverse proxy to route traffic to the correct internal service.

Remember to configure your **Split DNS (Host Overrides)** in OPNsense to point all your public hostnames (`vault.your-noip-domain.ddns.net`, etc.) to your Caddy server's DMZ IP (`192.168.83.2`).
