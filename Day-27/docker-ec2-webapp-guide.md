# Docker Web App on EC2 — Complete Guide
> Deploy a web app in Docker on EC2 and fix ERR_CONNECTION_REFUSED

---

## Table of Contents
1. [Launch EC2 with User Data](#1-launch-ec2-with-user-data)
2. [What the User Data Script Does](#2-what-the-user-data-script-does)
3. [Understanding the App Code](#3-understanding-the-app-code)
4. [Fix ERR_CONNECTION_REFUSED](#4-fix-err_connection_refused)
5. [Verify Everything Works](#5-verify-everything-works)
6. [Test Reboot Persistence](#6-test-reboot-persistence)

---

## 1. Launch EC2 with User Data

When launching your EC2 instance on AWS Console, paste this script in the **User Data** field under **Advanced Details**:

```bash
#!/bin/bash

# Step 1: Update the system
yum update -y

# Step 2: Install Docker
yum install -y docker

# Step 3: Start Docker service and enable it to start on every reboot
systemctl start docker
systemctl enable docker

# Step 4: Create the app folder at /myapp on EC2 filesystem
mkdir -p /myapp

# Step 5: Create the Node.js web application file
cat > /myapp/app.js << 'EOF'
const http = require('http');
const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/html'});
  res.end('<h1>Hello from Docker on EC2!</h1>');
});
server.listen(3000, () => console.log('Server running on port 3000'));
EOF

# Step 6: Create the Dockerfile
cat > /myapp/Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY app.js .
EXPOSE 3000
CMD ["node", "app.js"]
EOF

# Step 7: Build the Docker image
cd /myapp
docker build -t mywebapp .

# Step 8: Run the container
# -d            = run in background
# --name        = give it a name
# --restart always = auto-start after reboot
# -p 80:3000    = map EC2 port 80 to container port 3000
docker run -d \
  --name mywebapp \
  --restart always \
  -p 80:3000 \
  mywebapp
```

### Where Files Are Stored on EC2

```
/ (root)
├── bin
├── etc
├── home
├── myapp           ← Created by mkdir -p /myapp
│   ├── app.js      ← Your Node.js web app
│   └── Dockerfile  ← Docker build instructions
├── opt
└── var
```

---

## 2. What the User Data Script Does

| Step | Command | Purpose |
|------|---------|---------|
| 1 | `yum update -y` | Updates all system packages |
| 2 | `yum install -y docker` | Installs Docker |
| 3 | `systemctl start docker` | Starts Docker immediately |
| 3 | `systemctl enable docker` | Auto-starts Docker on every reboot |
| 4 | `mkdir -p /myapp` | Creates app folder at `/myapp` |
| 5 | `cat > app.js` | Writes the Node.js app file |
| 6 | `cat > Dockerfile` | Writes the Docker build file |
| 7 | `docker build` | Builds the Docker image |
| 8 | `docker run --restart always` | Runs container, survives reboots |

### Why `--restart always` is Critical

```
EC2 Reboots
     │
     ▼
systemd starts Docker        ← systemctl enable docker
     │
     ▼
Docker starts containers     ← --restart always
     │
     ▼
Your web app is running ✅
```

---

## 3. Understanding the App Code

```javascript
const http = require('http');
```
- Imports Node.js built-in HTTP module
- Gives you tools to create a web server

```javascript
const server = http.createServer((req, res) => {
```
- Creates a web server
- `req` = request from browser (what user asked for)
- `res` = response back to browser (what we send back)

```javascript
  res.writeHead(200, {'Content-Type': 'text/html'});
```
- `200` = HTTP status code meaning **OK / Success**
- `Content-Type: text/html` = tells browser we are sending HTML

```javascript
  res.end('<h1>Hello from Docker on EC2!</h1>');
```
- Sends the HTML response to the browser
- Closes the connection after sending

```javascript
server.listen(3000, () => console.log('Server running on port 3000'));
```
- Starts the server on **port 3000** inside the container
- `-p 80:3000` in docker run maps EC2 port 80 → container port 3000

### Request Flow

```
User types http://your-ec2-ip
          │
          ▼
   EC2 Port 80 receives request
          │
          ▼  (port mapping -p 80:3000)
   Docker Container Port 3000
          │
          ▼
   app.js handles request
          │
          ▼
   Sends back <h1>Hello from Docker on EC2!</h1>
          │
          ▼
   Browser displays the page ✅
```

---

## 4. Fix ERR_CONNECTION_REFUSED

This error means **Port 80 is blocked** by the AWS Security Group.

```
This site can't be reached
44.192.125.93 refused to connect.
ERR_CONNECTION_REFUSED
```

### Fix — Open Port 80 in AWS Security Group

**Step 1:** Go to AWS Console → EC2 → Instances

**Step 2:** Click your running instance

**Step 3:** Click the **Security** tab at the bottom

**Step 4:** Click on your **Security Group link**

**Step 5:** Click **Edit Inbound Rules**

**Step 6:** Click **Add Rule** and fill in:

| Field | Value |
|-------|-------|
| Type | HTTP |
| Protocol | TCP |
| Port Range | 80 |
| Source | 0.0.0.0/0 |

**Step 7:** Click **Save Rules** ✅

### Common Causes of This Error

| Cause | Fix |
|-------|-----|
| Port 80 not open in Security Group | Add HTTP inbound rule |
| Container not running | Run `docker ps` to check |
| Using https:// instead of http:// | Use `http://your-ip` |
| Docker not started | Run `systemctl start docker` |

---

## 5. Verify Everything Works

### Check Container is Running
```bash
docker ps
```
Expected output:
```
CONTAINER ID   IMAGE      PORTS                  STATUS        NAMES
abc123         mywebapp   0.0.0.0:80->3000/tcp   Up 2 minutes  mywebapp
```

### Test App Locally on EC2
```bash
curl http://localhost:80
```
Expected output:
```html
<h1>Hello from Docker on EC2!</h1>
```

### Check Port is Listening
```bash
ss -tlnp | grep :80
```
Expected output:
```
LISTEN   0.0.0.0:80
```

### Open in Browser
```
http://your-ec2-public-ip
```
> ⚠️ Always use `http://` NOT `https://`

---

## 6. Test Reboot Persistence

```bash
# Step 1: Reboot EC2
sudo reboot

# Step 2: Wait 1-2 minutes, then SSH back in
ssh -i your-key.pem ec2-user@your-ec2-public-ip

# Step 3: Check container auto-started
docker ps

# Step 4: Test app is still running
curl http://localhost:80
```

If container is running after reboot — **setup is complete** ✅

---

## Docker Restart Policy Options

| Flag | Behavior |
|------|----------|
| `--restart no` | Never auto-restart (default) |
| `--restart always` | Always restart including after reboot ✅ |
| `--restart unless-stopped` | Restart unless manually stopped |
| `--restart on-failure` | Only restart on crash or error |

---

## Quick Reference Commands

```bash
# View running containers
docker ps

# View all containers (including stopped)
docker ps -a

# View container logs
docker logs mywebapp

# Stop container
docker stop mywebapp

# Start container
docker start mywebapp

# Remove container
docker rm mywebapp

# View Docker images
docker images
```

---

*Guide covers: EC2 User Data → Docker setup → Node.js app → Port mapping → Security Group → Reboot persistence*
