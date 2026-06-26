# getmail (mailcow / MailPiler)

Retrieve mail from IMAP accounts via **IMAP IDLE** (instant push, not polled) and deliver
it into a local **Mailcow** mailbox (Dovecot LMTP) or a **MailPiler** archive (SMTP). Runs
as a Docker sidecar inside the Mailcow network. This fork adds **reconnect hardening** so a
dropped link (e.g. lost Wi-Fi) is detected and recovered automatically.

## What it does

- Watches one IMAP folder per account with IDLE → new mail is fetched immediately.
- Delivers via Dovecot **LMTP** (Sieve rules apply) or **SMTP** to MailPiler.
- Empties the source folder (delete) or moves fetched mail to a folder there.
- Optional **ClamAV** scan before delivery (LMTP bypasses Mailcow's own scanner).
- Adds `X-getmail-retrieved-from-mailbox-*` headers for Sieve filtering.

## Install

```bash
cd /opt
git clone https://github.com/Skydiver84de/getmail.git
cd /opt/getmail
# check for an existing override file first!
cp mailcow-dockerized_docker-compose.override.yml /opt/mailcow-dockerized/docker-compose.override.yml
cp conf/settings.ini.example conf/settings.ini
nano conf/settings.ini
cd /opt/mailcow-dockerized && docker compose up -d
docker compose logs -f getmail-mailcow
```

## Configuration

`conf/settings.ini` — `[DEFAULT]` applies to all accounts; each `[section]` is one IMAP
account (usually only host/user/password/recipient differ):

```ini
[example@gmx.de]
imap_hostname:   imap.gmx.net
imap_username:   example@gmx.de
imap_password:   xxx
lmtp_recipient:  me@my-mailcow-domain.tld   # target mailbox in Mailcow
# imap_sync_folder defaults to INBOX; one folder per account (add a 2nd section for Junk)
```

Key options:

| Option | Default | Meaning |
|---|---|---|
| `send_protocol` | `lmtp` | `lmtp` → Mailcow/Dovecot, `smtp` → MailPiler archive |
| `imap_move_enable` | `False` | `True` = move to `imap_move_folder` instead of deleting from the source |
| `clamd_active` | `False` | scan via ClamAV before delivery |

**Reconnect hardening** (optional, sensible defaults): TCP keepalive, a socket-timeout
backstop and a capped exponential backoff — a dead link is detected in ~2 min and recovers
in seconds. Individual knobs are documented in
[`conf/settings.ini.example`](conf/settings.ini.example).

**Timezone:** set `TZ` (e.g. `TZ=Europe/Berlin`) in the environment or `docker-compose.yml`.

**Sieve:** filter on the injected `X-getmail-retrieved-from-mailbox-user` header
(Mailcow → Mail Setup → Filters).

## Credits

- [christianbur/getmail](https://github.com/christianbur/getmail) — original author
- [mkrumbholz/getmail](https://github.com/mkrumbholz/getmail) — IMAP-IDLE robustness, SMTP/MailPiler delivery, ClamAV scanning (base of this fork)
