<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset=".github/logo-dark.webp">
    <img src=".github/logo-light.webp" alt="WEDOS.zone" width="420">
  </picture>
</p>

<p align="center">
  <a href="https://github.com/wedos-zone/wedos-zone-cpanel/releases/latest"><strong>Download the latest release</strong></a>
  · <a href="https://wedos.zone">wedos.zone</a>
</p>

# WEDOS.zone plugin — works with cPanel® & WHM®

Your DNS zones stay where you manage them — in cPanel®. This plugin reports every change
to WEDOS.zone, our global anycast DNS, so your customers' domains are answered from our
network instead of from a single server.

Full documentation: <https://kb.wedos.global/>

Tested on cPanel® & WHM® 138 on AlmaLinux 9, with PowerDNS (the cPanel default).

> **The plugin is provided "as is", without warranty of any kind, and changes the DNS
> configuration of your server. You install and use it at your own risk.** Back up your
> server configuration and DNS zones first, and try it on one server before deploying it
> more widely. Full terms: `LICENSE.txt` in the package.

---

## Before you install

**1. A WEDOS account with WAPI access.** WAPI is the API of the WEDOS administration.
You enable it there, set a WAPI password (it is not your login password) and allow the
IP addresses of this server.

**2. Allow this server's IP addresses — IPv4 and IPv6.** Without it every request is
refused. A server with IPv6 usually reaches WEDOS over IPv6, so allowing only the IPv4
address is not enough. The change takes **up to 30 minutes** to apply, so do it first.
If you are not sure, install anyway: the plugin tells you which address WEDOS saw.

**3. Open port 53 for us.** WEDOS.zone runs as your secondary DNS: the zones stay on
your server, the plugin reports only their names and we pull the contents. We pull them from
`46.28.104.64/27`. The installer allows that range in PowerDNS; **the firewall has to let
TCP and UDP port 53 through as well**. The plugin cannot see the firewall.

---

## Install

As root on the server, download the latest release, check that it arrived intact, then
install:

```
curl -fsSLO https://github.com/wedos-zone/wedos-zone-cpanel/releases/latest/download/wedos_zone-cpanel.tar.gz
curl -fsSLO https://github.com/wedos-zone/wedos-zone-cpanel/releases/latest/download/wedos_zone-cpanel.tar.gz.sha256
sha256sum -c wedos_zone-cpanel.tar.gz.sha256
tar xzf wedos_zone-cpanel.tar.gz && sh wedos_zone/cpanel/scripts/install.sh
```

The check must print `wedos_zone-cpanel.tar.gz: OK`. If it does not, do not install —
download the archive again. All releases are at
https://github.com/wedos-zone/wedos-zone-cpanel/releases.

Then in WHM open **Plugins → WEDOS**:

1. Read what the first screen asks of your server and continue.
2. Fill in your WAPI account (the e-mail of the WEDOS account) and the WAPI password.
3. The plugin starts in **test mode** — WAPI only checks the commands and changes
   nothing. Leave it on until the screen looks healthy.
4. Turn off test mode and tick **Report zone changes to WEDOS.zone**.

Nothing is sent until step 4. The plugin is deliberately inert on install.

### What the installer changes on the server

The installer only **adds**. Whatever you already have — another secondary DNS provider,
your own cluster nodes, your own transfer rules — keeps working as before.

- **DNS cluster.** cPanel tells plugins about zone changes through its DNS cluster, so
  the installer turns clustering on and adds one node, `wedoszone`, with the role
  *write-only*. You will see it in **WHM → Clusters → DNS Cluster**. It does not talk to
  any other server; it only puts the change into the plugin's queue.
- **dnsadmin** is taken off the list of dormant services, otherwise the cluster does not
  run.
- **PowerDNS zone transfers.** cPanel ships with transfers disabled. The installer
  enables them and adds `46.28.104.64/27` to `allow-axfr-ips` in `/etc/pdns/pdns.conf`,
  keeping every address that is already there. The original file is saved next to the
  plugin data as `pdns.conf.pred-wedos`.
- **BIND** instead of PowerDNS: the installer changes nothing. cPanel ships BIND with
  `allow-transfer { "none"; };` in the `options` block of `/etc/named.conf`. Replace
  `"none"` with `46.28.104.64/27` (or add the range to the list you already have) and
  run `rndc reconfig`. cPanel keeps this change when it rebuilds the configuration or
  adds zones.
- A cron job in `/etc/cron.d/wedos_zone` sends the queue once a minute.

Files: the plugin in `/usr/local/cpanel/3rdparty/wedos_zone`, its data (settings, queue,
log) in `/var/lib/wedos_zone`. The plugin runs on cPanel's own PHP, so changes to the PHP
your websites use do not affect it.

### Moving from your own scripts

If you already create secondary zones at WEDOS with your own WAPI scripts, the plugin
takes over from them. Nothing has to be deleted or created again.

1. Install the plugin and fill in the same WAPI account your scripts use. Zones belong to
   the account; with a different one the plugin reports them as held by another account.
2. While test mode is still on, run `wz.php reconcile`. It only compares and changes
   nothing. A zone on both sides is taken as it is. A zone only on this server would be
   created. A zone only at WEDOS is only reported, never deleted, so zones of your other
   servers on the same account are safe.
3. Zones that already exist keep the primary IP your scripts set. If it was this
   server's address, there is nothing to do. The plugin does not rewrite it.
4. Switch your scripts off, then turn off test mode and tick **Report zone changes to
   WEDOS.zone**. Two tools adding and removing the same zones would undo each other.

---

## Everyday use

There is nothing to do. Create, change or remove an account, an addon domain or a zone
record as usual and the plugin reports it. The WHM page shows the queue, the hourly
budget and anything that needs your attention.

---

## Branded nameservers (WhiteLabel)

If your zones are delegated to your own nameserver names on addresses reserved for you
by WEDOS, enter them in **Settings**:

- **Your own nameservers** — the names, one per line (`ns1.example.com`). Zones that
  list them are recognised immediately.
- **Reserved IP addresses** — the addresses WEDOS reserved for you, IPv4 and IPv6. With
  them, the plugin also recognises your nameservers by address during the nightly
  comparison.

**Check the new names only after they are published.** A resolver that asks for a name
before it exists remembers the negative answer for as long as the zone's SOA minimum
allows — up to 86400 seconds in a default cPanel zone, often capped at a few hours.
Create the `A`/`AAAA` records for the names first, wait until the nameservers of that
zone answer for them (`dig ns1.example.com @<one of those nameservers>`), and only then
test through public resolvers.

---

## WEDOS.protection

The **Protection** tab lists your domains and lets you put any of them behind
WEDOS.protection. You can try it right after installation on up to 50 domains on the
Free plan and 15 on NoLimit; higher plans are sold directly by WEDOS. Paid plans need a
WEDOS credit account. Without WhiteLabel CNAME, your customers see a `wedos.global`
name in their DNS; with it, the protection is shown under your own name and the limits
no longer apply — contact us to set it up.

**Switching on** takes about 15–25 minutes and needs nothing from you. The plugin
orders the service, waits until WEDOS is ready and then changes three things in the
domain's zone in cPanel:

- the apex `A` records point to the WEDOS.protection proxy,
- `www` becomes a `CNAME` to the proxy,
- `_acme-challenge` becomes a `CNAME` to WEDOS, which issues the certificate.

These records get a 300-second TTL so that switching off is quick. The domain has to be
delegated to the WEDOS nameservers (or your branded ones), otherwise WEDOS cannot verify
it and the plugin tells you so. Finish any change to the domain's nameservers before you
switch protection on: WEDOS checks the domain at intervals that can be a few hours long,
so a check that catches the change halfway delays switching on until the next one.

**Switching off** puts the original records back first and cancels the service only
when DNS no longer points to the proxy — about 10 minutes. A switch-on that never
finishes (for example a domain that is not registered) can be stopped with
**Cancel**; a domain stuck anywhere else has **Give up and clean up**.

**Deleting a domain or account switches its protection off.** The service is cancelled
once the domain no longer points to the proxy, so moving an account to another server
does not take the site down: while the copied zone still points to WEDOS.protection,
the service keeps running.

**Records you change by hand are reported, not undone.** Once a day the plugin checks
that the zone of every protected domain still carries the proxy records. If the apex
or `www` was changed, the Protection tab says so; the service is not cancelled.

**Where the proxy sends visitors.** When protection is switched on, the plugin gives
WEDOS the public IPv4 and IPv6 addresses of the apex and `www` from the zone as the
origin. Addresses of WEDOS.protection itself and private addresses are left out: the
proxy cannot reach them. If no address is left, protection is not switched on and the
Protection tab says why. Each protected domain shows its origin.

**After moving an account or changing the server IP, change the origin yourself.** The
proxy keeps sending visitors to the address it got when protection was switched on.
Once a day the plugin compares it with the IP that cPanel uses for the domain. When they
differ, the Protection tab shows both addresses and you get a notification. The plugin
cannot change the origin for you, because WAPI cannot. Change the target IP of the
protection in the WEDOS administration, then click **Target changed** (or run
`wz.php protection origin <domain>`). Only IPv4 is compared. Domains whose service was
not created by the plugin show no origin and are not checked.

**Before you switch protection off, make sure AutoSSL covers the domain.** While the
domain is protected, WEDOS serves the certificate. After switching off, visitors get
whatever certificate your server has — if AutoSSL is disabled or the domain is excluded
from it, that is the self-signed one and browsers show a warning. AutoSSL issues a new
certificate within a minute or two once it is allowed to run (WHM → SSL/TLS →
Manage AutoSSL, or *Run AutoSSL* for the user).

Wildcard certificates (`*.domain`) cannot be issued by AutoSSL for a domain served by
WEDOS: AutoSSL checks its DNS record immediately, and the change takes several minutes
to reach the WEDOS nameservers. Individual names (`www`, `mail`, …) are validated over
HTTP and work.

### Your own protection hostname (WhiteLabel CNAME)

With WhiteLabel CNAME, `www` of a protected domain points to a name of yours instead of
`<domain>.wedos.global`. WEDOS gives you the protection IP addresses; then:

1. In your own zone, add one wildcard record for a name of your choice, for example
   `*.proxy.example.com` → your protection IPv4 addresses (`A`) and IPv6 addresses
   (`AAAA`). Put **nothing else** under `proxy.example.com`: any other record there stops
   the wildcard from answering for that part of the name.
2. In **Settings → Protection hostname**, enter `proxy.example.com` and save. The plugin
   checks that a random name under it leads to WEDOS.protection and refuses to save it
   otherwise.

From then on, every domain you switch on gets `www` → `<domain>.proxy.example.com`.
Domains that are already protected keep their records until protection is switched off
and on again. The apex and `_acme-challenge` records are set as before. Empty the field
to go back to the WEDOS name.

---

## Limits worth knowing in advance

- **1000 WAPI requests per hour per account.** The plugin keeps to 800 and leaves the
  rest for whatever else uses the account. When the budget runs out the queue waits for
  the next hour; nothing is lost.
- **The shortest TTL we accept is 300 seconds.** The plugin raises shorter ones to 300 and
  tells you which zones and records. Your own server keeps the original value.
- **A change reaches the public nameservers in roughly ten minutes.**
- **Without WhiteLabel the plugin creates at most 500 zones at WEDOS.** Zones above the
  limit stay in the queue with the reason shown; zones that already run are not
  affected. WhiteLabel
  (your own nameserver names on a dedicated IP) lifts the limit.

---

## When something is not right

The WHM page says what happened and what to do about it.

**Wrong WAPI password or an IP that is not allowed.** WEDOS blocks an address after more
than ten failed requests, and the block grows with every further attempt. So the plugin
stops sending after the first refusal, waits (15 minutes, then longer) and says which
address to allow. Fix it, then **save the settings** — the plugin tries again within a
minute. Zone changes wait in the queue meanwhile; nothing is lost.

**A message says the plugin does not receive zone changes.** Something switched the DNS
cluster off or removed the node. Run the installer again; it puts back what is missing
and leaves your settings alone.

**A "Consent and Privacy" window opens on the plugin page every time.** It is the cPanel
analytics consent, not the plugin. WHM builds the header of PHP plugin pages without
knowing who is logged in, so the window opens even after you have answered — your answer
is saved, it is just not read there. To stop it, turn cPanel analytics off for the server
and rebuild the header:

```
whmapi1 participate_in_analytics enabled=0
/usr/local/cpanel/scripts/rebuild_whm_chrome
```

You can turn analytics back on in WHM → Server Configuration → Configure cPanel Analytics.

**You switched the DNS server in WHM (Nameserver Selection).** cPanel writes a fresh
`pdns.conf` when you switch to PowerDNS, which drops the transfer permission for WEDOS
(and any other changes you made there). The plugin reports it; run the installer again.

---

## Updating

**From the plugin page (version 0.1.5 and later).** Once a day the plugin checks the
latest release on GitHub (`github.com/wedos-zone/wedos-zone-cpanel`). When a newer
version is out, the plugin page in WHM says so and offers **Update**. The click only queues the update: within a
minute the cron downloads the package, verifies its signature, runs the installer and
writes the result to the log. Nothing is installed without the click. A package whose
signature does not match is refused and the installed version keeps running.

The server needs outgoing HTTPS to `github.com` and `release-assets.githubusercontent.com`.
From the command line:

```
/usr/local/cpanel/3rdparty/bin/php /usr/local/cpanel/3rdparty/wedos_zone/scripts/wz.php update --install
```

**By hand.** Run the install commands from *Install* again. Version 0.1.4 and older
update only this way.

Either way, settings, the queue and the zone memory live in `/var/lib/wedos_zone` and
survive the update untouched. If a cPanel update ever removes something the plugin
needs, the same command puts it back.

What changed in each version is in `CHANGELOG.md` next to this file, and with every
release on https://github.com/wedos-zone/wedos-zone-cpanel/releases.


## Removing

```
sh /usr/local/cpanel/3rdparty/wedos_zone/cpanel/scripts/uninstall.sh
```

It removes the node, the cron job and the WHM page, and takes back only what the
installer added: the WEDOS range in `pdns.conf`, and clustering and transfers only if
it was the plugin that turned them on. The WAPI password is deleted; other data stays
unless you chose *Remove all data when the plugin is uninstalled*.

**Your zones in WEDOS.zone are never deleted.** Removing the plugin is a decision about
this server, not about your customers' DNS.

---

## From the command line

```
PHP=/usr/local/cpanel/3rdparty/bin/php
WZ=/usr/local/cpanel/3rdparty/wedos_zone/scripts/wz.php

$PHP $WZ status                     settings, budget and queue
$PHP $WZ set wapi_pass              set the WAPI password (asks for it, never as an argument)
$PHP $WZ test-auth                  verify the login and the allowed IP
$PHP $WZ queue                      what is waiting
$PHP $WZ reconcile [--apply]        compare this server against WEDOS.zone
$PHP $WZ audit                      compare zone contents
$PHP $WZ update [--install]         check for a new version, install it
```

---

## The log

Everything the plugin does goes to `/var/lib/wedos_zone/wedos_zone.log`, one line per
event, with the level `INFO`, `WARN` or `ERROR`. The log records what was sent and what
came back; it never contains your password.

Follow it live:

```bash
tail -F /var/lib/wedos_zone/wedos_zone.log
```

Only warnings and errors, live or from the whole file:

```bash
tail -F /var/lib/wedos_zone/wedos_zone.log | grep --line-buffered -E ' (WARN|ERROR) '
grep -E ' (WARN|ERROR) ' /var/lib/wedos_zone/wedos_zone.log
```

Use `-F`, not `-f`. At 5 MB the log is renamed to `wedos_zone.log.1` and a new one starts;
`-F` follows the new file, `-f` would stay on the old one.

In WHM, Settings shows the end of the log, with *show only those* when there are warnings.

---

## Getting help

Include the output of `wz.php status` and the last lines of
`/var/lib/wedos_zone/wedos_zone.log`.

---

## Warranty, licence and trademarks

The plugin is provided "as is", without warranty of any kind; see `LICENSE.txt` for the
full disclaimer of warranty and limitation of liability. The DNS service is governed by
the WEDOS terms and conditions that apply to your account. The plugin is not licensed for
redistribution or modification. © 2026 WEDOS Internet, a.s. All rights reserved.

cPanel® and WHM® are trademarks of WebPros International LLC.
