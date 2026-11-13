# AGENTS.md --- EA Jellyfin GeoIP Routing and Nginx VPS

## 1. Purpose

This document describes the Nginx reverse-proxy, GeoIP routing, Jellyfin
media delivery, and health-monitoring configuration for `eajelly.xyz`.

The primary goal is to maintain one public Jellyfin URL for all clients:

`https://eajelly.xyz`

Media requests are routed according to the visitor's country:

-   Philippines and selected Southeast Asian countries: redirect
    directly to the home Jellyfin server.
-   Other countries: use the Los Angeles VPS reverse proxy.
-   Jellyfin frontend, authentication, API, and metadata requests:
    continue through the VPS.
-   Streamyfin uses the same public server URL without separate
    client-side routing configuration.

**Important architectural rule: There must be no server-side image,
artwork, or metadata caching on the VPS.**

Do not reintroduce Nginx caching, stale-response caching, or artificial
browser cache lifetimes for Jellyfin artwork.

Inspect existing configurations before making changes. Prefer minimal,
targeted modifications.

## 2. Architecture

### Public endpoints

-   Main Jellyfin frontend: `https://eajelly.xyz`
-   Public alias: `https://www.eajelly.xyz`
-   Direct home Jellyfin endpoint: `https://media.eajelly.xyz:9000`

### VPS-to-home connection

-   VPS operating system: Ubuntu
-   Web server: Nginx 1.28.3
-   Home Jellyfin upstream: `https://192.168.250.254:9000`
-   Upstream TLS SNI: `media.eajelly.xyz`
-   Upstream HTTP Host: `media.eajelly.xyz`
-   VPS-to-home connection uses the existing WireGuard network.

### Routing architecture

The main Nginx virtual host handles the public frontend, API requests,
artwork requests, and media streams.

Separate location blocks provide country-based routing for artwork,
video, and audio.

The direct media endpoint does not use the VPS artwork cache.

## 3. GeoIP2 country routing

### Country database

-   Database: GeoLite2 Country, MaxMind MMDB format
-   Path: `/usr/share/GeoIP/GeoLite2-Country.mmdb`
-   Nginx module: `/usr/lib/nginx/modules/ngx_http_geoip2_module.so`
-   Installed module package: `libnginx-mod-http-geoip2`
-   GeoIPUpdate version previously observed: `7.1.1`

The GeoIP2 module is enabled through:

`/etc/nginx/modules-enabled/50-mod-http-geoip2.conf`

Module-loading directive:

``` nginx
load_module modules/ngx_http_geoip2_module.so;
```

### Country lookup configuration

File:

`/etc/nginx/conf.d/geoip2-country.conf`

``` nginx
geoip2 /usr/share/GeoIP/GeoLite2-Country.mmdb {
    auto_reload 5m;
    $geoip2_country_code default=ZZ source=$remote_addr country iso_code;
}
```

This creates `$geoip2_country_code` using the visitor IP address seen by
Nginx.

### Country routing map

File:

`/etc/nginx/conf.d/geoip2-routing.conf`

``` nginx
map $geoip2_country_code $jellyfin_media_route {
    default proxy;
    PH direct;
    SG direct;
    MY direct;
    ID direct;
    TH direct;
    VN direct;
    BN direct;
    KH direct;
    LA direct;
    MM direct;
    TL direct;
}
```

Countries configured for direct delivery:

  Country       Code
  ------------- ------
  Philippines   PH
  Singapore     SG
  Malaysia      MY
  Indonesia     ID
  Thailand      TH
  Vietnam       VN
  Brunei        BN
  Cambodia      KH
  Laos          LA
  Myanmar       MM
  Timor-Leste   TL

All other countries default to `proxy`.

GeoIP is an estimate based on the public IP address. VPNs, mobile
carrier gateways, and other network routing can affect country
classification.

Do not change the country list or routing behavior without an explicit
requirement.

## 4. Upstream routing

File:

`/etc/nginx/conf.d/jellyfin-upstream-routing.conf`

The configuration defines the home Jellyfin upstreams:

``` nginx
upstream jellyfin_direct {
    server media.eajelly.xyz:9000;
}

upstream jellyfin_vps {
    server 192.168.250.254:9000;
}

map $jellyfin_media_route $jellyfin_upstream {
    direct  jellyfin_direct;
    default jellyfin_vps;
}
```

The active artwork configuration uses `$jellyfin_upstream` for proxied
requests.

Preserve both upstream definitions and the country-based routing map.

The obsolete `$jellyfin_artwork_cache_bypass` map must not be
reintroduced. It is no longer necessary because artwork caching has been
removed.

## 5. Video stream routing

The video location handles requests matching:

``` nginx
location ~* ^/videos/
```

Selected countries use:

``` nginx
if ($jellyfin_media_route = direct) {
    return 302 https://media.eajelly.xyz:9000$request_uri;
}
```

Other countries use the existing VPS-to-home proxy.

Preserve:

-   TLS SNI and Host configuration.
-   Forwarded client IP headers.
-   Disabled response buffering.
-   Disabled request buffering.
-   Long media read and send timeouts.
-   Existing range-request and streaming behavior.

Do not introduce response caching for video streams.

## 6. Audio stream routing

Audio requests use the existing country-based routing.

Preserve the existing audio location matching and redirect behavior.

The configuration must maintain:

-   Direct delivery for selected countries.
-   VPS proxying for other countries.
-   Existing upstream TLS configuration.
-   Existing streaming timeouts.
-   Existing audio and lyrics-specific routing.

The special lyrics location must remain on the public `eajelly.xyz`
origin unless an explicitly tested change requires otherwise.

Do not introduce response caching for audio streams.

## 7. Artwork and image routing

The artwork location matches requests under:

``` nginx
location ~* ^/(Items|Users|Genres|Studios|Persons|Artists|MusicGenres|Playlists)/[^/]+/Images/
```

### Routing behavior

Selected Southeast Asian countries:

``` nginx
if ($jellyfin_media_route = direct) {
    return 302 https://media.eajelly.xyz:9000$request_uri;
}
```

Other countries:

-   Proxy artwork requests to the home Jellyfin server through the VPS.
-   Do not cache artwork on the VPS.
-   Fetch the current artwork from Jellyfin for each request.

### Artwork response headers

The VPS artwork proxy must use:

``` nginx
proxy_hide_header Cache-Control;
proxy_hide_header Expires;

add_header Cache-Control "no-store, no-cache, must-revalidate" always;
add_header Pragma "no-cache" always;
```

These headers prevent browser and compliant intermediary caching of
artwork responses through this route.

### Prohibited artwork caching

The following must not be introduced:

-   `proxy_cache`
-   `proxy_cache_path`
-   `proxy_cache_valid`
-   `proxy_cache_key`
-   `proxy_cache_bypass`
-   `proxy_no_cache`
-   `proxy_cache_lock`
-   `proxy_cache_revalidate`
-   `proxy_cache_use_stale`
-   `X-Artwork-Cache`
-   Artificial long-lived `Cache-Control` headers.
-   Forced `immutable` artwork responses.
-   Any other VPS-side artwork cache implementation.

The obsolete cache zone was:

`jellyfin_artwork`

The old cache configuration file was:

`/etc/nginx/conf.d/jellyfin-cache.conf`

The artwork cache zone and its associated cache directives have been
removed.

The remaining `jellyfin-cache.conf` file contains the WebSocket
connection map:

``` nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

Preserve this map because the main Jellyfin proxy uses it for WebSocket
connections.

The filename is historical; do not assume that this file still
configures a cache.

## 8. Frontend, API, and metadata

The default Jellyfin location remains:

``` nginx
location / {
    proxy_pass https://$jellyfin_upstream;
}
```

It retains the existing:

-   TLS SNI and Host headers.
-   Forwarded client IP headers.
-   WebSocket upgrade support.
-   Request buffering configuration.
-   Response buffering configuration.
-   Proxy buffer settings.
-   Long connection and streaming timeouts.
-   Maximum request body size.

### Metadata and API caching policy

There must be no Nginx server-side caching for:

-   Jellyfin API responses.
-   Library metadata.
-   Item information.
-   Playback information.
-   User information.
-   Session information.
-   Other Jellyfin metadata responses.

The previously inspected configuration did not contain a separate
metadata/API `proxy_cache` implementation. The general API and frontend
location already proxied requests directly to Jellyfin without an Nginx
proxy cache.

Do not introduce any caching layer for these requests.

Do not apply blanket `no-store` headers to the entire frontend location
because this would unnecessarily disable normal caching of static
frontend resources.

If browser caching of a specific API response becomes an issue,
investigate that endpoint and apply an appropriately scoped policy
rather than changing all frontend responses.

The general frontend and API location must remain on the public
`eajelly.xyz` origin.

Authentication, library browsing, playback information, and session
requests must continue functioning normally.

## 9. Playback and special Jellyfin endpoints

Preserve the existing special routing for:

-   `/Items/{id}/PlaybackInfo`
-   `/Playback/BitrateTest`
-   Audio lyrics.
-   Video streams.
-   Audio streams.
-   Artwork requests.

The current active configuration contains special locations for
PlaybackInfo and BitrateTest.

Inspect the actual active configuration before modifying these
locations.

Do not replace the entire virtual host with a simplified example that
removes existing Jellyfin functionality.

## 10. Browser JavaScript selector

The earlier browser JS Injector used a connectivity probe to:

`https://media.eajelly.xyz:9000/System/Info/Public`

It also set a `jellyfin_media_route` cookie.

The current Nginx routing uses `$jellyfin_media_route`, derived from
GeoIP2, rather than relying on the cookie.

The old JS Injector is therefore not required for country-based routing.

Before removing or changing any remaining client-side script, verify
that no other desired functionality depends on it.

Streamyfin's configured server URL remains:

`https://eajelly.xyz`

## 11. GeoLite database updates

GeoLite2 Country is installed at:

`/usr/share/GeoIP/GeoLite2-Country.mmdb`

GeoIPUpdate configuration is stored in:

`/etc/GeoIP.conf`

Relevant settings:

``` ini
AccountID YOUR_ACCOUNT_ID
LicenseKey YOUR_LICENSE_KEY
EditionIDs GeoLite2-Country
DatabaseDirectory /usr/share/GeoIP
```

Never include the actual license key in this document, logs, or source
control.

Update the database manually:

``` bash
sudo geoipupdate
```

Verify the database:

``` bash
sudo ls -lh /usr/share/GeoIP/GeoLite2-Country.mmdb
```

GeoLite downloads require valid MaxMind account credentials and an
active license key.

The GeoIP2 configuration uses `auto_reload 5m`, allowing Nginx to check
for database updates.

## 12. Nginx validation and reload procedure

Before applying any Nginx configuration change:

``` bash
sudo nginx -t
```

Only reload when the syntax test succeeds:

``` bash
sudo systemctl reload nginx
```

Verify Nginx is active:

``` bash
systemctl is-active nginx
```

Expected output:

``` text
active
```

Prefer reloads over restarts for normal configuration changes.

### Check active cache directives

Use:

``` bash
sudo nginx -T 2>&1 | grep -nEi \
'proxy_cache|fastcgi_cache|uwsgi_cache|scgi_cache|jellyfin_artwork|X-Artwork-Cache'
```

Expected output: no matches.

This check must cover the active Nginx configuration, not just the main
site file.

Old backups may contain historical cache directives. They are not active
configurations and must remain outside Nginx's included configuration
directories.

### Check artwork headers

Test a real artwork request through the VPS and inspect its response
headers.

Expected:

``` text
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
```

The response must not contain:

``` text
X-Artwork-Cache
```

### Check GeoIP routing

Verify that:

-   Requests from listed countries receive the direct media redirect.
-   Requests from other countries continue through the VPS.
-   The artwork, video, and audio locations use the intended routing
    variable.

A VPS-originated request is classified using the VPS public IP, not the
Philippine viewer's IP.

A `302` confirms that a redirect was issued; it does not independently
prove that the direct endpoint delivered the requested content.

## 13. Health monitoring and Discord integration

### Purpose

Maintain Jellyfin availability monitoring through the VPS, display
custom outage pages, and update two existing Discord status messages
when the confirmed status changes.

### Health-check schedule

-   Systemd timer: `jellyfin-health-check.timer`
-   Service: `jellyfin-health-check.service`
-   Interval: 15 seconds.
-   Scheduled at `:00`, `:15`, `:30`, and `:45`.
-   Boot delay: 1 minute.
-   Timer accuracy: 1 second.
-   Require two consecutive failures of the same type before confirming
    an outage.

Continue checking during an outage, but do not repeatedly reload Nginx
or send Discord updates while the confirmed status remains unchanged.

### Health-check classification

**UP**

-   TCP connection succeeds.
-   `https://media.eajelly.xyz:9000/health` returns HTTP 200.

**CONNECTIVITY_ISSUE**

-   TCP connection to `192.168.250.254:9000` fails.
-   Nginx returns 503.
-   Display `jellyfin-connectivity.html`.

**JELLYFIN_DOWN**

-   TCP connection succeeds.
-   The HTTP health check does not return 200.
-   Nginx returns 504.
-   Display `jellyfin-down.html`.

On recovery:

-   Restore normal proxying.
-   Record recovery timestamp and outage duration.
-   Update Discord.

### Nginx outage pages

-   Main site: `https://eajelly.xyz/`
-   Connectivity page: `/var/www/html/jellyfin-connectivity.html`
-   Jellyfin-down page: `/var/www/html/jellyfin-down.html`
-   Health-state include:
    `/etc/nginx/snippets/jellyfin-health-state.conf`

Keep health-monitor rules scoped to the `eajelly.xyz` HTTPS virtual
host.

Both outage pages automatically retry the main site every 15 seconds and
provide a Retry now button.

Preserve existing GeoIP routing and no-cache artwork behavior when
modifying health-monitor configurations.

### Discord status updates

-   Sender: `/usr/local/sbin/jellyfin-discord-update`
-   Private webhook configuration: `/etc/jellyfin-health/discord.env`
-   Existing message IDs:
    `/var/lib/jellyfin-health/discord-messages.json`
-   Notification tracking:
    `/var/lib/jellyfin-health/last-notified-status`

Operational rules:

-   Edit the two existing Discord messages instead of posting new
    messages during normal updates.
-   Notify only on confirmed status transitions.
-   Retry failed notifications during later health checks.
-   Preserve the existing EA Jelly status-message template and links.
-   Recovery messages include the recovery timestamp and calculated
    downtime duration.
-   Use real newline characters, not literal `\n` text.
-   Never expose webhook URLs, tokens, or message IDs in logs or
    documentation.

### Configuration, state, and logs

-   Health configuration: `/etc/jellyfin-health/config`
-   Persistent state: `/var/lib/jellyfin-health/state`
-   Health log: `/var/log/jellyfin-health.log`
-   Health script: `/usr/local/sbin/jellyfin-health-check`
-   Systemd timer: `/etc/systemd/system/jellyfin-health-check.timer`
-   Systemd service: `/etc/systemd/system/jellyfin-health-check.service`

Keep sensitive configuration files root-owned and restricted.

## 14. Troubleshooting

### Nginx configuration fails

``` bash
sudo nginx -t
```

Inspect the reported file and line before making further changes.

### Nginx is not active

``` bash
systemctl status nginx --no-pager
sudo journalctl -u nginx -n 50 --no-pager
```

### Check GeoIP configuration

``` bash
sudo cat /etc/nginx/conf.d/geoip2-country.conf
sudo cat /etc/nginx/conf.d/geoip2-routing.conf
sudo ls -lh /usr/share/GeoIP/GeoLite2-Country.mmdb
```

Do not publish license keys or secrets.

### Check active site configuration

``` bash
sudo grep -nE \
'location .*Images|location .*videos|location .*audio|jellyfin_media_route|proxy_pass' \
/etc/nginx/sites-enabled/eajelly.xyz
```

### Check recent media routing

``` bash
sudo tail -n 30 /var/log/nginx/jellyfin-media-access.log
```

Correlate client IP, user agent, and status code.

Avoid logging full query strings because they may contain authentication
tokens or other sensitive information.

### Check for accidental caching

``` bash
sudo grep -RniE \
'proxy_cache|fastcgi_cache|uwsgi_cache|scgi_cache|jellyfin_artwork|X-Artwork-Cache' \
/etc/nginx/conf.d/ \
/etc/nginx/sites-enabled/
```

Historical backup files may contain old cache configuration. Do not
confuse those with active configuration.

## 15. Change-management rules

Before future changes:

1.  Inspect the actual active configuration. Do not assume it matches
    this document.
2.  Back up the specific file being edited.
3.  Keep backups outside Nginx included configuration directories,
    preferably `/etc/nginx/backups/`.
4.  Make one minimal change at a time.
5.  Preserve GeoIP2 country detection and country-based routing.
6.  Preserve TLS SNI and Host headers.
7.  Preserve WebSocket support.
8.  Preserve existing streaming and proxy timeouts.
9.  Never introduce server-side artwork or metadata caching.
10. Never introduce stale artwork serving.
11. Do not impose artificial long-lived browser cache headers on
    artwork.
12. Run `sudo nginx -t` before reloading.
13. Reload only after successful validation.
14. Test the relevant route from a real client.
15. Inspect logs and response headers after changes.
16. Never store API keys, GeoLite license keys, webhook URLs, or other
    secrets in this document.

## 16. Current implementation status

Confirmed during the setup and subsequent cache-removal work:

-   GeoIP2 Nginx module installed and loaded.
-   GeoLite2 Country database installed.
-   Country lookup configured using `$geoip2_country_code`.
-   Country routing map configured for selected Southeast Asian
    countries.
-   Video and audio locations use country-based routing.
-   Artwork location uses the same country-based routing.
-   Direct artwork requests use `media.eajelly.xyz:9000`.
-   Other artwork requests proxy through the VPS.
-   VPS artwork proxy caching has been removed.
-   The `jellyfin_artwork` cache zone has been removed.
-   The obsolete artwork cache bypass map has been removed.
-   The forced seven-day artwork browser cache headers have been
    removed.
-   Artwork proxy responses use
    `Cache-Control: no-store, no-cache, must-revalidate`.
-   The `X-Artwork-Cache` response header has been removed.
-   No separate Nginx metadata/API proxy cache was present in the
    inspected configuration.
-   The general frontend and API proxy remains operationally separate
    from artwork routing.
-   The WebSocket connection map has been retained.
-   The old artwork cache directory was identified and scheduled for
    cleanup.
-   Nginx configuration validation succeeded after removal of the active
    artwork cache directives.

The user confirmed that the direct media endpoint does not implement
caching.

### Items to verify after future changes

-   Direct connectivity from clients in the listed countries.
-   Successful artwork delivery through both routing paths.
-   Updated artwork is retrieved from the current Jellyfin server.
-   No active Nginx caching directives have been reintroduced.
-   Frontend, authentication, metadata, and playback functionality
    remain operational.
-   Health monitoring and Discord notifications continue functioning.

Do not claim that all countries or all client types have been tested
unless they have actually been verified.

------------------------------------------------------------------------

**End of AGENTS.md**
