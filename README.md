## **Claude Code on LANTA: Quick Setup**

### **1. The Problem**

* **SSH FS:** Fails because it looks for `/project` on your **local computer**, not the supercomputer.
* **LANTA Environment:** Most HPCs do not have Node.js or `npm` pre-installed. You need a local user-space installation.

### **2. The Solution (3 Steps)**

Run these commands in your LANTA terminal session:

1. **Install NVM (Node Version Manager):**
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc

```


2. **Install Node.js:**
```bash
nvm install --lts

```


3. **Install Claude Code CLI:**
```bash
npm install -g @anthropic-ai/claude-code

```



### **3. Usage**

```bash
cd /project/zz992000-zdevb
claude

```

### **⚠️ Quick Rules**

* **Privacy:** Your account and chat history are stored in your private home directory (`~`), so **teammates cannot access your Claude account.**
* **Etiquette:** Use Claude for coding assistance and scripting. Do not run heavy AI training or indexing on the Login Node.
* **Network:** LANTA blocks `api.anthropic.com` at the DNS level. If Claude says `Unable to connect to API`, follow the **[Network Fix Guide](NETWORK_FIX.md)**.
