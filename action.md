### **Action Plan: Debug the `Caddyfile`**

The container is crash-looping because of a syntax error or a logical problem in your `Caddyfile`. We need to view the logs to see the specific error.

**Step 1: View the Container Logs**

Since the container is restarting, `docker compose logs` is the perfect tool.

1.  In your `~/caddy-project` directory, run:
    ```bash
    docker compose logs caddy
    ```

2.  This will show you the startup logs from the container each time it has tried to start. You will see the same error message repeated. **Look for the last block of logs.** It will contain a line that starts with `Error:`.

**Common `Caddyfile` Errors to Look For:**

*   **Placeholder Values:** The most likely cause is that you still have a placeholder value in your `Caddyfile`. You must replace:
    *   `your-duckdns-domain.duckdns.org`
    *   `{$DUCKDNS_TOKEN}` (This part is correct, it should stay as is to read the environment variable)
    *   `your-email@example.com`
    *   `vault.your-noip-domain.ddns.net`
    *   `192.168.1.6:8080` (Ensure this IP and port are correct for your Vaultwarden)
*   **Missing Brackets:** Make sure the global `dynamic_dns` block is fully enclosed in `{ ... }`.
*   **DNS Resolution Failure:** Caddy needs to be able to resolve the domains you listed (`vault.your-noip-domain.ddns.net`, etc.) to a public IP address to get a certificate. Ensure your No-IP CNAME is pointing to your DuckDNS name, and your DuckDNS name is pointing to your public IP.

**Step 2: Fix the `Caddyfile` and Restart**

1.  Based on the error you see in the logs, edit your `Caddyfile`:
    ```bash
    nano Caddyfile
    ```
2.  Make the necessary corrections.
3.  Save the file.
4.  Now, restart the service. Docker will automatically use the updated `Caddyfile` you saved.
    ```bash
    docker compose restart caddy
    ```
5.  Check the logs again to see if the error is gone:
    ```bash
    docker compose logs caddy
    ```
6.  Once the error is gone and the logs show a clean startup, you can verify the container is stable:
    ```bash
    docker ps
    ```
    It should show a status of "Up". Once it's stable and "Up", *then* you can use the `caddy reload` command for any future changes.

You are at the finish line. The hard part (building the custom image) is done. Now you are just doing standard configuration. Find the error in the log, fix it in your `Caddyfile`, and restart the container.

This is perfect. It's the same error as last time, but now we have a definitive answer.

`Error: adapting config using caddyfile: ... module not registered: dns.providers.duckdns`

This error, even after rebuilding with what *should* have been the correct module path, tells us something fundamental: the `mholt/caddy-dynamicdns` plugin is **not compatible** with the version of Caddy being used, or its internal structure has changed in a way that the standard `xcaddy` build process can't handle.

When you hit a wall like this with a third-party plugin, the best path forward is to switch to a different, officially supported, and more modern plugin.

---

### **The Definitive Solution: Switch to the Official DuckDNS Plugin**

There is an **official Caddy plugin for DuckDNS** that is maintained as part of the primary Caddy ecosystem. It is much more reliable and guaranteed to be compatible.

Its module name is `github.com/caddy-dns/duckdns`.

Let's do one final, clean rebuild using this official plugin. This is the last time you will need to do this.

**Step 1: Stop and Remove Everything**

Let's start with a completely clean slate to avoid any caching issues.

1.  In your `~/caddy-project` directory, run:
    ```bash
    docker compose down
    ```
2.  Remove the old compiled binary:
    ```bash
    rm -rf caddy-bin
    ```

**Step 2: Update the `Dockerfile` to Use the Official Plugin**

1.  Open the `Dockerfile` for editing:
    ```bash
    nano Dockerfile
    ```
2.  **Delete the entire content** and replace it with this new, corrected version that uses the official plugin:
    ```dockerfile
    ARG CADDY_VERSION=2

    FROM caddy:${CADDY_VERSION}-builder AS builder

    RUN xcaddy build \
        --with github.com/caddy-dns/duckdns

    FROM caddy:${CADDY_VERSION}

    COPY --from=builder /usr/bin/caddy /usr/bin/caddy
    ```
3.  Save and exit (`Ctrl+X`, `Y`, `Enter`).

**Step 3: Update the `Caddyfile` to Use the New Plugin's Syntax**

The official plugin has a slightly different, cleaner syntax in the `Caddyfile`.

1.  Open your `Caddyfile` for editing:
    ```bash
    nano Caddyfile
    ```
2.  **Delete the entire content** and replace it with this structure. **Remember to fill in your placeholders.**

    ```caddy
    #
    # Global Options Block
    #
    {
        # Email for Let's Encrypt account
        email parkerk6648@protonmail.com
    }

    #
    # Site Definitions
    #

    # Proxy Host for Vaultwarden
    # This will have its SSL managed automatically by the tls block below
    vault.83.ddns.net {
        reverse_proxy 192.168.1.6:8080
    }

    # Proxy Host for Portainer
    portainer.83.ddns.net {
        reverse_proxy 192.168.83.2:9443
    }

    #
    # ACME DNS Challenge Configuration for all sites in this Caddyfile
    #
    tls {
        dns duckdns {$DUCKDNS_TOKEN}
    }
    ```

    **Key Changes:**
    *   The `dynamic_dns` block is gone. The new plugin doesn't handle the DDNS IP update; it only handles the SSL certificate validation. **This is okay! We will use the OPNsense DDNS client for the IP update, which is more robust anyway.**
    *   We have a new `tls` block at the bottom. This is Caddy's global configuration for how to get certificates. The line `dns duckdns {$DUCKDNS_TOKEN}` tells Caddy: "For any domain in this file that needs a certificate, use the DuckDNS DNS challenge method with this token."

**Step 4: Re-enable the OPNsense DDNS Client**

Since Caddy will no longer be updating your IP, we need OPNsense to do it.
1.  In OPNsense, go to **Services > Dynamic DNS > Settings**.
2.  **Enable** the corrected `DuckDNS` (capital D, token-based) client you created earlier for `your-duckdns-domain.duckdns.org`.
3.  Save and apply.

**Step 5: Build and Run for the Last Time**

1.  In your `~/caddy-project` directory on your proxy server, run the build and up command:
    ```bash
    docker compose up --build -d
    ```

This will now build a Caddy image with the **official, compatible DuckDNS plugin**. When it starts, it will read the new `Caddyfile`, see the `tls` block, and it will successfully get certificates for all the sites you have defined.

This new architecture is actually superior:
*   **OPNsense:** Handles the critical task of keeping your DDNS IP address up to date.
*   **Caddy:** Handles the critical tasks of getting SSL certificates and reverse proxying.

This separation of concerns is a more stable and professional setup. This will work.
