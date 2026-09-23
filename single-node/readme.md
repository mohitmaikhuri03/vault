# HashiCorp Vault (OSS) Setup — Raft Storage + TLS

Target VM: `172.16.2.155`
Storage backend: **Integrated Raft** (built-in, no external storage needed)
Vault edition: **Open Source (free)**

---

## 1. Prerequisites

Run these on the VM (RHEL / CentOS / Rocky / AlmaLinux):

```bash
sudo dnf install -y openssl unzip wget curl jq yum-utils
```

- `openssl` → generate the self-signed TLS cert/key for the Vault listener
- `unzip`/`wget`/`curl` → download and unpack the Vault binary
- `jq` → parse Vault's JSON output when initializing/unsealing

---

## 2. Install Vault (latest OSS)

```bash
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo dnf install -y vault
vault version
```

This installs the free, open-source `vault` package (not Vault Enterprise).

---

## 3. Create directories

```bash
sudo mkdir -p /opt/vault/data /opt/vault/tls
sudo useradd --system --home /opt/vault --shell /bin/false vault || true
sudo chown -R vault:vault /opt/vault
```

---

## 4. Generate TLS certificate with OpenSSL

Since there's no CA here, generate a self-signed cert valid for the VM's IP:

```bash
cd /opt/vault/tls

sudo openssl req -x509 -newkey rsa:4096 -sha256 -days 825 -nodes \
  -keyout vault-key.pem \
  -out vault-cert.pem \
  -subj "/CN=172.16.2.155" \
  -addext "subjectAltName=IP:172.16.2.155,IP:127.0.0.1"

sudo chown vault:vault vault-key.pem vault-cert.pem
sudo chmod 600 vault-key.pem
sudo chmod 644 vault-cert.pem
```

Since it's self-signed, your browser/CLI will warn about an untrusted cert — that's expected. You'll either accept the warning in the browser, or use `-tls-skip-verify`/`VAULT_SKIP_VERIFY` with the CLI, or import `vault-cert.pem` into your local trust store if you want a clean browser experience.

---

## 5. Configure Vault (`vault.hcl`)


```bash
sudo mkdir -p /etc/vault.d
sudo chown -R vault:vault /etc/vault.d
```

```bash
sudo tee /etc/vault.d/vault.hcl > /dev/null << 'EOF'
ui = true

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "node1"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/opt/vault/tls/vault-cert.pem"
  tls_key_file  = "/opt/vault/tls/vault-key.pem"
}

api_addr     = "https://172.16.2.155:8200"
cluster_addr = "https://172.16.2.155:8201"
disable_mlock = true
EOF
```

> `disable_mlock = true` is only needed if you're on a VM without the ability to lock memory (common in some virtualized/containerized setups). Remove it and instead run `sudo setcap cap_ipc_lock=+ep $(readlink -f $(which vault))` if you want the safer option.



---

## 6. Run Vault as a systemd service

```bash
sudo tee /etc/systemd/system/vault.service > /dev/null << 'EOF'
[Unit]
Description=HashiCorp Vault
Requires=network-online.target
After=network-online.target

[Service]
User=vault
Group=vault
ExecStart=/usr/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now vault
sudo systemctl status vault
```

---

## 7. Point the CLI at your Vault

On the VM (or any machine that can reach `172.16.2.155:8200`):

```bash
export VAULT_ADDR="https://172.16.2.155:8200"
export VAULT_SKIP_VERIFY="true"   # only because the cert is self-signed
```

---

## 8. Initialize & Unseal Vault

This only happens **once**, on a fresh Vault. You can do it either from the CLI or directly from the UI — pick one.

### Method A — CLI

**Initialize:**

```bash
vault operator init -key-shares=5 -key-threshold=3 -format=json > /opt/vault/init.json
cat /opt/vault/init.json | jq
```

This gives you:
- 5 **unseal keys** (any 3 of them unseal Vault)
- 1 **initial root token**

**Save `/opt/vault/init.json` somewhere safe outside the VM immediately** — if you lose it, you lose access to your data. Then restrict/remove it from the VM once backed up.

**Unseal:**

Vault starts **sealed** every time it (re)starts. Unseal with any 3 of the 5 keys:

```bash
vault operator unseal <unseal_key_1>
vault operator unseal <unseal_key_2>
vault operator unseal <unseal_key_3>
```

Check status:

```bash
vault status
```

`Sealed` should now show `false`.

### Method B — UI (no CLI needed)

1. Open a browser to `https://172.16.2.155:8200/ui`
2. On first load, Vault shows an **Initialize** screen. Set:
   - **Key Shares** → `5`
   - **Key Threshold** → `3`
3. Click **Initialize**
4. The UI immediately displays:
   - 5 **Unseal Keys**
   - 1 **Initial Root Token**
   - A **Download keys** button to save them as JSON
5. **Copy/save these somewhere safe right now** — they are shown only once.
6. On the same screen (or the login screen after refresh), enter 3 of the 5 unseal keys, one at a time, into the **Unseal Key** field.
7. After the 3rd key, Vault unseals automatically.

> Use CLI for scripted/automated setups (Terraform, Ansible, CI pipelines). Use the UI for a quick manual one-time setup — no terminal required.

---

## 9. Log in

Once Vault is unsealed (via Method A or B above), logging in requires the **root token** either way — whether you use the CLI or the UI, the same token has to be entered. It's the same root token you got during init (from `init.json` on CLI, or the "Initial Root Token" shown on the UI init screen).

### Method A — Log in via CLI

```bash
vault login <root_token>
```

This authenticates your terminal session with Vault — use this if you're going to interact with Vault via scripts or automation.

### Method B — Log in directly via UI (no CLI needed)

1. Open a browser to `https://172.16.2.155:8200/ui`
2. You'll see a self-signed certificate warning — click "Advanced" → "Proceed anyway"
3. On the login page, set **Method** to **Token**
4. Paste the root token into the **Token** field
5. Click **Sign In**

That's it — no terminal required, you're logged straight in from the browser.

> If you only plan to work through the UI (never opening a terminal), Method B alone is enough — Method A is optional.

Either method gets you into the same Vault UI, backed by Raft storage, with the setup fully complete.

---

## Notes / Next Steps (optional, not required to log in)

- Root token is for bootstrapping only — create proper auth methods (userpass, LDAP, OIDC) and policies for real use.
- For a production setup you'd want a real TLS cert (e.g. from an internal CA or Let's Encrypt with a real domain) instead of a self-signed one.
- If you add more Vault nodes later, Raft supports clustering via `retry_join` stanzas — ping if you want that config.
