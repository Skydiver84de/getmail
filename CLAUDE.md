# CLAUDE.md

Project context and conventions for working in this repository. User-facing setup
lives in [README.md](README.md), the change history in [CHANGELOG.md](CHANGELOG.md).

## What this is

A small tool that retrieves mail from IMAP accounts via **IMAP IDLE** (instant, not
polled) and delivers it into a local Mailcow/Dovecot mailbox. One Python script,
`Dockerfiles/getmail_imap2lmtp.py`, run as a Docker sidecar inside the Mailcow Docker
network. Each `[section]` in `conf/settings.ini` is one IMAP account, handled by its
own `Getmail` thread; `[DEFAULT]` applies to all.

## Fork relationship

- `origin` = `Skydiver84de/getmail` (this repo, a real GitHub fork).
- `upstream` = `mkrumbholz/getmail`; root of the network is `christianbur/getmail`.
- Pull upstream updates with `git fetch upstream && git merge upstream/master`
  (upstream's default branch is `master`; ours is `main`).
- **This fork's own contribution is the reconnect hardening** (see below). Everything
  else comes from upstream — when touching upstream code, prefer keeping it diff-able.

## Architecture

- Single script, no module structure. Per-account `threading.Thread` running an IMAP
  IDLE loop; `idle_check()` polls, `EXISTS` / a periodic safety-fetch triggers a fetch.
- Delivery is pluggable via `send_protocol`: **`lmtp`** (default, local Dovecot, no TLS)
  or **`smtp`** (STARTTLS, intended to feed a **MailPiler** email archive — the example
  ships `smtp_hostname: piler`). These are two separate *destinations* (Mailcow vs. an
  archiver), not two ways into Mailcow.
- Optional inline **ClamAV** scan (`clamd_active`) runs in `imap_fetch_mail` **before**
  the `send_protocol` dispatch, so it applies to whichever destination is chosen.
- The source is authoritative: a message is only deleted/moved on the source **after**
  successful delivery. A spurious reconnect/refetch is therefore always safe — never
  add a code path that deletes before delivery confirms.

## Domain facts worth remembering (don't re-derive these)

- **LMTP delivery bypasses Mailcow's rspamd + ClamAV.** Those run as Postfix `smtpd`
  milters (SMTP-gateway path only); LMTP to Dovecot (port 24) is the final delivery
  step, downstream of the milters. Sieve still runs (Dovecot LMTP does it). The inline
  ClamAV in this script exists specifically to plug the virus half of that gap; spam is
  not scanned on the LMTP path. Because the scan runs before the protocol dispatch it
  also covers the SMTP path, but it is most useful for LMTP→Mailcow; for an archiving
  target it is questionable (archives usually keep everything, infected mail included).
- **The SMTP path targets a MailPiler archive, not Mailcow.** `smtp_hostname: piler` in
  the example confirms it. MailPiler ingests mail over SMTP and just stores it (it is an
  archiver, not an MTA enforcing sender auth), so there is no SPF/DKIM rejection problem.
  Do **not** mistake SMTP for a way to re-inject into Mailcow with scanning: re-injecting
  into the same Mailcow is not a clean spam solution (trusted → rspamd skips scanning;
  untrusted → the original sender's SPF/DKIM fail because the connecting IP is this
  container, penalizing legitimate mail).
- **INTERNALDATE (received time) is not preserved** on the LMTP/SMTP path — Dovecot sets
  it to delivery time. The visible `Date:` header is preserved. Preserving INTERNALDATE
  would require IMAP `APPEND` (loses Sieve) or Dovecot dsync (both ends Dovecot); a known
  trade-off, intentionally not solved here.

## Reconnect model (this fork's core)

The goal: a dropped link (e.g. lost Wi-Fi) is detected within ~2 min and recovers within
seconds, instead of sitting half-open until the OS TCP timeout (~15 min). Four pieces in
`getmail_imap2lmtp.py`:

1. **TCP keepalive** (`enable_tcp_keepalive`) — the primary detector. During IDLE the
   connection is silent, so only active kernel probing notices a blackholed link.
   `TCP_KEEPIDLE/INTVL/CNT` are Linux-only and guarded with `getattr` (container is
   Alpine/Linux).
2. **Socket timeout** (constructor `timeout=`) — backstop so no blocking call hangs
   forever.
3. **Capped exponential backoff + jitter** (in `run()`) — replaced the old uncapped
   `60 * counter**2`. Exponent is clamped so a permanently-down server can't blow up
   `2**counter`.
4. **`safe_close()`** — `shutdown()` (no LOGOUT round-trip) before reconnecting, to avoid
   leaking a socket/FD on every reconnect.

All knobs are config options with fallbacks → existing `settings.ini` keeps working.

## Conventions

- **Language:** code comments, README and CHANGELOG are in **English** (match the
  existing upstream code).
- **New config options** go into `__init__` with a `configparser` `fallback=` (never
  break an existing `settings.ini`) and are documented in `conf/settings.ini.example`.
- Before adding functionality, `grep` first — reuse existing patterns/field names rather
  than adding a parallel mechanism.

## Validation

After changing `getmail_imap2lmtp.py`:

```bash
python3 -m py_compile Dockerfiles/getmail_imap2lmtp.py      # syntax
# stronger: import the module (catches def-level NameErrors), needs imapclient + humanfriendly
python3 -c "import importlib.util as u; s=u.spec_from_file_location('m','Dockerfiles/getmail_imap2lmtp.py'); mod=u.module_from_spec(s); s.loader.exec_module(mod)"
```

Real IMAP/IDLE and the actual reconnect behaviour can only be tested in the container on
the target Mailcow host (simulate a link drop). The `imapclient` APIs relied on
(`IMAPClient(timeout=...)`, `.socket()`, `.shutdown()`) exist as of imapclient 3.1.0.
