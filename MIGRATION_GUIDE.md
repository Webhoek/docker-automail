# SSL Migration Guide for Existing Automail Container

This guide helps you add SSL to your existing `automail-app` container without disrupting service.

## Current Setup
- Existing container: `automail-app` running on port 80
- Database: `automail-db` on port 3307
- No SSL configured

## Target Setup
- Same containers continue running
- Nginx reverse proxy handles SSL
- Support for adding more instances later

## Step-by-Step Migration

### Step 1: Copy SSL Certificates

Copy your certificates from the old server:

```bash
# Create SSL directory
mkdir -p /path/to/docker-automail/ssl

# Copy from old server
scp user@oldserver:/etc/nginx/ssl/certificate_sebn-automail_Base64.cer.cer ./ssl/
scp user@oldserver:/etc/nginx/ssl/sebn-automail.key ./ssl/
scp user@oldserver:/etc/nginx/ssl/dhparam ./ssl/

# Set permissions
chmod 644 ssl/certificate_sebn-automail_Base64.cer.cer
chmod 600 ssl/sebn-automail.key
chmod 644 ssl/dhparam
```

### Step 2: Update Docker Compose (Without Stopping Containers)

The docker-compose.yml has been updated to:
- Change port mapping from `"80:80"` to `"127.0.0.1:8080:80"`
- Change database port from `"3307:3306"` to `"127.0.0.1:3307:3306"`

To apply without downtime:

```bash
# Option A: If containers are not running yet
docker-compose up -d

# Option B: If containers are already running
docker-compose up -d --no-recreate  # This will update port mappings on next restart
```

### Step 3: Install and Configure Nginx

1. Install Nginx on the host:
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nginx

# RHEL/CentOS
sudo yum install nginx
```

2. Copy the Nginx configuration:
```bash
sudo cp nginx-proxy/automail-existing.conf /etc/nginx/sites-available/automail
```

3. Update certificate paths in the config:
```bash
sudo nano /etc/nginx/sites-available/automail
```

Update these lines with your actual paths:
```nginx
ssl_certificate /path/to/docker-automail/ssl/certificate_sebn-automail_Base64.cer.cer;
ssl_certificate_key /path/to/docker-automail/ssl/sebn-automail.key;
ssl_dhparam /path/to/docker-automail/ssl/dhparam;
```

4. Enable the site:
```bash
# Ubuntu/Debian
sudo ln -s /etc/nginx/sites-available/automail /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### Step 4: Update Container for Proxy Support

Apply the new port mapping and restart:

```bash
# This will briefly interrupt service (few seconds)
docker-compose down
docker-compose up -d
```

The container will now be accessible on port 8080 instead of 80.

### Step 5: Update DNS/Firewall

1. Ensure DNS points to your server
2. Open port 443 in firewall
3. Close port 80 external access (Nginx will handle redirects)

```bash
# Example for UFW
sudo ufw allow 443/tcp
sudo ufw allow 80/tcp  # For redirect only
```

### Step 6: Test the Setup

1. Test direct container access (should work):
```bash
curl http://127.0.0.1:8080
```

2. Test HTTPS access:
```bash
curl https://sebn.automail.eu.corp.samsungelectronics.net
```

3. Test HTTP redirect:
```bash
curl -I http://sebn.automail.eu.corp.samsungelectronics.net
# Should return 301 redirect to HTTPS
```

## Adding Future Instances

When ready to add more instances:

1. Keep existing container running as-is
2. Add new instances in `instance-1/`, `instance-2/` directories
3. Update Nginx upstream to include new instances:

```nginx
upstream automail_backend {
    server 127.0.0.1:8080;  # existing
    server 127.0.0.1:8081;  # instance-1
    server 127.0.0.1:8082;  # instance-2
}
```

## Rollback Plan

If issues occur, you can quickly rollback:

```bash
# Stop Nginx
sudo systemctl stop nginx

# Revert docker-compose.yml port mapping
# Change from "127.0.0.1:8080:80" back to "80:80"
nano docker-compose.yml

# Restart container with old config
docker-compose down
docker-compose up -d
```

## Troubleshooting

### Container not accessible after migration
- Check if container is running: `docker ps`
- Verify port mapping: `docker port automail-app`
- Test local access: `curl http://127.0.0.1:8080`

### SSL certificate errors
- Verify certificate files exist in `ssl/` directory
- Check Nginx error log: `sudo tail -f /var/log/nginx/error.log`
- Test SSL: `openssl s_client -connect localhost:443`

### Performance issues
- Check Nginx access log: `sudo tail -f /var/log/nginx/automail_access.log`
- Monitor container: `docker logs -f automail-app`
- Check resource usage: `docker stats`

## Benefits of This Approach

1. **Zero data migration**: Your existing container and data remain unchanged
2. **Minimal downtime**: Only brief restart to apply port changes
3. **Easy rollback**: Can quickly revert if issues occur
4. **Future scalability**: Ready to add more instances when needed
5. **Security**: SSL termination at proxy level
6. **Performance**: Nginx efficiently handles SSL