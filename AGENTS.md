# AGENTS.md — Jellyfin GeoIP Routing and Nginx VPS

## 1. Purpose

This document records the Nginx reverse-proxy and GeoIP routing configuration for Jellyfin.

The goal is to keep one public server URL for all clients:

`https://eajelly.xyz`

Media and artwork requests are routed according to the visitor's country:

- Philippines and selected Southeast Asian countries: redirect directly to the home Jellyfin server.
- Other countries: continue through the Los Angeles VPS reverse proxy.
- Jellyfin frontend and API: continue through the VPS.
- Streamyfin requires no separate server URL or client-side routing configuration.

**Important:** Do not replace the working configuration wholesale. Inspect the existing configuration and make minimal changes.

## 2. Architecture

### Public endpoints

- Public Jellyfin frontend: `https://eajelly.xyz`
- Public alias: `https://www.eajelly.xyz`
- Direct home Jellyfin endpoint: `https://media.eajelly.xyz:9000`

### VPS-to-home connection

- VPS operating system: Ubuntu
- Web server: Nginx 1.28.3
- Home Jellyfin upstream: `https://192.168.250.254:9000`
- Upstream TLS SNI: `media.eajelly.xyz`
- Upstream HTTP Host: `media.eajelly.xyz`
- VPS-to-home connection uses the existing WireGuard network.

The public frontend and default upstream proxy use the same Nginx virtual host. Separate locations handle artwork and media streams.

## 3. GeoIP country routing

### Country database

- Database: GeoLite2 Country, MaxMind MMDB format
- Path: `/usr/share/GeoIP/GeoLite2-Country.mmdb`
- Nginx module: `/usr/lib/nginx/modules/ngx_http_geoip2_module.so`
- Installed module package: `libnginx-mod-http-geoip2`
- GeoIPUpdate version observed: `7.1.1`

The GeoIP2 module is enabled through:

`/etc/nginx/modules-enabled/50-mod-http-geoip2.conf`

Its module-loading directive is:

```nginx
load_module modules/ngx_http_geoip2_module.so;
```

### Country lookup configuration

File: `/etc/nginx/conf.d/geoip2-country.conf`

```nginx
geoip2 /usr/share/GeoIP/GeoLite2-Country.mmdb {
    auto_reload 5m;
    $geoip2_country_code default=ZZ source=$remote_addr country iso_code;
}
```

This creates `$geoip2_country_code`, based on the visitor IP seen by Nginx.

### Country routing map

File: `/etc/nginx/conf.d/geoip2-routing.conf`

```nginx
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

- Philippines — `PH`
- Singapore — `SG`
- Malaysia — `MY`
- Indonesia — `ID`
- Thailand — `TH`
- Vietnam — `VN`
- Brunei — `BN`
- Cambodia — `KH`
- Laos — `LA`
- Myanmar — `MM`
- Timor-Leste — `TL`

All other country codes default to `proxy`.

GeoIP is an estimate based on the public IP address, not a guarantee of the visitor's physical location. VPNs, mobile carrier gateways, and other egress routing can affect classification.

## 4. Video stream routing

The active Nginx site is:

`/etc/nginx/sites-enabled/eajelly.xyz`

The video location matches:

```nginx
location ~* ^/videos/ {
```

The routing decision uses:

```nginx
if ($jellyfin_media_route = direct) {
    return 302 https://media.eajelly.xyz:9000$request_uri;
}
```

If the country map returns `direct`, Nginx redirects the client to the direct home endpoint. Otherwise, Nginx proxies the stream to:

`https://192.168.250.254:9000`

The existing proxy settings include TLS SNI and Host `media.eajelly.xyz`, forwarding headers, disabled response buffering, disabled request buffering, and long media timeouts.

Keep the existing video location's proxy configuration intact unless a separately tested change is needed.

## 5. Audio stream routing

The audio location matches:

```nginx
location ~* ^/audio/[^/]+/(stream|universal)(\.[^/]*)?$ {
```

It uses the same country-based redirect:

```nginx
if ($jellyfin_media_route = direct) {
    return 302 https://media.eajelly.xyz:9000$request_uri;
}
```

All other requests use the existing home Jellyfin upstream through the VPS.

Preserve the current audio proxy TLS settings and timeouts.

## 6. Artwork and image routing

The artwork location matches image requests under these resource types:

```nginx
location ~* ^/(Items|Users|Genres|Studios|Persons|Artists|MusicGenres|Playlists)/[^/]+/Images/ {
```

A GeoIP redirect was added inside this location:

```nginx
if ($jellyfin_media_route = direct) {
    return 302 https://media.eajelly.xyz:9000$request_uri;
}
```

Expected behavior:

- Listed Southeast Asian countries: image requests redirect to the direct home endpoint.
- Other countries: requests continue through the existing VPS artwork proxy and cache.

### Existing artwork cache settings

The existing artwork cache must be preserved.

The artwork location uses:

- Cache zone: `jellyfin_artwork`
- Cache key: `"$scheme|$proxy_host|$request_uri"`
- Successful responses: 7 days
- 404 responses: 15 minutes
- Cache locking and revalidation enabled
- Stale cache use for selected upstream failures
- Authorization-header requests bypass caching
- Response header: `X-Artwork-Cache`
- `Cache-Control: public, max-age=604800, immutable`
- Response buffering enabled
- Read and send timeouts: 60 seconds

Cache storage is configured in:

`/etc/nginx/conf.d/jellyfin-cache.conf`

The artwork redirect executes before the cache proxy directives. Therefore, direct-routed clients bypass the VPS artwork cache; other clients retain the existing cache behavior.

## 7. Frontend, API, and metadata

The default location remains:

```nginx
location / {
    proxy_pass https://192.168.250.254:9000;
}
```

It also retains the existing TLS SNI, Host, forwarded headers, WebSocket support, buffering settings, and long timeouts.

The general frontend and API location has not been changed to use GeoIP redirects.

This is intentional. Authentication, library browsing, playback information, and session requests continue to use the public `eajelly.xyz` origin and the VPS proxy.

**Do not assume that every metadata request is routed directly.** The implemented change covers the specific artwork location above; general metadata and API routing remain on the VPS.

## 8. Browser JavaScript selector

The earlier browser JS Injector used a short connectivity probe to `https://media.eajelly.xyz:9000/System/Info/Public` and set a `jellyfin_media_route` cookie.

Nginx media and artwork routing now uses `$jellyfin_media_route`, derived from GeoIP, rather than the cookie.

The JS Injector is therefore no longer needed for the current country-based routing behavior. Before removing it, verify that both video/audio locations and the artwork location use `$jellyfin_media_route`, and that no other desired functionality depends on the script.

Streamyfin's configured URL remains:

`https://eajelly.xyz`

## 9. GeoLite database updates

The database was successfully downloaded using GeoIPUpdate after credentials were configured in `/etc/GeoIP.conf`.

Relevant settings:

```ini
AccountID YOUR_ACCOUNT_ID
LicenseKey YOUR_LICENSE_KEY
EditionIDs GeoLite2-Country
DatabaseDirectory /usr/share/GeoIP
```

Never put the actual license key in this document, logs shared for troubleshooting, or source control.

Update the database manually:

```bash
sudo geoipupdate
```

Verify the database file:

```bash
sudo ls -lh /usr/share/GeoIP/GeoLite2-Country.mmdb
```

GeoLite downloads require valid MaxMind account credentials and an active license key. If GeoIPUpdate returns `401`, verify the account ID, key, and database access in the MaxMind account portal. Do not repeatedly retry invalid credentials.

The GeoIP2 configuration specifies `auto_reload 5m`, allowing the module to check the database for updates.

## 10. Validation and reload procedure

Before applying any Nginx configuration change:

```bash
sudo nginx -t
```

Only if the syntax test succeeds:

```bash
sudo systemctl reload nginx
```

Verify that Nginx remains active:

```bash
systemctl is-active nginx
```

Expected output:

```text
active
```

Prefer reloads over restarts for normal configuration changes.

### Confirm the GeoIP country

The temporary public test endpoint used during initial setup returned `PH` from a Philippine browser and was then removed.

Do not leave a diagnostic endpoint publicly accessible. If country detection needs to be tested again, use a temporary, controlled diagnostic method and remove it afterward.

### Check media request logs

The media access log is:

`/var/log/nginx/jellyfin-media-access.log`

Inspect recent requests:

```bash
sudo tail -n 20 /var/log/nginx/jellyfin-media-access.log
```

The log format records the client IP, time, request method and path, HTTP status, bytes sent, and user agent. It deliberately omits query strings to reduce the risk of exposing tokens or other sensitive parameters.

Useful observations:

- `302` for a listed-country media request indicates that Nginx issued a redirect.
- `206` commonly indicates successful partial-content delivery through the proxy.
- `200` may indicate a normal response, including a response to a request that did not use range handling.
- A `302` confirms the redirect was issued; it does not alone prove that the direct endpoint successfully delivered the media.

### Test a client request carefully

A VPS-originated request is classified using the VPS public IP, not the Philippine viewer's IP. Therefore, a test from the VPS may correctly follow the default proxy route.

Avoid testing a large video with `curl` without a timeout. A `200` response can download the entire file even when a small `Range` header is supplied.

For redirect-only diagnostics, use a request that does not download a large response, and inspect the `Location` header. Be aware that a `HEAD` request can behave differently from a client's `GET` request.

## 11. Important limitation: no transparent fallback

The direct routing uses an HTTP `302` redirect to:

`https://media.eajelly.xyz:9000`

After the redirect, the client connects directly to the home endpoint. If the client cannot reach that endpoint, Nginx cannot transparently rescue the already-redirected connection.

A proxy-side health check only verifies that the VPS can reach home Jellyfin. It does not prove that an individual viewer can reach the direct endpoint.

If reliable fallback is required, design and test an explicit fallback mechanism rather than assuming a redirect can fail over automatically.

## 12. Troubleshooting checklist

### Nginx configuration fails

```bash
sudo nginx -t
```

Inspect the reported file and line before changing anything else.

### Nginx is not active

```bash
systemctl status nginx --no-pager
sudo journalctl -u nginx -n 50 --no-pager
```

### Check GeoIP configuration

```bash
sudo cat /etc/nginx/conf.d/geoip2-country.conf
sudo cat /etc/nginx/conf.d/geoip2-routing.conf
sudo ls -lh /usr/share/GeoIP/GeoLite2-Country.mmdb
```

Do not publish license keys or other secrets.

### Check the active site locations

```bash
sudo grep -nE 'location .*Images|location .*videos|location .*audio|jellyfin_media_route|proxy_pass' \
  /etc/nginx/sites-enabled/eajelly.xyz
```

Verify that video, audio, and artwork locations all use `$jellyfin_media_route` and that their existing proxy configurations remain intact.

### Check recent media routing

```bash
sudo tail -n 30 /var/log/nginx/jellyfin-media-access.log
```

Correlate the client IP, user agent, and status code. Avoid logging full query strings, which may contain sensitive values.

### Artwork cache

The header `X-Artwork-Cache` reports the cache status for requests that reach the artwork proxy. Direct-redirected artwork requests bypass that proxy and therefore do not use the VPS artwork cache.

## 13. Change-management rules

Before future changes:

1. Inspect the current configuration; do not assume it still matches this document.
2. Back up the specific file being edited.
3. Make one minimal change at a time.
4. Run `sudo nginx -t`.
5. Reload only after the syntax test succeeds.
6. Test the relevant route from a real client.
7. Check logs for actual status codes and user agents.
8. Preserve artwork caching, TLS SNI, Host headers, WebSocket support, and long media timeouts unless a change explicitly requires modifying them.
9. Never store API keys, GeoLite license keys, or other secrets in this document.

## 14. Current implementation status

Confirmed during the setup session:

- GeoIP2 Nginx module installed and loaded.
- GeoLite2 Country MMDB installed at `/usr/share/GeoIP/GeoLite2-Country.mmdb`.
- Country lookup configured using `$geoip2_country_code`.
- Routing map created for the listed Southeast Asian countries.
- Video and audio locations changed to use `$jellyfin_media_route`.
- GeoIP routing produced `302` responses for a Philippine browser and a Streamyfin `libmpv` stream request.
- Artwork location updated to use the same country-based redirect.
- Nginx configuration tests passed and the service was reloaded.
- The user reported that direct media and artwork loading worked smoothly.

Not independently verified:

- Direct connectivity and fallback behavior from clients in every listed country.
- Successful direct delivery for every type of artwork request.
- Any GeoIP routing changes to general metadata/API requests; none are documented as implemented.
