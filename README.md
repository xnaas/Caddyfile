# Caddyfile
[This is my Caddyfile](https://github.com/xnaas/Caddyfile/blob/master/Caddyfile). There are many like it, but this one is mine.

See also: [my Caddy Docker container](https://github.com/xnaas/caddy)

## Global Settings
[Global Options @ Caddy docs](https://caddyserver.com/docs/caddyfile/options#global-options)

```
{
  default_sni xnaas.info
  acme_ca https://acme-v02.api.letsencrypt.org/directory
  acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}
  email {$ACMEEMAIL}
}
```

### `default_sni xnaas.info`
Because multiple domains are hosted on the same machine, [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) is somewhat important. I chose my personal domain as the primary domain.

### `acme_ca https://acme-v02.api.letsencrypt.org/directory`
I simply manually set Let's Encrypt's ACME v2 server in case the default is ever changed in the future.

### `acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}`
This globally sets the ACME challenges to a specific DNS provider.

### `email {$ACMEEMAIL}`
This sets the email for ACME challenges to the `$ACMEEMAIL` environment variable.

## Snippets
[Snippets @ Caddy docs](https://caddyserver.com/docs/caddyfile/concepts#snippets)

```
(proxyheaders) {
  flush_interval -1
  header_up X-Real-IP {http.request.header.CF-Connecting-IP}
  header_up X-Forwarded-For {http.request.header.CF-Connecting-IP}
  header_up X-Forwarded-Host {hostport}
}
(main) {
  tls {
    client_auth {
      mode require_and_verify
      trusted_ca_cert_file /data/origin-pull-ca.pem
    }
  }
  header {
    Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    Content-Security-Policy "upgrade-insecure-requests"
    X-Frame-Options "SAMEORIGIN"
    X-Content-Type-Options "nosniff"
    X-XSS-Protection "1; mode=block"
    Referrer-Policy "strict-origin-when-cross-origin"
  }
  log {
    output file /data/logs/access.log {
      roll_size 10MiB
      roll_keep 10
      roll_keep_for 168h
    }
    format json
    level ERROR
  }
}
(asak) {
  basicauth {
    {$ASAK}
  }
}
(xadmin) {
  basicauth {
    {$XADMIN}
  }
}
(errors) {
  handle_errors {
    rewrite * /{http.error.status_code}
    reverse_proxy https://http.cat {
      header_up Host http.cat
    }
  }
}
```

### `(proxyheaders)`
This snippet sets `flush_interval -1` for proxied applicaitons to disable buffering. See: [Streaming @ Caddy docs](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy#streaming).

This snippet also makes sure that certain headers are passed through correctly to proxied applications for IP logging.

### `(main)`
#### tls
This snippet section forces [Authenticated Origin Pulls](https://support.cloudflare.com/hc/en-us/articles/204899617-Authenticated-Origin-Pulls).

#### header
This snippet section simply sets some key headers.

#### log
This defines logging in a certain way so that I can parse the output with fail2ban.

### `(asak)` | `(xadmin)`
These snippets define [basicauth](https://caddyserver.com/docs/caddyfile/directives/basicauth) for a certain logins.

### `(errors)`
This snippet defines custom error images for different HTTP error codes using the kindly provided [http.cat](https://http.cat) service.

## Addresses / Site Blocks
[Addresses @ Caddy docs](https://caddyserver.com/docs/caddyfile/concepts#addresses)

* Site Blocks are formatted as `https://example.com` or `https://sub.example.com` and all include `import main` at the very least to force proxying of traffic through Cloudflare
* `reverse_proxy` blocks must contain `import proxyheaders` for accurate IP logging
* `import errors` is optional and cannot be used in any site block with `import asak` or `import xadmin` as it, unfortunately, breaks basicauth to have custom error handling
* I've made an effort to move to subdomains for all applications instead of `/sub/paths` because it's just easier and cleaner in a Caddyfile
* If you want an emoji domain or subdomain, you must [convert the emoji to punycode](https://www.punycoder.com/) first
