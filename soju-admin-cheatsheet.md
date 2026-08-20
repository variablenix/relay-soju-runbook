# Soju administration cheat sheet

This assumes a Debian/Ubuntu installation using the packaged service and configuration file:

~~~text
/etc/soju/config
~~~

The examples use placeholders such as <SOJU_USER>, <NETWORK>, and <IRC_ACCOUNT>. Replace them before running commands.

## Quick variables and helper functions

Run this once in a shell. The functions disappear when the shell closes; copy them into ~/.bashrc or ~/.zshrc if desired.

~~~bash
export SOJU_CONFIG=/etc/soju/config

sj() {
  sudo -n sojuctl -config "$SOJU_CONFIG" "$@"
}

sj-user() {
  local user="$1"
  shift
  sj user run "$user" "$@"
}

sj-networks() {
  sj-user "$1" network status
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
  sudo -n journalctl -u soju -n "${1:-100}" --no-pager
}

sj-follow-logs() {
  sudo -n journalctl -u soju -f
}
~~~

No separate sudo password-validation step is required when your sudo policy allows passwordless Soju administration. Test the helper with:

~~~bash
sj help
~~~

The `-n` option prevents password prompts and fails immediately if the command is not covered by your sudo policy. If your system requires a password, remove `-n` from the helper or run the command as a sudo-capable administrator. Soju's admin socket normally requires root or suitable permissions. Keep sudo password protection unless you have a deliberate, narrowly scoped sudoers policy.

## Optional: passwordless sudo for Soju administration

The setup used in this runbook grants the Linux account passwordless access to `sojuctl` only. The sudoers filename is arbitrary, but this setup uses `/etc/sudoers.d/sojuctl`. Do not grant `NOPASSWD: ALL`.

This must be installed by `root` or by an administrator who already has permission to edit sudoers. Replace `<LINUX_USER>` with the Linux login that will administer Soju:

~~~bash
# Run this as root. If you already have suitable sudo access, prefix it with sudo.
visudo -f /etc/sudoers.d/sojuctl
~~~

Put this exact rule in the editor:

~~~sudoers
# /etc/sudoers.d/sojuctl
<LINUX_USER> ALL=(root) NOPASSWD: /usr/bin/sojuctl -config /etc/soju/config *
~~~

Save and validate it as root:

~~~bash
chmod 0440 /etc/sudoers.d/sojuctl
visudo -cf /etc/sudoers.d/sojuctl
~~~

Then test from the target account. This must not prompt for a password:

~~~bash
sudo -n /usr/bin/sojuctl -config /etc/soju/config help
sudo -n /usr/bin/sojuctl -config /etc/soju/config user status
~~~

This rule covers the `sj`, `sj-user`, `sj-networks`, `sj-channels`, `sj-sasl`, and `sj-cert` helpers. It does not cover `systemctl`, `journalctl`, `sed`, or other root commands. The `sj-logs` helpers therefore fail cleanly with `sudo -n` unless you separately add a narrow journalctl rule. If you cannot become root or do not have an existing sudo-capable administrator, you cannot create this exception from the target account alone.

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

Changing SASL settings requires an upstream reconnect to apply. Verify with sasl status.

### Upstream CertFP / SASL EXTERNAL

Generate and use a client certificate for one Soju user/network:

~~~bash
sj-user <SOJU_USER> certfp generate -network <NETWORK>
sj-cert <SOJU_USER> <NETWORK>
sj-sasl <SOJU_USER> <NETWORK>
~~~

certfp generate is the Soju command for SASL EXTERNAL. Do not use sasl set-external; that command does not exist in current Soju releases.

The certificate is scoped to the Soju user/network. If two separate Soju users connect to the same upstream NickServ account, each user's certificate must be registered separately, or both users must use upstream SASL PLAIN.

Disable stored upstream SASL credentials when they are no longer needed:

~~~bash
sj-user <SOJU_USER> sasl reset -network <NETWORK>
~~~

The certfp fingerprint output is the client certificate fingerprint. Do not confuse it with the -certfp network option, which pins an upstream server certificate and uses a different purpose/format.

To pin an upstream server certificate when a network uses a self-signed certificate, use the server's SHA-512 fingerprint:

~~~bash
sj-user <SOJU_USER> network update <NETWORK> -certfp <UPSTREAM_SERVER_SHA512_FINGERPRINT>
~~~

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
Unauthenticated on upstream network  The TCP/TLS connection works, but services did not identify the account.
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
  </dev/null 2>&1 | grep -E 'Verify return code|subject=|issuer='
~~~

If a network reports Unauthenticated on upstream network, check sasl status, then either configure sasl set-plain or identify once with NickServ and register the Soju certificate if the network supports CertFP.

If an upstream network reports TAGMSG Unknown command, the network may advertise CLIENTTAGDENY=*. Use a Relay build that honors that capability; do not expose raw upstream details or passwords in debug logs.

## Official references

- [soju manual](https://soju.im/doc/soju.1.html)
- [sojuctl manual](https://soju.im/doc/sojuctl.1.html)

