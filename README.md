##### TASK 1 - Linux & Server Setup

1]
ssh -i "private.key" root@13.229.223.118

lsb_release -a

2]
sudo adduser deploy
sudo usermod -aG sudo deploy

# Local machine
ssh-keygen -t ed25519 -C "deploy@devops-test" -f ~/.ssh/devops_test_key

cat ~/.ssh/devops_test_key.pub 

ssh deploy@13.229.223.118

sudo mkdir -p /home/deploy/.ssh 
sudo tee -a /home/deploy/.ssh/authorized_keys 
sudo chown -R deploy:deploy /home/deploy/.ssh 
sudo chmod 700 /home/deploy/.ssh 
sudo chmod 600 /home/deploy/.ssh/authorized_keys

vi /home/deploy/.ssh/authorized_keys

paste the public key

:wq

exit

log agin with new user

ssh -i ~/.ssh/devops_test_key deploy@13.229.223.118

sudo apt update && sudo apt upgrade -y

# fixed the nginx.conf issue #

3]
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release ufw nginx git unzip fail2ban

# Docker (official repo, not apt's docker.io)
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker

sudo sed -n '78,90p' /etc/nginx/nginx.conf

sudo systemctl status docker
sudo systemctl start docker
sudo systemctl enable docker 
sudo docker run hello-world

4]
sudo ufw status verbose

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH       
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose

# or: sudo ufw allow 22/tcp

# from a new local terminal
ssh -i ~/.ssh/devops_test_key deploy@13.229.223.118

free -h
df -h
uptime
sudo ss -tulpn
docker --version && docker compose version

curl -I http://localhost

http://13.229.223.118/ (in browser)

========
##### TASK 2 - Containerization 

DIMESH@DESKTOP-B5UT235 MINGW64 K:/4projects/web-lankan/nextjs-mysql-example (main)
$ git remote -v
origin  https://github.com/dimeshk1/nextjs-mysql-example.git (fetch)
origin  https://github.com/dimeshk1/nextjs-mysql-example.git (push)

cd ~
git clone https://github.com/dimeshk1/nextjs-mysql-example.git
cd nextjs-mysql-example

vi .env
NODE_ENV=production
MYSQL_ROOT_PASSWORD=<>
MYSQL_DATABASE=appdb
MYSQL_USER=appuser
MYSQL_PASSWORD=<>

vi .gitignore
.gitignore additions
.env
.env.*
!.env.example
node_modules
.next

cat knexfile.ts

docker compose build
docker compose run --rm migrate
docker compose up -d
docker compose ps

#enable port
nano docker-compose.yml

    ports:
      - "127.0.0.1:3000:3000"
	  

docker compose up -d app
docker compose ps


DIMESH@DESKTOP-B5UT235 MINGW64 K:/4projects/web-lankan/nextjs-mysql-example (main)
$ curl -I --max-time 5 http://13.229.223.118:3000
curl: (28) Connection timed out after 5005 milliseconds

DIMESH@DESKTOP-B5UT235 MINGW64 K:/4projects/web-lankan/nextjs-mysql-example (main)
$ curl -I --max-time 5 http://13.229.223.118
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Tue, 25 Aug 2026 09:37:31 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 25 Aug 2026 06:20:43 GMT
Connection: keep-alive
ETag: "6a8d343b-267"
Accept-Ranges: bytes



Task 3 - Nginx Reverse Proxy & Domain Setup


##### Task 3

1]
https://www.duckdns.org/

login via google account

success: domain appaug25.duckdns.org added to your account

copy token
copy server public ip

curl "https://www.duckdns.org/update?domains=appaug25&token=7bfc0c75-ce93-4169-a27e-f1204f6c1ebc&ip=13.229.223.118"

nslookup appaug25.duckdns.org

2]
vi /etc/nginx/sites-available/

server {
    listen 80;
    server_name appaug25.duckdns.org;

    # Enable gzip compression for common text-based responses.
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    gzip_min_length 256;

    # Security headers.
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Reverse proxy requests to the Next.js application.
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_cache_bypass $http_upgrade;
    }

    # Cache Next.js static assets for one year.
    location /_next/static/ {
        proxy_pass http://127.0.0.1:3000;
        proxy_cache_valid 200 60m;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }
}

sudo ln -s /etc/nginx/sites-available/nextjs-app /etc/nginx/sites-enabled/nextjs-app
sudo rm -f /etc/nginx/sites-enabled/default

sudo nginx -t
sudo systemctl reload nginx
sudo systemctl status nginx --no-pager

curl -I http://127.0.0.1:3000

curl http://appaug25.duckdns.org

sudo nginx -t
sudo systemctl reload nginx
sudo nginx -T | grep -n -A40 -B5 "appaug25.duckdns.org"

testing:
created a sample user
http://appaug25.duckdns.org/

http://appaug25.duckdns.org/api/users/1


#### Task 6

# secure login
sudo su -
vi /etc/ssh/sshd_config

cp sshd_config sshd_config.bk.25aug2026

vi sshd_config

```bash
# SSH hardening
sudo nano /etc/ssh/sshd_config
```
Set:
```
PermitRootLogin no
PasswordAuthentication no

sudo systemctl restart ssh

with new session with deploy user 
sudo whoami


# Certbot setup
sudo apt install -y certbot python3-certbot-nginx

sudo certbot --nginx \
  -d appaug25.duckdns.org \
  --non-interactive \
  --agree-tos \
  -m dimeshdk@email.com \
  --redirect

sudo certbot renew --dry-run       # proves auto-renewal works

sudo systemctl status certbot.timer

sudo certbot certificates

sudo nginx -t
sudo systemctl reload nginx

sudo certbot --nginx \
  -d appaug25.duckdns.org \
  --non-interactive \
  --agree-tos \
  -m dimeshdk@email.com \
  --redirect
  
