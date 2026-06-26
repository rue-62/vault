# Vault

A minimalist, client-side AES-256-GCM encryptor. Built for one specific workflow: pre-encrypt sensitive values before storing them in Bitwarden, so a Bitwarden breach still leaves your data unreadable.

**Live at:** `https://YOUR_USERNAME.github.io/vault`

---

## Security design

### Why passphrase-based symmetric encryption, not a public/private key?

The key question when designing this was: _should the site have a hardcoded private key, use asymmetric (RSA/EC) keypairs, or just use a passphrase?_

**Hardcoded key in the site: no.**
This repo is public. Any secret baked into the source code is visible to everyone. A hardcoded key provides zero security.

**Public/private key pair: not useful here.**
Asymmetric crypto shines when you need to encrypt for _someone else_ without knowing their decryption secret. Here, _you_ are both the encryptor and decryptor. Asymmetric keys don't add meaningful security over a strong passphrase for this use case — and they introduce a new problem: where do you store the private key? On the website (public → visible), in your head (= a passphrase), or somewhere else (a new attack surface). There's no win.

**Passphrase-based symmetric encryption: correct.**
- The only secret is something that lives exclusively in your memory
- The website can be fully public with zero secret material in the source
- PBKDF2 with 600 000 iterations makes brute-force attacks computationally expensive even if an attacker obtains the ciphertext
- AES-256-GCM provides authenticated encryption: a wrong passphrase fails loudly rather than silently returning garbled data

### Cryptographic details

| Property         | Value                                                               |
|------------------|---------------------------------------------------------------------|
| Encryption       | AES-256-GCM (authenticated encryption)                             |
| Key derivation   | PBKDF2-HMAC-SHA256, 600 000 iterations (OWASP 2024 minimum)        |
| Salt             | 128-bit, cryptographically random, unique per encryption           |
| IV / nonce       | 96-bit, cryptographically random, unique per encryption            |
| Output format    | `ENC:v1:<base64(version ‖ salt ‖ iv ‖ ciphertext+GCM-tag)>`       |
| Runtime          | Web Crypto API — browser-native, no third-party libraries          |

Because salt and IV are random and unique per encryption, encrypting the same plaintext twice always produces different output.

### Threat model

**What this protects against**

| Threat | How |
|--------|-----|
| Bitwarden database breach | Attacker reads `ENC:v1:…` tokens — useless without your passphrase |
| Forced access to Bitwarden | Same: they see tokens, not plaintext |
| Shoulder surfing your Bitwarden | Tokens are opaque |

**What this does NOT protect against**

- A keylogger or malware on your device (captures keystrokes before encryption)
- A supply-chain attack on this GitHub Pages site (mitigate: review the source before typing your passphrase, or self-host)
- Physical coercion for the passphrase itself (no software solves this)

### Passphrase advice

- Use a **Diceware** phrase: 4–5 randomly chosen words from a wordlist. Easy to memorize, enormous entropy (~51–64 bits).
- **Do not reuse** your Bitwarden master password or any other existing password.
- There is no passphrase recovery. If you forget it, the data is gone. Consider keeping a paper backup of the phrase in a physically secure location.

---

## Deploying to GitHub Pages

### Requirements

- A GitHub account (free tier works)
- The repository must be **public** to use GitHub Pages on a free account

### One-time setup

**Step 1 — Create the repository**

Go to https://github.com/new and create a public repository. Name it `vault` (or whatever you like).

**Step 2 — Push the files**

```bash
git clone https://github.com/YOUR_USERNAME/vault.git
cd vault
# Copy index.html and README.md into this folder, then:
git add .
git commit -m "Initial commit"
git push origin main
```

**Step 3 — Enable GitHub Pages**

1. Open your repository on GitHub
2. Click **Settings** (top tab)
3. In the left sidebar click **Pages**
4. Under **Source**, choose: *Deploy from a branch*
5. Under **Branch**, select: `main` / `/ (root)`
6. Click **Save**

Within about 60 seconds your site will be live at:

```
https://YOUR_USERNAME.github.io/vault
```

GitHub Pages provides HTTPS automatically — this is required for the Web Crypto API to function.

### Updating the site

```bash
# After editing index.html locally
git add index.html
git commit -m "Update"
git push
```

GitHub Actions redeploys automatically; allow ~30–60 seconds.

### Private hosting (optional)

If you'd prefer not to have a public repository, you can:
- Use GitHub Pages with a **GitHub Pro / Team** account (supports private repos)
- Self-host the single `index.html` file on any static host (Cloudflare Pages, Netlify, Vercel — all have free tiers and serve over HTTPS)
- Run it locally: open `index.html` directly in a browser at `localhost` — the Web Crypto API works on `localhost` even without HTTPS

---

## Usage

**Encrypting a password before storing it in Bitwarden**

1. Open `https://YOUR_USERNAME.github.io/vault`
2. On the **Encrypt** tab, enter your passphrase and the plaintext value
3. Click **Encrypt** — this takes 1–2 seconds (intentional; PBKDF2 is slow by design)
4. Copy the `ENC:v1:…` token
5. Store the token in Bitwarden instead of the raw value

**Decrypting when you need the value**

1. Copy the `ENC:v1:…` token from Bitwarden
2. Open the vault site, go to the **Decrypt** tab
3. Enter your passphrase and paste the token
4. The plaintext appears — copy it, use it, close the tab

---

## Verification

Before trusting any tool with your data, verify the source code yourself. This entire application is a single `index.html` file. Open it in a text editor and confirm:

- No external scripts or stylesheets are loaded
- The crypto section uses only `crypto.subtle` (Web Crypto API)
- There are no `fetch` or `XMLHttpRequest` calls

You can also run it locally without a network connection to confirm it works offline.
