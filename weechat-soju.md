# WeeChat + Soju

[WeeChat](https://weechat.org/) is a terminal IRC client. Use it instead of, or alongside, Relay: WeeChat connects directly to Soju over IRC/TLS. Neither the Relay web app nor an HTTP reverse proxy is needed for that connection. WeeChat's own optional “relay” plugin is a different component from the Relay web IRC client in this runbook; neither is required here.

## Before you start

Complete the [Soju setup](relay-soju-setup-and-migration.md) through upstream authentication first. You need a reachable Soju TLS hostname/port, a Soju account/password, and an existing network name. Check DNS and routing from the machine running WeeChat, which may differ from your Relay server.

Install WeeChat using your distribution's packages and start `weechat`. The examples use current WeeChat 4.x syntax. Consult `/help server` and `/help secure` if your installed release differs. Commands below run **inside WeeChat**, not in a shell.

## Connect one network

This explicit setup needs no additional scripts. Replace all angle-bracket placeholders; `soju-example` is only a local WeeChat server label.

First store the **Soju password**, not the upstream NickServ password. If secured data already has a passphrase, keep it; otherwise set one before saving credentials:

```text
/secure passphrase <NEW_WEECHAT_SECURE_DATA_PASSPHRASE>
/secure set soju_password <SOJU_PASSWORD>
```

Do not paste real credentials into shared logs or screenshots. Secured data without a passphrase is not encrypted at rest. See [WeeChat's secured-data documentation](https://weechat.org/files/doc/stable/weechat_user.en.html#secured_data).

Configure the new connection:

```text
/server add soju-example <SOJU_HOST>/6697 -tls
/set irc.server.soju-example.tls_verify on
/set irc.server.soju-example.tls_fingerprint ""
/set irc.server.soju-example.nicks "<IRC_NICK>"
/set irc.server.soju-example.username "<SOJU_USER>/<NETWORK>@weechat"
/set irc.server.soju-example.sasl_mechanism plain
/set irc.server.soju-example.sasl_username "<SOJU_USER>/<NETWORK>@weechat"
/set irc.server.soju-example.sasl_password "${sec.data.soju_password}"
/set irc.server.soju-example.sasl_fail disconnect
/set irc.server.soju-example.password ""
/set irc.server.soju-example.command ""
/save
/connect soju-example
```

Keep certificate verification enabled. The empty fingerprint selects normal CA/hostname verification rather than a pin; fix trust or hostname errors instead of bypassing them. Empty `password` and `command` settings avoid inherited IRC PASS or post-connect identification commands. The [WeeChat user guide](https://weechat.org/files/doc/stable/weechat_user.en.html) documents these server options.

`<NETWORK>` must match the network saved in Soju. The stable `@weechat` suffix identifies this client for per-client backlog tracking; use distinct names such as `@weechat-laptop` for separate installations. Repeat with another local server label and network selector for each network. These are Soju's [client connection conventions](https://soju.im/doc/soju.1.html#DESCRIPTION), not upstream NickServ credentials.

After confirming login, optionally enable startup connection:

```text
/set irc.server.soju-example.autoconnect on
/save
```

WeeChat will need its secured-data passphrase when starting. See the [official quick start](https://weechat.org/files/doc/weechat/stable/weechat_quickstart.en.html) for navigation and channel commands.

## Verify and troubleshoot

- Confirm the server buffer reports a successful TLS connection and SASL login. If authentication fails, check the Soju password and network selector.
- Saved channels should appear; use `/join #example` in the appropriate network buffer for another channel. Soju remembers joined channels. Parting a channel changes the shared upstream session, including what other clients see.
- WeeChat authenticating to Soju does **not** prove upstream NickServ authentication. Use the [administration cheat sheet](soju-admin-cheatsheet.md) and the network's WHOIS/NickServ checks for that separate connection.
- Disconnecting/reconnecting WeeChat only reconnects the client to the bouncer. After changing upstream SASL or generating an upstream certificate, reconnect that network **in Soju**; do not restart every network unnecessarily.
- If migrating from a direct WeeChat connection, back up its configuration and disconnect the old direct server before enabling the corresponding staged Soju network. Keep the old profile until verification is complete.

## Optional automatic network discovery

Prefer automatic network setup? Soju's [client notes](https://github.com/emersion/soju/blob/master/contrib/clients.md#weechat) reference the optional [soju.py script](https://weechat.org/scripts/source/soju.py.html/) for discovering and connecting to your networks, plus `read_marker.py` for read-marker synchronization. Follow their current instructions and review third-party scripts before installation. Do not layer automatic discovery over duplicate manual profiles for the same networks.

The manual approach above remains usable without either script. Capability/history support depends on the installed client and scripts; do not assume every Soju extension is implemented.

## Compatibility

These examples use WeeChat 4.x. If an option differs in your version, check WeeChat's built-in `/help` before applying it.
