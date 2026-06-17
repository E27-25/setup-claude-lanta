# Claude Code on LANTA: API Connection Fix

LANTA blocks `api.anthropic.com` at the DNS level (it resolves to `127.0.0.1` instead of the real server). This prevents Claude Code from connecting even after installation.

## How to Check if You're Affected

Run this on LANTA:

```bash
curl -v https://api.anthropic.com 2>&1 | grep "IPv4"
```

If you see `IPv4: 127.0.0.1` — you're affected. Follow the fix below.

---

## The Fix: LD_PRELOAD DNS Override

We compile a tiny C library that intercepts the system's hostname lookup and returns the real IP for Anthropic's servers. No root required.

### Step 1 — Compile the DNS fix (one time only)

Run this on LANTA:

```bash
cat > /tmp/dns_fix.c << 'EOF'
#define _GNU_SOURCE
#include <netdb.h>
#include <string.h>
#include <stdlib.h>
#include <arpa/inet.h>
#include <dlfcn.h>

static const char *overrides[][2] = {
    {"api.anthropic.com", "160.79.104.10"},
    {"claude.ai",         "160.79.104.10"},
    {NULL, NULL}
};

int getaddrinfo(const char *name, const char *service,
                const struct addrinfo *hints, struct addrinfo **res) {
    if (name) {
        for (int i = 0; overrides[i][0]; i++) {
            if (strcmp(name, overrides[i][0]) == 0) {
                struct addrinfo *ai = calloc(1, sizeof(*ai));
                struct sockaddr_in *sa = calloc(1, sizeof(*sa));
                sa->sin_family = AF_INET;
                sa->sin_port = service ? htons(atoi(service)) : 0;
                inet_pton(AF_INET, overrides[i][1], &sa->sin_addr);
                ai->ai_family    = AF_INET;
                ai->ai_socktype  = hints ? hints->ai_socktype : SOCK_STREAM;
                ai->ai_protocol  = hints ? hints->ai_protocol : 0;
                ai->ai_addrlen   = sizeof(*sa);
                ai->ai_addr      = (struct sockaddr *)sa;
                *res = ai;
                return 0;
            }
        }
    }
    static int (*orig)(const char*, const char*, const struct addrinfo*,
                       struct addrinfo**) = NULL;
    if (!orig) orig = dlsym(RTLD_NEXT, "getaddrinfo");
    return orig(name, service, hints, res);
}
EOF

gcc -shared -fPIC -o ~/.dns_fix.so /tmp/dns_fix.c -ldl
echo "Compiled OK"
```

### Step 2 — Add to ~/.bashrc (one time only)

```bash
echo 'export LD_PRELOAD=~/.dns_fix.so' >> ~/.bashrc
source ~/.bashrc
```

### Step 3 — Test

```bash
curl -s https://api.anthropic.com -o /dev/null -w 'HTTP:%{http_code}'
# Expected: HTTP:404  (means you reached the server — 404 is normal for the root path)
```

### Step 4 — Run Claude

```bash
claude
```

That's it. The fix is permanent — `~/.bashrc` loads it automatically every session.

---

## If the IP Changes

Anthropic may change their server IP. If Claude stops connecting, get the current IP from a machine with internet access:

```bash
dig +short api.anthropic.com
```

Then recompile with the new IP (replace `160.79.104.10` in Step 1 and redo Steps 1–2).

---

## Alternative: SSH Tunnel from Mac (advanced)

If the LD_PRELOAD fix stops working or LANTA also blocks the IP directly, you can route Claude's traffic through your Mac.

### On your Mac — save this as `~/lanta-tunnel.sh`

```bash
#!/bin/bash
echo "Starting LANTA tunnel... (Ctrl+C to stop)"
while true; do
    ssh -i ~/.ssh/lanta \
        -R 1080 \
        -N \
        -o ServerAliveInterval=30 \
        -o ServerAliveCountMax=3 \
        -o ExitOnForwardFailure=yes \
        ub308@transfer.lanta.nstda.or.th
    echo "Tunnel disconnected. Reconnecting in 5s..."
    sleep 5
done
```

```bash
chmod +x ~/lanta-tunnel.sh
```

Run it before using Claude on LANTA:

```bash
~/lanta-tunnel.sh
```

### On LANTA — create a Python SOCKS forwarder

```bash
cat > ~/.claude-proxy.py << 'EOF'
import socket, struct, threading, select, sys

def pipe(a, b):
    while True:
        r, _, _ = select.select([a, b], [], [], 30)
        if not r:
            break
        for s in r:
            try:
                d = s.recv(4096)
                if not d:
                    return
                (b if s is a else a).sendall(d)
            except:
                return

def handle(client):
    try:
        socks = socket.create_connection(('127.0.0.1', 1080), timeout=10)
        socks.send(b'\x05\x01\x00')
        socks.recv(2)
        host = b'api.anthropic.com'
        socks.send(b'\x05\x01\x00\x03' + bytes([len(host)]) + host + struct.pack('>H', 443))
        socks.recv(10)
        t = threading.Thread(target=pipe, args=(client, socks), daemon=True)
        t.start()
        pipe(socks, client)
    except Exception as e:
        print(f'Error: {e}', file=sys.stderr)
    finally:
        client.close()

srv = socket.socket()
srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind(('127.0.0.1', 8443))
srv.listen(20)
print('Claude proxy ready on 127.0.0.1:8443', flush=True)
while True:
    c, _ = srv.accept()
    threading.Thread(target=handle, args=(c,), daemon=True).start()
EOF

nohup python3 ~/.claude-proxy.py > ~/.claude-proxy.log 2>&1 &
```

---

## Summary

| Method | Requirements | Permanent? |
|--------|-------------|-----------|
| LD_PRELOAD (recommended) | `gcc` on LANTA | Yes (via `~/.bashrc`) |
| SSH Tunnel (backup) | Mac with internet + open terminal | No (run each session) |
