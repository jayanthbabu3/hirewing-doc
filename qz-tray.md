# QZ Tray Setup — Silent Printing for TDQ Commerce POS

Every POS machine needs two things: **QZ Tray installed** and **our signing certificate copied**. That's it.

---

## New Machine Setup (3 Steps)

### Step 1: Install QZ Tray

1. Download from **https://qz.io/download/**
2. Run the installer
3. QZ Tray starts automatically and runs in the background

---

### Step 2: Copy the Signing Certificate

This is the key step. Without this, QZ Tray shows a popup on every print.

The certificate file is in our project repo at:

```
ui-tdqc-partnerintegration/qz-certs/digital-certificate.txt
```

You need to copy this file into the QZ Tray installation folder and rename it to `override.crt`.

#### Windows (most POS machines):

1. Open **Command Prompt as Administrator** (right-click → Run as administrator)
2. Run:

```cmd
copy "C:\path\to\project\qz-certs\digital-certificate.txt" "C:\Program Files\QZ Tray\override.crt"
```

Or if you have the file on a USB/shared drive:

```cmd
copy "D:\qz-certs\digital-certificate.txt" "C:\Program Files\QZ Tray\override.crt"
```

#### macOS:

```bash
sudo cp /path/to/project/qz-certs/digital-certificate.txt "/Applications/QZ Tray.app/Contents/Resources/override.crt"
```

#### Linux:

```bash
sudo cp /path/to/project/qz-certs/digital-certificate.txt /opt/qz-tray/override.crt
```

---

### Step 3: Restart QZ Tray

- **Windows**: Right-click QZ icon in system tray (bottom-right near clock) → **Exit**, then reopen from Start Menu → QZ Tray
- **macOS**: Right-click QZ icon in menu bar → **Quit**, then reopen from Applications
- **Linux**: `pkill qz` then restart from your application launcher

---

## Done! Now Configure in the App

1. Open the POS app in your browser and log in
2. Click the **printer icon** in the header (top-right)
3. Click **Connect to QZ Tray** — it should connect with no popup
4. Select your **printer** from the dropdown
5. Select **Printer Type**:
   - **Regular / PDF** — for regular printers
   - **Thermal (ESC/POS)** — for thermal receipt printers
6. Toggle **Auto-Print Orders** to **ON**

New orders will now print automatically.

---

## Where Is the Certificate File?

| What | Where |
|------|-------|
| Certificate file (in repo) | `ui-tdqc-partnerintegration/qz-certs/digital-certificate.txt` |
| Where to copy it (Windows) | `C:\Program Files\QZ Tray\override.crt` |
| Where to copy it (macOS) | `/Applications/QZ Tray.app/Contents/Resources/override.crt` |
| Where to copy it (Linux) | `/opt/qz-tray/override.crt` |

> **Tip for bulk deployment**: Copy `digital-certificate.txt` to a shared network drive or USB stick. Then on each machine just run the copy command pointing to that location.

---

## Troubleshooting

### Still seeing "Action Required" popup

1. **Did you copy the certificate?** Check that `override.crt` exists in the QZ Tray installation folder (see table above)
2. **Did you restart QZ Tray?** It only reads the certificate on startup
3. **Did you run as Administrator?** On Windows, copying to `C:\Program Files\` requires admin rights

### No printers found

- Make sure at least one printer is installed in the OS (Settings → Printers)
- Click **Refresh** in Print Settings

### QZ Tray won't connect

- Make sure QZ Tray is running (look for the icon in the system tray / menu bar)
- Refresh the browser page

### Currency shows as `?` on thermal printer

- Select **Thermal (ESC/POS)** as Printer Type — the app uses `Rs.` instead of `₹` for thermal

### Orders not printing for some users

- Non-admin users only get prints for their assigned stores
- Check with admin that correct stores are assigned
