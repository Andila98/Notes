# Nginx Setup Guide on Ubuntu

A practical guide for setting up Nginx on Ubuntu as a reverse proxy and static/Express file server.

---

## 1. Install Nginx

Update your package index and install Nginx:

```bash
sudo apt update
sudo apt install nginx -y
```

Verify the installation:

```bash
nginx -v
```

---

## 2. Start and Enable Nginx

Start the service and enable it to run on boot:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

Check that it's running:

```bash
sudo systemctl status nginx
```

You should see `active (running)` in the output. Opening your server's IP address in a browser should display the default Nginx welcome page.

---

## 3. Understand the Directory Structure

Before editing configs, get familiar with where things live:

| Path | Purpose |
|------|---------|
| `/etc/nginx/nginx.conf` | Main Nginx configuration file |
| `/etc/nginx/sites-available/` | Configuration files for your sites (inactive) |
| `/etc/nginx/sites-enabled/` | Symlinks to active site configs |
| `/var/www/html/` | Default web root for static files |
| `/var/log/nginx/access.log` | Access logs |
| `/var/log/nginx/error.log` | Error logs |

The standard workflow is: write a config in `sites-available/`, then symlink it into `sites-enabled/` to activate it.

---

## 4. Configure the Firewall

If UFW is active, allow Nginx traffic:

```bash
sudo ufw allow 'Nginx Full'
sudo ufw status
```

`Nginx Full` opens both port 80 (HTTP) and port 443 (HTTPS).

---

## 5. Serve Static Files

To serve static files (HTML, CSS, JS, images), create a site config:

```bash
sudo nano /etc/nginx/sites-available/mysite
```

Add the following configuration:

```nginx
server {
    listen 80;
    server_name your_domain_or_ip;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Create the web root and add a test file:

```bash
sudo mkdir -p /var/www/mysite
echo "<h1>Hello from Nginx</h1>" | sudo tee /var/www/mysite/index.html
```

Set the correct ownership:

```bash
sudo chown -R www-data:www-data /var/www/mysite
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
```

---

## 6. Configure a Reverse Proxy (Express App)

If your Express app is running locally (e.g. on port 3000), add a `location` block to proxy traffic to it:

```nginx
server {
    listen 80;
    server_name your_domain_or_ip;

    # Serve static files directly
    root /var/www/mysite;
    index index.html;

    location /static/ {
        try_files $uri $uri/ =404;
    }

    # Proxy API or app requests to Express
    location /api/ {
        proxy_pass http://127.0.0.1:3000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }
}
```

If you want Nginx to proxy **all** traffic to Express (i.e. Express handles everything including static files):

```nginx
server {
    listen 80;
    server_name your_domain_or_ip;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## 7. Test and Reload Nginx

Always test your config before reloading:

```bash
sudo nginx -t
```

You should see:
```
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If the test passes, reload Nginx to apply changes:

```bash
sudo systemctl reload nginx
```

---

## 8. Remove the Default Site

The default Nginx site can conflict with your custom config. Disable it:

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl reload nginx
```

---

## 9. Common Commands Reference

| Command | Purpose |
|---------|---------|
| `sudo systemctl start nginx` | Start Nginx |
| `sudo systemctl stop nginx` | Stop Nginx |
| `sudo systemctl restart nginx` | Restart Nginx (full restart) |
| `sudo systemctl reload nginx` | Reload config without downtime |
| `sudo nginx -t` | Test config for syntax errors |
| `sudo tail -f /var/log/nginx/error.log` | Watch error logs in real time |
| `sudo tail -f /var/log/nginx/access.log` | Watch access logs in real time |

---

## 10. Troubleshooting

**Nginx fails to start** — check for config syntax errors:
```bash
sudo nginx -t
sudo journalctl -xe | grep nginx
```

**502 Bad Gateway** — your upstream (Express) isn't running or is on the wrong port. Verify:
```bash
curl http://127.0.0.1:3000
```

**403 Forbidden on static files** — check file ownership and permissions:
```bash
sudo chown -R www-data:www-data /var/www/mysite
sudo chmod -R 755 /var/www/mysite
```

**Port 80 already in use** — check what's occupying the port:
```bash
sudo lsof -i :80
```

---

*Guide covers Ubuntu 20.04 LTS and later. Nginx version may vary slightly across releases.*
