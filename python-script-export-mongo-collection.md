To **log in to Git on your local machine**, the method depends on whether you use **HTTPS** or **SSH** to access GitHub/GitLab/Bitbucket.

Below are the **most common ways to log in locally**.

---

# ✅ **1. Git Login via HTTPS (Recommended for GitHub)**

When you push/pull for the first time, Git will ask for username & password —
BUT GitHub *no longer accepts passwords*. You must use a **Personal Access Token (PAT)**.

### **Step A — Generate PAT**

1. Go to GitHub → **Settings**
2. Developer settings → **Personal Access Tokens**
3. Generate **Fine-grained token**
4. Copy the token (save it safely).

### **Step B — Configure Git credentials**

Run these in terminal:

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

### **Step C — Login**

Do a push:

```bash
git push
```

Git will ask:

```
Username: your-github-username
Password: <paste your PAT here>
```

Then Git will store it in credential manager.

---

# ✅ **2. Git Login via SSH (No password needed)**

Recommended if you want **password-less login**.

### **Step A — Generate SSH key**

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

Press enter → enter → done.

Keys stored at:

```
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

### **Step B — Copy SSH public key**

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the output.

### **Step C — Add to GitHub**

GitHub → Settings → **SSH and GPG keys** → New SSH key → paste.

### **Step D — Test connection**

```bash
ssh -T git@github.com
```

If successful:

```
Hi <username>! You’ve successfully authenticated.
```

---

# ✅ **3. Check Git login status**

```bash
git config --global --list
```

---

# ✅ **4. To change logged-in user**

Clear stored credentials:

### **Mac**

```bash
git credential-osxkeychain erase
```

### **Windows**

Use **Credential Manager** → remove "git:[https://github.com](https://github.com)" entry.

---

# If you want, tell me which platform you use (Mac / Windows / Linux),

and I can give you **exact commands** step-by-step.
