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

Copy `wedos_zone-cpanel.tar.gz` and `SHA256SUMS.txt` to the server. As root, check
that the archive arrived intact, then install:

```
sha256sum -c SHA256SUMS.txt
tar xzf wedos_zone-cpanel.tar.gz && sh wedos_zone/cpanel/scripts/install.sh
```

The first command must print `wedos_zone-cpanel.tar.gz: OK`. If it does not, do not
install — download the archive again.

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
switch protection on: WEDOS checks the domain in intervals of about an hour and a half,
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

**From the plugin page (version 0.1.4 and later).** Once a day the plugin checks
`https://wedos.zone/plugins/cpanel/version.txt`. When a newer version is out, the plugin
page in WHM says so and offers **Update**. The click only queues the update: within a
minute the cron downloads the package, verifies its signature, runs the installer and
writes the result to the log. Nothing is installed without the click. A package whose
signature does not match is refused and the installed version keeps running.

The server needs outgoing HTTPS to `wedos.zone`. From the command line:

```
/usr/local/cpanel/3rdparty/bin/php /usr/local/cpanel/3rdparty/wedos_zone/scripts/wz.php update --install
```

**By hand.** Run the install command with the new archive. Version 0.1.3 and older
update only this way.

Either way, settings, the queue and the zone memory live in `/var/lib/wedos_zone` and
survive the update untouched. If a cPanel update ever removes something the plugin
needs, the same command puts it back.


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

## Getting help

Include the output of `wz.php status` and the last lines of
`/var/lib/wedos_zone/wedos_zone.log`. The log records what was sent and what came back;
it never contains your password.

---

## Warranty, licence and trademarks

The plugin is provided "as is", without warranty of any kind; see `LICENSE.txt` for the
full disclaimer of warranty and limitation of liability. The DNS service is governed by
the WEDOS terms and conditions that apply to your account. The plugin is not licensed for
redistribution or modification. © 2026 WEDOS Internet, a.s. All rights reserved.

cPanel® and WHM® are trademarks of WebPros International LLC.
