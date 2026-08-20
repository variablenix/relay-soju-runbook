# Relay + Soju Runbook

Operational documentation for running Relay with [Soju](https://soju.im/) as a persistent IRC bouncer.

This runbook is intended for self-hosted deployments where Soju maintains the upstream IRC connections and Relay connects to Soju as an IRC client.

## What this covers

- Installing and configuring Soju on Debian/Ubuntu.
- Private/VPN-reachable IRC access.
- Let's Encrypt certificates using DNS-01 validation.
- Automatic certificate renewal and Soju TLS reloads.
- Upstream SASL PLAIN and CertFP/SASL EXTERNAL authentication.
- Migrating existing Relay networks to Soju.
- Multiple Relay accounts and persistent channel connections.
- Common administration, verification, rollback, and troubleshooting commands.

## Documents

Read these in order:

1. [Relay + Soju setup and migration guide](relay-soju-setup-and-migration.md) — complete installation and migration procedure.
2. [Soju administration cheat sheet](soju-admin-cheatsheet.md) — day-to-day commands, helper functions, and troubleshooting.

## High-level architecture

~~~text
Relay browser/app
        |
        | TLS + SASL PLAIN
        v
Soju IRC bouncer
        |
        | TLS + upstream authentication
        v
IRC networks
~~~

The important separation is:

- Relay authenticates to Soju with the Soju account and password.
- Soju authenticates to each IRC network with upstream SASL credentials or its own registered client certificate.

## Migration principle

Keep the existing Relay data and networks until the Soju connection is verified. Prepare Soju first, edit each Relay network in place, save once, then verify the upstream account and channels.

After migration, restarting only Relay should reconnect the Relay client to Soju without making the upstream IRC network see a quit/part. Soju must remain running as its separate systemd service.

## Security notes

The examples use placeholders. Before publishing or sharing this repository, confirm that it contains no:

- IRC or Soju passwords.
- API tokens or private keys.
- Private IP addresses or internal hostnames.
- Personal nicknames, account names, or certificate fingerprints.

Keep Soju's administrative socket private, restrict TCP/6697 to trusted clients, and avoid enabling debug logging during normal operation.

## Status

This is an operational runbook, not a turnkey installer. Review provider-specific DNS plugin options, IRC network authentication requirements, and local firewall/DNS behavior before applying commands.
