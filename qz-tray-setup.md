# QZ Tray Setup — Silent Printing for TDQ Commerce POS

Follow these steps on **every new POS machine**. Takes 5 minutes.

---

## Windows Setup (4 Steps)

### Step 1 — Install QZ Tray

1. Go to **https://qz.io/download/**
2. Download the **Windows** installer
3. Run it, click **Next → Next → Finish**

That's it. QZ Tray is now running in the background (look for the small QZ icon near the clock, bottom-right of the screen).

---

### Step 2 — Download the Certificate File

1. Open this link in the browser on the POS machine:

   **https://github.com/techdoquest/ui-tdqc-partnerintegration/blob/main/qz-certs/digital-certificate.txt**

2. Click the **Download raw file** button (top-right of the file viewer — the small download arrow icon).
3. Save the file. It will go into your **Downloads** folder as `digital-certificate.txt`.

> If you don't have GitHub access on the POS machine, download it once on your own laptop and put the file on a **USB stick** so you can carry it to each POS machine.

---

### Step 3 — Copy the Certificate into QZ Tray (Most Important Step)

This is the step everyone gets wrong. Read carefully.

1. Open **File Explorer**
2. In the address bar at the top, paste this and press Enter:

   ```
   C:\Program Files\QZ Tray
   ```

3. You should now see the QZ Tray installation folder. Leave this window open.
4. Open a second **File Explorer** window and go to your **Downloads** folder (where you saved `digital-certificate.txt`).
5. **Copy** `digital-certificate.txt` and **paste** it into the `C:\Program Files\QZ Tray` folder.
   - Windows will ask for admin permission — click **Continue**.
6. The file is now in the QZ Tray folder. **Rename it** to exactly:

   ```
   override.crt
   ```

   - Right-click the file → **Rename** → type `override.crt` → press Enter.
   - If Windows warns about changing the file extension, click **Yes**.

> Make sure the file is named **exactly** `override.crt` — not `override.crt.txt`. If you don't see the `.txt` extension, turn on **View → File name extensions** in File Explorer first.

---

### Step 4 — Restart QZ Tray

QZ Tray only reads the certificate when it starts up, so we need to restart it:

1. Find the **QZ icon** near the clock (bottom-right). You may need to click the small **^** arrow to see hidden icons.
2. Right-click the QZ icon → **Exit**
3. Open the Start Menu → search for **QZ Tray** → click it to start again.

Done. QZ Tray will now print without showing any popup.

---

## Configure in the POS App

1. Open the POS app in the browser and log in
2. Click the **printer icon** in the header (top-right)
3. Click **Connect to QZ Tray** — should connect with **no popup**
   - If a popup appears, the certificate isn't installed correctly. Go back to Step 3.
4. Select your **printer** from the dropdown
5. Select **Printer Type**:
   - **Regular / PDF** — for normal A4 printers
   - **Thermal (ESC/POS)** — for thermal receipt printers
6. Toggle **Auto-Print Orders** to **ON**

New orders will now print automatically.

---

## Quick Check — Did It Work?

| Test | Expected Result |
|------|-----------------|
| Open POS → click printer icon → Connect | Connects with **no popup** |
| Place a test order | Prints automatically |
| File at `C:\Program Files\QZ Tray\override.crt` exists | Yes |

---

## Troubleshooting

### Still seeing the "Action Required" popup

99% of the time it's one of these:

1. **File is named wrong** — go to `C:\Program Files\QZ Tray\` and check the file is exactly `override.crt` (not `override.crt.txt`, not `digital-certificate.txt`).
2. **QZ Tray wasn't restarted** — right-click the QZ tray icon → Exit, then reopen it from the Start Menu.
3. **Copied the wrong file** — make sure you copied `digital-certificate.txt` from `qz-certs/`, not some other file.

### "You need permission to do this" when copying

You're trying to paste into `C:\Program Files\QZ Tray\` without admin rights. When Windows asks for permission, click **Continue**. If you're not an admin on the machine, ask whoever set it up.

### No printers found in the dropdown

- Make sure the printer is installed in Windows (Settings → Bluetooth & devices → Printers & scanners)
- Click **Refresh** in the POS Print Settings

### QZ Tray won't connect at all

- Check that QZ Tray is running (QZ icon near the clock)
- If not running, open it from the Start Menu
- Refresh the browser page

### Currency shows as `?` on thermal printer

Select **Thermal (ESC/POS)** as Printer Type — the app uses `Rs.` instead of `₹` for thermal printers.

### Orders not printing for some users

Non-admin users only get prints for their assigned stores. Ask an admin to check store assignments.

---

## macOS / Linux (Rare — for dev machines only)

Most POS machines are Windows. For Mac/Linux dev machines, download the certificate from:

**https://github.com/techdoquest/ui-tdqc-partnerintegration/blob/main/qz-certs/digital-certificate.txt**

**macOS** (assuming the file is in `~/Downloads`):

```bash
sudo cp ~/Downloads/digital-certificate.txt "/Applications/QZ Tray.app/Contents/Resources/override.crt"
```

Then quit QZ Tray from the menu bar and reopen it.

**Linux**:

```bash
sudo cp ~/Downloads/digital-certificate.txt /opt/qz-tray/override.crt
pkill qz
```

Then restart QZ Tray.
