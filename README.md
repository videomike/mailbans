# mailbans

Two small CLI tools for triaging fail2ban-driven mail-server bans on iRedMail (and compatible Postfix + Dovecot setups). Built to answer the practical question: *“who is locked out right now, why, and which of their devices is to blame?”*

## Tools

### `mailbans`

Lists every currently banned IP across all relevant fail2ban jails (`dovecot`, `postfix`, `postfix-sasl`, `sogo`, `nginx-http-auth`) and enriches each entry with:

- Reverse-DNS of the banned IP
- Auth-failure user / `sasl_username` aggregation (which accounts the IP attempted)
- Total time span of the failure burst (first → last failure, with duration)
- **Per mail client**, identified via Dovecot’s logged IMAP-ID command (RFC 2971) and linked to failures via the dovecot session-id: name, version, OS, plus failure count and time range. The client with the **most recently observed failure** is highlighted in red — useful when fixing passwords across multiple devices: at the end, only the one device still misconfigured stays red.
- TLS cipher and SASL method of the most recent attempt

Sample output:

```
[dovecot] 91.21.184.160  (p5b15b8a0.dip0.t-ipconnect.de)
    70 user=<office@example.com>
     5 sasl_username=office@example.com
   75 Failures, 2026-04-28  18:21:22 – 23:48:23 (5 h 27 min)
   Client: Mac OS X Notes 4.13 (Mac OS X 26.3.1) — 18× 18:21:24 – 23:45:23 (5 h 23 min)   ← red
   Client: Mac OS X Mail 16.0 (Mac OS X 26.3.1) — 4× 18:21:22 – 21:53:31 (3 h 32 min)
   Client unknown (kein ID-Command): 53× 18:21:26 – 23:48:23 (5 h 26 min)
   Last: TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384, method=PLAIN
```

There is also a `--debug-ip <IP> [<IP>...]` mode that runs the full enrichment on arbitrary IPs (for testing, or post-hoc analysis after the ban has expired).

### `mailunban`

One-line wrapper around `fail2ban-client unban <IP>` — removes an IP from **all** jails in a single call, instead of having to guess which jail caught it.

```
mailunban 91.21.184.160
```

## Requirements

- iRedMail (or any Postfix + Dovecot stack with the same fail2ban jail names)
- `fail2ban` reachable as `fail2ban-client` (root)
- Python 3 (uses only standard library)
- Dovecot logging IMAP-ID. On modern Dovecot this happens automatically as separate `imap-login: ID sent: ...` lines. If you want to be explicit, set in your dovecot config:

  ```
  imap_id_log = name version os os-version vendor
  ```

  On iRedMail this belongs in `/etc/dovecot/iredmail/99-local.conf` (iRedMail does **not** include the standard `/etc/dovecot/conf.d/`).

## Install

```sh
sudo install -m 0755 mailbans  /usr/local/bin/mailbans
sudo install -m 0755 mailunban /usr/local/bin/mailunban
```

## Caveats

- Bans on iRedMail are **per IP, not per account** — a user with the right password can still log in from a different IP. `mailbans` reflects this: it groups by IP.
- The session-id linkage between IMAP-ID and auth-failure relies on Dovecot logging both within the same TCP connection. Clients that reconnect frequently (Apple Mail, iOS) often only send `ID` on the first connection; subsequent reconnects show up in the `Client unknown` bucket. The set of *known* clients is still indicative — the unknowns almost always belong to one of them.
- Postfix-side bans (e.g. `postfix` jail) don’t carry a Dovecot session-id; the per-client breakdown only applies to Dovecot-driven bans.

## License

MIT — see [LICENSE](LICENSE).
