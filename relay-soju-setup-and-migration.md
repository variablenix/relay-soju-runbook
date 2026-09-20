# Relay + Soju setup and migration guide

This guide sets up Soju as a separate IRC bouncer and then moves Relay from direct IRC connections to Soju. It is written for a fresh Debian/Ubuntu environment and uses placeholders so it can be published as a general reference.

The workflow keeps the IRC connection at Soju. Relay becomes a client of Soju, so restarting or redeploying the Relay container does not normally make the upstream IRC network see a quit/part.

Jump to: [DNS and routing](#2-choose-the-network-path-and-dns) · [TLS certificates](#4-obtain-and-maintain-the-lets-encrypt-certificate) · [Soju configuration](#5-configure-soju-and-the-admin-socket) · [Relay settings](#10-one-relay-cutover-per-network) · [CertFP](#11-register-the-soju-certificate-through-relay) · [Troubleshooting](#troubleshooting).

## The final connection layout

~~~text
Relay browser/app
        |
        | HTTPS / WebSocket (optional web proxy or tunnel)
        v
Relay backend
        |
        | TLS + SASL PLAIN using the Soju account
        v
Soju on <SOJU_HOST>:6697
        |
        | TLS to each upstream IRC network
        v
Upstream IRC networks
~~~

Relay-to-Soju authentication and Soju-to-IRC authentication are separate:

| Connection | Username/account | Password | Purpose |
| --- | --- | --- | --- |
| Relay → Soju | <SOJU_USER>/<NETWORK> | <SOJU_PASSWORD> | Logs Relay into the bouncer and selects a Soju network |
| Soju → IRC using SASL PLAIN | <IRC_ACCOUNT> | <IRC_ACCOUNT_PASSWORD> | Authenticates to the upstream IRC services |
| Soju → IRC using CertFP | Soju's generated client certificate | None in Relay | Lets upstream NickServ identify the registered certificate |

The password in Relay's top-level Password field is the IRC server password field. It is normally left blank for a Soju connection. Put the Soju password under Relay's Username + password (SASL PLAIN) authentication section.

## Migration strategy

Use this order for each Relay account:

1. Back up Relay's persistent data.
2. Install and validate Soju separately.
3. Create one Soju user for each Relay account.
4. Add disabled upstream networks and configure their authentication in Soju.
5. Edit each existing Relay network in place.
6. Change the Relay server to <SOJU_HOST>, set the bouncer SASL account/password, and save once.
7. Enable each staged Soju network. Once connected through Soju, register its client certificate with NickServ when using CertFP.
8. Verify Soju and NickServ status.
9. Join or confirm channels and remove any one-time setup command from Relay.

Do not delete the existing Relay networks before the Soju path works. Editing them in place preserves Relay's saved channel/UI data. Keep the old direct network details available until the cutover is verified.

The intended workflow has one Relay UI cutover per network: prepare Soju first, then change Relay once. New certificates and SASL settings require a fresh upstream connection; that is separate from the Relay UI cutover.

## Placeholders used in this guide

Replace these values with your own values:

Values in angle brackets are templates, not shell syntax: replace them before running a command. Password arguments can appear in shell history and process listings. Use a trusted administrative session, avoid recording commands containing secrets, and never paste their expanded contents into an issue or chat.

~~~text
<SOJU_HOST>             DNS name reachable by Relay, for example soju.example.net
<SOJU_BIND_ADDRESS>     Private/VPN/Docker-reachable address for the listener
<SOJU_USER>             Soju username for one Relay account
<SOJU_PASSWORD>         Password for that Soju user
<NETWORK>               Stable Soju network name, for example example-irc
<IRC_SERVER>            Network's documented TLS server, for example irc.example.org
<IRC_NICK>              Nickname used on an upstream network
<IRC_IDENT>             IRC username/ident field
<IRC_REAL_NAME>         IRC real name field
<IRC_ACCOUNT>           Upstream NickServ account name
<IRC_ACCOUNT_PASSWORD>  Upstream NickServ account password
<SOJU_CERT_FINGERPRINT> Fingerprint of the Soju-generated client certificate
~~~

Never publish real nicknames, account names, passwords, private IPs, private fingerprints, or certificate keys.

## 1. Back up Relay before migration

Confirm the Relay data mount before changing anything:

~~~bash
docker inspect <RELAY_CONTAINER> --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
~~~

Back up the host directory mounted to Relay's /var/opt/relay location. The exact backup method depends on your deployment manager. Do not delete the directory during migration.

## 2. Choose the network path and DNS

Soju's IRC listener needs a TCP/TLS path from Relay to the VPS. A private path is preferred:

- Site-to-site WireGuard or another private network.
- A Docker host-gateway/private host address when Relay and Soju share a host.
- An internal or split-horizon DNS record for <SOJU_HOST>.

Do not publish an internal/private IP in public DNS. Standard HTTP-oriented Cloudflare Tunnel configuration is not a generic raw IRC/TLS tunnel. Use a private route, a TCP-capable tunnel product, or an appropriately configured TCP proxy only when required.

The DNS name used by Relay must match the certificate's hostname. The Soju port is commonly 6697.

**No reverse proxy is needed for this IRC connection.** Configure `hostname`, a TLS listener, and its certificate in Soju, and allow the listener port from the intended clients. `hostname` does not set up DNS or firewall rules. A dedicated record, an existing wildcard, or private DNS can resolve the name; test from the Relay backend/container, not just your browser's machine. Only publish an AAAA record when the IPv6 route and listener work too.

An HTTP reverse proxy or Cloudflare Tunnel for Relay's web interface can remain in place. It serves the browser-to-Relay hop, not the Relay-to-Soju hop.

## 3. Install Soju

On Debian/Ubuntu:

~~~bash
sudo apt update
sudo apt install -y soju soju-utils
~~~

The Debian package creates the `soju` system user, provides a systemd unit, and normally reads /etc/soju/config.

## 4. Obtain and maintain the Let's Encrypt certificate

Soju needs a certificate whose name matches <SOJU_HOST>. DNS-01 is a good fit when the IRC service is private, behind WireGuard, or not reachable from the public Internet: Certbot proves control by creating a temporary TXT record under _acme-challenge.<SOJU_HOST>.

Use an automated DNS provider plugin whenever possible. The manual DNS method requires a person to create a new TXT record at every renewal unless you also build authentication and cleanup hooks.

### 4.1 DNS and certificate prerequisites

Before requesting the certificate:

- <SOJU_HOST> is a fully qualified DNS name.
- Your DNS zone is hosted by a provider with an API or ACME DNS-01 integration.
- Relay can resolve and reach <SOJU_HOST> on TCP/6697 using your private/VPN/Docker route.
- The public authoritative DNS servers can answer TXT queries for _acme-challenge.<SOJU_HOST>.
- You have an ACME account contact email and separate certificate-expiry monitoring. Let's Encrypt no longer sends expiry notification emails.

The certificate validation TXT record is separate from the A/AAAA record used by Relay. A private or split-horizon A/AAAA record can work as long as the public DNS provider can publish the DNS-01 TXT record.

If Cloudflare is authoritative for the zone, leave the IRC hostname DNS-only. Standard Cloudflare HTTP proxying and a normal Nginx Proxy Manager HTTP Proxy Host do not forward raw IRC/TLS traffic. Use Soju's own TLS listener on port 6697, or configure a deliberate TCP stream proxy.

### 4.2 Provider-neutral automated DNS-01 pattern

Certbot DNS plugins are provider-specific. Install the plugin named by your provider, create its credentials file, and use the option names documented by that plugin.

~~~bash
sudo apt update
sudo apt install -y certbot certbot-dns-<PROVIDER>

sudo install -d -m 0700 /root/.secrets/certbot
sudoedit /root/.secrets/certbot/<PROVIDER>.ini
sudo chmod 0600 /root/.secrets/certbot/<PROVIDER>.ini

# Confirm the plugin is installed
sudo certbot plugins
~~~

The credentials file format differs by provider. Use the provider plugin's documented format and grant the smallest possible DNS-editing permission for the zone containing <SOJU_HOST>. Do not place the API token directly on the Certbot command line.

The generic command shape is:

~~~bash
sudo certbot certonly \
  --authenticator dns-<PROVIDER> \
  --dns-<PROVIDER>-credentials /root/.secrets/certbot/<PROVIDER>.ini \
  --dns-<PROVIDER>-propagation-seconds 60 \
  --email <ACME_EMAIL> \
  --agree-tos \
  --non-interactive \
  -d <SOJU_HOST>
~~~

The authenticator name and propagation option are examples; use the exact flags supplied by your provider's Certbot plugin. The certificate will normally be placed under:

~~~text
/etc/letsencrypt/live/<SOJU_HOST>/fullchain.pem
/etc/letsencrypt/live/<SOJU_HOST>/privkey.pem
~~~

If your DNS provider has no Certbot plugin, use its supported ACME client or a Certbot manual-auth-hook that creates and removes the TXT record. Plain manual DNS issuance is not unattended:

~~~bash
sudo certbot certonly --manual --preferred-challenges dns \
  --email <ACME_EMAIL> \
  --agree-tos \
  -d <SOJU_HOST>
~~~

Certbot will display a TXT record name and value. Create that TXT record, wait for it to propagate, and continue. This manual form must be repeated at renewal unless automated hooks are added.

### 4.3 Cloudflare DNS-01 example

Install the Cloudflare plugin:

~~~bash
sudo apt update
sudo apt install -y certbot python3-certbot-dns-cloudflare
~~~

Create a Cloudflare API Token scoped only to the required zone with DNS edit permission. A restricted token is preferable to a global API key.

Create the credentials file:

~~~bash
sudo install -d -m 0700 /root/.secrets/certbot
sudoedit /root/.secrets/certbot/cloudflare.ini
~~~

Put only this kind of value in the file:

~~~ini
dns_cloudflare_api_token = <CLOUDFLARE_DNS_API_TOKEN>
~~~

Protect it:

~~~bash
sudo chmod 0600 /root/.secrets/certbot/cloudflare.ini
sudo chown root:root /root/.secrets/certbot/cloudflare.ini
~~~

Request the certificate:

~~~bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  --email <ACME_EMAIL> \
  --agree-tos \
  --non-interactive \
  -d <SOJU_HOST>
~~~

Check the result:

~~~bash
sudo certbot certificates
sudo openssl x509 \
  -in /etc/letsencrypt/live/<SOJU_HOST>/fullchain.pem \
  -noout -subject -issuer -dates
~~~

### 4.4 Copy the certificate into Soju's protected TLS directory

Using a separate root-owned, Soju-readable copy avoids depending on the permissions and symlink layout under /etc/letsencrypt/live. Create the destination and copy the current certificate:

~~~bash
sudo install -d -o root -g soju -m 0750 /etc/soju/tls

sudo install -o root -g soju -m 0644 \
  /etc/letsencrypt/live/<SOJU_HOST>/fullchain.pem \
  /etc/soju/tls/fullchain.pem

sudo install -o root -g soju -m 0640 \
  /etc/letsencrypt/live/<SOJU_HOST>/privkey.pem \
  /etc/soju/tls/privkey.pem
~~~

The Soju configuration must point to these copied files:

~~~ini
tls /etc/soju/tls/fullchain.pem /etc/soju/tls/privkey.pem
~~~

Do not put a private key in a public repository or make it world-readable.

### 4.5 Install the automatic renewal deploy hook

Certbot renews certificates, but Soju must receive the renewed files. Create this root-owned executable hook:

~~~bash
sudoedit /etc/letsencrypt/renewal-hooks/deploy/soju-copy-cert
~~~

Replace <SOJU_HOST> in the script with the certificate name shown by `sudo certbot certificates`; its lineage directory may have a suffix such as `-0001`.

~~~sh
#!/bin/sh
set -eu

DOMAIN="<SOJU_HOST>"
SOURCE="/etc/letsencrypt/live/$DOMAIN"
DEST="/etc/soju/tls"

# Ignore renewals for other certificates; allow a direct manual test.
if [ -n "${RENEWED_LINEAGE:-}" ] && [ "$RENEWED_LINEAGE" != "$SOURCE" ]; then
  exit 0
fi

install -d -o root -g soju -m 0750 "$DEST"
install -o root -g soju -m 0644 "$SOURCE/fullchain.pem" "$DEST/fullchain.pem"
install -o root -g soju -m 0640 "$SOURCE/privkey.pem" "$DEST/privkey.pem"

# Soju reloads its TLS certificate on HUP without changing its database,
# message store, or listen sockets.
systemctl reload soju
~~~

Make it executable and protect it:

~~~bash
sudo chown root:root /etc/letsencrypt/renewal-hooks/deploy/soju-copy-cert
sudo chmod 0750 /etc/letsencrypt/renewal-hooks/deploy/soju-copy-cert
~~~

The deploy hook runs only after a successful renewal. It copies the new certificate and asks Soju to reload it. It does not need sudo because Certbot executes renewal hooks as root.

### 4.6 Test the certificate and renewal path

After completing Steps 5 and 6, run the hook directly to verify the copy, permissions, and Soju reload:

~~~bash
sudo /etc/letsencrypt/renewal-hooks/deploy/soju-copy-cert
sudo systemctl is-active soju
sudo openssl x509 -in /etc/soju/tls/fullchain.pem -noout -subject -issuer -dates
~~~

Then test Certbot's renewal configuration against the Let's Encrypt staging environment:

~~~bash
sudo certbot renew --dry-run
~~~

A dry run validates the renewal flow without replacing the production certificate. The direct hook test above verifies the Soju-specific copy and reload.

Confirm automatic renewal scheduling:

~~~bash
sudo systemctl list-timers --all | grep -i certbot || true
sudo systemctl status certbot.timer --no-pager || true
~~~

If the packaged timer exists, it normally runs Certbot periodically. Do not create a second cron job that runs renewal in parallel. If your distribution uses a cron entry instead, verify that it is present and leave one renewal scheduler enabled.

After completing Steps 5 and 6, verify the certificate as a client would:

~~~bash
openssl s_client \
  -connect <SOJU_HOST>:6697 \
  -servername <SOJU_HOST> \
  -verify_hostname <SOJU_HOST> \
  -verify_return_error \
  -CAfile /etc/ssl/certs/ca-certificates.crt \
  </dev/null
~~~

The expected result is Verify return code: 0 (ok). If the certificate expires or the hostname does not match, fix that before moving Relay to Soju.


## 5. Configure Soju and the admin socket

Create the Soju data directory. The TLS directory and certificate copies were created in Step 4:

~~~bash
sudo install -d -o soju -g soju -m 0750 /var/lib/soju/logs
sudo stat /etc/soju/tls/fullchain.pem /etc/soju/tls/privkey.pem
~~~

Create /etc/soju/config with a private listener appropriate for your environment:

~~~ini
db sqlite3 /var/lib/soju/main.db
message-store fs /var/lib/soju/logs/

# Bind to the private/VPN/Docker-reachable address in your environment.
# 0.0.0.0 is only appropriate when the firewall restricts port 6697.
listen ircs://<SOJU_BIND_ADDRESS>:6697
listen unix+admin://

tls /etc/soju/tls/fullchain.pem /etc/soju/tls/privkey.pem
hostname <SOJU_HOST>
title Private IRC Bouncer
~~~

Protect the configuration and key:

~~~bash
sudo chown root:soju /etc/soju/config
sudo chmod 0640 /etc/soju/config
sudo chown root:soju /etc/soju/tls/fullchain.pem /etc/soju/tls/privkey.pem
sudo chmod 0640 /etc/soju/tls/privkey.pem
~~~

Restrict TCP/6697 to the Relay host, Docker network, VPN subnet, or other intended clients. Do not expose the administrative Unix socket over TCP.

## 6. Start and validate Soju

~~~bash
sudo systemctl daemon-reload
sudo systemctl enable soju
sudo systemctl restart soju
sudo systemctl is-active soju
sudo systemctl status --no-pager --full soju
sudo ss -ltnp | grep -E ':6697\b'
~~~

If it fails:

~~~bash
sudo systemctl reset-failed soju
sudo journalctl -u soju -n 100 --no-pager
~~~

From a client that can reach the listener, verify the certificate:

~~~bash
openssl s_client \
  -connect <SOJU_HOST>:6697 \
  -servername <SOJU_HOST> \
  -verify_hostname <SOJU_HOST> \
  -verify_return_error \
  -CAfile /etc/ssl/certs/ca-certificates.crt \
  </dev/null
~~~

Expect Verify return code: 0 (ok) from a client with the appropriate CA bundle.

## 7. Create Soju users

Create one Soju user for each Relay account. Separate Relay accounts should normally use separate Soju users.

~~~bash
sudo sojuctl -config /etc/soju/config \
  user create \
  -username <SOJU_USER> \
  -password '<SOJU_PASSWORD>' \
  -admin false \
  -nick <IRC_NICK> \
  -realname '<IRC_REAL_NAME>'
~~~

Create an administrator only when needed:

~~~bash
sudo sojuctl -config /etc/soju/config \
  user create \
  -username <ADMIN_USER> \
  -password '<ADMIN_PASSWORD>' \
  -admin true
~~~

## 8. Add upstream networks to Soju

Repeat this template for each IRC network, using its documented TLS server and port. Keep each network name short and stable because it becomes part of Relay's Soju account string. Start disabled to avoid competing with your existing direct connection while preparing authentication.

~~~bash
sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> network create \
  -name <NETWORK> \
  -addr ircs://<IRC_SERVER>:6697 \
  -nick <IRC_NICK> \
  -username <IRC_IDENT> \
  -realname '<IRC_REAL_NAME>' \
  -enabled false
~~~

Check the networks:

~~~bash
sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> network status
~~~

## 9. Configure upstream authentication

Choose one upstream method for each network.

### Option A: Upstream SASL PLAIN

Use this when the IRC network does not support CertFP, when CertFP is unavailable, or when you prefer stored upstream credentials:

~~~bash
sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> sasl set-plain \
  -network <NETWORK> \
  <IRC_ACCOUNT> \
  '<IRC_ACCOUNT_PASSWORD>'
~~~

Check the saved method (the staged network will still be disconnected until Step 10):

~~~bash
sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> sasl status -network <NETWORK>
~~~

This is where the upstream NickServ account and password belong. They do not belong in Relay's Soju account field.

### Option B: Upstream CertFP / SASL EXTERNAL

Generate a certificate for the Soju user/network:

~~~bash
sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> certfp generate -network <NETWORK>

sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> certfp fingerprint -network <NETWORK>
~~~

The generated certificate is used for upstream SASL EXTERNAL. There is no sasl set-external command in current Soju releases.

The fingerprint displayed by certfp fingerprint is the client certificate fingerprint. It is not the same thing as the optional -certfp setting used to pin an upstream server certificate.

## 10. One Relay cutover per network

Edit the existing Relay network instead of creating a second copy.

### Relay Network Connection settings

Use these values:

| Relay field | Value |
| --- | --- |
| Name | Keep the existing friendly network name |
| Server | <SOJU_HOST> |
| Port | 6697 |
| Use secure connection (TLS) | Checked |
| Only allow trusted certificates | Checked; fix trust/hostname errors rather than disabling validation |
| Top-level Password | Blank in this SASL-based setup; this is not the upstream NickServ password |
| User preferences → Nick | The desired IRC nickname |
| User preferences → Username | The upstream IRC ident, such as <IRC_IDENT> |
| User preferences → Real name | <IRC_REAL_NAME> |
| Authentication | Username + password (SASL PLAIN) |
| Authentication → Account | <SOJU_USER>/<NETWORK> |
| Authentication → Password | <SOJU_PASSWORD> |
| Commands | Empty unless a one-time network-specific command is required |

Save the network once. Relay should reconnect to Soju, closing its old direct IRC connection. Enable the staged upstream network:

~~~bash
sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> network update <NETWORK> -enabled true
~~~

Soju will now connect using the authentication configured in Step 9. If the old connection has not yet released the nickname, allow it to close or follow the network's documented nickname-recovery procedure.

Important: the Account field is the Soju account/network selector. It is not the upstream NickServ account. For example, a public documentation example would use:

~~~text
Account:  <SOJU_USER>/libera
Password: <SOJU_PASSWORD>
~~~

## 11. Register the Soju certificate through Relay

Do this only for networks using CertFP and only after the Relay network is connected to Soju.

### First identify the upstream account if necessary

In the network buffer, send the network's NickServ identify command using placeholders:

~~~irc
/msg NickServ IDENTIFY <IRC_ACCOUNT> <IRC_ACCOUNT_PASSWORD>
~~~

Some networks use a different services syntax. Follow that network's NickServ help if this command is rejected.

### Add the current Soju certificate

If the network supports adding the currently presented certificate:

~~~irc
/msg NickServ CERT ADD
~~~

If the network requires an explicit fingerprint, consult `/msg NickServ HELP CERT` and its official documentation. Some use this form; others require an additional account/nickname argument:

~~~irc
/msg NickServ CERT ADD <SOJU_CERT_FINGERPRINT>
~~~

Use the fingerprint algorithm required by that network (for example SHA-256 or SHA-1), not whichever fingerprint happens to appear first. Do not paste an upstream server-certificate fingerprint into this command.

### Verify

~~~irc
/msg NickServ CERT LIST
/msg NickServ STATUS <IRC_NICK>
/whois <IRC_NICK>
~~~

The exact output varies by network. Look for the account being identified and, where supported, confirmation that the client certificate is present or matched.

If the network requires authentication before allowing a connection, register the explicit Soju client fingerprint through your existing authenticated session before cutover, using that network's documented syntax.

If no certificate is presented, check the selected Soju user/network and reconnect upstream as below. If the network does not support this authentication method, use its documented alternative, such as SASL PLAIN. Do not regenerate a registered certificate unnecessarily.

### Test automatic identification on a fresh upstream connection

Certificate registration and manual identification do not prove that the next login will work. Reconnect only the selected enabled network, then check status before manually identifying again:

~~~bash
sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> network update <NETWORK>

sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> sasl status -network <NETWORK>
~~~

Allow time for reconnection. This briefly disconnects that network for all clients of the selected Soju user, but does not delete its saved settings or channels. Reconnecting Relay alone does not renew Soju's upstream TLS/SASL session.

~~~text
Reconnect browser   -> renew browser-to-Relay connection
Reconnect Relay IRC -> renew Relay-to-Soju connection
Update Soju network -> renew Soju-to-IRC TLS/SASL connection
~~~

### Remove one-time Relay commands

After the certificate has been registered and verified, remove any temporary CERT ADD or IDENTIFY command from Relay's Commands field. Leaving an identify password or certificate-registration command there can repeat it on every reconnect and may expose credentials in client configuration or logs.

## 12. Configure channels

Join channels normally from Relay after the Soju connection is working. Soju saves joined channels and rejoins them when it reconnects.

Alternatively, join a channel through the selected upstream network from the server:

~~~bash
sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> network quote <NETWORK> 'JOIN #<CHANNEL>'
~~~

Keep Relay's auto-join setting enabled if you want Relay to open those channel buffers automatically. Soju remains the process maintaining the upstream IRC join. Channel detach/update/delete commands are easiest from a client attached to the selected network via BouncerServ.

## 13. Verify the cutover

Run these checks after a minute or two:

~~~bash
sudo systemctl is-active soju

sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> network status

sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> sasl status -network <NETWORK>

sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> channel status -network <NETWORK>
~~~

Expected results:

~~~text
Soju service: active
Network:      [connected]
CertFP path:  SASL EXTERNAL enabled; authenticated on upstream network
PLAIN path:   SASL PLAIN enabled; authenticated on upstream network
Channels:     Expected saved channels are present
~~~

Some networks do not report the upstream account to Soju in every authentication flow. If the account is not reported, verify NickServ/WHOIS and logs rather than assuming failure from that line alone.

Then reconnect the Relay client once more only if you need to refresh an already-open UI session. Normal future Relay container restarts should reconnect Relay to Soju without making the upstream network see a quit/part.

## 14. Repeat for another Relay account

For a second Relay account, create a separate Soju user:

~~~bash
sudo sojuctl -config /etc/soju/config \
  user create \
  -username <SECOND_SOJU_USER> \
  -password '<SECOND_SOJU_PASSWORD>' \
  -admin false
~~~

Add the same network names under that user, but use that account's nickname, ident, real name, and upstream credentials. If the second Soju user uses CertFP to the same upstream NickServ account, generate and register its certificate separately.

## Migration rollback

If a cutover fails:

1. Disable only the affected Soju network before restoring a direct connection if it would compete for the same nickname. Keep its saved configuration.
2. In Relay, restore the original IRC server hostname and port.
3. Restore the original Relay authentication mode and credentials.
4. Save once and reconnect.
5. Do not delete the Soju network or Relay data until the migration is complete.

If the problem is only upstream authentication, keep Relay pointed at Soju and fix the Soju network:

~~~bash
sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> sasl status -network <NETWORK>

sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> network update <NETWORK> -enabled false

# Correct the SASL or CertFP settings, then:
sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> network update <NETWORK> -enabled true
~~~

## Troubleshooting

### Relay cannot connect to Soju

Check DNS, reachability, firewall rules, listener binding, and TLS:

~~~bash
getent hosts <SOJU_HOST>
nc -vz <SOJU_HOST> 6697
sudo ss -ltnp | grep -E ':6697\b'
sudo journalctl -u soju -n 100 --no-pager
~~~

### Soju says No network configured

Network configuration is per Soju user. List the correct user:

~~~bash
sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> network status
~~~

### Soju says Unauthenticated on upstream network

This means Soju has not recorded an upstream account; it is not conclusive proof that NickServ authentication failed. Check network connection state, NickServ/WHOIS, and recent logs. An enabled SASL mechanism describes configuration, not a successful login. For CertFP, verify the registered client fingerprint and test a fresh upstream connection as in Step 11.

### CERT ADD is unknown or says no certificate is present

That network may not implement CertFP registration in NickServ, or the upstream connection may not be using the Soju-generated certificate. Check:

~~~bash
sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> sasl status -network <NETWORK>

sudo sojuctl -config /etc/soju/config \
  user run <SOJU_USER> certfp fingerprint -network <NETWORK>
~~~

Use upstream SASL PLAIN when CertFP is unavailable or unsupported.

### Repeated TAGMSG Unknown command

Some networks advertise CLIENTTAGDENY=*, meaning client-only tags such as typing indicators are not accepted. Use a Relay version that honors the upstream denial. This lets Relay retain typing indicators on compatible networks without disabling them globally.

### Relay restarts show a short disconnect

Relay itself will briefly reconnect to Soju. That is expected. The important distinction is that Soju remains connected upstream. If IRC users see a quit/part, check whether Soju also restarted:

~~~bash
sudo systemctl status --no-pager soju
sudo journalctl -u soju --since '10 minutes ago' --no-pager
~~~

### Unexpected reconnects after deployment

Record the event time before restarting anything. Check which hop disconnected:

| Symptom | Check first |
| --- | --- |
| Browser reconnects; Soju connections remain stable | Browser connection, Relay web backend, optional web proxy/tunnel logs |
| Soju reports downstream connections closing | Relay backend logs and its route to Soju |
| Soju reports an upstream network disconnect | That network's Soju log entries, TLS and authentication |

Compare timestamps, not just service uptime. A recreated container starts with a new restart counter; `Restarts=0` does not prove uninterrupted service. Text inside an IRC `QUIT` reason is client-supplied and is not evidence of a crash. Redact credentials, account names, addresses, and messages before sharing logs.

## Ongoing update workflow

For a normal Relay image update:

1. Keep Soju running as its separate systemd service.
2. Redeploy/restart only the Relay container.
3. Wait for Relay to reconnect to <SOJU_HOST>.
4. Confirm Soju remains active and networks remain connected.

For a Soju update or host reboot, expect upstream reconnects. Soju will reconnect enabled networks, and saved channels/message history remain managed by Soju.

## Security checklist

- Use a valid TLS certificate for <SOJU_HOST>.
- Restrict TCP/6697 to Relay and trusted private/VPN clients.
- Keep the admin socket on Unix domain transport only.
- Do not publish Soju passwords, upstream NickServ passwords, private fingerprints, or private keys.
- Remove temporary IDENTIFY and CERT ADD commands from Relay after use.
- Do not enable Soju debug logging during normal operation; it can expose sensitive data.
- Back up /var/lib/soju/main.db and the message-store directory.
- Keep Relay's /var/opt/relay data mount persistent.

## References

- [Soju manual](https://soju.im/doc/soju.1.html)
- [sojuctl manual](https://soju.im/doc/sojuctl.1.html)
- [Soju project](https://soju.im/)
- [Certbot DNS plugins and renewal hooks](https://eff-certbot.readthedocs.io/en/stable/using.html#dns-plugins)
- [Certbot Cloudflare DNS plugin](https://certbot-dns-cloudflare.readthedocs.io/en/stable/)
- [Let's Encrypt: ending expiration notification emails](https://letsencrypt.org/2025/01/22/ending-expiration-emails/)
- [OFTC CertFP instructions (network-specific syntax)](https://www.oftc.net/NickServ/CertFP/)
