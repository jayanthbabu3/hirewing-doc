# QZ Tray Setup Guide — Silent Printing for TDQ Commerce POS

This guide walks you through installing and configuring QZ Tray on your machine so that orders print silently (no browser dialog) when they come in.

---

## Step 1: Install QZ Tray

1. Go to **https://qz.io/download/**
2. Download the installer for your OS:
   - **Windows**: `.exe` installer
   - **macOS**: `.pkg` installer
   - **Linux**: `.run` installer
3. Run the installer and follow the prompts
4. After installation, QZ Tray runs in the background:
   - **Windows**: Look for the QZ icon in the **system tray** (bottom-right near the clock)
   - **macOS**: Look for the QZ icon in the **menu bar** (top-right)

---

## Step 2: Whitelist Your Domain (Skip the Trust Dialog)

By default, QZ Tray asks "Allow this website to connect?" every time a page loads. To prevent this, you must create an **override.properties** file.

> **IMPORTANT**: Do NOT edit `prefs.properties` — QZ Tray overwrites it on restart. Use `override.properties` instead, which is permanent.

### Find the QZ Tray config folder

| OS | Config Folder |
|----|---------------|
| **macOS** | `~/Library/Application Support/qz/` |
| **Windows** | `C:\Users\<YourUsername>\AppData\Roaming\qz\` |
| **Linux** | `~/.qz/` |

### Create the override.properties file

Open a terminal (or Command Prompt on Windows) and run **one** of the following:

#### macOS:

```bash
printf 'wss.whitelist=localhost,localhost\\:5173,app.flex.dev.tdqcommerce.com,flex.tdqcommerce.com\n' > ~/Library/Application\ Support/qz/override.properties
```

#### Linux:

```bash
printf 'wss.whitelist=localhost,localhost\\:5173,app.flex.dev.tdqcommerce.com,flex.tdqcommerce.com\n' > ~/.qz/override.properties
```

#### Windows (Command Prompt):

```cmd
echo wss.whitelist=localhost,localhost:5173,app.flex.dev.tdqcommerce.com,flex.tdqcommerce.com > "%APPDATA%\qz\override.properties"
```

#### Windows (PowerShell):

```powershell
Set-Content "$env:APPDATA\qz\override.properties" "wss.whitelist=localhost,localhost:5173,app.flex.dev.tdqcommerce.com,flex.tdqcommerce.com"
```

### Common domain examples

| Environment | Domain to whitelist |
|---|---|
| Local development | `localhost` or `localhost:5173` |
| Dev / Staging | `app.flex.dev.tdqcommerce.com` |
| Production | `flex.tdqcommerce.com` |

All environments are whitelisted by default in the commands above:

```
wss.whitelist=localhost,localhost:5173,app.flex.dev.tdqcommerce.com,flex.tdqcommerce.com
```

### Restart QZ Tray

After creating the file, restart QZ Tray for changes to take effect:

- **macOS**: Right-click QZ icon in menu bar → **Quit**, then reopen from Applications
- **Windows**: Right-click QZ icon in system tray → **Exit**, then reopen from Start Menu
- **Linux**: `pkill qz` then restart from your application launcher

### Verify the file was created

Run this to confirm:

**macOS:**
```bash
cat ~/Library/Application\ Support/qz/override.properties
```

**Windows (Command Prompt):**
```cmd
type "%APPDATA%\qz\override.properties"
```

**Linux:**
```bash
cat ~/.qz/override.properties
```

You should see:
```
wss.whitelist=localhost,localhost:5173,app.flex.dev.tdqcommerce.com,flex.tdqcommerce.com
```

---

## Step 3: Verify It Works

1. Open the POS app in your browser and log in
2. Click the **printer icon** in the header (top-right, next to notifications bell)
3. Click **"Connect to QZ Tray"**
   - It should connect **without** showing any "Action Required" trust dialog
   - The badge should say **"Connected"** (green)
4. The printer list auto-loads — select your printer
5. Click **"Test Print"** — a test receipt should print silently

If you still see the trust dialog, see Troubleshooting below.

---

## Step 4: Enable Auto-Print

1. In the Print Settings dropdown, toggle **"Auto-Print Orders"** to **ON**
2. Select your **Printer Type**:
   - **Regular / PDF** — for regular printers
   - **Thermal (ESC/POS)** — for thermal receipt printers (80mm / 58mm)
3. Make sure a **printer is selected** from the list

From now on, every new order that comes in (for your assigned stores) will print automatically — no clicks needed.

### Store-Based Filtering

- **Admin users**: Receive notifications and auto-print for all orders from all stores
- **Other roles (User, Reporting)**: Only receive notifications and auto-print for orders from stores assigned to them
- If a user doesn't have the order's store assigned, they won't see the notification and it won't print on their machine

---

## Troubleshooting

### Trust dialog ("Action Required") still appears after whitelisting

This is the most common issue. Check these in order:

1. **Did you use `override.properties` (not `prefs.properties`)?**
   - `prefs.properties` gets overwritten by QZ Tray on restart — your changes are lost
   - `override.properties` is permanent and never overwritten
   - Verify: run the "Verify the file was created" command from Step 2

2. **Did you restart QZ Tray after creating the file?**
   - QZ Tray only reads config on startup
   - Quit it completely and reopen

3. **Is the domain correct?**
   - For localhost with a port, use `localhost:5173` (or whatever port your dev server uses)
   - For production, use the exact domain (no `https://` prefix, no trailing `/`)

4. **Check file location**
   - The file must be in the QZ Tray config folder (see table in Step 2)
   - The filename must be exactly `override.properties`

### "No printers found"

- Make sure at least one printer is installed in your OS
  - **Windows**: Settings → Devices → Printers & Scanners
  - **macOS**: System Settings → Printers & Scanners
- Click **Refresh** in Print Settings after adding a printer
- "Microsoft Print to PDF" (Windows) works for testing

### QZ Tray won't connect

- Make sure QZ Tray is running (check system tray / menu bar for the icon)
- If it's not running, open it from:
  - **Windows**: Start Menu → QZ Tray
  - **macOS**: Applications → QZ Tray
- Try refreshing the browser page after starting QZ Tray

### Currency shows as `?` on thermal printer

- Thermal printers only support ASCII characters
- The app automatically uses `Rs.` instead of `₹` for thermal mode
- Make sure **Thermal (ESC/POS)** is selected in Printer Type

### Print comes out blank or garbled

- Switch between **Regular / PDF** and **Thermal (ESC/POS)** in Print Settings
- Regular printers should use **Regular / PDF**
- Thermal receipt printers (Epson, Star, Bixolon, etc.) should use **Thermal (ESC/POS)**

### Orders not printing / no notifications for some users

- This is expected if the order's store is not assigned to that user
- Admin users see all orders; other roles only see orders for their assigned stores
- Check with your admin that the correct stores are assigned to the user

---

## Quick Reference

| Setting | Where |
|---|---|
| QZ Tray download | https://qz.io/download/ |
| Override config (macOS) | `~/Library/Application Support/qz/override.properties` |
| Override config (Windows) | `%APPDATA%\qz\override.properties` |
| Override config (Linux) | `~/.qz/override.properties` |
| Whitelist format | `wss.whitelist=domain1,domain2,domain3` |
| Print Settings in app | Printer icon in the header (top-right) |

---

## One-Liner Setup (Copy-Paste)

For quick setup on a new machine, just run one command after installing QZ Tray:

**macOS:**
```bash
printf 'wss.whitelist=localhost,localhost\\:5173,app.flex.dev.tdqcommerce.com,flex.tdqcommerce.com\n' > ~/Library/Application\ Support/qz/override.properties && echo "Done! Now restart QZ Tray."
```

**Windows (Command Prompt):**
```cmd
echo wss.whitelist=localhost,localhost:5173,app.flex.dev.tdqcommerce.com,flex.tdqcommerce.com > "%APPDATA%\qz\override.properties" && echo Done! Now restart QZ Tray.
```

**Linux:**
```bash
printf 'wss.whitelist=localhost,localhost\\:5173,app.flex.dev.tdqcommerce.com,flex.tdqcommerce.com\n' > ~/.qz/override.properties && echo "Done! Now restart QZ Tray."
```

Then restart QZ Tray and you're good to go.
