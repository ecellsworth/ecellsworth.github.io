---
title: "Setting Up LittleLink on GitHub Pages
description: "Create your own Linktree page for free using LittleLink on GitHub Pages
category: "MCP & Agents"
pubDate: 2026-08-10
difficulty: "beginner"
---

A step-by-step tutorial for creating your own free, self-hosted [LittleLink](https://github.com/sethcottle/littlelink) page (a lightweight Linktree alternative) and hosting it at `https://YOUR_USERNAME.github.io`.

---

## Prerequisites

- A free [GitHub](https://github.com) account
- Basic comfort with a terminal (Command Prompt, PowerShell, Terminal, or Git Bash)
- A text editor (VS Code, Notepad++, nano, etc.)

---

## 1. Create your free GitHub account and user site repository

1. Go to [https://github.com](https://github.com) and sign up (or log in).
2. Click the **+** (top right) → **New repository**.
3. **Repository name** must be exactly `YOUR_USERNAME.github.io`  
   (replace `YOUR_USERNAME` with your actual GitHub username — this is required for the root URL).
4. Set visibility to **Public**.
5. **Do not** check “Add a README”, “Add .gitignore”, or “Choose a license” (we will populate it with LittleLink).
6. Click **Create repository**.

Your empty repo now exists at `https://github.com/YOUR_USERNAME/YOUR_USERNAME.github.io`.

---

## 2. Set up Git on your local machine

### Install Git

- **Windows**: Download and install from [https://git-scm.com](https://git-scm.com). Use the defaults (it includes Git Bash).
- **macOS**: `xcode-select --install` or install via Homebrew (`brew install git`).
- **Linux**: `sudo apt install git` (Debian/Ubuntu) or equivalent.

### Configure Git (one-time)

Open a terminal and run:

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

(Use the email associated with your GitHub account.)

### Authentication (choose one)

**Option A – GitHub CLI (easiest, recommended)**

```bash
# Install: https://cli.github.com
gh auth login
```

Follow the prompts (HTTPS or SSH, browser login).

**Option B – HTTPS + Personal Access Token**

1. GitHub → Settings → Developer settings → Personal access tokens → Generate new token (classic).
2. Give it `repo` scope.
3. Copy the token. When Git asks for a password later, paste the token.

**Option C – SSH keys**

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
# Press Enter for defaults
cat ~/.ssh/id_ed25519.pub   # copy this
```

Add the public key at GitHub → Settings → SSH and GPG keys → New SSH key.

---

## 3. Get LittleLink onto your machine and into the repo

In a terminal:

```bash
# Clone the official LittleLink repo temporarily
git clone https://github.com/sethcottle/littlelink.git littlelink-temp
cd littlelink-temp

# Remove the original Git history
rm -rf .git

# Initialize a fresh Git repo
git init
git branch -M main

# Point it at your new GitHub Pages repo
git remote add origin https://github.com/YOUR_USERNAME/YOUR_USERNAME.github.io.git
# (or git@github.com:YOUR_USERNAME/YOUR_USERNAME.github.io.git if using SSH)

# First commit & push
git add .
git commit -m "Initial LittleLink setup"
git push -u origin main
```

### Enable GitHub Pages

1. Go to your repo on GitHub → **Settings** → **Pages** (left sidebar).
2. Under **Build and deployment** → **Source**, choose **Deploy from a branch**.
3. Branch: `main`, folder: `/ (root)`.
4. Click **Save**.

After 1–2 minutes your site will be live at:

**https://YOUR_USERNAME.github.io**

---

## 4. Customize the basics (avatar, name, description)

Still inside the `littlelink-temp` folder (or re-clone later if needed):

1. Replace the avatar images:
   - `images/avatar.png` and `images/avatar@2x.png`  
     (recommended size ~200–400 px square, PNG or JPG).

2. Open `index.html` in a text editor and change:
   - The `<title>` tag
   - The `<h1>` name
   - The short description `<p>` under the name
   - The theme on the `<html>` tag (`theme-auto`, `theme-light`, or `theme-dark`)

3. Commit and push to Github when ready:

```bash
git add .
git commit -m "Update avatar, name and description"
git push
```

---

## 5. Manage links with a simple text file + script

Create two files in the same folder as `index.html`.

### `links.txt` (the source of truth)

```text
# Format: brand|URL|Display Label
# Lines starting with # are ignored
# Order in this file = order of buttons on the page
# Brand names should match LittleLink’s icon/class names (github, x, instagram, youtube, linkedin, discord, etc.)

github|https://github.com/YOUR_USERNAME|GitHub
x|https://x.com/YOUR_USERNAME|Follow on X
instagram|https://instagram.com/YOUR_USERNAME|Instagram
youtube|https://youtube.com/@YOUR_CHANNEL|YouTube
linkedin|https://linkedin.com/in/YOUR_USERNAME|LinkedIn
# Add or remove lines as needed. Re-run the script after editing.
```

### `update_links.py` (the updater script)

Save this exact file:

```python
#!/usr/bin/env python3
"""
LittleLink Link Updater
Reads links.txt (in order) and rewrites the buttons inside index.html.
Edit links.txt → run this script → git commit & push.
"""

import re
from pathlib import Path

LINKS_FILE = Path("links.txt")
INDEX_FILE = Path("index.html")

# Known exceptions (brand → (css_suffix, icon_filename_without_svg, default_label))
OVERRIDES = {
    "youtube": ("yt", "youtube", "YouTube"),
    "linkedin": ("linked", "linkedin", "LinkedIn"),
    "twitter": ("x", "x", "X"),
    "x": ("x", "x", "X"),
    "default": ("default", "littlelink", "Link"),
}

def load_links():
    links = []
    if not LINKS_FILE.exists():
        print("links.txt not found. Create it first.")
        return []
    for line in LINKS_FILE.read_text(encoding="utf-8").splitlines():
        line = line.strip()
        if not line or line.startswith("#"):
            continue
        parts = [p.strip() for p in line.split("|")]
        if len(parts) < 2:
            continue
        brand = parts[0].lower()
        url = parts[1]
        label = parts[2] if len(parts) > 2 else None
        links.append((brand, url, label))
    return links

def make_button(brand, url, label):
    if brand in OVERRIDES:
        css, icon, default_label = OVERRIDES[brand]
    else:
        # Most brands use the same name for class and icon
        css = brand
        icon = brand
        default_label = brand.capitalize()

    if not label:
        label = default_label

    return f'''      <a class="button button-{css}" href="{url}" target="_blank" rel="noopener" role="button">
        <img class="icon" aria-hidden="true" src="images/icons/{icon}.svg" alt="{label} Logo">
        {label}
      </a>'''

def update_index(links):
    if not INDEX_FILE.exists():
        print("index.html not found.")
        return

    content = INDEX_FILE.read_text(encoding="utf-8")
    buttons = "\n".join(make_button(*item) for item in links)

    # Primary: replace everything inside the button-stack / nav
    pattern = r'(<(?:nav|div)[^>]*class="[^"]*button-stack[^"]*"[^>]*>)(.*?)(</(?:nav|div)>)'
    new_content, n = re.subn(
        pattern,
        lambda m: m.group(1) + "\n" + buttons + "\n    " + m.group(3),
        content,
        count=1,
        flags=re.DOTALL | re.IGNORECASE,
    )

    # Fallback: use explicit markers if present
    if n == 0 and "<!-- LINKS-START -->" in content:
        pattern2 = r'(<!-- LINKS-START -->)(.*?)(<!-- LINKS-END -->)'
        new_content, n = re.subn(
            pattern2,
            r"\1\n" + buttons + r"\n    \3",
            content,
            count=1,
            flags=re.DOTALL,
        )

    if n == 0:
        print("Could not locate the button area.")
        print("Either keep the original button-stack or add markers:")
        print("  <!-- LINKS-START -->")
        print("  ...buttons...")
        print("  <!-- LINKS-END -->")
        return

    INDEX_FILE.write_text(new_content, encoding="utf-8")
    print(f"Updated index.html with {len(links)} link(s) in the order of links.txt")

if __name__ == "__main__":
    links = load_links()
    if links:
        update_index(links)
    else:
        print("No valid links found in links.txt")
```

Make it executable (optional):

```bash
chmod +x update_links.py
```

### How to use the script

1. Edit `links.txt` (add, remove, or reorder lines).
2. Run:

```bash
python3 update_links.py
```

3. Review the change in `index.html` if you want.
4. Push:

```bash
git add index.html links.txt update_links.py
git commit -m "Update links"
git push
```

The buttons will appear in exactly the order they are listed in `links.txt`. Re-running the script completely replaces the previous set of buttons, so adding or removing a link is just editing the text file and re-running.

**Notes on brands**

- Most common services work out of the box (`github`, `x`, `instagram`, `discord`, `twitch`, `tiktok`, `spotify`, etc.).
- For rarer brands or fully custom buttons, either:
  - Use the [LittleLink Button Builder](https://builder.littlelink.io), or
  - Manually add the CSS to `css/brands.css` and the SVG to `images/icons/`, then reference the brand name in `links.txt`.
- **Adding More Brands**: See the [LittleLink Extended](https://github.com/sethcottle/littlelink-extended) repo and the [wiki](https://github.com/sethcottle/littlelink/wiki) for adding custom buttons.

---

## 6. Local preview

To preview your page locally:

1. Open the folder that contains `index.html`.
2. Double-click `index.html` (or right-click → Open with your preferred web browser).

The page will open in your browser so you can check the layout, avatar, and links before pushing.

---

## 7. Daily workflow after the initial setup

```bash
# Workflow for future changes
cd path/to/littlelink-temp   # or whatever you named the folder

# Edit links.txt (or any other files)
# Then:
python3 update_links.py
git add .
git commit -m "Describe your change"
git push
```

GitHub Pages will rebuild automatically within a minute or two.

You now have a fully self-hosted, free LittleLink page at `https://YOUR_USERNAME.github.io` that you can update by simply editing a plain text file and running one command.


---

## Project Files

You can download the complete files used in this tutorial from the repository below:

<a class="btn-github" href="https://github.com/ecellsworth/littlelink-page">View repository on GitHub</a>


