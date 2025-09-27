# SSL Certificates Directory

Place your SSL certificates in this directory:

1. Copy your certificate file: `certificate_sebn-automail_Base64.cer.cer`
2. Copy your private key: `sebn-automail.key`
3. Copy your dhparam file: `dhparam`

From your old server, these files are located at:
- `/etc/nginx/ssl/certificate_sebn-automail_Base64.cer.cer`
- `/etc/nginx/ssl/sebn-automail.key`
- `/etc/nginx/ssl/dhparam`

## Copy commands from old server:
```bash
# On old server
scp /etc/nginx/ssl/certificate_sebn-automail_Base64.cer.cer user@newserver:/path/to/docker-automail/ssl/
scp /etc/nginx/ssl/sebn-automail.key user@newserver:/path/to/docker-automail/ssl/
scp /etc/nginx/ssl/dhparam user@newserver:/path/to/docker-automail/ssl/
```

## Set proper permissions:
```bash
chmod 644 certificate_sebn-automail_Base64.cer.cer
chmod 600 sebn-automail.key
chmod 644 dhparam
```