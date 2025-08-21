# Windows-Guide
## Windows Cron Jobs using Task Scheduler

### Configuring AWS in Windows

<details>
  <summary>Click to view the steps</summary>

## 1) Create the profile (one-time)

In **CMD or PowerShell**:

```bash
aws configure --profile my-sync-profile
```

Enter your Access Key, Secret, and Region (e.g. `ap-south-1`).

This writes:

* `%UserProfile%\.aws\credentials`
* `%UserProfile%\.aws\config`

---

## 2) Make it the default for your terminals

Pick **one** of these (both work):

### A) Persist for future sessions (recommended)

**CMD:**

```cmd
setx AWS_DEFAULT_PROFILE "my-sync-profile"
setx AWS_DEFAULT_REGION "ap-south-1"
```

**PowerShell:**

```powershell
setx AWS_DEFAULT_PROFILE "my-sync-profile"
setx AWS_DEFAULT_REGION "ap-south-1"
```

> Close & reopen the terminal after `setx`.

### B) Just for the current window (temporary)

**CMD:**

```cmd
set AWS_PROFILE=my-sync-profile
set AWS_DEFAULT_REGION=ap-south-1
```

**PowerShell:**

```powershell
$env:AWS_PROFILE = "my-sync-profile"
$env:AWS_DEFAULT_REGION = "ap-south-1"
```

> `AWS_PROFILE` and `AWS_DEFAULT_PROFILE` behave the same for choosing the default.

---

## 3) Make it the default for Task Scheduler

You have two tidy options. Use whichever matches how your task runs.

### Option 3A — Inject env var in the **Action** (works with any user/SYSTEM)

**If your task runs a batch (`.bat`) via CMD:**

```cmd
SCHTASKS /Change /TN "S3DailySync" /TR "cmd.exe /c set AWS_PROFILE=my-sync-profile&& set AWS_DEFAULT_REGION=ap-south-1&& C:\scripts\s3sync.bat"
```

**If your task runs a PowerShell script:**

```cmd
SCHTASKS /Change /TN "S3DailySync" /TR "powershell.exe -NoProfile -ExecutionPolicy Bypass -Command \"$env:AWS_PROFILE='my-sync-profile'; $env:AWS_DEFAULT_REGION='ap-south-1'; & 'C:\scripts\s3sync.ps1'\""
```

> This guarantees the task uses the right profile, even when it runs as **SYSTEM** or a different account.

### Option 3B — Set inside your script (simple)

**Batch (`s3sync.bat`), add at the top:**

```bat
set AWS_PROFILE=my-sync-profile
set AWS_DEFAULT_REGION=ap-south-1
```

**PowerShell (`s3sync.ps1`), add at the top:**

```powershell
$env:AWS_PROFILE = "my-sync-profile"
$env:AWS_DEFAULT_REGION = "ap-south-1"
```

---

## 4) Verify

Run these from a new terminal (or trigger the task), then check:

```bash
aws configure list
aws sts get-caller-identity
```

You should see the profile in use and the expected IAM identity.
Optionally list your bucket to confirm access:

```bash
aws s3 ls s3://elasticbeanstalk-ap-south-1-508351649560/resources/environments/logs/
```

---

## 5) (Optional) Make `my-sync-profile` the literal `[default]`

If you **really** want no env vars at all, you can copy the credentials into the `[default]` section:

**`%UserProfile%\.aws\credentials`**

```ini
[default]
aws_access_key_id=AKIA...
aws_secret_access_key=...

[my-sync-profile]
aws_access_key_id=AKIA...
aws_secret_access_key=...
```

**`%UserProfile%\.aws\config`**

```ini
[default]
region=ap-south-1
output=json

[profile my-sync-profile]
region=ap-south-1
output=json
```

> Caution: this changes the default for **everything** on that machine/user.

---

### Quick recap

* Create it: `aws configure --profile my-sync-profile`
* Make it default:

  * Persist: `setx AWS_DEFAULT_PROFILE my-sync-profile`
  * Or inject in task action / script (`AWS_PROFILE=my-sync-profile`)
* Verify: `aws sts get-caller-identity`, `aws configure list`

That’s it — now `aws` will behave as if `my-sync-profile` is the default everywhere.

</details>

---



### **Step 1: Create the AWS profile**

Run this in PowerShell (replace with your real values):

```powershell
aws configure --profile mfa-session
```

It will ask:

```
AWS Access Key ID [None]: ASIAxxxx
AWS Secret Access Key [None]: xxxxx
Default region name [None]: ap-south-1
Default output format [None]: json
```

👉 After this, open the file
`C:\Users\<YourUser>\.aws\credentials`
and **add the session token** manually under `[mfa-session]`: as well as add the aws access key and secret access key after the mfa command is given in the cli

```ini
[mfa-session]
aws_access_key_id = ASIAxxxx
aws_secret_access_key = xxxxx
aws_session_token = IQoJb3JpZ2luX2Vj....
```

That’s it. ✅

---

### **Step 2: Create the PowerShell script**

Save this as `C:\scripts\s3sync.ps1`:

```powershell
# Ensure log directory exists
$LogDir = "C:\Logs"
if (!(Test-Path $LogDir)) {
    New-Item -ItemType Directory -Path $LogDir | Out-Null
}

# Date format: YYYYMMDD_HHmmss
$DateTime = (Get-Date).ToString("yyyyMMdd_HHmmss")
$LogFile  = Join-Path $LogDir "s3sync_$DateTime.log"

# Write header
"Starting sync at $DateTime" | Out-File -FilePath $LogFile -Encoding utf8

# Run sync with profile and log output
aws s3 sync "C:\Data\Reports" "s3://my-company-backups/reports/" --profile mfa-session *>> $LogFile

# Write footer
"Finished sync at $DateTime" | Out-File -FilePath $LogFile -Append -Encoding utf8
```

---
#### Before Executing the steps task
<img width="1200" height="490" alt="image" src="https://github.com/user-attachments/assets/2ee18984-e253-4944-b6e4-ce3fe6769bf8" />

### **Step 3: Schedule the Task**

Run this in PowerShell **as Administrator**:

```powershell
$Action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -ExecutionPolicy Bypass -File `"`"C:\scripts\s3sync.ps1`"`""
$Trigger = New-ScheduledTaskTrigger -Daily -At 05:32
$Principal = New-ScheduledTaskPrincipal -UserId "$env:USERDOMAIN\$env:USERNAME" -LogonType Interactive
Register-ScheduledTask -TaskName "S3DailySync" -Action $Action -Trigger $Trigger -Principal $Principal -Description "Daily sync C:\Data\Reports to S3"
```

---

### **Step 4: Verify**

* Check task info:

  ```powershell
  Get-ScheduledTaskInfo -TaskName "S3DailySync"
  ```
* Check logs:

  ```powershell
  Get-Content (Get-ChildItem "C:\Logs\s3sync_*.log" | Sort-Object LastWriteTime -Descending | Select-Object -First 1)
  ```

---

#### After Executing the steps
<img width="1186" height="483" alt="image" src="https://github.com/user-attachments/assets/ce08d0fe-f4cc-454b-858a-63fea8f4ed3e" />


✅ That’s the **simplest setup**:

* Profile is stored once (`mfa-session`).
* Script always runs with `--profile mfa-session`.
* No environment variables, no exporting.

---

