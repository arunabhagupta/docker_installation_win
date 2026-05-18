# Docker on Windows — Complete D Drive Installation Guide

> ✅ **Verified on a fresh Windows PC with Docker Desktop 4.20+**
> All steps are based on a real installation walkthrough with all errors encountered and fixed.

**Goal:** Install Docker Desktop and all dependencies entirely on D drive.
- All programs → `D:\ProgramFiles\`
- All Docker images, containers, volumes → `D:\WORK\Docker_Images\`
- Nothing significant on C drive

---

## Table of Contents

1. [System Requirements](#1-system-requirements)
2. [Enable BIOS Virtualization](#2-enable-bios-virtualization)
3. [Create D Drive Directories](#3-create-d-drive-directories)
4. [Enable WSL2 & Windows Features](#4-enable-wsl2--windows-features)
5. [Create .wslconfig](#5-create-wslconfig)
6. [Install Ubuntu WSL2 to D Drive](#6-install-ubuntu-wsl2-to-d-drive)
7. [Install Docker Desktop to D Drive](#7-install-docker-desktop-to-d-drive)
8. [Add Docker CLI to System PATH](#8-add-docker-cli-to-system-path)
9. [Launch Docker Desktop & Initialize](#9-launch-docker-desktop--initialize)
10. [Fix Docker Context Error](#10-fix-docker-context-error)
11. [Move Docker WSL Disk to D Drive](#11-move-docker-wsl-disk-to-d-drive)
12. [Update Docker Desktop Disk Image Location](#12-update-docker-desktop-disk-image-location)
13. [Verify Installation](#13-verify-installation)
14. [Final Path Summary](#14-final-path-summary)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. System Requirements

| Requirement | Detail |
|---|---|
| Windows Version | Windows 10 64-bit Build 19041 (20H1) or higher, or Windows 11 |
| RAM | 4 GB minimum — 8 GB or more recommended |
| Processor | 64-bit with SLAT support |
| BIOS | Hardware virtualization must be enabled (VT-x for Intel, AMD-V for AMD) |
| Hyper-V | Required for Windows Pro / Enterprise / Education only — not available on Windows Home |

### Check your Windows version

Open PowerShell as Administrator and run:

```powershell
Get-ComputerInfo -Property OsName, OsVersion, OsBuildNumber
```

Build number must be **19041 or higher** for Windows 10. Any Windows 11 build is fine.

---

## 2. Enable BIOS Virtualization

Docker uses WSL2 which requires hardware-level virtualization to be turned on in your motherboard BIOS. **Most new PCs have this OFF by default.**

1. Restart your PC and press the BIOS entry key during POST — typically `DEL`, `F2`, `F10`, or `F12` (shown briefly on screen and varies by motherboard brand)
2. Find the virtualization setting:
   - **Intel CPU:** Navigate to `Advanced` → `CPU Configuration` → Enable **Intel Virtualization Technology (VT-x)** and **Intel VT-d**
   - **AMD CPU:** Navigate to `Advanced` → Enable **SVM Mode** (also called AMD-V)
3. Save and Exit — your PC will reboot into Windows
4. After Windows loads, verify: open **Task Manager** → **Performance** → **CPU** → confirm **Virtualization: Enabled**

> ⚠️ If virtualization shows as Disabled after following the steps above, consult your motherboard manual. The exact menu location varies between manufacturers (ASUS, MSI, Gigabyte, ASRock, etc.).

---

## 3. Create D Drive Directories

Create all required folders on D drive before installing anything. This ensures paths are ready when the installers run.

Open **PowerShell as Administrator** and run:

```powershell
New-Item -ItemType Directory -Force -Path "D:\ProgramFiles"
New-Item -ItemType Directory -Force -Path "D:\ProgramFiles\Docker"
New-Item -ItemType Directory -Force -Path "D:\ProgramFiles\WSL"
New-Item -ItemType Directory -Force -Path "D:\WORK\Docker_Images"
New-Item -ItemType Directory -Force -Path "D:\WORK\Docker_Images\wsl"

# Verify all folders exist
Get-ChildItem D:\ -Directory
```

---

## 4. Enable WSL2 & Windows Features

Run the following in **PowerShell as Administrator**. A mandatory restart is required after this step.

```powershell
# Enable Windows Subsystem for Linux
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# Enable Virtual Machine Platform (required for WSL2)
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Enable Hyper-V — SKIP THIS if you are on Windows Home edition
dism.exe /online /enable-feature /featurename:Microsoft-Hyper-V-All /all /norestart

# Restart your PC now — this is mandatory before continuing
Restart-Computer
```

> ⚠️ **Do not skip the restart.** The WSL2 Linux kernel will not be available until after the reboot.

### After reboot — set WSL2 as default

Open **PowerShell as Administrator** again:

```powershell
# Set WSL2 as the default version
wsl --set-default-version 2

# Update WSL to the latest kernel version
wsl --update

# Confirm WSL version
wsl --version
```

---

## 5. Create .wslconfig

> ⚠️ **Important:** The `.wslconfig` file does **not** exist by default on Windows. You must create it yourself. Windows never creates this file automatically.

This file lives in your Windows user home folder (`C:\Users\YourUsername\`). It is a tiny configuration file — not an application — so it is fine for it to live on C drive. It controls WSL2 memory and CPU limits only.

### Create it via PowerShell (recommended)

```powershell
$config = @"
[wsl2]
memory=4GB
processors=2
swap=2GB
"@

Set-Content -Path "$env:USERPROFILE\.wslconfig" -Value $config
Write-Host "Created at: $env:USERPROFILE\.wslconfig"
```

### Verify it was created

```powershell
Get-Content "$env:USERPROFILE\.wslconfig"
```

### Create it manually via File Explorer (alternative)

1. Open File Explorer and navigate to `C:\Users\YourUsername\`
2. Enable hidden files: **View → Show → Hidden items**
3. Enable file extensions: **View → Show → File name extensions**
4. Right-click in the folder → **New → Text Document**
5. Name it exactly `.wslconfig` — make sure Windows does not append `.txt`
6. Open it and paste the config content from above, then save

---

## 6. Install Ubuntu WSL2 to D Drive

Docker Desktop requires a Linux distro running under WSL2. This step installs Ubuntu 22.04 with its virtual disk stored entirely on D drive.

### Download the Ubuntu appx bundle

```powershell
Invoke-WebRequest `
  -Uri "https://aka.ms/wslubuntu2204" `
  -OutFile "D:\ProgramFiles\WSL\Ubuntu2204.appx" `
  -UseBasicParsing

Write-Host "Download complete"
```

### Extract the bundle — two levels required

The downloaded file is a bundle that contains multiple appx files inside it. You must extract twice to reach the actual Linux filesystem archive.

```powershell
# Level 1: Extract the outer bundle
Copy-Item "D:\ProgramFiles\WSL\Ubuntu2204.appx" "D:\ProgramFiles\WSL\Ubuntu2204.zip"
Expand-Archive "D:\ProgramFiles\WSL\Ubuntu2204.zip" "D:\ProgramFiles\WSL\Ubuntu2204" -Force

# Check what is inside — you will see multiple appx files
Get-ChildItem "D:\ProgramFiles\WSL\Ubuntu2204"
```

You will see files like:
```
Ubuntu_2204.1.7.0_ARM64.appx   ← do NOT use this one
Ubuntu_2204.1.7.0_x64.appx     ← use this one (standard Windows PCs)
Ubuntu_2204.1.7.0_scale-100.appx
...
```

> ⚠️ **Always use the `_x64.appx` file.** The `_ARM64.appx` file is for ARM-based devices (Surface Pro X, Snapdragon laptops) only. Using it on a standard desktop or laptop will fail.

```powershell
# Level 2: Extract the x64 appx
# Update the filename below if your version number differs
Copy-Item "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_2204.1.7.0_x64.appx" `
          "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64.zip"

Expand-Archive "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64.zip" `
               "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64" -Force

# Confirm install.tar.gz is now present
Get-ChildItem "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64" -Filter "*.tar.gz"
```

You should see `install.tar.gz` in the output. If the version number in the filename differs from above, use whatever `_x64.appx` filename `Get-ChildItem` showed you.

### Import Ubuntu directly into D drive

```powershell
wsl --import Ubuntu-22.04 `
    "D:\ProgramFiles\WSL\Ubuntu2204-Data" `
    "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64\install.tar.gz" `
    --version 2
```

### Verify Ubuntu is registered and on WSL2

```powershell
wsl --list --verbose
```

Expected output:
```
  NAME          STATE    VERSION
* Ubuntu-22.04  Stopped  2
```

The Ubuntu virtual disk is now stored at `D:\ProgramFiles\WSL\Ubuntu2204-Data\` — not on C drive.

---

## 7. Install Docker Desktop to D Drive

### Download the installer to D drive

```powershell
Invoke-WebRequest `
  -Uri "https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe" `
  -OutFile "D:\ProgramFiles\Docker\DockerDesktopInstaller.exe" `
  -UseBasicParsing

Write-Host "Download complete"
```

### Run the silent installer with custom path flags

```powershell
Start-Process "D:\ProgramFiles\Docker\DockerDesktopInstaller.exe" -ArgumentList `
  "install",
  "--quiet",
  "--accept-license",
  "--installation-dir=D:\ProgramFiles\Docker\App",
  "--wsl-default-data-root=D:\WORK\Docker_Images\wsl",
  "--backend=wsl-2",
  "--no-windows-containers" `
  -Wait -NoNewWindow

Write-Host "Docker Desktop installation complete"
```

**What each flag does:**

| Flag | Purpose |
|---|---|
| `--installation-dir` | Where Docker Desktop app files are installed |
| `--wsl-default-data-root` | Where Docker's WSL2 virtual disks go |
| `--backend=wsl-2` | Forces the WSL2 engine (not Hyper-V) |
| `--quiet` | Silent install with no UI wizard |
| `--accept-license` | Auto-accepts the Docker license agreement |
| `--no-windows-containers` | Skips Windows container feature (not needed for Linux containers) |

> **Note:** Docker Desktop writes a shortcut and a few registry entries to Windows default system locations. This is unavoidable for any Windows application. All actual binaries, images, and container data are on D drive.

### Set Docker environment variable

```powershell
[System.Environment]::SetEnvironmentVariable(
  "DOCKER_CONFIG",
  "D:\ProgramFiles\Docker\config",
  "Machine"
)

Write-Host "DOCKER_CONFIG = $([System.Environment]::GetEnvironmentVariable('DOCKER_CONFIG','Machine'))"
```

---

## 8. Add Docker CLI to System PATH

Because Docker Desktop was installed to a custom path on D drive, Windows does not automatically know where the `docker` command is. Without this step, running `docker` in PowerShell will give a **"not recognized"** error.

### Find the exact Docker CLI location

```powershell
Get-ChildItem "D:\ProgramFiles\Docker" -Filter "docker.exe" -Recurse
```

This will return a path like:
```
D:\ProgramFiles\Docker\App\resources\bin\docker.exe
```

### Add the folder to the system PATH

```powershell
$dockerBin = "D:\ProgramFiles\Docker\App\resources\bin"
$currentPath = [System.Environment]::GetEnvironmentVariable("Path", "Machine")

if ($currentPath -notlike "*$dockerBin*") {
    [System.Environment]::SetEnvironmentVariable(
        "Path",
        "$currentPath;$dockerBin",
        "Machine"
    )
    Write-Host "Docker CLI added to system PATH"
} else {
    Write-Host "Docker CLI already in PATH"
}
```

> ⚠️ **Close and reopen PowerShell after this step.** PATH changes only take effect in newly opened terminal windows.

Confirm it works:

```powershell
docker --version
```

---

## 9. Launch Docker Desktop & Initialize

Docker Desktop must run at least once to create its internal WSL2 distros before you can move them to D drive.

1. Launch Docker Desktop from the Start menu or desktop shortcut
2. Wait for the **whale icon** in the taskbar system tray (bottom-right corner) to show **"Docker Desktop is running"** — this can take 2–3 minutes on first launch
3. Accept the license agreement when prompted
4. Skip the tutorial and any onboarding prompts
5. Once fully running — **right-click the whale icon → Quit Docker Desktop**
6. Then shut down WSL completely:

```powershell
wsl --shutdown
```

### Confirm Docker's WSL distros were created

```powershell
wsl --list --verbose
```

Expected output:

```
  NAME              STATE    VERSION
* Ubuntu-22.04      Stopped  2
  docker-desktop    Stopped  2
```

> **Docker Desktop 4.20+ Note:** Older versions of Docker Desktop created two distros: `docker-desktop` and `docker-desktop-data`. In Docker Desktop 4.20 and newer, these were **merged into a single `docker-desktop` distro**. If you only see `docker-desktop` and not `docker-desktop-data` — that is correct.

---

## 10. Fix Docker Context Error

After installation, running `docker` commands may fail with this error:

```
Failed to initialize: unable to resolve docker endpoint: context "desktop-linux": context not found
```

This happens because Docker Desktop created its context files in `C:\Users\YourUsername\.docker\` but the `DOCKER_CONFIG` environment variable is pointing to `D:\ProgramFiles\Docker\config`. Fix it by copying the config over.

```powershell
# Replace YourUsername with your actual Windows username
# To find your username run: echo $env:USERNAME

Copy-Item -Path "C:\Users\$env:USERNAME\.docker" `
          -Destination "D:\ProgramFiles\Docker\config" `
          -Recurse -Force

Write-Host "Docker config copied to D drive successfully"
```

Verify the context files are in place:

```powershell
Get-ChildItem "D:\ProgramFiles\Docker\config" -Recurse -Depth 3
```

You should see a `contexts\meta\...\meta.json` file in the output.

**Close and reopen PowerShell**, then confirm Docker context is working:

```powershell
docker context list
```

---

## 11. Move Docker WSL Disk to D Drive

The `docker-desktop` WSL2 distro holds all Docker images, containers, and volumes. By default it may have been placed on C drive. This step moves it permanently to `D:\WORK\Docker_Images\wsl\`.

> ⚠️ Make sure Docker Desktop is fully quit and WSL is shut down (`wsl --shutdown`) before running these commands.

```powershell
# Step 1: Export the docker-desktop distro to a tar file on D drive
# This may take a minute — no output while it runs, that is normal
wsl --export docker-desktop "D:\WORK\Docker_Images\wsl\docker-desktop.tar"

# Step 2: Unregister it from its current location
wsl --unregister docker-desktop

# Step 3: Re-import it into D:\WORK\Docker_Images\wsl\
wsl --import docker-desktop `
    "D:\WORK\Docker_Images\wsl" `
    "D:\WORK\Docker_Images\wsl\docker-desktop.tar" `
    --version 2

# Step 4: Remove the tar file to free up space
Remove-Item "D:\WORK\Docker_Images\wsl\docker-desktop.tar"

# Step 5: Verify all distros are present and on WSL2
wsl --list --verbose
```

Expected output:
```
  NAME              STATE    VERSION
* Ubuntu-22.04      Stopped  2
  docker-desktop    Stopped  2
```

---

## 12. Update Docker Desktop Disk Image Location

Tell Docker Desktop's UI where the virtual disk now lives.

1. Launch Docker Desktop
2. Click the **gear icon** (top right) to open Settings
3. Go to **Resources → Advanced**
4. Find the **Disk image location** field and set it to:
   ```
   D:\WORK\Docker_Images\wsl\disk
   ```
5. Click **Apply & Restart**

> ✅ **This is expected:** Docker Desktop will automatically append `\DockerDesktopWSL` to the path you entered. The field will display `D:\WORK\Docker_Images\wsl\disk\DockerDesktopWSL` after you apply. This is correct — do not change it.

Your final disk structure on D drive will look like this:

```
D:\WORK\Docker_Images\
└── wsl\
    ├── disk\
    │   └── DockerDesktopWSL\       ← Docker Desktop created this automatically
    │       ├── disk\
    │       │   └── docker_data.vhdx   ← all images & containers live here (grows with use)
    │       └── main\
    │           └── ext4.vhdx          ← Docker engine files
    └── main\
```

---

## 13. Verify Installation

Run the following after Docker Desktop has fully restarted:

```powershell
# 1. Confirm Docker CLI version
docker --version
docker compose --version

# 2. Run the official test container
docker run hello-world

# 3. Confirm the vhdx file exists on D drive and is non-zero in size
Get-ChildItem "D:\WORK\Docker_Images\wsl\disk\DockerDesktopWSL" -Recurse

# 4. Check WSL distros are healthy
wsl --list --verbose

# 5. Confirm no vhdx files exist on C drive
Get-ChildItem "C:\Users\$env:USERNAME\AppData\Local\Docker" -Recurse -ErrorAction SilentlyContinue
```

### What correct output looks like

**`docker run hello-world`** should print:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

**`Get-ChildItem` on D drive** should show:
```
D:\WORK\Docker_Images\wsl\disk\DockerDesktopWSL\disk\docker_data.vhdx   ← grows as you pull images
D:\WORK\Docker_Images\wsl\disk\DockerDesktopWSL\main\ext4.vhdx
```

**`docker info | findstr "Docker Root Dir"`** returns:
```
Docker Root Dir: /var/lib/docker
```
> This Linux-style path is correct. Inside WSL2, Docker always shows its internal Linux path. The physical storage is the `docker_data.vhdx` on your D drive.

**C drive check** should show only small log and lock files — **no `.vhdx` files**. A few KB of logs and lock files in `AppData\Local\Docker\` are unavoidable and expected for any Windows application.

---

## 14. Final Path Summary

### Program installation paths

| Component | Path on D Drive |
|---|---|
| Docker Desktop app files | `D:\ProgramFiles\Docker\App` |
| Docker CLI (`docker.exe`) | `D:\ProgramFiles\Docker\App\resources\bin` |
| Docker Desktop installer | `D:\ProgramFiles\Docker\DockerDesktopInstaller.exe` |
| Docker config & context files | `D:\ProgramFiles\Docker\config` |
| Ubuntu 22.04 WSL2 distro | `D:\ProgramFiles\WSL\Ubuntu2204-Data` |
| Ubuntu installer files | `D:\ProgramFiles\WSL\` |

### Docker data paths

| Component | Path on D Drive |
|---|---|
| Docker images & containers | `D:\WORK\Docker_Images\wsl\disk\DockerDesktopWSL\disk\docker_data.vhdx` |
| Docker engine files | `D:\WORK\Docker_Images\wsl\disk\DockerDesktopWSL\main\ext4.vhdx` |

### Environment variables set

| Variable | Value |
|---|---|
| `DOCKER_CONFIG` | `D:\ProgramFiles\Docker\config` |
| `Path` (appended) | `D:\ProgramFiles\Docker\App\resources\bin` |

### Dependencies installed

| Dependency | How it was installed | Required |
|---|---|---|
| BIOS Virtualization (VT-x / AMD-V) | Enabled manually in BIOS/UEFI | ✅ Yes |
| Windows Subsystem for Linux (WSL) | `dism.exe` — built into Windows | ✅ Yes |
| Virtual Machine Platform | `dism.exe` — built into Windows | ✅ Yes |
| Hyper-V | `dism.exe` — Pro/Enterprise/Education only | ✅ Yes (skip on Home) |
| WSL2 Linux kernel update | `wsl --update` | ✅ Yes |
| Ubuntu 22.04 LTS | Downloaded from `aka.ms/wslubuntu2204` | ✅ Yes |
| Docker Desktop | Downloaded from `docker.com` | ✅ Yes |

---

## 15. Troubleshooting

### `docker : The term 'docker' is not recognized`

Docker CLI is not in your system PATH. Go to [Step 8](#8-add-docker-cli-to-system-path), find the `docker.exe` location, and add it to PATH. Then close and reopen PowerShell.

---

### `context "desktop-linux": context not found`

Docker CLI cannot find its context files because they were created on C drive but `DOCKER_CONFIG` points to D drive. Go to [Step 10](#10-fix-docker-context-error) and copy the `.docker` folder from C to D.

---

### `There is no distribution with the supplied name` when exporting docker-desktop

Docker Desktop has not run yet and has not created its WSL distros. Go to [Step 9](#9-launch-docker-desktop--initialize), launch Docker Desktop, wait for it to fully start, quit it, run `wsl --shutdown`, then retry.

---

### `The system cannot find the file specified` when importing Ubuntu

The `install.tar.gz` was not found at the path you provided. This usually means you used the outer bundle path instead of the inner `_x64.appx` path. Follow [Step 6](#6-install-ubuntu-wsl2-to-d-drive) carefully — extraction is required twice, and you must use `_x64.appx` not `_ARM64.appx`.

---

### `.wslconfig` file not found or not working

This file does not exist by default. You must create it. See [Step 5](#5-create-wslconfig). Also ensure the filename is exactly `.wslconfig` with no `.txt` extension — enable file extensions in File Explorer to verify.

---

### Docker Desktop shows `docker-desktop-data` distro is missing

This is not an error. Docker Desktop 4.20 and newer merged `docker-desktop-data` into `docker-desktop`. Only `docker-desktop` needs to be exported and moved. See [Step 11](#11-move-docker-wsl-disk-to-d-drive).

---

### Docker Desktop stuck on "Starting..." at launch

This is usually caused by one of:
- Virtualization not enabled in BIOS — verify via Task Manager → Performance → CPU → Virtualization: Enabled
- WSL kernel not updated — run `wsl --update` in Administrator PowerShell
- Windows features not enabled — recheck [Step 4](#4-enable-wsl2--windows-features)

---

### Disk image location reverts to C drive after restarting Docker Desktop

This can happen if Docker Desktop cannot write to the D drive path. Verify:
- The folder `D:\WORK\Docker_Images\wsl\disk` exists
- You have write permissions to that folder
- Docker Desktop was fully restarted (not just closed and reopened)

---

### `docker run hello-world` fails with a network or pull error

This is usually a network or DNS issue inside WSL2, not a Docker installation problem. Try:

```powershell
# Restart WSL completely
wsl --shutdown
# Then relaunch Docker Desktop and try again
```

---

*Guide version: Docker Desktop 4.20+ · WSL2 backend · Windows 10/11 · Last verified May 2026*
