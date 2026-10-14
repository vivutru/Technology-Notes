# Apache HTTP Server: A Practical Guide

Apache HTTP Server (`httpd`) is one of the most widely used open-source web servers. It serves static files, runs dynamic content through modules or FastCGI, acts as a reverse proxy, and is configured through plain-text files.

---

## 1. Key Concepts

| Concept | Meaning |
|---|---|
| **httpd / apache2** | The server binary. Named `httpd` on RHEL/Fedora/Arch, `apache2` on Debian/Ubuntu |
| **Module** | A plug-in that adds features (SSL, rewrite, proxy, etc.) |
| **MPM** | Multi-Processing Module: decides how Apache handles concurrent connections |
| **Virtual host** | Lets one server host many sites, selected by IP, port, or hostname |
| **Directive** | A configuration instruction, e.g. `DocumentRoot /var/www/html` |
| **Context** | Where a directive can appear: server config, `<VirtualHost>`, `<Directory>`, `.htaccess` |
| **DocumentRoot** | The folder served as the site's root |

---

## 2. Installation

```bash
# Debian / Ubuntu
sudo apt update
sudo apt install apache2

# RHEL / Rocky / Alma / Fedora
sudo dnf install httpd

# Start and enable at boot
sudo systemctl enable --now apache2      # Debian/Ubuntu
sudo systemctl enable --now httpd        # RHEL family
```

Open the firewall if needed:

```bash
# UFW (Ubuntu)
sudo ufw allow 'Apache Full'

# firewalld (RHEL family)
sudo firewall-cmd --permanent --add-service={http,https}
sudo firewall-cmd --reload
```

Test by visiting `http://your-server-ip/` or running `curl -I http://localhost`.

---

## 3. File Layout

| Item | Debian / Ubuntu | RHEL family |
|---|---|---|
| Main config | `/etc/apache2/apache2.conf` | `/etc/httpd/conf/httpd.conf` |
| Extra config | `/etc/apache2/conf-available/`, `conf-enabled/` | `/etc/httpd/conf.d/*.conf` |
| Sites | `/etc/apache2/sites-available/`, `sites-enabled/` | `/etc/httpd/conf.d/` |
| Modules | `mods-available/`, `mods-enabled/` | `/etc/httpd/conf.modules.d/` |
| Default web root | `/var/www/html` | `/var/www/html` |
| Logs | `/var/log/apache2/` | `/var/log/httpd/` |
| Runs as user | `www-data` | `apache` |

Debian-style systems use helper commands that create or remove symlinks between `*-available` and `*-enabled`:

```bash
sudo a2ensite  mysite.conf     # enable site
sudo a2dissite mysite.conf     # disable site
sudo a2enmod   rewrite ssl     # enable modules
sudo a2dismod  status
sudo a2enconf  security        # enable a config snippet
```

On RHEL-family systems, add or remove files in `conf.d/` and edit `LoadModule` lines directly.

---

## 4. Managing the Service

```bash
sudo systemctl start   apache2
sudo systemctl stop    apache2
sudo systemctl restart apache2     # full stop/start, drops connections
sudo systemctl reload  apache2     # graceful: re-reads config without dropping requests
sudo systemctl status  apache2

# Always test config before reloading
sudo apachectl configtest          # or: apache2ctl -t / httpd -t
sudo apachectl -S                  # show parsed virtual hosts
sudo apachectl -M                  # list loaded modules
apache2 -V                         # version and build info
```

> **Habit to build:** run `configtest` before every reload. A syntax error can stop the server from coming back up.

---

## 5. Core Configuration Directives

```apache
ServerName   www.example.com        # the main hostname
ServerAdmin  admin@example.com
DocumentRoot "/var/www/example"
Listen 80
ErrorLog  ${APACHE_LOG_DIR}/error.log
LogLevel  warn
DirectoryIndex index.html index.php
Timeout 60
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5
ServerTokens Prod                   # hide version details in headers
ServerSignature Off
```

### Directory permissions

```apache
<Directory "/var/www/example">
    Options -Indexes +FollowSymLinks
    AllowOverride None              # ignore .htaccess (faster, safer)
    Require all granted
</Directory>
```

| `Options` value | Effect |
|---|---|
| `Indexes` | Show directory listing when no index file exists (usually disable) |
| `FollowSymLinks` | Follow symbolic links |
| `ExecCGI` | Allow CGI scripts |
| `MultiViews` | Content negotiation by filename |

### Access control (Apache 2.4 syntax)

```apache
Require all granted
Require all denied
Require ip 192.168.1.0/24
Require host example.com
Require user alice bob
Require valid-user
```

Apache 2.2's `Order`/`Allow`/`Deny` is obsolete; use `Require` on 2.4+.

---

## 6. Virtual Hosts

Host several sites on one server.

### Name-based virtual host

```apache
<VirtualHost *:80>
    ServerName  example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/example.com/public

    <Directory /var/www/example.com/public>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog  ${APACHE_LOG_DIR}/example.com-error.log
    CustomLog ${APACHE_LOG_DIR}/example.com-access.log combined
</VirtualHost>
```

Steps on Debian/Ubuntu:

```bash
sudo mkdir -p /var/www/example.com/public
sudo chown -R www-data:www-data /var/www/example.com
sudo nano /etc/apache2/sites-available/example.com.conf
sudo a2ensite example.com.conf
sudo apachectl configtest && sudo systemctl reload apache2
```

The first virtual host defined for a given IP:port is the **default** used when no `ServerName` matches. Use `apachectl -S` to see order and matches.

---

## 7. HTTPS / TLS

### Manual SSL virtual host

```bash
sudo a2enmod ssl headers
```

```apache
<VirtualHost *:443>
    ServerName example.com
    DocumentRoot /var/www/example.com/public

    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/example.com.crt
    SSLCertificateKeyFile /etc/ssl/private/example.com.key
    SSLCertificateChainFile /etc/ssl/certs/chain.pem   # or use fullchain in CertificateFile

    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
    SSLHonorCipherOrder off

    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
</VirtualHost>
```

### Free certificates with Let's Encrypt

```bash
sudo apt install certbot python3-certbot-apache
sudo certbot --apache -d example.com -d www.example.com
sudo certbot renew --dry-run        # test auto-renewal
```

### Redirect HTTP to HTTPS

```apache
<VirtualHost *:80>
    ServerName example.com
    Redirect permanent / https://example.com/
</VirtualHost>
```

Test your setup with SSL Labs' server test or `openssl s_client -connect example.com:443`.

---

## 8. Essential Modules

| Module | Purpose | Enable (Debian) |
|---|---|---|
| `mod_rewrite` | URL rewriting and redirects | `a2enmod rewrite` |
| `mod_ssl` | HTTPS | `a2enmod ssl` |
| `mod_headers` | Set/modify HTTP headers | `a2enmod headers` |
| `mod_proxy` + `proxy_http` | Reverse proxy | `a2enmod proxy proxy_http` |
| `mod_proxy_fcgi` | Talk to PHP-FPM and other FastCGI apps | `a2enmod proxy_fcgi` |
| `mod_deflate` | Gzip compression | `a2enmod deflate` |
| `mod_expires` | Cache-control for static files | `a2enmod expires` |
| `mod_http2` | HTTP/2 support | `a2enmod http2` |
| `mod_security2` | Web application firewall | `apt install libapache2-mod-security2` |
| `mod_status` | Live server status page | `a2enmod status` |
| `mod_auth_basic` | Password prompts | `a2enmod auth_basic authn_file` |

---

## 9. URL Rewriting and Redirects

```apache
RewriteEngine On

# Force HTTPS
RewriteCond %{HTTPS} off
RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]

# Remove www
RewriteCond %{HTTP_HOST} ^www\.(.+)$ [NC]
RewriteRule ^ https://%1%{REQUEST_URI} [L,R=301]

# Front controller for single-page apps / frameworks
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^ /index.php [L]
```

Simple redirects don't need rewrite:

```apache
Redirect permanent /old-page https://example.com/new-page
```

Common flags: `L` (last rule), `R=301` (redirect), `NC` (case-insensitive), `QSA` (keep query string), `F` (403), `P` (proxy).

### .htaccess

A `.htaccess` file applies directives per directory without restarting the server, but only if `AllowOverride` permits it. It is read on every request, so prefer putting rules in the main config when you control the server. Use `.htaccess` for shared hosting or apps that ship with their own rules (WordPress, Laravel).

---

## 10. Reverse Proxy and Load Balancing

### Proxy to an app server

```apache
<VirtualHost *:443>
    ServerName app.example.com

    ProxyPreserveHost On
    ProxyPass        / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/

    RequestHeader set X-Forwarded-Proto "https"
</VirtualHost>
```

### WebSockets

```apache
ProxyPass "/ws/" "ws://127.0.0.1:3000/ws/"
```
(requires `proxy_wstunnel`)

### Load balancer

```bash
sudo a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests
```
```apache
<Proxy "balancer://mycluster">
    BalancerMember http://10.0.0.11:8080
    BalancerMember http://10.0.0.12:8080
    ProxySet lbmethod=byrequests
</Proxy>
ProxyPass        / balancer://mycluster/
ProxyPassReverse / balancer://mycluster/
```

---

## 11. Running PHP

**Recommended: PHP-FPM with the event MPM** (faster and more memory-efficient than `mod_php`).

```bash
sudo apt install php-fpm
sudo a2dismod php8.3 mpm_prefork           # if previously using mod_php
sudo a2enmod mpm_event proxy_fcgi setenvif
sudo a2enconf php8.3-fpm
sudo systemctl restart apache2
```

Manual handler per site:

```apache
<FilesMatch "\.php$">
    SetHandler "proxy:unix:/run/php/php8.3-fpm.sock|fcgi://localhost"
</FilesMatch>
```

(Adjust the PHP version and socket path for your system.)

---

## 12. Multi-Processing Modules (MPMs)

| MPM | Model | Best for |
|---|---|---|
| **event** | Threads + dedicated threads for keep-alive connections | Default on modern distros; best general choice; works with PHP-FPM |
| **worker** | Multiple processes, each with multiple threads | Thread-safe workloads |
| **prefork** | One process per connection, no threads | Non-thread-safe modules such as legacy `mod_php` |

Only one MPM can be active. Check with `apachectl -V | grep -i mpm`.

Example event MPM tuning:

```apache
<IfModule mpm_event_module>
    StartServers             3
    MinSpareThreads         75
    MaxSpareThreads        250
    ThreadsPerChild         25
    MaxRequestWorkers      400
    MaxConnectionsPerChild   0
</IfModule>
```

Size `MaxRequestWorkers` so that workers × average memory use fits comfortably in RAM, or the server will swap.

---

## 13. Performance Tuning

```apache
# Compression
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html text/css application/javascript application/json image/svg+xml
</IfModule>

# Browser caching
<IfModule mod_expires.c>
    ExpiresActive On
    ExpiresByType image/webp  "access plus 1 year"
    ExpiresByType text/css    "access plus 1 month"
    ExpiresByType application/javascript "access plus 1 month"
</IfModule>

# HTTP/2
Protocols h2 http/1.1
```

Other tips:
- Set `AllowOverride None` wherever possible to avoid `.htaccess` lookups.
- Keep `KeepAlive On` with a short timeout (2–5 seconds).
- Use the event MPM and PHP-FPM.
- Serve static assets via a CDN.
- Disable unused modules.
- Measure with `ab` (ApacheBench), `wrk`, or `hey` before and after changes.

---

## 14. Security Hardening

```apache
ServerTokens Prod
ServerSignature Off
TraceEnable Off

<Directory />
    AllowOverride None
    Require all denied
</Directory>

# Security headers
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "SAMEORIGIN"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Content-Security-Policy "default-src 'self'"

# Block hidden files (.git, .env, .htaccess)
<FilesMatch "^\.">
    Require all denied
</FilesMatch>
```

Checklist:
1. Keep Apache and the OS patched.
2. Run under a dedicated unprivileged user (default).
3. Disable directory listing (`-Indexes`).
4. Enable HTTPS everywhere with modern TLS versions only.
5. Restrict file permissions: web content readable, not writable, by the Apache user.
6. Protect admin areas with authentication and IP restrictions.
7. Consider `mod_security` with the OWASP Core Rule Set and `fail2ban` for repeated abusive requests.
8. On RHEL/CentOS, leave **SELinux** enabled and set proper contexts, e.g. `httpd_sys_content_t`, and booleans such as `httpd_can_network_connect` for proxying.

### Password-protecting a directory

```bash
sudo htpasswd -c /etc/apache2/.htpasswd alice
```
```apache
<Directory /var/www/example.com/private>
    AuthType Basic
    AuthName "Restricted"
    AuthUserFile /etc/apache2/.htpasswd
    Require valid-user
</Directory>
```

Use this only over HTTPS, since Basic auth sends credentials in an easily decoded form.

---

## 15. Logging

Default log files:

| Log | Content |
|---|---|
| `access.log` | Every request |
| `error.log` | Errors and diagnostics |

### Custom format

```apache
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
CustomLog ${APACHE_LOG_DIR}/access.log combined
```

| Token | Meaning |
|---|---|
| `%h` | Client IP (use `%{X-Forwarded-For}i` behind a proxy, or `mod_remoteip`) |
| `%t` | Time |
| `%r` | Request line |
| `%>s` | Final status code |
| `%b` | Response size in bytes |
| `%D` | Time taken in microseconds |

### Useful log commands

```bash
sudo tail -f /var/log/apache2/error.log
sudo awk '{print $9}' /var/log/apache2/access.log | sort | uniq -c | sort -rn   # status code counts
sudo awk '{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -rn | head   # top IPs
```

Logs are rotated by `logrotate` (`/etc/logrotate.d/apache2`). Raise verbosity temporarily with `LogLevel debug` or per module: `LogLevel warn rewrite:trace3`.

---

## 16. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| Server won't start | Run `apachectl configtest`; check `journalctl -xeu apache2` and the error log |
| "Address already in use" | Another service (often nginx) holds port 80/443: `sudo ss -tlnp \| grep ':80'` |
| 403 Forbidden | Missing `Require all granted`, wrong file permissions, SELinux context, or no index file with `-Indexes` |
| 404 Not Found | Wrong `DocumentRoot`, wrong filename case (Linux is case-sensitive), or rewrite rule issue |
| 500 Internal Server Error | Check error log; often a bad `.htaccess` directive, missing module, or PHP error |
| 502 / 503 from proxy | Backend app is down, wrong socket or port, or `MaxRequestWorkers` reached |
| `.htaccess` ignored | `AllowOverride` is `None`, or the module (e.g. `rewrite`) is not enabled |
| Wrong site is served | Check `ServerName`/`ServerAlias` and `apachectl -S`; first vhost is the default |
| "Could not reliably determine the server's fully qualified domain name" | Set a global `ServerName` (can be `localhost`) |
| Changes not showing | Did you reload? Browser/CDN cache? Test with `curl -I` |
| SSL errors | Wrong chain, expired cert, key mismatch, or `ssl` module not enabled |

Handy diagnostics:

```bash
curl -I http://localhost
curl -vk https://example.com
apachectl -S
apachectl -M | sort
ls -lZ /var/www/html                 # SELinux contexts
```

---

## 17. Apache vs Nginx (Short Comparison)

| | Apache | Nginx |
|---|---|---|
| Model | Process/thread (event MPM is event-driven hybrid) | Event-driven, asynchronous |
| Config | Directives, `.htaccess` per-directory overrides | Central config only |
| Dynamic content | Modules or FastCGI | FastCGI/proxy only |
| Flexibility | Very wide module ecosystem | Leaner, strong as proxy/static server |
| Typical use | Shared hosting, legacy PHP apps, rich per-dir config | High-concurrency static serving, reverse proxy, load balancing |

Both are solid; many setups put Nginx in front as a reverse proxy and use Apache behind it.

---

## 18. Cheat Sheet

| Task | Command |
|---|---|
| Install (Debian) | `sudo apt install apache2` |
| Install (RHEL) | `sudo dnf install httpd` |
| Test config | `sudo apachectl configtest` |
| Reload gracefully | `sudo systemctl reload apache2` |
| Restart | `sudo systemctl restart apache2` |
| Enable site | `sudo a2ensite site.conf` |
| Disable site | `sudo a2dissite site.conf` |
| Enable module | `sudo a2enmod rewrite` |
| List vhosts | `sudo apachectl -S` |
| List modules | `sudo apachectl -M` |
| Version | `apache2 -v` |
| Follow error log | `sudo tail -f /var/log/apache2/error.log` |
| Get TLS cert | `sudo certbot --apache -d example.com` |
| Create password file | `sudo htpasswd -c /etc/apache2/.htpasswd user` |
| Benchmark | `ab -n 1000 -c 50 http://localhost/` |

---

## 19. Best Practices

1. **Test, then reload**: `configtest` every time.
2. **One vhost file per site**, named clearly, with its own logs.
3. **Prefer main config over `.htaccess`** for speed and control.
4. **Use HTTPS everywhere** and redirect HTTP to it.
5. **Hide version info**, disable directory listings, and add security headers.
6. **Use event MPM + PHP-FPM** for PHP workloads.
7. **Keep configs in version control** (e.g. Git) and automate with Ansible or similar.
8. **Monitor** logs, `mod_status`, and resource usage; set up log rotation and alerts.
9. **Back up** `/etc/apache2` (or `/etc/httpd`) and your web root.

---

*Notes: paths and package names assume Debian/Ubuntu unless noted; adjust for your distribution. Directives shown target Apache 2.4. Official documentation: <https://httpd.apache.org/docs/2.4/>.*
