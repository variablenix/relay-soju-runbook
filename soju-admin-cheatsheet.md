# Soju administration cheat sheet

These Soju administration commands are independent of your IRC client; [Relay](https://relayirc.com) is simply the web IRC client used in the companion guide.

The service and filesystem examples assume Debian/Ubuntu with systemd and this configuration file. On other Linux distributions, adjust the package paths, service commands, and permissions to match your installation:

~~~text
/etc/soju/config
~~~

The examples use placeholders such as <SOJU_USER>, <NETWORK>, and <IRC_ACCOUNT>. Replace them before running commands.

## Quick variables and helper functions

Run this once in Bash or Zsh. The functions disappear when the shell closes; copy them into ~/.bashrc or ~/.zshrc if desired.

~~~bash
export SOJU_CONFIG=/etc/soju/config

sj() {
  sudo sojuctl -config "$SOJU_CONFIG" "$@"
}

sj-user() {
  if [ "$#" -eq 0 ]; then
    sj user status
    return
  fi

  local user="$1"
  shift
  if [ "$#" -eq 0 ]; then
    sj user run "$user" network status
  else
    sj user run "$user" "$@"
  fi
}

sj-networks() {
  if [ -z "${1:-}" ]; then
    printf 'usage: sj-networks <SOJU_USER>\n' >&2
    return 2
  fi
  sj-user "$1"
}

sj-channels() {
  sj-user "$1" channel status -network "$2"
}

sj-sasl() {
  sj-user "$1" sasl status -network "$2"
}

sj-cert() {
  sj-user "$1" certfp fingerprint -network "$2"
}

sj-logs() {
  sudo journalctl -u soju -n "${1:-100}" --no-pager
}

sj-follow-logs() {
  sudo journalctl -u soju -f
}
~~~

These helpers use your existing sudo policy and prompt when required. Test with:

~~~bash
sj help
~~~

The `sj-user` helper has convenient defaults:

~~~bash
sj-user                                      # List all Soju users
sj-user <SOJU_USER>                          # List that user's networks
sj-user <SOJU_USER> sasl status -network <NETWORK>
~~~

## Admin access

For an optional menu-driven alternative to these commands, see [Soju-TUI](https://github.com/variablenix/soju-tui). It uses `sojuctl` and needs the same authorized admin-socket access; it is not an IRC chat client.

Soju's admin socket grants full bouncer administration, including other users' networks. Use a trusted administrator account. Do not make the socket world-writable or copy a broad passwordless sudo rule just to avoid a prompt. If you deliberately delegate access, have the system administrator review the socket permissions or sudo policy; that is separate from connecting Relay as a regular Soju user.

If Soju is running but the TUI or `sojuctl` reports permission denied, check access to both `/run/soju` and `/run/soju/admin`. Socket permissions may be recreated on restart. A missing socket instead calls for checking the service and `listen unix+admin://` configuration.

## Service and configuration

~~~bash
# Service state
sudo systemctl is-active soju
sudo systemctl status --no-pager --full soju
sudo systemctl enable soju
sudo systemctl start soju
sudo systemctl stop soju
sudo systemctl restart soju

# Reload configuration, TLS certificates, and MOTD
sudo systemctl reload soju

# Logs
sudo journalctl -u soju -n 100 --no-pager
sudo journalctl -u soju -f

# Confirm the listener
sudo ss -ltnp | grep -E ':6697\b' || true
~~~

The reload operation does not change the database, message store, or listen sockets. Restart Soju when changing those settings.

Inspect the active configuration carefully; it contains paths and security-sensitive settings:

~~~bash
sudo sed -n '1,240p' /etc/soju/config
sudo systemctl cat soju
~~~

## Server-wide commands

~~~bash
# List users and basic statistics; admin only
sj user status
sj user status <SOJU_USER>

# Server statistics; admin only
sj server status

# Broadcast a notice to connected bouncer users; admin only
sj server notice "Scheduled maintenance in 10 minutes"

# Temporarily enable or disable debug logging.
# Debug logging can expose passwords and other sensitive data.
sj server debug true
sj server debug false

# Command help
sj help
sj help network
sj help sasl
sj help certfp
~~~

## User administration

Create a user:

~~~bash
sj user create \
  -username <SOJU_USER> \
  -password '<SOJU_PASSWORD>' \
  -admin false \
  -nick <DEFAULT_NICK> \
  -realname '<REAL_NAME>'
~~~

For a first administrative account, use -admin true. Avoid putting real passwords in shell history; use a private prompt or a password manager when possible.

~~~bash
# Enable or disable a user
sj user update <SOJU_USER> -enabled true
sj user update <SOJU_USER> -enabled false

# Change another user's password or admin status
sj user update <SOJU_USER> -password '<NEW_SOJU_PASSWORD>'
sj user update <SOJU_USER> -admin false

# Delete a user. This disconnects and removes all of that user's networks/channels.
sj user delete <SOJU_USER>
~~~

## Network commands

Run these as the Soju user who owns the networks:

~~~bash
# List saved networks and connection state
sj-networks <SOJU_USER>

# Create a TLS network
sj-user <SOJU_USER> network create \
  -name <NETWORK> \
  -addr ircs://<IRC_SERVER>:6697 \
  -nick <IRC_NICK> \
  -username <IRC_IDENT> \
  -realname '<IRC_REAL_NAME>' \
  -enabled true

# Create it disabled while staging configuration
sj-user <SOJU_USER> network create \
  -name <NETWORK> \
  -addr ircs://<IRC_SERVER>:6697 \
  -enabled false

# Update a network; this disconnects and reconnects the upstream network
# Force a reconnect without changing saved settings (enabled networks only)
sj-user <SOJU_USER> network update <NETWORK>

sj-user <SOJU_USER> network update <NETWORK> -enabled true
sj-user <SOJU_USER> network update <NETWORK> -enabled false
sj-user <SOJU_USER> network update <NETWORK> -nick <IRC_NICK>
sj-user <SOJU_USER> network update <NETWORK> -username <IRC_IDENT>
sj-user <SOJU_USER> network update <NETWORK> -realname '<IRC_REAL_NAME>'

# Change a network's address or server password
sj-user <SOJU_USER> network update <NETWORK> -addr ircs://<IRC_SERVER>:6697
sj-user <SOJU_USER> network update <NETWORK> -pass '<UPSTREAM_SERVER_PASSWORD>'

# Optional network behavior
sj-user <SOJU_USER> network update <NETWORK> -auto-away true
sj-user <SOJU_USER> network update <NETWORK> -auto-away false

# Last-resort identify command for networks without usable SASL.
# Avoid storing plaintext passwords in configuration when possible.
sj-user <SOJU_USER> network update <NETWORK> -connect-command 'PRIVMSG NickServ :IDENTIFY <IRC_ACCOUNT_PASSWORD>'

# Clear all saved connect commands
sj-user <SOJU_USER> network update <NETWORK> -connect-command ''

# Send an exact raw IRC line to a network
sj-user <SOJU_USER> network quote <NETWORK> 'PRIVMSG NickServ :STATUS <IRC_NICK>'

# Delete a network; this disconnects and forgets it
sj-user <SOJU_USER> network delete <NETWORK>
~~~

The -pass option is the IRC server password, if the IRC network has one. It is not normally the NickServ password and is not the Soju bouncer password.

## Upstream authentication

### Upstream SASL PLAIN

Use this when a network does not support CertFP or when you intentionally want the upstream NickServ account/password stored in Soju:

~~~bash
sj-user <SOJU_USER> sasl set-plain \
  -network <NETWORK> \
  <IRC_ACCOUNT> \
  '<IRC_ACCOUNT_PASSWORD>'

sj-sasl <SOJU_USER> <NETWORK>
~~~

The IRC account and password are upstream network credentials. They are different from the Soju bouncer username/password used by Relay.

Changing SASL settings requires an upstream reconnect to apply: `sj-user <SOJU_USER> network update <NETWORK>`. Then verify with `sasl status`. Reconnecting only Relay does not reconnect the upstream network.

### Upstream CertFP / SASL EXTERNAL

Generate and use a client certificate for one Soju user/network:

~~~bash
sj-user <SOJU_USER> certfp generate -network <NETWORK>
sj-cert <SOJU_USER> <NETWORK>
sj-sasl <SOJU_USER> <NETWORK>
~~~

certfp generate is the Soju command for SASL EXTERNAL. Do not use sasl set-external; that command does not exist in current Soju releases.

The certificate is scoped to the Soju user/network. If two separate Soju users connect to the same upstream NickServ account, each user's certificate must be registered separately, or both users must use upstream SASL PLAIN.

Reconnect upstream after generation so the server sees the certificate. Register the client fingerprint using that network's NickServ instructions, then reconnect upstream again to test automatic login. Generating another certificate replaces the identity you need to register.

Only when intentionally disabling SASL entirely, remove its stored credentials:

~~~bash
sj-user <SOJU_USER> sasl reset -network <NETWORK>
~~~

Do not run `sasl reset` as a cleanup step after configuring EXTERNAL: it disables that method too. Switching methods may remove the old stored credentials/certificate; plan to register a replacement certificate if you later regenerate one.

The certfp fingerprint output is the client certificate fingerprint. Do not confuse it with the -certfp network option, which pins an upstream server certificate and uses a different purpose/format.

To pin an upstream server certificate when a network uses a self-signed certificate, obtain its SHA-512 fingerprint from the administrator over a trusted channel. Do not blindly trust a fingerprint from an error or an unverified connection. Pinning replaces normal CA validation:

~~~bash
sj-user <SOJU_USER> network update <NETWORK> -certfp <UPSTREAM_SERVER_SHA512_FINGERPRINT>
~~~

If that network now uses a valid public certificate for its configured hostname, remove the old server pin to restore CA validation:

~~~bash
sj-user <SOJU_USER> network update <NETWORK> -certfp=
~~~

This does not remove the client certificate used for SASL EXTERNAL. Likewise, `-pass=` clears a stale upstream IRC PASS value; do that only when the server no longer requires it.

## Channel commands

You can join channels normally from Relay. Connect to the desired Soju network and use Relay's usual `/join #channel` command. Soju saves the joined channel and automatically joins it again on the next connection. You do not need to use `sojuctl` or BouncerServ for ordinary channel use.

~~~bash
# List saved channels
sj-channels <SOJU_USER> <NETWORK>
~~~

The following commands are optional administrative alternatives. Use them when no Relay client is attached, when you need to manage a detached channel, or when administering another Soju user:

~~~bash
# Raw IRC fallback; normally use Relay's /join instead
sj-user <SOJU_USER> network quote <NETWORK> 'JOIN #<CHANNEL>'
~~~

From a client already attached to the network, you can also use BouncerServ for channel administration, including `channel create`:

~~~irc
/msg BouncerServ channel create '#<CHANNEL>'
/msg BouncerServ channel update '#<CHANNEL>' -detached true
/msg BouncerServ channel update '#<CHANNEL>' -detached false
/msg BouncerServ channel delete '#<CHANNEL>'
~~~

Useful detached-channel modes include message, highlight, none, and default. Run channel update from a client attached to the selected network:

~~~irc
/msg BouncerServ channel update '#<CHANNEL>' -detached true -relay-detached highlight -reattach-on highlight
~~~

## Fast status checks

~~~bash
sudo systemctl is-active soju
sj user status
sj-networks <SOJU_USER>
sj-sasl <SOJU_USER> <NETWORK>
sj-channels <SOJU_USER> <NETWORK>
sj-logs 80
~~~

Interpretation:

~~~text
[connected]                         Soju has an upstream connection.
[disabled]                          The network is intentionally disabled.
Unauthenticated on upstream network  Soju has not recorded an upstream account; verify with services/logs.
SASL PLAIN enabled                   Soju is configured with upstream account/password authentication.
SASL EXTERNAL enabled                Soju is configured to present its client certificate.
~~~

## Common fixes

~~~bash
# Network is disabled
sj-user <SOJU_USER> network update <NETWORK> -enabled true

# Check why the service failed
sudo systemctl reset-failed soju
sudo journalctl -u soju -n 100 --no-pager

# Check TLS and hostname validation from a trusted client
openssl s_client \
  -connect <SOJU_HOST>:6697 \
  -servername <SOJU_HOST> \
  -verify_hostname <SOJU_HOST> \
  -verify_return_error \
  -CAfile /etc/ssl/certs/ca-certificates.crt \
  </dev/null
~~~

If a network reports Unauthenticated on upstream network (or a TUI says the account was not reported), verify with NickServ/WHOIS and recent logs before changing credentials. Some authentication flows do not report the account to Soju. A configured SASL mechanism or a certificate in NickServ's list alone does not prove automatic login succeeded.

If an upstream network reports TAGMSG Unknown command, the network may advertise CLIENTTAGDENY=*. Use a Relay build that honors that capability; do not expose raw upstream details or passwords in debug logs.

## Official references

- [soju manual](https://soju.im/doc/soju.1.html)
- [sojuctl manual](https://soju.im/doc/sojuctl.1.html)
