# Windows-Guide
## Install AWS CLI in Windows
Find it in AWS documentation.

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

## Windows Cron Jobs using Task Scheduler to backup In Daily basis to S3 Bucket

<details>
  <summary>Using Task Scheduler to Backup In Daily basis to S3 Bucket, Click to view the steps</summary>

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

### Go to task scheduler GUI
- then Go to Task Scheduler Library
- Find S3DailySync and CLick on it
- In the left hand side you will find `Run` , `End` , `Disable` and other options.
- Added some files in the source local folder in the system
- Then clicked on Run
- Followed Step 4 from the above task
* Check task info:

  ```powershell
  Get-ScheduledTaskInfo -TaskName "S3DailySync"
  ```
* Check logs:

  ```powershell
  Get-Content (Get-ChildItem "C:\Logs\s3sync_*.log" | Sort-Object LastWriteTime -Descending | Select-Object -First 1)
  ```
- New task has run and the output of the above command looks like
```powershell
PS C:\Users\Mallick\.aws> Get-ScheduledTaskInfo -TaskName "S3DailySync"
LastRunTime        : 21-08-2025 16:24:15
LastTaskResult     : 0
NextRunTime        : 22-08-2025 16:13:00
NumberOfMissedRuns : 0
TaskName           : S3DailySync
TaskPath           :
PSComputerName     :
```

- In the S3 bucket folder
<img width="1200" height="572" alt="image" src="https://github.com/user-attachments/assets/bc990648-c88a-4be0-8b53-481466bd6c9c" />

</details>

---

## Using Task Scheduler to Backup every minute to S3 Bucket
- Tested Every Minute with Multiple Folders
- Tested with Restarting the system multiple times.
- Tested with shuting down the system multiple times.

<details>
  <summary>Click to view the configuration</summary>

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
[default]
aws_access_key_id = ASIAxxxx
aws_secret_access_key = xxxxx

[mfa-session]
aws_access_key_id = ASIAxxxx ## --> Generated from MFA Session for CLI
aws_secret_access_key = xxxxx ## --> Generated from MFA Session for CLI
aws_session_token = IQoJb3JpZ2luX2Vj.... ## --> Generated from MFA Session for CLI
```

That’s it. ✅

### **Step 2: Create the PowerShell script**
- Save this as `C:\scripts\s3sync.ps1`:

```powershell
# Ensure log directory exists
$LogDir = "C:\Logs"
if (!(Test-Path $LogDir)) {
    New-Item -ItemType Directory -Path $LogDir | Out-Null
}

# Define persistent log file
$LogFile = Join-Path $LogDir "s3sync.log"

# Add header with timestamp
$DateTime = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
Add-Content $LogFile "==== Sync started at $DateTime ===="

# Run sync and append output to the log file (single-line command)
aws s3 sync "C:\Users\Testing\scripts\Data\Reports" "s3://elasticbeanstalk-ap-south-1-508351649560/sync-commands-test/" --profile mfa-session 2>&1 | Add-Content $LogFile

# Add footer with timestamp
$EndTime = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
Add-Content $LogFile "==== Sync finished at $EndTime ===="
Add-Content $LogFile ""
```
- _What it will do:_ It will mirror the contents of a specific local folder to an S3 bucket (uploading new/changed files, skipping unchanged ones), while logging the entire process to a file for auditing. The sync is one-way (local to S3) and idempotent (safe to run multiple times).
- _What it will not do:_ It won't delete files (from local or S3), handle authentication failures gracefully (e.g., no retry on expired MFA tokens), send notifications on errors, or perform any pre/post-validation (e.g., checking disk space or internet connectivity). It's a simple, fire-and-forget operation focused solely on sync and logging.


### **Step 3: Schedule the Task**
Run this in PowerShell **as Administrator**:
- Repetation Duration 3 Days
```powershell
$Action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -ExecutionPolicy Bypass -File "C:\Users\Testing\scripts\s3sync.ps1""
$Trigger = New-ScheduledTaskTrigger -Once -At (Get-Date).Date.AddMinutes(1) ``
    -RepetitionInterval (New-TimeSpan -Minutes 1) ``
    -RepetitionDuration (New-TimeSpan -Days 3)
$Principal = New-ScheduledTaskPrincipal -UserId "$env:USERDOMAIN\$env:USERNAME" -LogonType Interactive
Register-ScheduledTask -TaskName "S3EveryMinuteSync" -Action $Action -Trigger $Trigger -Principal $Principal -Description "Sync C:\Data\Reports to S3 every minute"
```

- Repetation Duration Max
```powershell
$Action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -ExecutionPolicy Bypass -File "C:\Users\Testing\scripts\s3sync.ps1""
$Trigger = New-ScheduledTaskTrigger -Once -At (Get-Date).Date.AddMinutes(1) ``
    -RepetitionInterval (New-TimeSpan -Minutes 1) ``
    -RepetitionDuration ([TimeSpan]::MaxValue)
$Principal = New-ScheduledTaskPrincipal -UserId "$env:USERDOMAIN\$env:USERNAME" -LogonType Interactive
Register-ScheduledTask -TaskName "S3EveryMinuteSync" -Action $Action -Trigger $Trigger -Principal $Principal -Description "Sync C:\Data\Reports to S3 every minute"
$RepetitionDuration = New-TimeSpan -Days 3 ## --> We can change at the later stage anytime.
```

### **Step 4: Verify**

* Check task info:

  ```powershell
  Get-ScheduledTaskInfo -TaskName "S3DailySync"
  Get-ScheduledTask | Get-ScheduledTaskInfo | Select-Object TaskName, State, LastRunTime, NextRunTime
  Get-ScheduledTask -TaskName "S3EveryMinuteSync" | Get-ScheduledTaskInfo | Select-Object TaskName, State, LastRunTime, NextRunTime
  Get-ScheduledTask -TaskName "S3EveryMinuteSync" | Get-ScheduledTaskInfo | Select-Object TaskName, LastRunTime, LastTaskResult
  ```
* Check logs:

  ```powershell
  Get-Content (Get-ChildItem "C:\Logs\s3sync.log" | Sort-Object LastWriteTime -Descending | Select-Object -First 1)
  ```

---

### **Steps to Modify a Scheduled Task in Task Scheduler GUI**

1. **Open Task Scheduler**

   * Press **`Win + R`**
   * Type `taskschd.msc` and press **Enter**
   * This opens the **Task Scheduler** console.

2. **Locate Your Task**

   * In the **left pane**, expand:

     * **Task Scheduler Library**
     * Navigate through the scheduled tasks and find your tasks.

3. **Open Task Properties**

   * Right-click the task → Select **Properties** or After clicking your task in Task Scheduler Library, Check the Right Side Panel Under → Selected Item  → Select **Properties**
   * You’ll see multiple tabs for different settings.

4. **Modify Configuration as Needed**

   * **General Tab**

     * Change the task **name**, **description**, or **security options** (e.g., run only when user is logged on, highest privileges).
   * **Triggers Tab**

     * Click **Edit** to modify an existing trigger (like time, startup, logon, etc.)
     * Or click **New** to add another trigger.
   * **Actions Tab**

     * Edit the action (script or program to run).
     * Or add/remove actions as needed.
   * **Conditions Tab**

     * Configure conditions such as *“Start only if on AC power”* or *“Wake computer to run task.”*
   * **Settings Tab**

     * Adjust advanced settings like allowing the task to be run on demand, retry attempts, or stopping the task if it runs too long.

5. **Apply Changes**

   * Once you’ve made changes, click **OK**.
   * If prompted for credentials (when running under a different account), enter them.

6. **Test Your Task**

   * Right-click the task → Select **Run** to make sure it executes with the new configuration.
  
<details>
  <summary>Click to view the Images which was configured</summary>

- General Tab
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/85c580a2-a607-430c-ad16-e41132e199bc" />

- Triggers Tab
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/0e59e79e-e0dc-41e8-b72f-e302a03a32fc" />

- Actions Tab
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/eefd33d6-cfa6-4cc7-92a6-a319f1f02073" />

- Conditions Tab
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/8c5508b1-a1b5-428a-af89-8bb7fe0e118a" />

- Settings Tab
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/7df15510-44d1-4b49-9baa-65e5da631ff2" />

- History Tab
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/92d5542f-5c90-4e5c-84ba-6e10da9bcc6c" />
  
</details>

---

  
</details>
