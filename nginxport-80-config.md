Here are the **two separate Nginx configuration files** as per your request:

---

## 🔹 1. **Nginx Config: Proxy to Docker App on Port (e.g., 8080)**

Assuming your Docker app is running on port `8080`, and you're exposing it via Nginx on port `80`.

### 📄 `/etc/nginx/sites-available/nginx.examples.com`

```nginx
server {
    listen 80;
    server_name nginx.examples.com;

    access_log /var/log/nginx/nginx.examples.com_access.log;
    error_log /var/log/nginx/nginx.examples.com_error.log;

    location / {
        proxy_pass http://localhost:8080;  # Change to your Docker app port
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

To enable this config:

```bash
sudo ln -s /etc/nginx/sites-available/nginx.examples.com /etc/nginx/sites-enabled/
```

---

## 🔹 2. **Nginx Config: Serve Static Files from `/var/www/html`**

### 📄 `/etc/nginx/sites-available/path.examples.com`

```nginx
server {
    listen 80;
    server_name path.examples.com;

    access_log /var/log/nginx/path.examples.com_access.log;
    error_log /var/log/nginx/path.examples.com_error.log;

    root /var/www/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

To enable this config:

```bash
sudo ln -s /etc/nginx/sites-available/path.examples.com /etc/nginx/sites-enabled/
```

---

### ✅ Final Steps

1. **Test Nginx config:**

```bash
sudo nginx -t
```

2. **Reload Nginx:**

```bash
sudo systemctl reload nginx
```
