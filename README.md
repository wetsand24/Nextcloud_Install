# Nextcloud on Ubuntu 22.04 (Apache, MariaDB, PHP, HTTPS)

Every command from the video, in order, to run your own Nextcloud instead of Google Drive or OneDrive.

▶️ Video: https://youtu.be/dD95YXpNwO0

**Change these to match your setup:**
| In the video | Yours |
|---|---|
| `nextcloud.itera.ca` (`yourdomain.com` in the config files) | the name you want for Nextcloud, e.g. `nextcloud.example.com` |
| `admin123` | your own database password (`YOUR-DB-PASSWORD` below) |
| `10.37.0.4` | your Ubuntu server's IP |

## What you need
- A 64-bit CPU and 64-bit Linux
- At least 128 MB RAM per PHP process (512 MB per process recommended)
- A supported OS, e.g. Ubuntu 24.04 LTS or 26.04 LTS, Debian 13, Red Hat Enterprise Linux 10 ([full list](https://docs.nextcloud.com/server/latest/admin_manual/installation/system_requirements.html))
- PHP 8.3, 8.4 or 8.5
- MariaDB 10.11, 11.4, 11.8 or 12.3 (or MySQL / PostgreSQL)
- Apache 2.4
- A local DNS server (Pi-hole, pfSense or your router) for the domain name (step 8)

> **Note (2026):** The video uses Ubuntu 22.04 and Nextcloud 24. Nextcloud 24 is end of life and gets no more
> security updates. Current Nextcloud needs PHP 8.3 or newer, and Ubuntu 22.04 installs PHP 8.1. For a new install,
> use one of the OS versions above and download `latest.zip` in step 5.

## 1. Update and install Apache + PHP
```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php -y
```

## 2. Install MariaDB
```bash
sudo apt install mariadb-server -y
```

## 3. Create the database and user
```bash
sudo mysql -u root
```
No password is asked. Then:
```sql
CREATE DATABASE nextcloud;
CREATE USER 'admin'@'localhost' IDENTIFIED BY 'YOUR-DB-PASSWORD';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'admin'@'localhost';
FLUSH PRIVILEGES;
exit;
```

## 4. Install the PHP modules
```bash
sudo apt install php-gd php-mysql php-curl php-xml php-mbstring php-zip php-intl -y
```

## 5. Download and extract Nextcloud
```bash
cd /var/www/html
sudo wget https://download.nextcloud.com/server/releases/nextcloud-24.0.1.zip
```
`unzip` is not on Ubuntu by default. Install it (in the video it was already there):
```bash
sudo apt install unzip
```
```bash
sudo unzip nextcloud-24.0.1.zip
```
This creates the folder `/var/www/html/nextcloud`.

> **Note (2026):** For the current version use
> `sudo wget https://download.nextcloud.com/server/releases/latest.zip` and `sudo unzip latest.zip` (see the note at the top).

## 6. Permissions and Apache virtual host
Make the web server user (`www-data`) the owner of the folder:
```bash
sudo chown -R www-data:www-data /var/www/html/nextcloud
sudo touch /etc/apache2/sites-available/nextcloud.conf
sudo nano /etc/apache2/sites-available/nextcloud.conf
```
Paste this and change `yourdomain.com` to your name (the same file is in this repo: [nextcloud.conf](nextcloud.conf)):
```apache
<VirtualHost *:80>
ServerName yourdomain.com
DocumentRoot /var/www/html/nextcloud

<Directory /var/www/html/nextcloud/>
 Require all granted
 Options FollowSymlinks MultiViews
 AllowOverride All
 <IfModule mod_dav.c>
 Dav off
 </IfModule>
</Directory>

ErrorLog /var/log/apache2/yourdomain.com.error_log
CustomLog /var/log/apache2/yourdomain.com.access_log common
</VirtualHost>
```
Save with `Ctrl+O`, `Enter`, exit with `Ctrl+X`. Then:
```bash
sudo a2ensite nextcloud.conf
sudo a2enmod rewrite
service apache2 restart
```

## 7. AllowOverride in apache2.conf
```bash
sudo nano /etc/apache2/apache2.conf
```
Press `Ctrl+W`, search for `override`. In the `<Directory /var/www/>` block, change `AllowOverride None` to:
```apache
AllowOverride All
```
Save, exit, restart Apache again:
```bash
service apache2 restart
```

## 8. Local DNS record
Because the virtual host uses a `ServerName`, your PC has to find that name. In the Pi-hole web page:
**Local DNS → DNS Records** → Domain: `nextcloud.itera.ca` → IP: `10.37.0.4` → **Add** (change both to yours).

> **Note (2026):** In Pi-hole v6 this page moved to **Settings → Local DNS Records**.

No Pi-hole yet? Setup video: https://youtu.be/k9GPXPp1KU0 (pfSense or your router work too).

## 9. Nextcloud setup wizard
Open `http://yourdomain.com` (your name) in the browser and fill in:
- Admin username and password: pick your own (this is your Nextcloud login)
- Data folder: leave the default (`/var/www/html/nextcloud/data`)
- Database user: `admin`
- Database password: `YOUR-DB-PASSWORD`
- Database name: `nextcloud`
- Database host: leave `localhost`

Click **Install** and wait a few minutes. On the next screen click **Install recommended apps**.

## 10. Nextcloud dashboard
From here you can upload files and photos, work on documents together, and use video calls, chat, mail, contacts
and calendar.

## 11. HTTPS with a self-signed certificate
Right now Nextcloud only runs on port 80 (not encrypted). Make a copy of the virtual host for port 443:
```bash
cd /etc/apache2/sites-available
ls
sudo cp nextcloud.conf nextcloudssl.conf
```
Open the default SSL site to see where Ubuntu's built-in self-signed certificate is. Copy the
`SSLCertificateFile` and `SSLCertificateKeyFile` lines, then exit without saving (`Ctrl+X`):
```bash
sudo nano default-ssl.conf
```
Edit the copy:
```bash
sudo nano nextcloudssl.conf
```
Change `<VirtualHost *:80>` to `<VirtualHost *:443>`, and under `</Directory>` add `SSLEngine on` plus the two lines
you copied. It should look like this ([nextcloudssl.conf](nextcloudssl.conf)):
```apache
<VirtualHost *:443>
ServerName yourdomain.com
DocumentRoot /var/www/html/nextcloud

<Directory /var/www/html/nextcloud/>
 Require all granted
 Options FollowSymlinks MultiViews
 AllowOverride All
 <IfModule mod_dav.c>
 Dav off
 </IfModule>
</Directory>

SSLEngine on
SSLCertificateFile      /etc/ssl/certs/ssl-cert-snakeoil.pem
SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key

ErrorLog /var/log/apache2/yourdomain.com.error_log
CustomLog /var/log/apache2/yourdomain.com.access_log common
</VirtualHost>
```
Save, exit, then enable the site and the SSL module:
```bash
sudo a2ensite nextcloudssl.conf
sudo a2enmod ssl
service apache2 restart
```
Open `https://yourdomain.com`. The browser says **Your connection is not private** because the certificate is self-signed. The traffic is
still encrypted.

**Want a trusted certificate without the warning?** Put Nextcloud behind Nginx Proxy Manager with Let's Encrypt:
https://youtu.be/_PhecuWHe4M?t=376

## Links
- Nextcloud: https://nextcloud.com · admin docs: https://docs.nextcloud.com/server/latest/admin_manual/
- Nextcloud downloads: https://download.nextcloud.com/server/releases/
- More self-hosting: [Vaultwarden](https://youtu.be/VUgmH43ek1g) · [Docker + Portainer](https://youtu.be/3W9Gl5AHSMo) · [WordPress on Ubuntu](https://youtu.be/NI_XZAp1tPo)
