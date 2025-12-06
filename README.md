# Ubuntu Express Server Setup (Proxmox + AT&T BGW320)

This guide documents how to:

- Set up a basic **Express.js** server on **Ubuntu** (running inside a Proxmox VM)
- Make it accessible from the internet using an **AT&T BGW320-505 router**
- Keep it running with **PM2** so it survives reboots

It is written around a real working example where the VM IP was `192.168.1.129` and the Express app ran on port `3000`.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [1. Create / use an Ubuntu VM on Proxmox](#1-create--use-an-ubuntu-vm-on-proxmox)
- [2. Install Node.js and npm](#2-install-nodejs-and-npm)
- [3. Cloning Or Creating Your App](#3-cloning-or-creating-your-app)
  - [3.1 Cloning Your App From GitHub](#31-cloning-your-app-from-github)
    - [3.1.1 Install Git (if not already installed)](#311-install-git-if-not-already-installed)
    - [3.1.2 Clone your repository](#312-clone-your-repository)
    - [3.1.3 Install project dependencies](#313-install-project-dependencies)
    - [3.1.4 Start the server (test mode)](#314-start-the-server-test-mode)
    - [3.1.5 Optional: Use PM2 to run it permanently](#315-optional-use-pm2-to-run-it-permanently)
    - [3.1.6 Pull updates from GitHub (when you make changes)](#316-pull-updates-from-github-when-you-make-changes)
  - [3.2 Create a basic Express server](#32-create-a-basic-express-server)
    - [3.2.1 Bind to 0.0.0.0 (important)](#321-bind-to-0000-important)
    - [3.2.2 Test locally](#322-test-locally)
- [5. Adjust npm scripts](#5-adjust-npm-scripts)
- [6. Verify LAN access to the VM](#6-verify-lan-access-to-the-vm)
- [7. Configure AT&T BGW320-505 port forwarding (NAT/Gaming)](#7-configure-att-bgw320-505-port-forwarding-natgaming)
  - [7.1 Create a custom service](#71-create-a-custom-service)
  - [7.2 Assign the service to the VM](#72-assign-the-service-to-the-vm)
  - [7.3 Find your public IP](#73-find-your-public-ip)
  - [7.4 Test from outside your network](#74-test-from-outside-your-network)
- [8. Keep the Express server running with PM2](#8-keep-the-express-server-running-with-pm2)
  - [8.1 Install PM2](#81-install-pm2)
  - [8.2 Start the app with PM2](#82-start-the-app-with-pm2)
  - [8.3 Enable PM2 startup on boot](#83-enable-pm2-startup-on-boot)
  - [8.4 Common PM2 commands](#84-common-pm2-commands)
    - [8.4.1 Fork mode](#841-fork-mode)
    - [8.4.2 Setup startup script](#842-setup-startup-script)
    - [8.4.3 Restart application on changes](#843-restart-application-on-changes)
    - [8.4.4 Check status, logs, metrics](#844-check-status-logs-metrics)
    - [8.4.5 Managing Processes](#845-managing-processes)
    - [8.4.6 Misc](#846-misc)
    - [8.4.7 Options that can be passed with the above commands](#847-options-that-can-be-passed-with-the-above-commands)
- [9. Security notes](#9-security-notes)
- [10. Troubleshooting checklist](#10-troubleshooting-checklist)

---

# Prerequisites

- Proxmox server up and running
- An **Ubuntu** VM created in Proxmox
- AT&T **BGW320-505** router as your main gateway
- Basic familiarity with SSH and the Linux command line
- Node.js + npm installed (we’ll cover this next)

> Note: The examples use:
>
> - VM LAN IP: `192.168.1.129`
> - Express port: `3000`

Adjust these values for your setup.

---

# 1. Create / use an Ubuntu VM on Proxmox

In Proxmox:

1. Create a new VM and install Ubuntu (Server or Desktop).
2. Use a **bridged network** (typically `vmbr0`) so the VM gets a normal LAN IP like `192.168.1.x` from your router.
3. Once installed, SSH into the VM or use the Proxmox console.

On the VM, find its IP:

```bash
ip a
```

Look for an address like:

```text
inet 192.168.1.129/24
```

We’ll call this `VM_IP` from here on.

---

# 2. Install Node.js and npm

On the Ubuntu VM, install Node.js and npm. One common way (for recent Ubuntu versions) is:

```bash
sudo apt update
sudo apt install -y nodejs npm
```

Check versions:

```bash
node -v
npm -v
```

If you prefer a specific Node version, you can use `nvm`, but the above is fine for a simple Express server.

---

# 3. Cloning Or Creating Your App

## 3.1 Cloning Your App From GitHub

If your Express server is already in a GitHub repository, you can pull it onto your Ubuntu VM with a single command.

### 3.1.1 Install Git (if not already installed)

```bash
sudo apt update
sudo apt install -y git
```

Verify:

```bash
git --version
```

---

### 3.1.2 Clone your repository

Navigate to the directory where you want the project to live:

```bash
cd ~
```

Then clone your repo:

```bash
git clone https://github.com/your-username/your-repo.git
```

Replace:

- `your-username` with your GitHub username
- `your-repo` with the repository name

After cloning, enter the project folder:

```bash
cd your-repo
```

---

### 3.1.3 Install project dependencies

If your app uses npm packages (like Express), install them:

```bash
npm install
```

This reads your **package.json** and installs everything required.

---

### 3.1.4 Start the server (test mode)

If your server file is named `app.js`, run:

```bash
node app.js
```

Or if it's something like `server.js`, use:

```bash
node server.js
```

You should now be able to test it from your LAN:

```bash
curl http://<VM_IP>:3000
```

---

### 3.1.5 Optional: Use PM2 to run it permanently

```bash
pm2 start app.js --name express-server
pm2 save
pm2 startup systemd
```

This keeps your app running after reboots.

---

### 3.1.6 Pull updates from GitHub (when you make changes)

When you update your code and want to deploy the new version:

```bash
cd your-repo
git pull
pm2 restart express-server
```

This makes updating your live server very easy.

---

### 3.1.7 Summary

Using `git clone` gives you a clean, repeatable way to deploy or redeploy your app.  
Once cloned:

- `npm install` loads dependencies
- `node app.js` runs the server for testing
- `pm2 start` keeps it running long‑term
- `git pull` updates the app at any time

You can now add this as a complete installation shortcut under Section #3.

---

## 3.2. Create a basic Express server

Inside your home directory or a project folder on the VM:

```bash
mkdir express-server
cd express-server
npm init -y
npm install express
```

Create a file `app.js`:

```bash
nano app.js
```

Paste this code:

```js
import express from "express"

const app = express()
const PORT = process.env.PORT || 3000

// Basic route
app.get("/", (req, res) => {
  res.send("Hello from Express!")
})

// IMPORTANT: bind to 0.0.0.0 so it listens on all interfaces
app.listen(PORT, "0.0.0.0", () => {
  console.log(`Server running on http://0.0.0.0:${PORT}`)
})
```

If you are not using ES modules, you can use `require` instead:

```js
const express = require("express")
const app = express()
const PORT = process.env.PORT || 3000

app.get("/", (req, res) => {
  res.send("Hello from Express!")
})

app.listen(PORT, "0.0.0.0", () => {
  console.log(`Server running on http://0.0.0.0:${PORT}`)
})
```

### 3.2.1 Bind to 0.0.0.0 (important)

Binding to `0.0.0.0` is crucial. If you only listen on `localhost` (`127.0.0.1`), the server will not be reachable from other machines on your LAN or from the internet.

### 3.2.2 Test locally

Start the server:

```bash
node app.js
```

From the **VM itself**, you can test:

```bash
curl http://127.0.0.1:3000
```

You should see:

```text
Hello from Express!
```

---

# 4. Handling the .env File on Ubuntu

Environment files beginning with a dot are hidden in Linux. This includes `.env`.  
Below are the correct ways to create, upload, and secure it.

## 4.1 Create `.env` directly on the server (recommended)

```bash
cd /var/www/your-app
nano .env
```

Add your variables:

```
PORT=3000
NODE_ENV=production
API_KEY=your_api_key_here
```

Save and exit Nano:

- Ctrl + O
- Enter
- Ctrl + X

---

## 4.2 Upload `.env` through SFTP tools (WinSCP, FileZilla)

These tools hide dotfiles by default.

Enable hidden files:

- **WinSCP:** Ctrl + Alt + H
- **FileZilla:** Server → Force showing hidden files

Then upload normally.

---

## 4.3 Upload using a temporary filename, then rename

If your tool blocks `.env` uploads:

```bash
mv env-temp .env
```

---

## 4.4 Upload from Windows via SCP

```powershell
scp .env username@SERVER_IP:/var/www/your-app/.env
```

---

## 4.5 Verify and secure `.env`

```bash
ls -la /var/www/your-app
chmod 600 .env
```

This ensures only your user can read it.

---

# 5. Adjust npm scripts

Open `package.json` and adjust it to:

```json
{
  "name": "express-server-demo",
  "version": "1.0.0",
  "description": "Simple Express server example",
  "main": "app.js",
  "type": "module",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.21.0"
  }
}
```

Key things to notice:

- `"type": "module"` lets you use `import`/`export` in Node.
- `"start": "node app.js"` tells npm how to start your server.

You can keep additional fields (`license`, `author`, etc.) that npm added automatically.

---

# 6. Verify LAN access to the VM

From another machine on your local network (for example, your Windows PC), run:

```powershell
curl http://192.168.1.129:3000
```

Replace `192.168.1.129` with your actual `VM_IP`.

If everything is working, you should see:

```text
Hello from Express!
```

At this point:

- Express is running
- The VM is reachable on your LAN
- The last step is to expose it to the outside world using the router

---

# 7. Configure AT&T BGW320-505 port forwarding (NAT/Gaming)

The AT&T BGW320 uses a section called **NAT/Gaming** for port forwarding.

---

### 7.1 Create a custom service

1. Open a browser on a device connected to your home network.
2. Go to `http://192.168.1.254`.
3. Log in using the Access Code from the label on the BGW320.
4. Navigate to: **Firewall → NAT/Gaming**.
5. Under **Custom Services**, create a new service, for example:

   - **Service Name:** `ExpressServer`
   - **Global Port Range:** `3000` to `3000`
   - **Protocol:** `TCP`
   - **Base Host Port:** `3000`

Save the custom service.

---

### 7.2 Assign the service to the VM

Still under **Firewall → NAT/Gaming**:

1. At the top of the page, select `ExpressServer` in the **Service** dropdown.
2. In **Needed by Device**, select your Ubuntu VM (by hostname) or the IP `192.168.1.129`.
3. Click **Add** / **Save**.

Now the BGW320 knows to forward internet traffic on port `3000` to your VM.

---

### 7.3 Find your public IP

From a device on your home network (no VPN):

- Open `https://ifconfig.me` in a browser, or
- Search “what is my IP” on Google, or
- Check the **Broadband / Status** page on the BGW320 for the **IPv4 Address**.

You’ll get something like:

```text
99.123.45.67
```

We’ll call this `PUBLIC_IP`.

---

### 7.4 Test from outside your network

AT&T gateways are often weird about loopback, so you should test from **outside** your LAN.

On your phone:

1. Turn **Wi‑Fi off** so it uses mobile data.
2. In a browser, go to:

   ```text
   http://PUBLIC_IP:3000
   ```

Example:

```text
http://99.123.45.67:3000
```

If everything is set up correctly, you should see:

```text
Hello from Express!
```

You can also test using a port-checking site like `canyouseeme.org` on port `3000`.

---

# 8. Keep the Express server running with PM2

Right now, if you close the terminal or reboot the VM, the server will stop. To fix that, use **PM2**, a process manager for Node.js.

### 8.1 Install PM2

On the Ubuntu VM:

```bash
sudo npm install -g pm2
```

---

### 8.2 Start the app with PM2

From inside your project directory (`express-server`):

```bash
pm2 start app.js --name express-server
```

This will:

- Run `app.js` in the background
- Give the process a friendly name: `express-server`

You can check running apps:

```bash
pm2 [list|ls|status]            # Show all managed processes
```

---

### 8.3 Enable PM2 startup on boot

Tell PM2 to generate a startup script and save the current process list.

```bash
pm2 startup systemd
```

This command will output something like:

```text
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u yourusername --hp /home/yourusername
```

Copy and run that exact command.

Then save your current processes so PM2 restarts them after reboot:

```bash
pm2 save
```

From now on:

- When the VM reboots, PM2 will start
- PM2 will restart your `express-server` app automatically

---

### 8.4 Common PM2 commands (Corrected & Simplified)

#### 8.4.1 Start your application

```bash
pm2 start app.js --name express-server
```

#### 8.4.2 Enable PM2 to start on reboot

```bash
pm2 startup systemd
pm2 save
```

#### 8.4.3 Restart your app after updates

```bash
pm2 restart express-server
```

#### 8.4.4 View logs

```bash
pm2 logs express-server
```

#### 8.4.5 List processes

```bash
pm2 list
```

#### 8.4.6 Stop or delete

```bash
pm2 stop express-server
pm2 delete express-server
```

#### 8.4.7 Optional: watch mode for development

```bash
pm2 start app.js --name express-server --watch
```

> Do not use --watch in production.

---

#### 8.4.1 Fork mode

```bash
pm2 start app.js --name my-api          # Name process
pm2 start app.js --no-daemon            # Launch PM2 in no daemon
pm2 start app.js --no-autorestart       # run 1-time scripts
```

#### 8.4.2 Setup startup script

```bash
pm2 startup                             # Auto-boot on restart
pm2 save                                # Save the current process list
```

#### 8.4.3 Restart application on changes

```bash
cd /path/to/my/app
$ pm2 start env.js --watch --ignore-watch="node_modules"
```

#### 8.4.4 Check status, logs, metrics

```bash
pm2 monit                       # Terminal Based Dashboard
pm2 [list|ls|status]            # Show all managed processes
pm2 logs                        # display logs in realtime for all apps
pm2 logs app-name               # logs for just this app
pm2 logs --lines 200            # dig in older logs
pm2 logs [--raw]                # Display all processes logs in streaming
pm2 flush                       # Empty all log files
pm2 reloadLogs                  # Reload all logs
pm2 describe 0                  # Display all information about a specific process
```

#### 8.4.5 Managing Processes

```bash
pm2 restart [app-name|all|id]
pm2 reload [app-name|all|id]
pm2 stop [app-name|all|id]
pm2 delete [app-name|all|id]
```

#### 8.4.6 Misc

```bash
pm2 reset <process>             # Reset meta data (restarted time...)
pm2 updatePM2                   # Update in memory pm2
pm2 ping                        # Ensure pm2 daemon has been launched
pm2 sendSignal SIGUSR2 my-app   # Send system signal to script
```

#### 8.4.7 Options that can be passed with the above commands:

```bash
# Specify an app name
--name <app_name>

# Watch and Restart app when files change
--watch

# Specify log file
--log <log_path>

# Prefix logs with time
--time

# Do not auto restart app
--no-autorestart

# Specify cron for forced restart
--cron <cron_pattern>

# Attach to application log
--no-daemon
```

> Note: PM2 itself is **free**. The “PM2 Plus / Enterprise” cloud dashboard is optional and paid, but you don’t need an account or a credit card to run PM2 locally on your server.

---

# 9. Security notes

Once your Express server is accessible from the internet:

- Avoid exposing sensitive routes that modify data without authentication.
- Do not serve `.env` files or any secrets.
- Consider:
  - Putting a reverse proxy like Nginx or Caddy in front of Node
  - Using HTTPS with Let’s Encrypt
  - Restricting open ports to only what you need (e.g. port `3000` or port `80/443` if you later use a proxy)
- Keep Ubuntu, Node.js, and npm packages updated.

For simple personal projects or learning, this basic setup is fine as long as you are careful about what routes you expose.

---

# 10. Troubleshooting checklist

If you can’t reach the server from outside, walk this list in order:

1. **Express working locally on VM?**  
   On the VM:

   ```bash
   curl http://127.0.0.1:3000
   ```

2. **Express reachable from LAN?**  
   From another machine on your home network:

   ```bash
   curl http://VM_IP:3000
   ```

3. **Is the VM getting a LAN IP?**  
   On the VM, `ip a` should show something like `192.168.1.x`. If not, check Proxmox networking (use a bridged interface like `vmbr0`).

4. **NAT/Gaming set up correctly on BGW320?**

   - Custom Service: `ExpressServer`, port `3000` TCP, host port `3000`
   - Service assigned to the correct device: `VM_IP`

5. **Testing with the correct public IP and from outside the LAN?**

   - Use `PUBLIC_IP` from `ifconfig.me` or router WAN info
   - Test from mobile data: `http://PUBLIC_IP:3000`

6. **PM2 running and app started?**
   - `pm2 list` should show `express-server` as `online`.

Once all of the above check out, your Express app on Ubuntu (inside Proxmox) should be reliably accessible from the internet through your AT&T BGW320-505 router.

---

Happy hacking! 🚀
