# Changelog

This fork of [mkrumbholz/getmail](https://github.com/mkrumbholz/getmail) records the
changes made on top of upstream. The full upstream history is preserved in the git log.

## Unreleased

### Added — reconnect hardening

- **TCP keepalive** on the IMAP socket (`imap_keepalive`, `imap_keepalive_idle`,
  `imap_keepalive_intvl`, `imap_keepalive_cnt`) so a dead link (e.g. lost Wi-Fi) is
  detected during IDLE silence within ~2 minutes, instead of staying half-open until the
  OS TCP timeout (~15 min).
- **Socket timeout backstop** (`imap_socket_timeout`) so no blocking call
  (login/fetch/idle_done) can hang forever.
- **Capped exponential backoff with jitter** (`reconnect_backoff_base`,
  `reconnect_backoff_max`) for the restart loop, replacing the previous uncapped
  `60 * counter**2` wait — a flapping link now recovers in seconds.
- `safe_close()` — best-effort teardown of a possibly-dead connection (via `shutdown()`,
  no LOGOUT round-trip) before reconnecting, to avoid leaking a socket/FD on every
  reconnect.

All new settings are optional with sensible defaults; an existing `settings.ini` works
unchanged. See [`conf/settings.ini.example`](conf/settings.ini.example).

### Forked from

- **mkrumbholz/getmail** — adds IMAP-IDLE robustness (periodic safety-fetch, graceful
  IDLE-timeout reconnect, envelope-sender sanitization), an optional SMTP delivery path
  and optional ClamAV scanning on top of the original **christianbur/getmail**.
