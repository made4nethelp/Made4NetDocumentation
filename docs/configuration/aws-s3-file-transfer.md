# Moving Magento Export Files to AWS S3

## What we were trying to achieve

Originally, the client files were transferred using FTP and WinSCP. The client later asked us to stop using the FTP location and send the files directly to an AWS S3 bucket.

The idea was to make the process completely automatic.

No one should have to:

- Log in to the AWS portal
- Open the S3 bucket manually
- Select files
- Upload files one by one

The Windows server should automatically find the files, copy them to S3, rename the local files after a successful transfer, write logs, and clean up older local files.

The complete flow looks like this:

```text
Local export folders
        |
        v
MagentoAwsFileTransfers.bat
        |
        v
MagentoAwsFileTransfers.ps1
        |
        v
AWS CLI
        |
        v
AWS S3 bucket
```

Windows Task Scheduler runs the `.bat` file automatically at the scheduled time.

---

## Folder Structure

We kept the scripts, logs, and export files in separate locations.

A sample local folder structure looks like this:

```text
C:\
│
├── Scripts\
│   ├── MagentoAwsFileTransfers.bat
│   ├── MagentoAwsFileTransfers.ps1
│   │
│   └── Logs\
│       ├── MagentoAws-transfer-2026-06.log
│       ├── MagentoAws-transfer-2026-07.log
│       └── MagentoAwsFileTransfers-batch.log
│
├── Exports\
│   ├── Snapshot20260701.xml
│   ├── Snapshot20260702.csv
│   └── Snapshot20260703_copied_to_s3.xml
│
└── MAGENTOEXPORT\
    └── putaway\
        ├── Putaway20260701.xml
        ├── Putaway20260702.csv
        └── Putaway20260703_copied_to_s3.csv
```

The main script files are stored here:

```text
C:\Scripts\MagentoAwsFileTransfers.bat
C:\Scripts\MagentoAwsFileTransfers.ps1
```

The monthly logs are stored here:

```text
C:\Scripts\Logs
```

The first local source folder is:

```text
C:\Exports
```

The second local source folder is:

```text
C:\MAGENTOEXPORT\putaway
```

---

## S3 Folder Structure

The S3 bucket used for this process was:

```text
s3://wms-snapshot/
```

The expected structure inside the bucket is similar to this:

```text
s3://wms-snapshot/
└── inventory/
    └── us/
        ├── magentoexport/
        │   └── snapshot/
        │       ├── Snapshot20260701.xml
        │       └── Snapshot20260702.csv
        │
        └── putaway/
            ├── Putaway20260701.xml
            └── Putaway20260702.csv
```

Files from:

```text
C:\Exports
```

are copied to:

```text
s3://wms-snapshot/inventory/us/magentoexport/snapshot/
```

Files from:

```text
C:\MAGENTOEXPORT\putaway
```

are copied to:

```text
s3://wms-snapshot/inventory/us/putaway/
```

Make sure the spelling in the script exactly matches the required S3 path.

For example, these are different paths:

```text
magentoexport
megentoexport
```

S3 treats them as different folder prefixes.

---

## Step 1: Install AWS CLI on the Windows Server

The PowerShell script uses AWS CLI commands such as:

```powershell
aws s3 cp
```

and:

```powershell
aws s3 ls
```

Because of that, AWS CLI must be installed on the same Windows server where the scheduled task runs.

Create a temporary folder if it does not already exist:

```powershell
New-Item -Path "C:\Temp" -ItemType Directory -Force
```

Download the AWS CLI installer:

```powershell
Invoke-WebRequest `
    "https://awscli.amazonaws.com/AWSCLIV2.msi" `
    -OutFile "C:\Temp\AWSCLIV2.msi"
```

Install it:

```powershell
Start-Process msiexec.exe `
    -ArgumentList '/i "C:\Temp\AWSCLIV2.msi" /quiet' `
    -Wait
```

After installation, close PowerShell and open it again.

Then check the installation:

```powershell
aws --version
```

You should see output similar to:

```text
aws-cli/2.x.x Python/3.x Windows/Server
```

When this error appears:

```text
aws is not recognized as the name of a cmdlet
```

it normally means one of these things:

- AWS CLI is not installed
- PowerShell was not reopened after installation
- The AWS CLI installation folder is not available in the system PATH

---

## Step 2: Configure AWS Access

AWS CLI needs permission to access the S3 bucket.

Run:

```powershell
aws configure
```

It asks for:

```text
AWS Access Key ID
AWS Secret Access Key
Default region name
Default output format
```

Example:

```text
AWS Access Key ID:     ***************
AWS Secret Access Key: ***************
Default region name:   us-east-1
Default output format: json
```

The AWS access key must belong to a user or role that has permission to:

```text
List the bucket
Upload files
Read files, if required
Delete files, if required
```

After configuring it, confirm which AWS identity is being used:

```powershell
aws sts get-caller-identity
```

The command should return account and user or role information.

### Better option for AWS servers

When the Windows server is an AWS EC2 server, the better option is to attach an IAM role to the EC2 instance.

With an EC2 IAM role:

- Access keys do not need to be stored in the script
- Access keys do not need to be stored in the `.bat` file
- AWS CLI automatically receives temporary credentials
- Credential rotation is handled by AWS

When an IAM role is available, we do not need to run:

```powershell
aws configure
```

The script can directly use the role assigned to the server.

---

## Step 3: Confirm Access to the S3 Bucket

Before working on the script, we tested the connection manually.

To list the top level of the bucket:

```powershell
aws s3 ls s3://wms-snapshot/
```

Example output:

```text
                           PRE inventory/
2025-06-24 16:53:54   15475631 Snapshot20250607.xml
```

To list everything inside the bucket recursively:

```powershell
aws s3 ls s3://wms-snapshot/ --recursive
```

This is useful because S3 sometimes appears to have repeated folder names.

For example:

```text
inventory/inventory/file.xml
```

may exist when a file was accidentally uploaded to:

```text
s3://wms-snapshot/inventory/inventory/
```

To check the snapshot location:

```powershell
aws s3 ls s3://wms-snapshot/inventory/us/magentoexport/snapshot/
```

To check the putaway location:

```powershell
aws s3 ls s3://wms-snapshot/inventory/us/putaway/
```

Always add the final slash when checking an S3 folder-like path:

```powershell
aws s3 ls s3://wms-snapshot/inventory/us/
```

---

## Step 4: Create S3 Folder Paths

S3 does not really have folders in the same way Windows does.

An S3 folder is only part of an object name called a key or prefix.

For example:

```text
inventory/us/magentoexport/snapshot/file.xml
```

The folder-like part is:

```text
inventory/us/magentoexport/snapshot/
```

Normally, there is no need to create the path first. Uploading a file automatically creates the path.

But when we need an empty folder-like path, we can run:

```powershell
aws s3api put-object `
    --bucket wms-snapshot `
    --key inventory/us/magentoexport/snapshot/
```

To create the putaway path:

```powershell
aws s3api put-object `
    --bucket wms-snapshot `
    --key inventory/us/putaway/
```

---

## Step 5: Create the PowerShell Script

Create this file:

```text
C:\Scripts\MagentoAwsFileTransfers.ps1
```

This PowerShell script does the main work.

It:

- Looks for `.xml` and `.csv` files
- Ignores files already ending with `_copied_to_s3`
- Copies eligible files to S3
- Renames local files after a successful upload
- Writes a monthly log
- Counts successful and failed transfers
- Deletes copied local files older than seven days

```powershell
# MagentoAwsFileTransfers.ps1

# ============================================================
# Configuration
# ============================================================

$bucketName = "wms-snapshot"

$exportSourceFolder = "C:\Exports"
$exportDestination = "inventory/us/magentoexport/snapshot/"

$putawaySourceFolder = "C:\MAGENTOEXPORT\putaway"
$putawayDestination = "inventory/us/putaway/"

$logDirectory = "C:\Scripts\Logs"
$retentionDays = 7

# ============================================================
# Monthly Logging
# ============================================================

$logMonth = (Get-Date).ToString("yyyy-MM")

$logFile = Join-Path `
    -Path $logDirectory `
    -ChildPath "MagentoAws-transfer-$logMonth.log"

if (-not (Test-Path -LiteralPath $logDirectory)) {
    New-Item `
        -Path $logDirectory `
        -ItemType Directory `
        -Force | Out-Null
}

function Write-Log {
    param (
        [Parameter(Mandatory = $true)]
        [string]$Message
    )

    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logEntry = "$timestamp  $Message"

    Add-Content `
        -LiteralPath $logFile `
        -Value $logEntry `
        -Encoding UTF8

    Write-Host $logEntry
}

# ============================================================
# Copy Files to S3
# ============================================================

function Copy-FilesToS3 {
    param (
        [Parameter(Mandatory = $true)]
        [string]$SourceFolder,

        [Parameter(Mandatory = $true)]
        [string]$DestinationPrefix
    )

    $result = [PSCustomObject]@{
        Found   = 0
        Copied  = 0
        Failed  = 0
        Renamed = 0
    }

    Write-Log "Checking source folder: $SourceFolder"

    if (-not (Test-Path -LiteralPath $SourceFolder)) {
        Write-Log "ERROR: Source folder does not exist: $SourceFolder"
        return $result
    }

    $files = Get-ChildItem `
        -LiteralPath $SourceFolder `
        -File `
        -ErrorAction Stop |
        Where-Object {
            $_.Extension.ToLowerInvariant() -in @(".xml", ".csv") -and
            $_.BaseName -notmatch "_copied_to_s3(?:_\d+)?$"
        }

    $result.Found = @($files).Count

    Write-Log "S3 destination: s3://$bucketName/$DestinationPrefix"
    Write-Log "Eligible files found: $($result.Found)"

    if ($result.Found -eq 0) {
        Write-Log "No new XML or CSV files were found in $SourceFolder"
        return $result
    }

    foreach ($file in $files) {
        $s3Key = "$DestinationPrefix$($file.Name)"
        $s3Location = "s3://$bucketName/$s3Key"

        Write-Log "Copying file: $($file.FullName)"
        Write-Log "Copying to: $s3Location"

        $awsOutput = & aws s3 cp `
            $file.FullName `
            $s3Location `
            --only-show-errors 2>&1

        $awsExitCode = $LASTEXITCODE

        if ($awsExitCode -ne 0) {
            $result.Failed++

            Write-Log "ERROR: Failed to copy $($file.Name)"
            Write-Log "AWS exit code: $awsExitCode"

            if ($awsOutput) {
                Write-Log "AWS response: $($awsOutput -join ' ')"
            }

            continue
        }

        $result.Copied++

        Write-Log "SUCCESS: Copied $($file.Name) to S3"

        $newFileName = "{0}_copied_to_s3{1}" -f `
            $file.BaseName,
            $file.Extension

        $newFilePath = Join-Path `
            -Path $file.DirectoryName `
            -ChildPath $newFileName

        if (Test-Path -LiteralPath $newFilePath) {
            $counter = 1

            do {
                $newFileName = "{0}_copied_to_s3_{1}{2}" -f `
                    $file.BaseName,
                    $counter,
                    $file.Extension

                $newFilePath = Join-Path `
                    -Path $file.DirectoryName `
                    -ChildPath $newFileName

                $counter++
            }
            while (Test-Path -LiteralPath $newFilePath)
        }

        try {
            Rename-Item `
                -LiteralPath $file.FullName `
                -NewName $newFileName `
                -ErrorAction Stop

            $result.Renamed++

            Write-Log "SUCCESS: Renamed local file to $newFileName"
        }
        catch {
            Write-Log "WARNING: File was copied to S3, but local rename failed."
            Write-Log "Rename error: $($_.Exception.Message)"
        }
    }

    return $result
}

# ============================================================
# Delete Old Local Copied Files
# ============================================================

function Remove-OldCopiedLocalFiles {
    param (
        [Parameter(Mandatory = $true)]
        [string]$FolderPath,

        [int]$Days = 7
    )

    $result = [PSCustomObject]@{
        Deleted = 0
        Failed  = 0
    }

    if (-not (Test-Path -LiteralPath $FolderPath)) {
        Write-Log "WARNING: Cleanup folder does not exist: $FolderPath"
        return $result
    }

    $cutoffDate = (Get-Date).AddDays(-$Days)

    Write-Log "Checking $FolderPath for copied files older than $Days days"
    Write-Log "Cleanup cutoff date: $cutoffDate"

    $oldFiles = Get-ChildItem `
        -LiteralPath $FolderPath `
        -File |
        Where-Object {
            $_.Extension.ToLowerInvariant() -in @(".xml", ".csv") -and
            $_.BaseName -match "_copied_to_s3(?:_\d+)?$" -and
            $_.LastWriteTime -lt $cutoffDate
        }

    if (@($oldFiles).Count -eq 0) {
        Write-Log "No copied files older than $Days days were found in $FolderPath"
        return $result
    }

    foreach ($file in $oldFiles) {
        try {
            Remove-Item `
                -LiteralPath $file.FullName `
                -Force `
                -ErrorAction Stop

            $result.Deleted++

            Write-Log "DELETED: $($file.FullName)"
            Write-Log "Last modified date: $($file.LastWriteTime)"
        }
        catch {
            $result.Failed++

            Write-Log "ERROR: Failed to delete $($file.FullName)"
            Write-Log "Delete error: $($_.Exception.Message)"
        }
    }

    return $result
}

# ============================================================
# Main Process
# ============================================================

Write-Log "============================================================"
Write-Log "Magento AWS file transfer started"
Write-Log "Server: $env:COMPUTERNAME"
Write-Log "Windows account: $env:USERDOMAIN\$env:USERNAME"

if (-not (Get-Command aws -ErrorAction SilentlyContinue)) {
    Write-Log "ERROR: AWS CLI is not installed or is not available in PATH"
    exit 1
}

Write-Log "AWS CLI was found"

$identityOutput = & aws sts get-caller-identity --output json 2>&1
$identityExitCode = $LASTEXITCODE

if ($identityExitCode -ne 0) {
    Write-Log "ERROR: AWS authentication failed"
    Write-Log "AWS response: $($identityOutput -join ' ')"
    exit 1
}

Write-Log "AWS authentication was successful"

$exportResult = Copy-FilesToS3 `
    -SourceFolder $exportSourceFolder `
    -DestinationPrefix $exportDestination

$putawayResult = Copy-FilesToS3 `
    -SourceFolder $putawaySourceFolder `
    -DestinationPrefix $putawayDestination

$exportCleanup = Remove-OldCopiedLocalFiles `
    -FolderPath $exportSourceFolder `
    -Days $retentionDays

$putawayCleanup = Remove-OldCopiedLocalFiles `
    -FolderPath $putawaySourceFolder `
    -Days $retentionDays

$totalFound = $exportResult.Found + $putawayResult.Found
$totalCopied = $exportResult.Copied + $putawayResult.Copied
$totalFailed = $exportResult.Failed + $putawayResult.Failed
$totalRenamed = $exportResult.Renamed + $putawayResult.Renamed
$totalDeleted = $exportCleanup.Deleted + $putawayCleanup.Deleted
$totalDeleteFailed = $exportCleanup.Failed + $putawayCleanup.Failed

Write-Log "------------------------------------------------------------"
Write-Log "Transfer summary"
Write-Log "Files found: $totalFound"
Write-Log "Files copied to S3: $totalCopied"
Write-Log "Copy failures: $totalFailed"
Write-Log "Local files renamed: $totalRenamed"
Write-Log "Old local copied files deleted: $totalDeleted"
Write-Log "Local cleanup failures: $totalDeleteFailed"
Write-Log "Magento AWS file transfer completed"
Write-Log "============================================================"

if ($totalFailed -gt 0 -or $totalDeleteFailed -gt 0) {
    exit 1
}

exit 0
```

---

## How the File Rename Works

Suppose the local source folder contains:

```text
Snapshot20260720.xml
```

After the file is copied successfully to S3, the local file becomes:

```text
Snapshot20260720_copied_to_s3.xml
```

For a CSV file:

```text
Snapshot20260720.csv
```

it becomes:

```text
Snapshot20260720_copied_to_s3.csv
```

The next time the script runs, it ignores files ending with:

```text
_copied_to_s3
```

This prevents the same file from being copied again.

The S3 file keeps its original name:

```text
Snapshot20260720.xml
```

Only the local file is renamed.

---

## Step 6: Create the Batch File

Create this file:

```text
C:\Scripts\MagentoAwsFileTransfers.bat
```

```bat
@echo off
setlocal

set "SCRIPT_PATH=C:\Scripts\MagentoAwsFileTransfers.ps1"
set "BAT_LOG=C:\Scripts\Logs\MagentoAwsFileTransfers-batch.log"

echo ============================================================ >> "%BAT_LOG%"
echo [%date% %time%] Magento AWS batch process started >> "%BAT_LOG%"

powershell.exe ^
  -NoProfile ^
  -NonInteractive ^
  -ExecutionPolicy Bypass ^
  -File "%SCRIPT_PATH%" >> "%BAT_LOG%" 2>&1

set "EXIT_CODE=%ERRORLEVEL%"

echo [%date% %time%] PowerShell exit code: %EXIT_CODE% >> "%BAT_LOG%"
echo [%date% %time%] Magento AWS batch process completed >> "%BAT_LOG%"
echo ============================================================ >> "%BAT_LOG%"

endlocal & exit /b %EXIT_CODE%
```

The batch file:

1. Starts PowerShell
2. Runs `MagentoAwsFileTransfers.ps1`
3. Writes console messages to a batch log
4. Returns the PowerShell exit code to Task Scheduler

---

## Step 7: Test Everything Manually

Before creating the scheduled task, test each part separately.

### Test AWS CLI

```powershell
aws --version
```

### Test AWS identity

```powershell
aws sts get-caller-identity
```

### Test bucket access

```powershell
aws s3 ls s3://wms-snapshot/
```

### Test one file manually

```powershell
aws s3 cp `
    "C:\Exports\TestSnapshot.xml" `
    "s3://wms-snapshot/inventory/us/magentoexport/snapshot/TestSnapshot.xml"
```

### Verify the uploaded file

```powershell
aws s3 ls `
    "s3://wms-snapshot/inventory/us/magentoexport/snapshot/" `
    --recursive
```

### Test the PowerShell script

```powershell
powershell.exe `
    -NoProfile `
    -ExecutionPolicy Bypass `
    -File "C:\Scripts\MagentoAwsFileTransfers.ps1"
```

### Test the batch file

```cmd
C:\Scripts\MagentoAwsFileTransfers.bat
```

Only after the PowerShell and batch files both work manually should the scheduled task be created.

---

## Step 8: Schedule the Batch File in Windows Task Scheduler

Open Windows Task Scheduler and select:

```text
Create Task
```

### General tab

Use a name such as:

```text
Magento AWS File Transfers
```

Select:

```text
Run whether user is logged on or not
Run with highest privileges
```

The selected account must have access to:

```text
C:\Scripts
C:\Scripts\Logs
C:\Exports
C:\MAGENTOEXPORT\putaway
```

The same account must also have AWS access.

### Triggers tab

Create the required schedule.

Example:

```text
Daily at 6:30 AM
```

Make sure:

```text
Enabled
```

is checked.

### Actions tab

Set:

```text
Program/script:
C:\Windows\System32\cmd.exe
```

```text
Add arguments:
/c "C:\Scripts\MagentoAwsFileTransfers.bat"
```

```text
Start in:
C:\Scripts
```

### Conditions tab

For a server task, uncheck conditions that may prevent execution:

```text
Start the task only if the computer is idle
Start the task only if the computer is on AC power
Stop if the computer switches to battery power
```

### Settings tab

Enable:

```text
Allow task to be run on demand
Run task as soon as possible after a scheduled start is missed
```

For failures:

```text
Restart every 5 minutes
Attempt to restart up to 3 times
```

For overlapping runs:

```text
Do not start a new instance
```

---

## Step 9: Test the Scheduled Task

After saving the task:

1. Right-click the task
2. Select **Run**
3. Wait for the task to finish
4. Refresh Task Scheduler
5. Check the last run result
6. Check the logs
7. Check the S3 destination
8. Check the renamed local files

A successful task normally shows:

```text
The operation completed successfully. (0x0)
```

When Task Scheduler shows:

```text
The task has not yet run. (0x41303)
```

check:

- The trigger is enabled
- The scheduled time is correct
- Idle and power conditions are disabled
- The task account has the correct permissions
- The task account password is current
- The batch file path is correct
- The `Start in` folder is configured

Enable task history through:

```text
Task Scheduler
→ Action
→ Enable All Tasks History
```

---

## Monthly Logging

The log file name changes every month.

Example:

```text
C:\Scripts\Logs\MagentoAws-transfer-2026-07.log
```

The script appends new entries instead of overwriting previous entries.

Example:

```text
2026-07-20 06:30:00  Magento AWS file transfer started
2026-07-20 06:30:01  AWS authentication was successful
2026-07-20 06:30:01  Eligible files found: 2
2026-07-20 06:30:02  SUCCESS: Copied Snapshot20260720.xml to S3
2026-07-20 06:30:02  SUCCESS: Renamed local file to Snapshot20260720_copied_to_s3.xml
2026-07-20 06:30:03  Files copied to S3: 2
2026-07-20 06:30:03  Copy failures: 0
2026-07-20 06:30:03  Magento AWS file transfer completed
```

---

## Local File Cleanup

After a successful copy, the local file remains with the copied flag:

```text
Snapshot20260720_copied_to_s3.xml
```

The cleanup function deletes only copied `.xml` and `.csv` files older than the retention period.

The default retention is:

```powershell
$retentionDays = 7
```

New files that have not yet been copied are not deleted.

---

## Useful AWS Commands

### List all files

```powershell
aws s3 ls s3://wms-snapshot/ --recursive
```

### List snapshot files

```powershell
aws s3 ls `
    s3://wms-snapshot/inventory/us/magentoexport/snapshot/ `
    --recursive
```

### List putaway files

```powershell
aws s3 ls `
    s3://wms-snapshot/inventory/us/putaway/ `
    --recursive
```

### Delete one incorrect S3 file

```powershell
aws s3 rm `
    "s3://wms-snapshot/inventory/us/magentoexport/snapshot/IncorrectFile.xml"
```

### Preview deleting a complete prefix

```powershell
aws s3 rm `
    "s3://wms-snapshot/inventory/us/unwanted-folder/" `
    --recursive `
    --dryrun
```

### Delete a complete prefix

```powershell
aws s3 rm `
    "s3://wms-snapshot/inventory/us/unwanted-folder/" `
    --recursive
```

---

## Full Process Summary

```text
1. Export files are created locally.

2. Files are placed in:
   C:\Exports
   or
   C:\MAGENTOEXPORT\putaway

3. Windows Task Scheduler starts:
   C:\Scripts\MagentoAwsFileTransfers.bat

4. The batch file starts:
   C:\Scripts\MagentoAwsFileTransfers.ps1

5. PowerShell checks:
   - AWS CLI is installed
   - AWS authentication works
   - Source folders exist

6. PowerShell finds new XML and CSV files.

7. Files already ending with _copied_to_s3 are ignored.

8. Snapshot files are copied to:
   s3://wms-snapshot/inventory/us/magentoexport/snapshot/

9. Putaway files are copied to:
   s3://wms-snapshot/inventory/us/putaway/

10. After a successful copy, the local file is renamed:
    filename.xml
    becomes
    filename_copied_to_s3.xml

11. Every action is added to the current monthly log.

12. Local copied files older than seven days are deleted.

13. The script writes totals for:
    - Files found
    - Files copied
    - Failures
    - Files renamed
    - Old local files deleted

14. Task Scheduler receives the final success or failure result.
```

The result is a fully automated file-transfer process with:

```text
No manual AWS login
No manual upload
No duplicate copying of completed files
Monthly transfer logs
Automatic local cleanup
Task Scheduler monitoring
```
