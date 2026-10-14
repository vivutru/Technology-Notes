# PHP Setup on Arch Linux

A step-by-step guide to set up PHP for web development using Apache, MariaDB, phpMyAdmin, Xdebug and Composer.

Follow the steps in order. Each step explains what the tool does, why you might want it, and how it changes the way you work.

---

## What you are building

When you open a PHP website in your browser, several programs work together:

1. The browser asks for a page.
2. Apache (the web server) receives the request.
3. PHP runs your code and builds the page.
4. MariaDB (the database) stores and returns data if your code needs it.
5. Apache sends the finished page back to the browser.

phpMyAdmin, Xdebug and Composer are helpers that make your life easier.

---

## Step 1: Install everything

```
sudo pacman -Syu
sudo pacman -S apache php php-apache php-gd php-intl php-sqlite mariadb phpmyadmin xdebug composer
```

| Package | What it is | Why you need it |
|---|---|---|
| `apache` | The web server | Shows your PHP pages in a browser |
| `php` | The PHP language itself | Runs your code |
| `php-apache` | A bridge between Apache and PHP | Lets Apache run `.php` files |
| `php-gd` | Image tools for PHP | Resizing images, making thumbnails, captchas |
| `php-intl` | Language and region tools | Date, number and currency formatting. Many frameworks need it |
| `php-sqlite` | Small file-based database support | Handy for tiny projects that do not need a full database |
| `mariadb` | The database | Stores users, posts, products and anything else your site remembers |
| `phpmyadmin` | Database manager in the browser | Lets you see and edit your data by clicking instead of typing commands |
| `xdebug` | A debugging tool | Lets you pause your code and inspect it step by step |
| `composer` | PHP package manager | Installs libraries and frameworks like Laravel with one command |

**How it affects your workflow:** After this step everything is on your computer, but nothing is connected yet. The next steps connect the pieces.

---

## Step 2: Connect Apache to PHP

Apache has several "modes" for handling visitors. The default mode does not work with PHP, so we switch to one that does (called `prefork`), then tell Apache to load PHP.

Switch the mode:

```
sudo sed -i 's/^LoadModule mpm_event_module/#LoadModule mpm_event_module/' /etc/httpd/conf/httpd.conf
sudo sed -i 's/^#LoadModule mpm_prefork_module/LoadModule mpm_prefork_module/' /etc/httpd/conf/httpd.conf
```

Add the PHP settings to the end of the Apache config file:

```
sudo tee -a /etc/httpd/conf/httpd.conf <<'EOF'
LoadModule php_module modules/libphp.so
AddHandler php-script .php
Include conf/extra/php_module.conf
ServerName localhost
EOF
```

What those lines do:

- `LoadModule php_module ...` loads PHP into Apache.
- `AddHandler php-script .php` tells Apache that files ending in `.php` should be run by PHP, not shown as plain text.
- `Include conf/extra/php_module.conf` loads extra PHP settings that come with the package.
- `ServerName localhost` gives Apache a name for itself. Without it you get a harmless but annoying warning every time it starts.

Check for mistakes, then start Apache:

```
sudo apachectl configtest
sudo systemctl enable --now httpd
```

You want to see `Syntax OK`. The `enable --now` part starts Apache immediately and makes it start automatically every time you turn on your computer.

**How it affects your workflow:** Apache now runs in the background all the time. Any change you make to Apache settings needs a restart to take effect:

```
sudo systemctl restart httpd
```

---

## Step 3: Set up your website folder

Apache serves files from `/srv/http`. By default only the system can edit it, so give yourself permission:

```
sudo chown -R $USER:http /srv/http
```

Make a test page:

```
echo '<?php phpinfo();' > /srv/http/info.php
```

Open `http://localhost/info.php` in your browser. If you see a purple and blue page full of PHP details, everything works so far.

**Delete the test page when you are done.** It shows details about your system that you do not want to leave lying around:

```
rm /srv/http/info.php
```

**How it affects your workflow:** Every project you want to view in the browser goes inside `/srv/http`. For example, a folder called `/srv/http/myproject` opens at `http://localhost/myproject`.

For a quick test of a single project you can also skip Apache and run this from inside the project folder:

```
php -S localhost:8000
```

---

## Step 4: Turn on PHP extensions

PHP comes with many optional features called extensions. They are off by default. This turns on the ones most projects need:

```
sudo sed -i -E 's/^;extension=(curl|mysqli|pdo_mysql|zip|gd|intl|iconv|pdo_sqlite|sqlite3)$/extension=\1/' /etc/php/php.ini
```

| Extension | What it does |
|---|---|
| `mysqli` and `pdo_mysql` | Let PHP talk to MariaDB. Almost every project needs at least one |
| `curl` | Lets PHP download things from the internet and call other websites |
| `zip` | Lets PHP open and create zip files. Composer uses it |
| `gd` | Image handling |
| `intl` | Language and region formatting |
| `iconv` | Text encoding conversion. phpMyAdmin needs it |
| `pdo_sqlite` and `sqlite3` | Let PHP use small file-based SQLite databases |

Check that they loaded:

```
php -m
```

This prints every active extension. Restart Apache afterward:

```
sudo systemctl restart httpd
```

**How it affects your workflow:** If a project shows an error like "Call to undefined function" or "could not find driver", an extension is missing. Open `/etc/php/php.ini`, search for the extension name, remove the `;` at the start of its line, and restart Apache. Some extensions need an extra package from `pacman` first.

---

## Step 5: Set up the database (MariaDB)

Create the database storage area and start the service:

```
sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
sudo systemctl enable --now mariadb
```

Run the security helper:

```
sudo mariadb-secure-installation
```

It asks several questions. Good answers for a personal computer:

| Question | Answer |
|---|---|
| Enter current password for root | Press Enter (there is none yet) |
| Switch to unix_socket authentication | `Y` |
| Change the root password | `Y`, then pick one you will remember |
| Remove anonymous users | `Y` |
| Disallow root login remotely | `Y` |
| Remove test database | `Y` |
| Reload privilege tables | `Y` |

**How it affects your workflow:** MariaDB now runs in the background. You can open its command line any time with `sudo mariadb`.

### Make your own database user

Your projects should not log in as root. Create a normal user:

```
sudo mariadb
```

Then type these one line at a time. Replace `your-password-here` with a real password:

```
CREATE USER 'london'@'localhost' IDENTIFIED BY 'your-password-here';
GRANT ALL PRIVILEGES ON *.* TO 'london'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

- The first line creates the user.
- The second gives it full control over all databases. This is fine on your own computer but not on a public server.
- The third applies the changes.

To change the password later:

```
ALTER USER 'london'@'localhost' IDENTIFIED BY 'new-password';
```

**How it affects your workflow:** Use this username and password in phpMyAdmin and in your PHP code. Root will not work in phpMyAdmin because it logs in through the system, not with a password.

---

## Step 6: Set up phpMyAdmin

phpMyAdmin is a website that lets you manage your databases with buttons and forms. You can create tables, browse data, import and export files, and run queries without using the command line.

Create its Apache settings file:

```
sudo tee /etc/httpd/conf/extra/phpmyadmin.conf <<'EOF'
Alias /phpmyadmin "/usr/share/webapps/phpMyAdmin"
<Directory "/usr/share/webapps/phpMyAdmin">
    DirectoryIndex index.php
    AllowOverride All
    Options FollowSymlinks
    Require all granted
</Directory>
EOF
```

This makes the address `http://localhost/phpmyadmin` point to where phpMyAdmin is installed.

Tell Apache to load that file:

```
echo 'Include conf/extra/phpmyadmin.conf' | sudo tee -a /etc/httpd/conf/httpd.conf
```

Generate a secret key. phpMyAdmin uses it to protect your login cookie:

```
openssl rand -hex 16
```

Copy the result, then open the phpMyAdmin settings:

```
sudo nano /etc/webapps/phpmyadmin/config.inc.php
```

Find this line and paste the key between the quotes:

```
$cfg['blowfish_secret'] = '';
```

Save and exit (Ctrl+O, Enter, Ctrl+X), then restart Apache:

```
sudo apachectl configtest
sudo systemctl restart httpd
```

Open `http://localhost/phpmyadmin` and log in with the database user from Step 5.

**How it affects your workflow:** You will use this a lot early on to create databases and check that your code is saving data correctly.

---

## Step 7: Turn on Xdebug

Installing Xdebug is not enough. It has to be switched on:

```
sudo sed -i 's/^;zend_extension/zend_extension/' /etc/php/conf.d/xdebug.ini
sudo systemctl restart httpd
```

Check that it is active:

```
php -m | grep -i xdebug
```

**What it does:** Normally when your code breaks, you guess where the problem is by adding `echo` lines everywhere. Xdebug lets your code editor pause the program at any line, so you can see what every variable contains at that moment. It also gives clearer error messages.

**How it affects your workflow:** It makes finding bugs much faster, but it needs an editor that supports it (VS Code with a PHP debug extension, or Neovim with a debug plugin). It also slows PHP down a little. If things feel slow and you are not debugging, turn it off by putting the `;` back at the start of the `zend_extension` line in `/etc/php/conf.d/xdebug.ini` and restarting Apache.

---

## Step 8: Composer

Composer is already installed from Step 1. Check it:

```
composer --version
```

**What it does:** Most modern PHP projects use ready-made code written by other people. Composer downloads that code and keeps track of it. You list what you need in a file called `composer.json` and Composer fetches it all.

Common uses:

```
composer create-project laravel/laravel myproject
composer require vendor/package-name
composer install
composer update
```

- `create-project` starts a brand new project from a template.
- `require` adds a library to your project.
- `install` downloads everything listed in an existing project. Run this after cloning someone else's project.
- `update` upgrades your libraries to newer versions.

**How it affects your workflow:** Any time you download a project from GitHub, run `composer install` inside it first. It creates a `vendor` folder with everything the project needs.

---

## Fixing slow or broken downloads (mirrors)

Pacman downloads packages from servers called mirrors. Sometimes one goes out of date and shows an error like:

```
error: failed retrieving file ... The requested URL returned error: 410
```

Pacman usually recovers by using another mirror, so the install still finishes. To stop seeing the error, disable the bad one:

```
sudo cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
sudo sed -i '/mirrors.logal.dev/s/^/#/' /etc/pacman.d/mirrorlist
```

Or let a tool pick fast mirrors for you. This can take a few minutes, so let it finish:

```
sudo pacman -S reflector
sudo reflector --latest 20 --sort rate --protocol https --download-timeout 15 --save /etc/pacman.d/mirrorlist
sudo pacman -Syu
```

---

## Everyday commands

| What you want | Command |
|---|---|
| Restart Apache after changing settings | `sudo systemctl restart httpd` |
| Check Apache settings for mistakes | `sudo apachectl configtest` |
| Restart the database | `sudo systemctl restart mariadb` |
| See if Apache is running | `systemctl status httpd` |
| See Apache error messages | `sudo tail -f /var/log/httpd/error_log` |
| See which PHP extensions are on | `php -m` |
| Open the database command line | `sudo mariadb` |
| Stop Apache and MariaDB to save resources | `sudo systemctl stop httpd mariadb` |
| Start them again | `sudo systemctl start httpd mariadb` |

## Important file locations

| What | Where |
|---|---|
| Your website files | `/srv/http` |
| Apache settings | `/etc/httpd/conf/httpd.conf` |
| PHP settings | `/etc/php/php.ini` |
| Xdebug settings | `/etc/php/conf.d/xdebug.ini` |
| phpMyAdmin settings | `/etc/webapps/phpmyadmin/config.inc.php` |
| Apache error log | `/var/log/httpd/error_log` |

## When something goes wrong

| Problem | Likely cause | Fix |
|---|---|---|
| Apache will not start | A typo in a config file, or a file that is missing | Run `sudo apachectl configtest`. Then run `journalctl -xeu httpd.service \| tail -20` |
| Browser downloads the PHP file or shows raw code | PHP is not connected to Apache | Recheck Step 2 and restart Apache |
| "Could not find driver" or "undefined function" | A PHP extension is off | Turn it on in `/etc/php/php.ini` (Step 4) |
| "Forbidden" or "Permission denied" in the browser | Wrong file permissions | Run `sudo chown -R $USER:http /srv/http` |
| phpMyAdmin says "Access denied" | Using root or the wrong password | Log in with the user you created in Step 5 |
| phpMyAdmin warns about `blowfish_secret` | The secret key is empty | Fill it in as shown in Step 6 |
| Config file "not found" | A typo in the filename, such as a stray `:` at the end | Check the exact name with `ls /etc/httpd/conf/extra/` |
| Changes to settings do nothing | Apache still has the old settings loaded | Restart Apache |
| Site is slow | Xdebug is on when you are not debugging | Turn it off (Step 7) |

## After system updates

When `sudo pacman -Syu` updates PHP or Apache, restart the services so they use the new versions:

```
sudo systemctl restart httpd mariadb
```

If PHP jumps to a new major version, run `php -m` again to check your extensions are still on, and read any messages pacman shows about `.pacnew` files.
