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
