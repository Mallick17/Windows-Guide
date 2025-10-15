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

<details>
    <summary>Click to view Explaination of PowerShell Commands</summary>

### Overview
These PowerShell commands use the `ScheduledTasks` module (available in Windows PowerShell 3.0+ and PowerShell Core) to create and register a scheduled task in Windows Task Scheduler. The task is named "S3EveryMinuteSync" and is designed to run the `s3sync.ps1` script (from your previous query) every minute, starting 1 minute from when the commands are executed. It runs under the current user's interactive session.

Key assumptions:
- You're running this in an elevated PowerShell session (as admin) to register tasks.
- The script `C:\Users\Testing\scripts\s3sync.ps1` exists and is the one you described earlier (syncing to S3).
- The task will run indefinitely (repeating forever) but can be modified later (e.g., to limit to 3 days as noted in the comment).

After running these, you can view/manage the task in Task Scheduler GUI (search for "Task Scheduler" in Start menu) under the "Task Scheduler Library".

### Step-by-Step Breakdown
I'll explain each command line by line, including syntax, parameters, and behavior. Note the backticks (`` ` ``) in the trigger for line continuation—common in PowerShell for readability.

#### 1. Define the Task Action
```powershell
$Action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -ExecutionPolicy Bypass -File "C:\Users\Testing\scripts\s3sync.ps1""
```
- **What it does**:
  - `New-ScheduledTaskAction`: Creates an action object that defines *what* the task runs.
  - `-Execute "powershell.exe"`: Specifies the executable to launch (PowerShell host).
  - `-Argument`: Passes command-line arguments to PowerShell:
    - `-NoProfile`: Skips loading user profiles (faster startup, avoids customizations that might interfere).
    - `-ExecutionPolicy Bypass`: Overrides execution policy restrictions (allows unsigned scripts like `s3sync.ps1` to run without warnings).
    - `-File "C:\Users\Testing\scripts\s3sync.ps1"`: Tells PowerShell to execute this specific script file (full path required).
  - Stores the result in `$Action` (a `CimInstance` object of type `MSFT_TaskExecAction`).
- **Why?** Encapsulates how to invoke your S3 sync script reliably.
- **What it won't do**: Validate if the script exists (fails at runtime if missing). No working directory set (defaults to system32; consider adding `-WorkingDirectory` if needed). The double quote at the end has an extra quote—likely a typo; it should be just one closing quote after the path.

#### 2. Define the Task Trigger
```powershell
$Trigger = New-ScheduledTaskTrigger -Once -At (Get-Date).Date.AddMinutes(1) `
    -RepetitionInterval (New-TimeSpan -Minutes 1) `
    -RepetitionDuration ([TimeSpan]::MaxValue)
```
- **What it does**:
  - `New-ScheduledTaskTrigger`: Creates a trigger object that defines *when* the task runs.
  - `-Once`: Sets a one-time initial trigger (not daily/weekly).
  - `-At (Get-Date).Date.AddMinutes(1)`: Calculates the start time:
    - `(Get-Date).Date`: Gets today's date at midnight (e.g., 2025-10-15 00:00:00).
    - `.AddMinutes(1)`: Adds 1 minute (e.g., starts at 00:01:00 today). If run mid-day, it floors to midnight +1 min, so adjust if you want "now +1 min" via `(Get-Date).AddMinutes(1)`.
  - `-RepetitionInterval (New-TimeSpan -Minutes 1)`: Repeats the action every 1 minute after the initial start.
  - `-RepetitionDuration ([TimeSpan]::MaxValue)`: Repeats indefinitely (`MaxValue` is ~10,000 years; effectively forever).
  - Backticks allow multi-line for readability.
  - Stores in `$Trigger` (a `CimInstance` of type `MSFT_TaskTimeTrigger`).
- **Why?** Enables frequent, automated execution without manual intervention.
- **What it won't do**: Account for time zones (uses local time). No randomization or conditions (e.g., only on idle). If the system time changes (e.g., DST), it might drift slightly.

#### 3. Define the Task Principal (Security Context)
```powershell
$Principal = New-ScheduledTaskPrincipal -UserId "$env:USERDOMAIN\$env:USERNAME" -LogonType Interactive
```
- **What it does**:
  - `New-ScheduledTaskPrincipal`: Creates a principal object for *who* runs the task (security/permissions).
  - `-UserId "$env:USERDOMAIN\$env:USERNAME"`: Uses the current user's domain-qualified name (e.g., "WORKGROUP\Testing"). `$env:USERDOMAIN` and `$env:USERNAME` are environment variables.
  - `-LogonType Interactive`: Requires the user to be logged in interactively (task won't run in background/service mode; uses the user's session).
  - Stores in `$Principal` (a `CimInstance` of type `MSFT_TaskPrincipal`).
- **Why?** Ensures the task runs with the current user's permissions (e.g., access to AWS profiles, local files).
- **What it won't do**: Prompt for password (assumes cached creds). If you need it to run when logged off, use `-LogonType S4U` or `ServiceAccount`. No group policy overrides.

#### 4. Register the Scheduled Task
```powershell
Register-ScheduledTask -TaskName "S3EveryMinuteSync" -Action $Action -Trigger $Trigger -Principal $Principal -Description "Sync C:\Data\Reports to S3 every minute"
```
- **What it does**:
  - `Register-ScheduledTask`: Creates and registers the task in the Task Scheduler store (root folder by default).
  - `-TaskName "S3EveryMinuteSync"`: Unique name for the task (visible in Task Scheduler).
  - `-Action $Action`: Attaches the previously defined action (run the PS script).
  - `-Trigger $Trigger`: Attaches the trigger (start in 1 min, repeat every min forever).
  - `-Principal $Principal`: Attaches the security context (run as current interactive user).
  - `-Description`: Adds a human-readable note (shown in Task Scheduler UI).
  - Returns a `CimInstance` of the registered task (e.g., for further queries).
- **Why?** Combines all pieces into a live, scheduled task.
- **What it won't do**: Overwrite an existing task with the same name (fails with error; use `-Force` to overwrite). No settings for power (e.g., wake on LAN) or conditions (e.g., only on AC power). The description mentions "C:\Data\Reports", but your script uses "C:\Users\Testing\scripts\Data\Reports"—minor mismatch, but doesn't affect functionality.

#### 5. Snippet for Later Modification (Not Executed Here)
```powershell
$RepetitionDuration = New-TimeSpan -Days 3 ## --> We can change at the later stage anytime.
```
- **What it does**: This is *not* part of the task creation—it's a standalone variable assignment. `New-TimeSpan -Days 3` creates a timespan of 3 days (e.g., `3.00:00:00`).
- **Why?** As per the comment, it's a placeholder to limit repetitions later. To apply it, you'd unregister the task (`Unregister-ScheduledTask -TaskName "S3EveryMinuteSync" -Confirm:$false`), recreate the trigger with `-RepetitionDuration $RepetitionDuration`, and re-register.
- **What it won't do**: Anything on its own—it's just prep for editing. Note: The code has a typo (`New-ScheduledTaskTrigger -Days 3` in your message, but I assume it's `New-TimeSpan` based on context).

### Overall Behavior
- **Execution flow**: Run these commands → Task registers → Waits ~1 min → Runs `s3sync.ps1` → Repeats every 1 min forever (or until stopped).
- **Monitoring**: Check status via `Get-ScheduledTask -TaskName "S3EveryMinuteSync" | Get-ScheduledTaskInfo` (shows last run time, result code). History in Task Scheduler UI under the task's "History" tab.
- **Stopping/Editing**: `Disable-ScheduledTask -TaskName "S3EveryMinuteSync"` to pause; `Unregister-ScheduledTask` to delete. For 3-day limit: Modify trigger as noted, re-register.
- **Exit codes**: Script runs exit 0 (success) or non-zero (e.g., AWS error); Task Scheduler logs these.

### Limitations and Recommendations
- **Performance**: Every-minute syncs could be resource-intensive (CPU/network for AWS calls). Start with longer intervals (e.g., 5-15 min) via `-RepetitionInterval (New-TimeSpan -Minutes 5)`.
- **Reliability**: If script takes >1 min, overlaps occur (no built-in queuing). Use `-Settings` in `Register-ScheduledTask` for `RestartInterval` or `MultipleInstances Policy=Queue`.
- **Security**: Interactive logon means it stops if user logs off. For always-on, switch to a service account.
- **Testing**: Run `Start-ScheduledTask -TaskName "S3EveryMinuteSync"` manually first. Monitor S3 costs (frequent uploads).
- **Improvements**: Add `-Force` to `Register-ScheduledTask` for idempotency. For production, wrap in a function or use XML exports for versioning.

This setup turns your script into a robust, repeating job—great for continuous syncing! 
  
</details>

- What it will do: Automate the S3 sync script to execute repeatedly every minute, mimicking a "cron" job on Windows. This is useful for near-real-time backups or monitoring, but be cautious—frequent runs could incur AWS costs or overload the system/network.
- What it will not do: Handle task failures (e.g., no retries or alerts), validate the script path beforehand, or clean up on uninstall. It won't run if the user is logged off (due to Interactive logon type) or if the machine is off. The last line appears to be a snippet for later modification but isn't part of the task creation.

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
