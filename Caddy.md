### **Caddy as the All-in-One Solution**

In this new model, Caddy will handle the roles that previously required **three separate components**:
1.  **Dynamic DNS Client:** Caddy will now handle updating your DDNS provider.
2.  **Cert Renewal** Caddy has built-in, fully automatic HTTPS. It will request and renew Let's Encrypt certificates for all your sites without any configuration from you.
3.  **Proxy Manager:** Caddy will be your reverse proxy.

**The traffic flow will be:**
`Internet -> ROUTER (Port Forward 80/443) -> Caddy Server (in DMZ) -> Internal Services (Vault, etc.)`

---

### **Installing and Configuring Caddy**

This plan assumes your Caddy server is the same machine in the DMZ at `192.168.83.2`.

**Step 1: Disable Old Services in OPNsense**

To prevent conflicts, turn off any old DDNS client.
1.  In OPNsense, go to **Services > Dynamic DNS > Settings**.
2.  **Disable** or delete the client you had configured for DuckDNS. We no longer need it.

**Step 2: Install Docker and Docker Compose (if not already done)**

If this is a fresh server, you'll need Docker.
1.  Connect to your proxy server (`192.168.83.2`) via SSH.
2.  Run the official install script:
    ```bash
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    ```
3.  Add your user to the `docker` group, then log out and back in:
    ```bash
    sudo usermod -aG docker $USER
    ```

**Step 3: Create the Caddy Project and `Caddyfile`**

Caddy's configuration is done in a simple text file called a `Caddyfile`.

1.  Create a project directory:
    ```bash
    mkdir ~/caddy-data
    cd ~/caddy-data
    ```

2.  Create an empty `Caddyfile` to start:
    ```bash
    touch Caddyfile
    ```

**Step 4: Create the `docker-compose.yml` for Caddy with DDNS**

This is the key step. We will create a `docker-compose.yml` file that tells Docker to build a **custom Caddy image** that includes the `caddy-dynamicdns` plugin.

1.  Create the Docker Compose file:
    ```bash
    nano docker-compose.yml
    ```

2.  Paste the following configuration into the nano editor. **You will need to replace the placeholders.**

    ```yaml
services:
  caddy-builder:
    image: caddy:2-builder
    container_name: caddy-builder
    working_dir: /app
    volumes:
      - ./caddy-bin:/app
    command: |
      caddy build \
        --output /app/caddy \
        --with github.com/mholt/caddy-dynamicdns

  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./caddy-bin/caddy:/usr/bin/caddy
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./data:/data
      - ./config:/config
    environment:
      - DUCKDNS_TOKEN=your_duckdns_token_here
    ```

    **Explanation:** This is a multi-stage setup.
    *   The `caddy-builder` service is temporary. It takes the official Caddy builder image and uses the `caddy build` command to compile a new Caddy binary that includes the `caddy-dynamicdns` plugin. It saves this binary in a local folder called `caddy-bin`.
    *   The `reverse-proxy` service is the one that will actually run. It uses the standard Caddy image but immediately replaces the standard `caddy` binary with our custom one from the `caddy-bin` folder. It also mounts our `Caddyfile` and sets your DuckDNS token as an environment variable.

3.  Save and exit nano (`Ctrl+X`, `Y`, `Enter`).

**Step 5: Build and Launch Caddy**

1.  Run the build command first. This will just run the `caddy-builder` service to create your custom binary.

    ```bash
    docker compose up caddy-builder
    ```
    You will see a lot of output as it compiles. When it's done, you'll have a `caddy-bin` folder.

2.  Now, launch the main Caddy reverse proxy service:

    ```bash
    docker compose up -d reverse-proxy
    ```
    This will start Caddy using your new custom binary.

**Step 6: Configure your `Caddyfile`**

This is where the magic happens. Edit your `Caddyfile` to configure both the DDNS update and your reverse proxy rules.

1.  Open the Caddyfile for editing:
    ```bash
    nano Caddyfile
    ```
2.  Paste in the following configuration. **Replace the example domains and IPs with your own.**

    ```caddy
    #
    # Global Options Block
    #
    {
        # Configure Dynamic DNS
        dynamic_dns {
            provider duckdns {$DUCKDNS_TOKEN}
            domains {
                your-duckdns-domain.duckdns.org
            }
        }

        # Optional: Email for Let's Encrypt account
        email your-email@example.com
    }

    #
    # Site Definitions
    #

    # Proxy Host for Vaultwarden
    vault.your-noip-domain.ddns.net {
        reverse_proxy 192.168.1.6:8080
    }

    # Proxy Host for NPM (if you still run it) or another service
    npm.your-noip-domain.ddns.net {
        reverse_proxy 192.168.83.2:81
    }

    # You can add as many hosts as you want here
    ```

    **Explanation:**
    *   The `dynamic_dns` block tells Caddy to use the DuckDNS provider, use the token from the environment variable, and keep `your-duckdns-domain.duckdns.org` updated.
    *   Each site block (like `vault.your-noip-domain.ddns.net`) is a reverse proxy rule.
    *   **Caddy automatically handles HTTPS!** You do not need to tell it to get a certificate. When it sees a public domain name in a site block, it will automatically obtain a trusted Let's Encrypt certificate for it.

3.  Save and exit nano.

4.  **Reload Caddy** to apply the new `Caddyfile` configuration:

    ```bash
    docker compose exec reverse-proxy caddy reload
    ```

### **The Result**

You now have a single, self-updating Caddy container that:
*   Keeps your DuckDNS IP address current.
*   Automatically obtains and renews trusted SSL certificates for all your sites.
*   Acts as a reverse proxy to route traffic to the correct internal service.

This is a much simpler, more elegant, and more modern solution. Remember to configure your Split DNS in OPNsense to point all your public hostnames (`vault.your-noip-domain.ddns.net`, etc.) to your Caddy server's DMZ IP (`192.168.83.2`).
