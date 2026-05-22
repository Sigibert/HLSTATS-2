# HLstatsZ — Real-time stats daemon for GoldSrc, Source, and Source 2 (CS2)

HLstatsZ is the only active HLstatsX fork with ***native*** **Counter-Strike 2 support** via Valve's HTTP log transport. It stays fully backward-compatible with all UDP-based Source and GoldSrc games.
 
> **Why Perl + Mojolicious?**  
> Mojolicious provides the async engine: actively maintained, truly non‑blocking, lightweight, and built with a clean, modern API. It handles timers, events, sockets, and async I/O without blocking — exactly what an event‑driven daemon needs.  
> Perl provides the regex horsepower: this daemon performs continuous, high‑volume, regex‑heavy log parsing, which is precisely what Perl’s mature text‑processing engine excels at.  
> Together, they deliver a fast, stable, non‑blocking daemon designed for long‑running, high‑throughput log processing at scale.  
---

## What makes HLstatsZ different

- **Hardened code and config loading** — Consistent use of `use strict` ensures no hidden bugs; database settings are applied through symbolic references rather than `eval`; raw log bytes are always decoded to prevent any compromised DB value from injecting code into the running daemon.
- **Dual-protocol listener on one port** — Mojo::IOLoop watches a raw UDP socket and a Mojo::Server::Daemon (HTTP) simultaneously. GoldSrc/Source logs arrive over UDP; CS2 logs arrive over HTTP. Both feed the same parser.
- **Non-blocking by design** — UDP and HTTP each have their own queue (`@UDPQ` / `@HTTPQ`) drained via `Mojo::IOLoop->next_tick()`. A burst of CS2 traffic never starves UDP servers or vice versa, and neither blocks the main loop.
- **Periodic tasks stay low-priority** — cleanup, stats snapshots, and DB flushes run on `Mojo::IOLoop->recurring()` timers, always yielding to incoming log traffic.
- **Buffered batch INSERTs** — events queue up and flush in bulk (configurable size, default 50), reducing InnoDB write pressure at high log rates.
- **CS2 userid quirk handling** — CS2 initially assigns every connecting player `userid=0`, then reassigns the real userid. The daemon tracks and migrates player keys transparently.
- **Automatic socket health-check** — detects stale connections and reconnects without restarting.
- **Built-in daily cronjob** — awards and bans run automatically; no external cron needed.
- **MariaDB or MySQL** — InnoDB-optimised queries, fastest collation (schema ≥ 84).
- **GeoIP2** — optional binary GeoIP2-City lookup for player location.
- **Cross-platform** — Linux and Windows (Strawberry Perl).

---

## Requirements

**Perl modules:**
- [Mojolicious](https://metacpan.org/pod/Mojolicious)
- [GeoIP2](https://metacpan.org/pod/GeoIP2) + GeoIP2::Database::Reader *(optional)*
- [DBD::MariaDB](https://metacpan.org/pod/DBD::MariaDB) *(optional, or DBD::mysql)*

**Install Mojolicious:**
```sh
# Linux
curl -L https://cpanmin.us | perl - -M https://cpan.metacpan.org -n Mojolicious

# Windows (Strawberry Perl)
cpan install Mojolicious
```

**Windows Perl:** [Strawberry Perl 5.38.0.1 64-bit](https://strawberryperl.com/release-notes/5.38.0.1-64bit.html) (must be compiled with MySQL support)

---

## Installation & Setup

### Database
Use your existing db or import the schema:
```sh
mysql -u root -p hlstats < sql/install.sql
```

### Firewall Port
```
allow inbound traffic on TCP 27500 and UDP 27500
```
### Daemon configuration
Edit `hlstats.conf`:
```ini
DBHost     "localhost"
DBName     "hlstats"
DBUsername "root"
DBPassword "yourpassword"
Port       "27500"
```

Built-in daily cronjob (optional):
```ini
PathPerl   "/usr/bin/perl"
PathAwards "/usr/bin/hlstats/hlstats-awards.pl"
PathBans   "/usr/bin/hlstats/Importbans/importbans.pl"
```

### Game server configuration

**GoldSrc (Half-Life 1)**
<br>Recommended plugin: [hlstats_cs.amxx](https://github.com/SnipeZilla/HLSTATS-2/blob/main/plugins/amxmodx/plugins/hlstats_cs.amxx)

```
rcon_password "PaSsWoRd"
log on
logaddress_delall
logaddress_add 64.74.97.164 27500
```

**Source (CS:GO, TF2, L4D2 …)**
<br>Recommended plugin: [hlstatsx.smx](https://github.com/SnipeZilla/HLSTATS-2/blob/main/plugins/sourcemod/plugins/hlstatsx.smx)
```
rcon_password "PaSsWoRd"
log on
logaddress_delall
logaddress_add "64.74.97.164:27500"
```

**Source 2 — Counter-Strike 2**
<br>Recommended plugin: [HLstatsZ CS2 plugin](https://github.com/SnipeZilla/CS2-HLstatsX-Plugin) or [HLstatsZ Classics plugin](https://github.com/SnipeZilla/HLstatsZ-Classics)

```
rcon_password "PaSsWoRd"
log on
logaddress_delall_http
logaddress_add_http "http://64.74.97.164:27500"
sv_visiblemaxplayers 32
```

Add `-usercon` to your server launch options.

Optional secondary address for testing against a different daemon:
```
logaddress_add_http_delayed 0.0 "http://64.74.97.164:27501"
```

---

## Running

```sh
# Linux
./run_hlstats

# Windows
run-hlstats.ps1
```

---

## Web frontend

Any HLstats‑compatible frontend will run — but most legacy ones are outdated, unmaintained or poorly patched and look 15+ years old.

The good news is that fully modern, actively maintained frontends are available today — and they drop in with the same database structure you’re already using.

**Recommended modern frontends**  
 - [HLstatsZ web](https://github.com/SnipeZilla/hlstatsz-web)  
 - [HLstatsX CE - Laravel Rebase](https://github.com/Royal-Multi-Gamers/hlstatsx-community-edition-laravel)  

 Both are modern, responsive, and drop‑in compatible.  
 And more are currently in development!

---

## FAQ

**Where should I run HLstatsZ?**  
On the machine that hosts your SQL database — not on your game server.

**Why don't daily awards run?**  
Set `PathPerl` and `PathAwards` / `PathBans` in `hlstats.conf`. The built-in cronjob fires automatically once per day.

**How do I ignore Warmup or End-of-Round?**  
In the server's database entry set `BonusRoundIgnore = 1`.

**Does it support multiple servers?**  
Yes. Add each server to the database; the daemon handles all of them in a single process.  
The clever architecture allows for virtually unlimited servers from just one daemon!

**Who's using it?**  
HLstatsZ has already been widely adopted by many hosting companies, internet cafes, and various gaming communities around the world.

---

## Lineage

```
HLstats (Simon Garner)
  └─ HLstatsX (Tobias Oetzel)
       └─ ELstatsNEO (Malte Bayer)
            └─ HLstatsX:CE (Nicholas Hastings)
                 └─ HLstatsZ (SnipeZilla) ← now you are stuck here
```

## Credits

Based on HLstatsX:CE 1.6.19  
Maintained and modernized by [SnipeZilla](https://github.com/SnipeZilla)  
Validation and help by [ghost-](https://github.com/ghostt187)

Licensed under the GNU General Public License v2 or later.
