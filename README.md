# ttyd-authelia-traefik
A secure, browser-based terminal using ttyd, Traefik reverse proxy, and Authelia authentication.

<p align="center">
  <a href="https://hub.docker.com/r/twistedsisters/ttyd-traefik">
    <img src="https://img.shields.io/badge/Docker%20Hub-twistedsisters%2Fttyd--traefik-2496ED?style=flat&logo=docker&logoColor=white" alt="Docker Hub">
  </a>
</p>

<p align="center">
  <a href="https://www.hostinger.com/vps/docker-hosting?compose_url=https://raw.githubusercontent.com/ivahyael/ttyd-authelia-traefik/main/docker-compose.yml">
    <img src="https://assets.hostinger.com/vps/deploy.svg" alt="Deploy on Hostinger"/>
  </a>
</p>


## Features

- 🖥️ **Browser-based terminal** - Access a containerized terminal directly from your web browser.
- 🔐 **Secure authentication** - Protected by Authelia with session management
- 🔒 **SSL/TLS encryption** - Automatic HTTPS with Let's Encrypt certificates
- 🐳 **Fully containerized** - Easy deployment with Docker Compose
- 🔄 **Auto-restart** - Services automatically restart on failure
- ⏱️ **Session timeout** - Configurable session expiration for security


## Architecture

```
User → Traefik (Reverse Proxy) → Authelia (Auth) → ttyd (Terminal)
         ↓
    Let's Encrypt (SSL)
```

## Prerequisites

- Docker and Docker Compose installed
- A domain name pointing to your server
- Ports 80 and 443 available
- Ubuntu/Debian server (or similar Linux distribution)
- Basic understanding of Docker and terminal usage

## Quick Start

### 1. Clone the repository
```bash
git clone https://github.com/ivahyael/ttyd-terminal-docker-with-auth.git
cd ttyd-terminal-docker-with-auth
```

### 2. Configure environment variables
Create a `.env` file:
```bash
ACME_EMAIL=your-email@example.com
HOSTNAME=your-domain.com
```

### 3. Set up Authelia

#### Generate secure secrets:
```bash
# Generate JWT secret
openssl rand -hex 32

# Generate session secret
openssl rand -hex 32

# Generate storage encryption key
openssl rand -hex 32
```

#### Update `authelia/configuration.yml`:
Replace the placeholder values with your generated secrets:
```yaml
identity_validation:
  reset_password:
    jwt_secret: [YOUR_JWT_SECRET_HERE]

session:
  secret: [YOUR_SESSION_SECRET_HERE]

storage:
  encryption_key: [YOUR_STORAGE_ENCRYPTION_KEY_HERE]
```

### 4. Create user credentials

#### Generate password hash:
```bash
docker run --rm authelia/authelia:latest \
  authelia crypto hash generate argon2 \
  --password 'your-secure-password'
```

#### Update `authelia/users_database.yml`:
```yaml
users:
  admin:
    displayname: "Admin"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..."  # Your generated hash
    email: admin@example.com
    groups:
      - admins
```

### 5. Set permissions
```bash
chmod 600 authelia/configuration.yml
chmod 600 authelia/users_database.yml
chmod 600 letsencrypt/acme.json 2>/dev/null || touch letsencrypt/acme.json && chmod 600 letsencrypt/acme.json
```

### 6. Deploy
```bash
docker-compose up -d
```

## Configuration Files

### Directory Structure
```
terminal/
├── docker-compose.yml      # Main orchestration file
├── .env                    # Environment variables (create this)
├── .gitignore              # Git ignore file
├── authelia/
│   ├── configuration.yml   # Authelia config
│   ├── users_database.yml  # User credentials
│   ├── db.sqlite3         # Session database (auto-created)
│   └── notification.txt   # Auth notifications (auto-created)
├── letsencrypt/
│   └── acme.json          # SSL certificates (auto-created)
└── .ssh/                  # SSH keys (optional mount)
```

### Important Files

#### docker-compose.yml
Defines three services:
- **Traefik**: Reverse proxy and SSL termination
- **Authelia**: Authentication and authorization
- **ttyd**: Web-based terminal

#### authelia/configuration.yml
Key settings:
- Session timeout: 1 hour
- Inactivity timeout: 10 minutes
- Authentication backend: File-based
- Access control: One-factor authentication

#### authelia/users_database.yml
User account definitions with Argon2id password hashing

## Usage

### Access the terminal
1. Navigate to `https://your-domain.com`
2. Login with your Authelia credentials
3. You'll be redirected to `/terminal` with full shell access

### Session management
- Sessions expire after 1 hour or 10 minutes of inactivity
- Logout available at the Authelia portal
- Credentials are securely stored with Argon2id hashing

### Managing users

#### Add a new user:
1. Generate password hash:
```bash
docker run --rm authelia/authelia:latest \
  authelia crypto hash generate argon2 \
  --password 'new-user-password'
```

2. Add to `authelia/users_database.yml`:
```yaml
users:
  john:
    displayname: "User1"
    password: "$argon2id$..."  # Generated hash
    email: user1@example.com
    groups:
      - users
```

3. Restart Authelia:
```bash
docker-compose restart authelia
```

## Security Considerations

### ⚠️ Critical Security Points

1. **Never commit secrets**
   - Add `.env` to `.gitignore`
   - Keep `users_database.yml` secure
   - Don't share encryption keys

2. **Use strong passwords**
   - Minimum 12 characters
   - Mix of letters, numbers, symbols
   - Unique for each user

3. **Regular updates**
   ```bash
   docker-compose pull
   docker-compose up -d
   ```

4. **Monitor access**
   ```bash
   docker logs terminal_authelia_1 --tail 100
   ```

5. **Firewall configuration**
   - Only ports 80 and 443 should be public
   - Restrict SSH access to specific IPs


## Troubleshooting

### Check service status
```bash
# View all services
docker-compose ps

# Check specific service logs
docker-compose logs -f ttyd-terminal-docker-with-auth_traefik_1
docker-compose logs -f ttyd-terminal-docker-with-auth_authelia_1
docker-compose logs -f ttyd-terminal-docker-with-auth_ttyd_1
```

### Common Issues and Solutions

#### 502 Bad Gateway
**Cause**: ttyd not running or misconfigured
```bash
# Check ttyd status
docker logs ttyd-terminal-docker-with-auth_ttyd_1

# Restart ttyd
docker-compose restart ttyd
```

#### SSL Certificate Issues
**Cause**: Let's Encrypt validation failing
```bash
# Check Traefik logs
docker logs terminal_traefik_1 | grep -i acme

# Verify DNS
nslookup your-domain.com

# Check firewall
sudo ufw status
```

#### Login Failed
**Cause**: Incorrect password or user configuration
```bash
# Verify user exists
cat authelia/users_database.yml

# Check Authelia logs
docker logs terminal_authelia_1 | grep -i auth

# Regenerate password hash
docker run --rm authelia/authelia:latest \
  authelia crypto hash generate argon2 \
  --password 'new-password'
```

#### White Page on Authelia
**Cause**: Path routing issues
- Ensure no conflicting routes in docker-compose.yml
- Check browser console for 404 errors
- Verify Authelia is running: `docker ps | grep authelia`

### Reset Everything
```bash
# Complete reset (WARNING: Deletes all data)
docker-compose down -v
rm -rf authelia/db.sqlite3
rm -rf authelia/notification.txt
rm -rf letsencrypt/acme.json
docker-compose up -d
```

## Advanced Configuration

### Custom terminal startup
Modify ttyd command:
```yaml
command: ttyd -W -p 7681 /bin/zsh  # Use zsh instead of bash
```

### Change session timeout
Edit `authelia/configuration.yml`:
```yaml
session:
  expiration: 4h      # 4 hours
  inactivity: 30m     # 30 minutes
```

### Add 2FA (TOTP)
Update `authelia/configuration.yml`:
```yaml
access_control:
  rules:
    - domain: your-domain.com
      policy: two_factor  # Change from one_factor
```

## System Requirements

- **Minimum**: 1 CPU, 1GB RAM, 10GB storage
- **Recommended**: 2 CPU, 2GB RAM, 20GB storage
- **Network**: Stable internet, ports 80/443 open
- **OS**: Linux (Ubuntu 20.04+ recommended)

## Maintenance

### Update procedure
```bash
# Pull latest images
docker-compose pull

# Deploy updates
docker-compose up -d

# Verify services
docker-compose ps
```

## License

MIT License - See LICENSE file for details

## Support

For issues and questions:
1. Check the troubleshooting section
2. Review closed issues
3. Open a new issue

## Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push to the branch
5. Open a Pull Request

## Acknowledgments

- [ttyd](https://github.com/tsl0922/ttyd) - Terminal over websocket
- [Traefik](https://traefik.io/) - Modern reverse proxy
- [Authelia](https://www.authelia.com/) - Authentication and authorization server
- [Let's Encrypt](https://letsencrypt.org/) - Free SSL certificates

---

**Security Notice**: This setup provides web access to a terminal with limited system privileges. Ensure you understand the security implications and follow best practices for authentication and network security.
