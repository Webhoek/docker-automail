# Automail Multi-Instance Setup with SSL

This guide explains how to set up Automail with SSL using Nginx reverse proxy, supporting multiple independent instances.

## Architecture Overview

```
Internet -> Nginx (SSL) -> Docker Containers
                        ├── Instance 1 (port 8081)
                        ├── Instance 2 (port 8082) [future]
                        └── Instance N (port 808N) [future]
```

Each instance has its own:
- Application container
- Database container
- Data storage
- Backup system

## Prerequisites

1. Docker and Docker Compose installed
2. Nginx installed on host system
3. SSL certificates from old server
4. GitHub credentials for Automail repository

## Step 1: Copy SSL Certificates

Copy your SSL certificates from the old server to the `ssl/` directory:

```bash
# From old server
scp /etc/nginx/ssl/certificate_sebn-automail_Base64.cer.cer user@newserver:/path/to/docker-automail/ssl/
scp /etc/nginx/ssl/sebn-automail.key user@newserver:/path/to/docker-automail/ssl/
scp /etc/nginx/ssl/dhparam user@newserver:/path/to/docker-automail/ssl/

# Set permissions
cd /path/to/docker-automail/ssl
chmod 644 certificate_sebn-automail_Base64.cer.cer
chmod 600 sebn-automail.key
chmod 644 dhparam
```

## Step 2: Configure Host Nginx

1. Copy the Nginx configuration:
```bash
sudo cp nginx-proxy/automail.conf /etc/nginx/sites-available/automail
```

2. Update the SSL certificate paths in the config file:
```bash
sudo nano /etc/nginx/sites-available/automail
# Update these lines with your actual paths:
# ssl_certificate /path/to/docker-automail/ssl/certificate_sebn-automail_Base64.cer.cer;
# ssl_certificate_key /path/to/docker-automail/ssl/sebn-automail.key;
# ssl_dhparam /path/to/docker-automail/ssl/dhparam;
```

3. Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/automail /etc/nginx/sites-enabled/
sudo nginx -t  # Test configuration
sudo systemctl reload nginx
```

## Step 3: Set Up Instance 1

1. Navigate to instance-1 directory:
```bash
cd instance-1
```

2. Create `.env` file with your credentials:
```bash
cat > .env << EOF
GITHUB_USERNAME=your_github_username
GITHUB_TOKEN=your_github_token
SITE_URL=https://sebn.automail.eu.corp.samsungelectronics.net
ADMIN_EMAIL=admin@example.com
ADMIN_PASS=initial_admin_password
DB_ROOT_PASS=secure_root_password
DB_NAME=automail
DB_USER=automail
DB_PASS=secure_db_password
EOF
```

3. Build and start the containers:
```bash
docker-compose build
docker-compose up -d
```

4. Monitor startup (first boot takes 2-5 minutes):
```bash
docker-compose logs -f automail-app-1
```

## Step 4: Verify Installation

1. Check container status:
```bash
docker-compose ps
```

2. Test direct access (bypassing SSL):
```bash
curl http://127.0.0.1:8081
```

3. Test through Nginx with SSL:
```bash
curl https://sebn.automail.eu.corp.samsungelectronics.net
```

## Step 5: Post-Installation

1. **Change admin password**: Login and immediately change the admin password
2. **Configure application settings**: Set up mail server, etc.
3. **Test email functionality**: Ensure notifications work correctly

## Adding Additional Instances

To add more instances (e.g., instance-2):

1. Copy instance-1 to instance-2:
```bash
cp -r instance-1 instance-2
```

2. Update instance-2/docker-compose.yml:
- Change all `-1` suffixes to `-2`
- Change port from 8081 to 8082
- Change database port from 33071 to 33072

3. Update instance-2/.env with different credentials if needed

4. Start instance-2:
```bash
cd instance-2
docker-compose up -d
```

5. Update Nginx configuration to include new instance:
```nginx
upstream automail_backend {
    server 127.0.0.1:8081 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8082 max_fails=3 fail_timeout=30s;  # New instance
}
```

6. Reload Nginx:
```bash
sudo systemctl reload nginx
```

## Management Commands

### Instance Management
```bash
# Start instance
cd instance-1
docker-compose up -d

# Stop instance
docker-compose down

# View logs
docker-compose logs -f automail-app-1

# Restart instance
docker-compose restart

# Access container shell
docker exec -it automail-app-1 bash
```

### Database Access
```bash
# Access database
docker exec -it automail-db-1 mysql -u automail -p automail

# Manual backup
docker exec automail-db-backup-1 backup-now
```

### Monitoring
```bash
# Check all instances
for dir in instance-*/; do
  echo "=== $dir ==="
  cd "$dir" && docker-compose ps
  cd ..
done

# View Nginx access logs
sudo tail -f /var/log/nginx/automail_access.log

# View Nginx error logs
sudo tail -f /var/log/nginx/automail_error.log
```

## Load Balancing Options

The Nginx configuration supports different load balancing methods:

1. **Round-robin** (default): Requests distributed evenly
2. **IP Hash** (sticky sessions): Users stay on same instance
   - Uncomment `ip_hash;` in upstream block
3. **Least connections**: Route to instance with fewest connections
   - Add `least_conn;` in upstream block

## Troubleshooting

### SSL Certificate Issues
- Verify certificate paths in Nginx config
- Check certificate permissions (readable by nginx user)
- Test with `openssl s_client -connect sebn.automail.eu.corp.samsungelectronics.net:443`

### Container Won't Start
- Check logs: `docker-compose logs automail-app-1`
- Verify GitHub credentials in .env
- Ensure ports aren't in use: `netstat -tuln | grep 8081`

### Database Connection Issues
- Verify database container is running
- Check database credentials match in .env
- Test connection: `docker exec -it automail-app-1 ping automail-db-1`

### Performance Issues
- Monitor resource usage: `docker stats`
- Check Nginx error logs for timeout issues
- Consider adjusting PHP memory limits in docker-compose.yml

## Security Considerations

1. **Rotate passwords**: Change all default passwords immediately
2. **Firewall**: Ensure only port 443 is exposed externally
3. **Updates**: Regularly update Docker images and host system
4. **Backups**: Verify automated backups are working
5. **Monitoring**: Set up monitoring for all instances

## Support

For Automail application issues: https://automail.net/support
For Docker setup issues: Check the project repository