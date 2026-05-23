# XAMPP MySQL Crash Fix Guide
## XAMPP MySQL क्रैश फिक्स गाइड

---

## 🔴 The Error / एरर क्या दिखता है

```
[mysql] Status change detected: stopped
[mysql] Error: MySQL shutdown unexpectedly.
[mysql] This may be due to a blocked port, missing dependencies,
[mysql] improper privileges, a crash, or a shutdown by another method.
```

---

## 📖 ENGLISH

### Why Does This Happen?

MySQL (MariaDB) in XAMPP stores its system/privilege tables using the **Aria storage engine** — a file format with `.MAD` (data) and `.MAI` (index) files. This crash happens when one or more of these files get **corrupted**.

**Common reasons for corruption:**

| Cause | Description |
|-------|-------------|
| Force shutdown | Turning off PC / laptop without shutting MySQL down first |
| Power cut | Sudden electricity loss while MySQL was writing data |
| XAMPP killed forcefully | Closing XAMPP via Task Manager or force-restart |
| Disk full | MySQL trying to write when drive had no space left |
| Windows crash / BSOD | System crash while MySQL was running |
| Antivirus interference | AV scanning/locking MySQL data files mid-write |

**The most corrupted file is almost always:**
`D:\xampp\mysql\data\mysql\db.MAI` — the privilege table index.

When this file is corrupted, MySQL logs this fatal error in Windows Event Viewer (not in XAMPP's log panel):
```
Fatal error: Can't open and lock privilege tables: Incorrect file format 'db'
```

---

### Step-by-Step Fix

> **Prerequisites:** XAMPP installed, MySQL is stopped (not running).

---

#### Step 1 — Find your XAMPP path

Open PowerShell and run:
```powershell
where.exe mysqld
```
This will show something like `D:\xampp\mysql\bin\mysqld.exe`.
Your XAMPP root is `D:\xampp` (replace with your actual drive).

---

#### Step 2 — Check Windows Event Viewer for real error

XAMPP's log panel hides the actual error. Check Event Viewer:

1. Press `Win + R` → type `eventvwr` → Enter
2. Go to **Windows Logs → Application**
3. Filter by Source: **MariaDB**
4. Look for **Error** entries — you'll see the real crash reason

Common errors to look for:
- `Fatal error: Can't open and lock privilege tables: Incorrect file format 'db'`
- `Got error 176 "Read page with wrong checksum" from storage engine Aria`

---

#### Step 3 — Backup the corrupted files first

Open PowerShell as Administrator:
```powershell
Copy-Item "D:\xampp\mysql\data\mysql\db.MAD" "D:\xampp\mysql\data\mysql\db.MAD.bak" -Force
Copy-Item "D:\xampp\mysql\data\mysql\db.MAI" "D:\xampp\mysql\data\mysql\db.MAI.bak" -Force
```

---

#### Step 4 — Replace corrupted files from XAMPP backup

XAMPP ships with a clean backup of system tables at `D:\xampp\mysql\backup\`.
```powershell
Copy-Item "D:\xampp\mysql\backup\mysql\db.MAD" "D:\xampp\mysql\data\mysql\db.MAD" -Force
Copy-Item "D:\xampp\mysql\backup\mysql\db.MAI" "D:\xampp\mysql\data\mysql\db.MAI" -Force
```

> ✅ This is safe — your user databases (`rimeso`, `myapp`, etc.) are stored in InnoDB (`ibdata1`), not in these system files. They will NOT be affected.

---

#### Step 5 — Start MySQL manually to repair all tables

```powershell
Start-Process -FilePath "D:\xampp\mysql\bin\mysqld.exe" -ArgumentList "--defaults-file=D:\xampp\mysql\bin\my.ini" -WindowStyle Hidden
Start-Sleep -Seconds 10
```

Verify it started (should show port 3306 LISTENING):
```powershell
netstat -ano | findstr ":3306"
```

---

#### Step 6 — Repair ALL Aria system tables

```powershell
cmd /c "D:\xampp\mysql\bin\mysql.exe -u root -e ""REPAIR TABLE mysql.columns_priv, mysql.db, mysql.event, mysql.func, mysql.global_priv, mysql.help_category, mysql.help_keyword, mysql.help_relation, mysql.help_topic, mysql.plugin, mysql.proc, mysql.procs_priv, mysql.proxies_priv, mysql.roles_mapping, mysql.servers, mysql.tables_priv, mysql.time_zone, mysql.time_zone_leap_second, mysql.time_zone_name, mysql.time_zone_transition, mysql.time_zone_transition_type USE_FRM;"" 2>&1"
```

All tables should show `status: OK`.

---

#### Step 7 — Create missing system user (if needed)

```powershell
cmd /c "D:\xampp\mysql\bin\mysql.exe -u root -e ""CREATE USER IF NOT EXISTS 'mariadb.sys'@'localhost' IDENTIFIED VIA mysql_native_password USING '' ACCOUNT LOCK;"" 2>&1"
```

---

#### Step 8 — Run mysql_upgrade

```powershell
cmd /c "D:\xampp\mysql\bin\mysql_upgrade.exe -u root --force 2>&1"
```

It should end with `Phase 7/7: Running 'FLUSH PRIVILEGES'` → `OK`

---

#### Step 9 — Shutdown and restart via XAMPP

```powershell
cmd /c "D:\xampp\mysql\bin\mysqladmin.exe -u root shutdown 2>&1"
```

Wait 5 seconds, then click **Start** in XAMPP Control Panel. MySQL should now start normally.

---

#### Step 10 — Cleanup backup files

```powershell
Remove-Item "D:\xampp\mysql\data\mysql\db.MAD.bak" -Force
Remove-Item "D:\xampp\mysql\data\mysql\db.MAI.bak" -Force
```

---

### How to Prevent This in Future

1. **Always stop MySQL from XAMPP** before shutting down your PC
2. **Never force-kill XAMPP** via Task Manager while MySQL is running
3. **Add XAMPP folder to Antivirus exclusions:**
   `D:\xampp\mysql\data\` → add to Windows Defender exclusion list
4. **Enable UPS/battery backup** if you're on a desktop PC
5. **Take periodic backups** using phpMyAdmin Export or `mysqldump`

---

---

## 📖 हिंदी (HINDI)

### यह समस्या क्यों होती है?

XAMPP में MySQL (MariaDB) अपनी system/privilege tables को **Aria storage engine** में स्टोर करता है। यह `.MAD` (data file) और `.MAI` (index file) फॉर्मेट में होती हैं। जब ये फाइलें **corrupt** हो जाती हैं, तो MySQL start होते ही crash कर देता है।

**Corruption के मुख्य कारण:**

| कारण | विवरण |
|------|-------|
| Force shutdown | MySQL बंद किए बिना PC/Laptop बंद करना |
| बिजली कटौती | MySQL चल रहा हो तब अचानक बिजली जाना |
| XAMPP जबरदस्ती बंद करना | Task Manager से XAMPP को kill करना |
| Disk full | Drive में जगह न हो तब MySQL data लिखने की कोशिश करे |
| Windows crash / BSOD | MySQL चलते समय system crash होना |
| Antivirus interference | Antivirus का MySQL data files को scan/lock करना |

**सबसे ज्यादा corrupt होने वाली फाइल:**
`D:\xampp\mysql\data\mysql\db.MAI` — privilege table की index file।

जब यह file corrupt हो जाती है, Windows Event Viewer में यह error दिखती है:
```
Fatal error: Can't open and lock privilege tables: Incorrect file format 'db'
```
(यह error XAMPP के log panel में नहीं दिखती — इसलिए Event Viewer देखना जरूरी है।)

---

### Step-by-Step Fix (हिंदी में)

> **जरूरी:** XAMPP installed हो, MySQL बंद हो।

---

#### Step 1 — XAMPP का path पता करें

PowerShell खोलें और यह command चलाएं:
```powershell
where.exe mysqld
```
Output: `D:\xampp\mysql\bin\mysqld.exe` जैसा कुछ दिखेगा।
आपका XAMPP root = `D:\xampp` (अपने drive letter से बदलें)।

---

#### Step 2 — Windows Event Viewer में असली error देखें

XAMPP का log panel असली error नहीं दिखाता। Event Viewer में देखें:

1. `Win + R` दबाएं → `eventvwr` type करें → Enter
2. **Windows Logs → Application** पर जाएं
3. Source: **MariaDB** से filter करें
4. **Error** entries देखें — असली crash reason मिलेगा

देखने वाली errors:
- `Fatal error: Can't open and lock privilege tables: Incorrect file format 'db'`
- `Got error 176 "Read page with wrong checksum" from storage engine Aria`

---

#### Step 3 — Corrupt files का backup बनाएं

PowerShell (Administrator) में:
```powershell
Copy-Item "D:\xampp\mysql\data\mysql\db.MAD" "D:\xampp\mysql\data\mysql\db.MAD.bak" -Force
Copy-Item "D:\xampp\mysql\data\mysql\db.MAI" "D:\xampp\mysql\data\mysql\db.MAI.bak" -Force
```

---

#### Step 4 — XAMPP के backup से clean files restore करें

XAMPP के साथ `D:\xampp\mysql\backup\` में clean system tables होती हैं:
```powershell
Copy-Item "D:\xampp\mysql\backup\mysql\db.MAD" "D:\xampp\mysql\data\mysql\db.MAD" -Force
Copy-Item "D:\xampp\mysql\backup\mysql\db.MAI" "D:\xampp\mysql\data\mysql\db.MAI" -Force
```

> ✅ यह बिल्कुल safe है — आपके user databases (`rimeso`, `myapp`, आदि) InnoDB में हैं (`ibdata1` में), इन system files में नहीं। आपका data safe रहेगा।

---

#### Step 5 — MySQL manually start करें (tables repair के लिए)

```powershell
Start-Process -FilePath "D:\xampp\mysql\bin\mysqld.exe" -ArgumentList "--defaults-file=D:\xampp\mysql\bin\my.ini" -WindowStyle Hidden
Start-Sleep -Seconds 10
```

Verify करें कि start हुआ (port 3306 LISTENING होना चाहिए):
```powershell
netstat -ano | findstr ":3306"
```

---

#### Step 6 — सभी Aria system tables repair करें

```powershell
cmd /c "D:\xampp\mysql\bin\mysql.exe -u root -e ""REPAIR TABLE mysql.columns_priv, mysql.db, mysql.event, mysql.func, mysql.global_priv, mysql.help_category, mysql.help_keyword, mysql.help_relation, mysql.help_topic, mysql.plugin, mysql.proc, mysql.procs_priv, mysql.proxies_priv, mysql.roles_mapping, mysql.servers, mysql.tables_priv, mysql.time_zone, mysql.time_zone_leap_second, mysql.time_zone_name, mysql.time_zone_transition, mysql.time_zone_transition_type USE_FRM;"" 2>&1"
```

सभी tables में `status: OK` दिखना चाहिए।

---

#### Step 7 — Missing system user बनाएं (अगर जरूरत हो)

```powershell
cmd /c "D:\xampp\mysql\bin\mysql.exe -u root -e ""CREATE USER IF NOT EXISTS 'mariadb.sys'@'localhost' IDENTIFIED VIA mysql_native_password USING '' ACCOUNT LOCK;"" 2>&1"
```

---

#### Step 8 — mysql_upgrade चलाएं

```powershell
cmd /c "D:\xampp\mysql\bin\mysql_upgrade.exe -u root --force 2>&1"
```

अंत में `Phase 7/7: Running 'FLUSH PRIVILEGES'` → `OK` दिखना चाहिए।

---

#### Step 9 — MySQL बंद करें और XAMPP से restart करें

```powershell
cmd /c "D:\xampp\mysql\bin\mysqladmin.exe -u root shutdown 2>&1"
```

5 seconds रुकें, फिर XAMPP Control Panel में **Start** दबाएं। MySQL अब normal start होगा।

---

#### Step 10 — Backup files delete करें

```powershell
Remove-Item "D:\xampp\mysql\data\mysql\db.MAD.bak" -Force
Remove-Item "D:\xampp\mysql\data\mysql\db.MAI.bak" -Force
```

---

### भविष्य में यह समस्या कैसे रोकें?

1. **PC बंद करने से पहले हमेशा XAMPP से MySQL बंद करें**
2. **MySQL चलते समय XAMPP को Task Manager से कभी kill न करें**
3. **Antivirus exclusion list में XAMPP folder add करें:**
   `D:\xampp\mysql\data\` → Windows Defender Exclusion में डालें
   *(Settings → Windows Security → Virus & Threat Protection → Exclusions)*
4. **Desktop PC है तो UPS लगाएं** ताकि बिजली जाने पर safely shutdown हो
5. **Regular backup लें** — phpMyAdmin Export या `mysqldump` से

---

## 🗂 Quick Reference / त्वरित संदर्भ

| File | Location | Purpose |
|------|----------|---------|
| MySQL Error Log | `D:\xampp\mysql\data\mysql_error.log` | XAMPP-level startup log |
| Real Error | Windows Event Viewer → Application → MariaDB | Actual crash reason |
| Corrupt File | `D:\xampp\mysql\data\mysql\db.MAI` | Usually the culprit |
| Clean Backup | `D:\xampp\mysql\backup\mysql\` | XAMPP's factory backup |
| Main Config | `D:\xampp\mysql\bin\my.ini` | MySQL configuration |

---

*Guide created after real incident on 2026-05-23 — XAMPP MariaDB 10.4.32 on Windows 11*
