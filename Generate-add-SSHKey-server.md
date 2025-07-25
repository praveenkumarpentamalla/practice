Sure! Here's a **step-by-step guide** to add your **SSH public key to a remote server**, so you can log in without entering a password each time.

---

### ✅ Step 1: Generate SSH Key (if not already done)

On your **local machine (your laptop/computer)**:

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

Just press `Enter` to accept the default file location (usually `~/.ssh/id_rsa`).

* Your **public key** will be: `~/.ssh/id_rsa.pub`
* Your **private key** will be: `~/.ssh/id_rsa`

---

### ✅ Step 2: Copy the Public Key to the Server

You need to copy your **public key** to the **remote server**.

#### Method 1: Use `ssh-copy-id` (easy and recommended)

```bash
ssh-copy-id username@server_ip
```

Example:

```bash
ssh-copy-id ubuntu@82.197.93.132
```

It will ask for your remote password once and copy the key to the server’s `~/.ssh/authorized_keys` file.

---

#### Method 2: Manual Copy (if `ssh-copy-id` is not available)

1. **Print your public key**:

```bash
cat ~/.ssh/id_rsa.pub
```

Copy the output (it starts with `ssh-rsa` or `ssh-ed25519`).

2. **SSH into your server**:

```bash
ssh username@server_ip
```

3. **On the server**, create the `.ssh` directory if it doesn't exist:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

4. **Open or create the `authorized_keys` file**:

```bash
nano ~/.ssh/authorized_keys
```

5. Paste your public key into this file, then:

* Save and exit (`Ctrl+O`, `Enter`, then `Ctrl+X` in `nano`).
* Set permissions:

```bash
chmod 600 ~/.ssh/authorized_keys
```

6. Exit the server:

```bash
exit
```

---

### ✅ Step 3: Test the SSH Login

Now try to login again:

```bash
ssh username@server_ip
```

If everything is set up correctly, you **shouldn’t be asked for a password**.

---

### 🛡️ Optional: Disable Password Authentication (for extra security)

Once key-based login works, you can disable password auth:

1. Edit SSH config on server:

```bash
sudo nano /etc/ssh/sshd_config
```

Set these values:

```conf
PasswordAuthentication no
ChallengeResponseAuthentication no
```

2. Restart SSH:

```bash
sudo systemctl restart ssh
```

