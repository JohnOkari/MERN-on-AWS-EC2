# MERN Stack Implementation on AWS EC2

## Overview
The **MERN stack** is a modern JavaScript stack consisting of:
- **M**ongoDB – NoSQL database
- **E**xpress – Backend web framework for Node.js
- **R**eact – Frontend library
- **N**ode.js – JavaScript runtime

We will deploy a production-grade MERN application on an Amazon EC2 instance running Ubuntu, serving the React build with Nginx and proxying API requests to a Node.js/Express server. You can use either MongoDB Atlas (managed) or install MongoDB locally on the EC2 instance.

---

## 1. Prerequisites
Before starting, ensure you have:
1. An **AWS Account**.
2. Basic knowledge of **Linux commands**.
3. A **key pair** for SSH access.
4. AWS **security group rules** allowing:
   - HTTP (Port 80)
   - HTTPS (Port 443) – if using SSL
   - SSH (Port 22)
5. A local terminal (Linux/Mac/git bash) or **PuTTY** (Windows).
6. Optional: A domain name managed in Route 53 or your DNS provider.

---

## 2. Step-by-Step Implementation

### Step 1: Launch an EC2 Instance
- Log into the AWS Management Console to set up the EC2 instance.
---
- Launch Instance and Confirm instance is running and status checks pass.
---
![ec2-success](./images/status_checks.png)
---
- Copy the public IP address of your instance.
---
![ec2-success](./images/copy_ip.png)
---
- Alternatively, retrieve the public IP with:
```bash
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"` && \
curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/public-ipv4
```

### Step 2: Connect to Your Instance
From your terminal, cd to the directory containing your key pair and run:
```bash
chmod 400 mern-stack-kp.pem
ssh -i mern-stack-kp.pem ubuntu@<EC2_PUBLIC_IP>
```
### Step 3: Update the System and Install Base Tools
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential curl git ufw
```

---

### Step 4: Install Node.js (LTS) and NPM
Use NodeSource for the latest LTS (recommended for servers):
```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```

---

### Step 5: Choose Your MongoDB Option
You can either use MongoDB Atlas (managed and recommended for production) or install MongoDB locally on EC2.

#### Option A (Recommended): MongoDB Atlas
1. Create a free/paid cluster on Atlas.
2. Create a database user and password.
3. Add your EC2 public IP (or 0.0.0.0/0 for testing only) to the IP Access List.
4. Obtain your connection string (URI), e.g.:
```
mongodb+srv://<USER>:<PASSWORD>@<CLUSTER_NAME>.mongodb.net/<DB_NAME>?retryWrites=true&w=majority
```

#### Option B: Install MongoDB on EC2 (Ubuntu 22.04)
```bash
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl enable --now mongod
sudo systemctl status mongod | cat
```
Enable authentication and keep MongoDB accessible only locally (recommended):
```bash
sudo sed -i 's/^#\?\s*authorization:.*/  authorization: enabled/' /etc/mongod.conf
sudo sed -i 's/^\s*bindIp:.*/  bindIp: 127.0.0.1/' /etc/mongod.conf
sudo systemctl restart mongod
```
Create admin and app users with `mongosh`:
```bash
mongosh
```
```javascript
use admin
db.createUser({ user: "admin", pwd: "StrongAdmin.P4ss", roles: [ { role: "root", db: "admin" } ] })
```
Reconnect with auth, then create an app database/user:
```bash
mongosh -u admin -p --authenticationDatabase admin
```
```javascript
use appdb
db.createUser({ user: "appuser", pwd: "PassWord.1", roles: [ { role: "readWrite", db: "appdb" } ] })
```
Your local MongoDB URI for the app will be:
```
mongodb://appuser:PassWord.1@localhost:27017/appdb?authSource=appdb
```

---

### Step 6: Prepare Application Directories
```bash
sudo mkdir -p /var/www/mernapp
sudo chown -R $USER:$USER /var/www/mernapp
cd /var/www/mernapp
```

---

### Step 7: Create and Configure the Express API (Backend)
```bash
cd /var/www/mernapp
mkdir server && cd server
npm init -y
npm install express mongoose dotenv cors
```

Create `server.js`:
```javascript
// server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(express.json());
app.use(cors());

const mongoUri = process.env.MONGODB_URI;
const port = process.env.PORT || 4000;

const TodoSchema = new mongoose.Schema({ content: { type: String, required: true } }, { timestamps: true });
const Todo = mongoose.model('Todo', TodoSchema);

app.get('/api/health', (_req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

app.get('/api/todos', async (_req, res) => {
  const items = await Todo.find().sort({ createdAt: -1 });
  res.json(items);
});

app.post('/api/todos', async (req, res) => {
  const created = await Todo.create({ content: req.body.content });
  res.status(201).json(created);
});

async function start() {
  try {
    await mongoose.connect(mongoUri);
    app.listen(port, () => console.log(`API listening on :${port}`));
  } catch (err) {
    console.error('Failed to start server:', err);
    process.exit(1);
  }
}

start();
```

Create `.env`:
```bash
cat > .env << 'EOF'
PORT=4000
# Use your Atlas URI or local Mongo depending on your choice
# MONGODB_URI=mongodb+srv://<USER>:<PASSWORD>@<CLUSTER>/<DB_NAME>?retryWrites=true&w=majority
MONGODB_URI=mongodb://appuser:PassWord.1@localhost:27017/appdb?authSource=appdb
EOF
```

Add helpful scripts to `package.json`:
```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "NODE_ENV=development node server.js"
  }
}
```

Test locally (temporarily):
```bash
node server.js
curl http://localhost:4000/api/health | cat
```

Stop the process (Ctrl+C) before moving on.

---

### Step 8: Create the React Frontend with Vite
```bash
cd /var/www/mernapp
npm create vite@latest client -- --template react
cd client
npm install
```

Replace `src/App.jsx` with a simple client calling the API:
```jsx
import { useEffect, useState } from 'react'

const API_BASE = import.meta.env.VITE_API_URL || ''

export default function App() {
  const [todos, setTodos] = useState([])
  const [content, setContent] = useState('')

  async function loadTodos() {
    const res = await fetch(`${API_BASE}/api/todos`)
    const data = await res.json()
    setTodos(data)
  }

  async function addTodo(e) {
    e.preventDefault()
    if (!content.trim()) return
    await fetch(`${API_BASE}/api/todos`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ content })
    })
    setContent('')
    loadTodos()
  }

  useEffect(() => { loadTodos() }, [])

  return (
    <div style={{ maxWidth: 600, margin: '2rem auto', fontFamily: 'system-ui, sans-serif' }}>
      <h1>Todo List</h1>
      <form onSubmit={addTodo}>
        <input value={content} onChange={e => setContent(e.target.value)} placeholder="New todo" />
        <button type="submit">Add</button>
      </form>
      <ol>
        {todos.map(t => <li key={t._id}>{t.content}</li>)}
      </ol>
    </div>
  )
}
```

Optionally configure an API base URL for development/production:
```bash
echo 'VITE_API_URL=' > .env
```

Build the production frontend:
```bash
npm run build
```
Vite outputs static files to `client/dist`.

---

### Step 9: Install and Configure Nginx (Frontend + Reverse Proxy)
```bash
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx | cat
```

Create a site configuration:
```bash
sudo nano /etc/nginx/sites-available/mernapp
```

Add the following configuration (adjust `server_name` and paths as needed):
```nginx
server {
    listen 80;
    server_name mern.example.com www.mern.example.com _;

    root /var/www/mernapp/client/dist;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api/ {
        proxy_pass http://localhost:4000/api/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Enable the site and disable default:
```bash
sudo ln -s /etc/nginx/sites-available/mernapp /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Copy the built frontend to the expected location if needed:
```bash
sudo mkdir -p /var/www/mernapp/client
sudo cp -r /var/www/mernapp/client/dist /var/www/mernapp/client/
sudo chown -R www-data:www-data /var/www/mernapp/client
```

Visit:
```
http://<EC2_PUBLIC_IP>
```

---

### Step 10: Run the API with PM2 (Process Manager)
Install PM2 globally:
```bash
sudo npm install -g pm2
```
Start the API and configure it to start on boot:
```bash
cd /var/www/mernapp/server
pm2 start server.js --name mern-api
pm2 status | cat
pm2 startup systemd -u $USER --hp $HOME
# Run the command PM2 outputs to finalize startup integration, then:
pm2 save
```

Tail logs if needed:
```bash
pm2 logs mern-api --lines 100
```

---

### Step 11: Firewall Configuration (UFW)
```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw --force enable
sudo ufw status | cat
```

---

### Step 12: Optional – Enable HTTPS with Let's Encrypt
If you have a domain pointed to the EC2 public IP:
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
sudo systemctl status certbot.timer | cat
```

---

### Step 13: Smoke Tests
- Backend health:
```bash
curl http://localhost:4000/api/health | cat
```
- Frontend in browser: `http://<EC2_PUBLIC_IP>` (or your domain). Add items and ensure they persist in MongoDB.

---

## 3. Troubleshooting
| Issue | Solution |
|-------|----------|
| Nginx 502/504 errors | Ensure API is running (`pm2 status`), and `proxy_pass` port matches API port. Check `sudo journalctl -u nginx -e` and `pm2 logs`. |
| React refresh leads to 404 | Ensure `try_files $uri /index.html;` exists in Nginx config. Reload Nginx. |
| MongoDB connection fails | Verify URI, credentials, IP whitelist (Atlas), and that `mongod` is running. Check `sudo journalctl -u mongod -e`. |
| Port 80/443 not reachable | Check AWS security group rules and any corporate firewall. |
| PM2 app not starting | Inspect `pm2 logs mern-api`; confirm `.env` variables and Node version. |
| SSL issuance fails | Ensure A/AAAA DNS records point to EC2 IP and port 80 is accessible publicly. |

---

## 4. Cleanup
If you no longer need the setup:
- Terminate the EC2 instance from AWS Console.
- Delete associated security groups, Elastic IPs, and key pairs.
- If using Atlas, pause/terminate the cluster to avoid charges.

---


**End of Guide**
