# 🚀 Chattingo Deployment with Docker, Nginx & Let's Encrypt

This guide documents the journey of deploying **Chattingo** (React frontend + backend) on a VPS using **Docker**, **Nginx**, **Let's Encrypt**, and **Jenkins CI/CD**.  
Along the way, we solved some common errors — this README explains them so you won’t struggle like we did.  

---

## 📌 Project Setup
Our services:
- **Frontend (React)** → runs on Nginx inside a container
- **Backend (Express / Spring Boot, etc.)** → separate container
- **Reverse Proxy (Nginx)** → handles SSL & routing
- **Certbot** → issues free SSL certificates
- **Jenkins** → CI/CD pipeline to build & deploy

---

## 🛑 Error 1: Nginx Config Mount Issue
**Error:**
```bash
error mounting "/var/lib/jenkins/workspace/chattingo/nginx.conf" \
to rootfs at "/etc/nginx/conf.d/default.conf": not a directory
Cause:
We tried to mount a file into /etc/nginx/conf.d/ incorrectly.

Fix:
We placed nginx.conf inside the frontend directory and mounted it correctly:
frontend:
  build:
    context: ./frontend
    dockerfile: Dockerfile
  image: chattingo/frontend:latest
  container_name: frontend
  ports:
    - "3000:80"
    - "443:443"
  volumes:
    # ✅ Correct file-to-file mount
    - ./frontend/nginx.conf:/etc/nginx/conf.d/default.conf:ro
Cause:
Certbot needs a webroot folder to complete the challenge.

Fix:
We created the directories and mounted them:
mkdir -p letsencrypt letsencrypt-www

volumes:
  # Certificates
  - ./letsencrypt:/etc/letsencrypt:rw
  - ./letsencrypt-www:/var/www/certbot:rw
Error 3: HTTPS Setup with Nginx & Certbot

We needed Nginx to serve both HTTP (for Certbot validation) and HTTPS with certificates.

Solution (nginx.conf):

server {
    listen 80;
    server_name chattingo.duckdns.org;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name chattingo.duckdns.org;

    ssl_certificate /etc/letsencrypt/live/chattingo.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chattingo.duckdns.org/privkey.pem;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri /index.html;
    }
}

🛑 Error 4: Jenkins + Docker Integration

We needed Jenkins to deploy automatically.

Solution (Jenkinsfile):

pipeline {
    agent any

    stages {
        stage('Build Docker Images') {
            steps {
                sh 'docker-compose build'
            }
        }

        stage('Obtain SSL Certificate') {
            steps {
                sh '''
                docker run --rm \
                  -v $(pwd)/letsencrypt:/etc/letsencrypt \
                  -v $(pwd)/letsencrypt-www:/var/www/certbot \
                  certbot/certbot certonly --webroot \
                  -w /var/www/certbot \
                  -d chattingo.duckdns.org --email you@example.com --agree-tos --no-eff-email
                '''
            }
        }

        stage('Deploy Containers') {
            steps {
                sh 'docker-compose down && docker-compose up -d'
            }
        }
    }
}

✅ Final Steps

Update DNS:

chattingo.duckdns.org → VPS IP (72.60.111.48)


Run Jenkins pipeline → builds frontend & backend

Certbot issues certificate → Nginx serves HTTPS

Open browser:
👉 https://chattingo.duckdns.org
 🎉

📊 Architecture Flow
Client ---> Nginx (Docker)
         |             |
         |             ---> Frontend (React, port 3000)
         |
         ---> Backend (API, port 5000)

🎯 Lessons Learned

Always mount file-to-file or dir-to-dir (don’t mix them).

Certbot requires a writable directory for challenges.

Jenkins + Docker works best with volumes for certificates.

Debugging deployment is painful 😅 but documenting saves future headaches.


---

Do you want me to also include a **ready-to-use `docker-compose.yml`** example inside this README so new contributors can just copy it and run?

