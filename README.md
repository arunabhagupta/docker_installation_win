# Docker on Windows — Full D Drive Installation Guide

> **Zero C: Drive footprint** — All programs under `D:\ProgramFiles\` · All Docker images and data under `D:\WORK\Docker_Images\`

This guide is based on a real installation walkthrough on a fresh Windows PC with Docker Desktop 4.20+. All commands are verified and corrected from actual output.

---

## Table of Contents

1. [Prerequisites & BIOS](#1-prerequisites--bios-virtualization)
2. [Create D Drive Directories](#2-create-d-drive-directories)
3. [Enable WSL2 & Windows Features](#3-enable-wsl2--windows-features)
4. [Create .wslconfig](#4-create-wslconfig)
5. [Install Ubuntu WSL2 to D Drive](#5-install-ubuntu-wsl2-to-d-drive)
6. [Install Docker Desktop to D Drive](#6-install-docker-desktop-to-d-drive)
7. [Add Docker CLI to PATH](#7-add-docker-cli-to-system-path)
8. [Fix Docker Context](#8-fix-docker-context)
9. [Move Docker WSL Disk to D Drive](#9-move-docker-wsl-disk-to-d-drive)
10. [Verify Installation](#10-verify-installation)
11. [Final Path Summary](#11-final-path-summary)

---

## System Requirements

| Requirement | Minimum |
|---|---|
| Windows Version | Windows 10 Build 19041 (20H1) or Windows 11 — 64-bit |
| RAM | 4 GB minimum (8 GB+ recommended) |
| Processor | 64-bit with SLAT support |
| BIOS | Hardware virtualization enabled (VT-x or AMD-V) |

---

## 1. Prerequisites & BIOS Virtualization

Docker requires hardware virtualization enabled at the BIOS level. This must be done before booting Windows.

### Check your Windows version

Open PowerShell as Administrator and run:

```powershell
Get-ComputerInfo -Property OsName, OsVersion, OsBuildNumber
```

### Enable virtualization in BIOS/UEFI

1. Restart your PC and press the BIOS key during POST — usually `DEL`, `F2`, `F10`, or `F12` (shown briefly on screen, depends on your motherboard brand)
2. Navigate to the CPU settings:
   - **Intel:** Advanced → CPU Configuration → Enable `Intel Virtualization Technology (VT-x)` and `Intel VT-d`
   - **AMD:** Advanced → Enable `AMD-V (SVM Mode)`
3. Save and Exit — your PC will reboot normally
4. Verify in Windows: open **Task Manager → Performance → CPU** and confirm **Virtualization: Enabled**

> **Note:** Most new PC builds ship with virtualization OFF by default. If you skip this step, WSL2 and Docker will fail with a cryptic error.

---

## 2. Create D Drive Directories

Create all required folders on D drive before installing anything.

```powershell
# Run PowerShell as Administrator

New-Item -ItemType Directory -Force -Path "D:\ProgramFiles"
New-Item -ItemType Directory -Force -Path "D:\ProgramFiles\Docker"
New-Item -ItemType Directory -Force -Path "D:\ProgramFiles\WSL"
New-Item -ItemType Directory -Force -Path "D:\WORK\Docker_Images"
New-Item -ItemType Directory -Force -Path "D:\WORK\Docker_Images\wsl"

# Verify
Get-ChildItem D:\ -Directory
```

---

## 3. Enable WSL2 & Windows Features

Run these commands in PowerShell as Administrator. A restart is required after this step.

```powershell
# Enable Windows Subsystem for Linux
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# Enable Virtual Machine Platform (required for WSL2)
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Enable Hyper-V (skip this if you are on Windows Home edition)
dism.exe /online /enable-feature /featurename:Microsoft-Hyper-V-All /all /norestart

# Restart your PC now — do not skip this
Restart-Computer
```

> **Windows Home users:** Hyper-V is not available on Windows Home. Skip that command. WSL2 backend works without it.

After reboot, open PowerShell as Administrator again:

```powershell
# Set WSL2 as the default version
wsl --set-default-version 2

# Update WSL to the latest version
wsl --update

# Confirm version
wsl --version
```

---

## 4. Create .wslconfig

The `.wslconfig` file does **not** exist by default — you must create it yourself. It lives in your Windows user home folder, not on D drive (it is a tiny config file, not application data).

```powershell
# This creates the file and sets WSL2 memory/CPU limits
$config = @"
[wsl2]
memory=4GB
processors=2
swap=2GB
"@

Set-Content -Path "$env:USERPROFILE\.wslconfig" -Value $config
Write-Host "Created: $env:USERPROFILE\.wslconfig"
```

To verify it was created:

```powershell
Get-Content "$env:USERPROFILE\.wslconfig"
```

> **Tip:** If you prefer to create it manually in File Explorer, go to `C:\Users\YourUsername\`, enable hidden file visibility via **View → Show → Hidden items**, then create a new text file and name it exactly `.wslconfig` — make sure Windows does not append `.txt`.

---

## 5. Install Ubuntu WSL2 to D Drive

Docker Desktop needs a Linux distro. This installs Ubuntu 22.04 with its data VHD stored on D drive.

### Download the Ubuntu appx bundle

```powershell
Invoke-WebRequest -Uri "https://aka.ms/wslubuntu2204" `
  -OutFile "D:\ProgramFiles\WSL\Ubuntu2204.appx" `
  -UseBasicParsing
```

### Extract the bundle (two levels deep)

The downloaded file is a bundle containing multiple appx files inside. You need to extract twice to reach the actual `install.tar.gz`.

```powershell
# Step 1: Rename and extract the outer bundle
Copy-Item "D:\ProgramFiles\WSL\Ubuntu2204.appx" "D:\ProgramFiles\WSL\Ubuntu2204.zip"
Expand-Archive "D:\ProgramFiles\WSL\Ubuntu2204.zip" "D:\ProgramFiles\WSL\Ubuntu2204" -Force

# Step 2: Check what's inside — you will see files like Ubuntu_2204.1.x.x_x64.appx
Get-ChildItem "D:\ProgramFiles\WSL\Ubuntu2204"
```

```powershell
# Step 3: Extract the x64 appx (the ARM64 one is NOT for regular Windows PCs)
Copy-Item "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_2204.1.7.0_x64.appx" `
          "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64.zip"
Expand-Archive "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64.zip" `
               "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64" -Force

# Step 4: Confirm install.tar.gz is now present
Get-ChildItem "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64" -Filter "*.tar.gz"
```

> **Important:** Use the `_x64.appx` file, not `_ARM64.appx`. ARM64 is for Surface Pro X and similar ARM-based devices only.

### Import Ubuntu directly to D drive

```powershell
wsl --import Ubuntu-22.04 `
    "D:\ProgramFiles\WSL\Ubuntu2204-Data" `
    "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64\install.tar.gz" `
    --version 2

# Verify it is registered on WSL2
wsl --list --verbose
```
### Step 07 Alternative in case it does not work
```powershell
# Step 1: Copy the x64 appx and rename to zip
Copy-Item "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_2204.1.7.0_x64.appx" `
          "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64.zip"

# Step 2: Extract it
Expand-Archive "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64.zip" `
               "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64" -Force

# Step 3: See what's inside
Get-ChildItem "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64"

# Step 4: Import using the correct path
wsl --import Ubuntu-22.04 `
    "D:\ProgramFiles\WSL\Ubuntu2204-Data" `
    "D:\ProgramFiles\WSL\Ubuntu2204\Ubuntu_x64\install.tar.gz" `
    --version 2

# Step 5: Verify
wsl --list --verbose
```

Expected output:
```
  NAME          STATE    VERSION
* Ubuntu-22.04  Stopped  2
```

The Ubuntu VHD (`ext4.vhdx`) is now stored at `D:\ProgramFiles\WSL\Ubuntu2204-Data\` — not on C drive.

---

## 6. Install Docker Desktop to D Drive

### Download the installer to D drive

```powershell
Invoke-WebRequest `
  -Uri "https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe" `
  -OutFile "D:\ProgramFiles\Docker\DockerDesktopInstaller.exe" `
  -UseBasicParsing
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
```

**Flag reference:**

| Flag | Purpose |
|---|---|
| `--installation-dir` | Where Docker Desktop app files are installed |
| `--wsl-default-data-root` | Where Docker's WSL2 virtual disks go |
| `--backend=wsl-2` | Forces WSL2 engine (not Hyper-V) |
| `--quiet` | Silent install, no UI wizard |

> **Note:** Docker Desktop will still write a small shortcut and a few registry entries to Windows default locations. This is unavoidable for any Windows application. All actual application binaries and image data will be on D drive.

### Set Docker environment variables

```powershell
[System.Environment]::SetEnvironmentVariable(
  "DOCKER_CONFIG",
  "D:\ProgramFiles\Docker\config",
  "Machine"
)

Write-Host "DOCKER_CONFIG set to D:\ProgramFiles\Docker\config"
```

---

## 7. Add Docker CLI to System PATH

Because Docker Desktop was installed to a custom D drive path, Windows does not automatically know where the `docker` command lives. You must add it to the system PATH manually.

### Find the Docker CLI location

```powershell
Get-ChildItem "D:\ProgramFiles\Docker" -Filter "docker.exe" -Recurse
```

This will return a path like:
```
D:\ProgramFiles\Docker\App\resources\bin\docker.exe
```

### Add it to the system PATH

```powershell
$dockerBin = "D:\ProgramFiles\Docker\App\resources\bin"
$currentPath = [System.Environment]::GetEnvironmentVariable("Path", "Machine")

if ($currentPath -notlike "*$dockerBin*") {
    [System.Environment]::SetEnvironmentVariable(
        "Path",
        "$currentPath;$dockerBin",
        "Machine"
    )
    Write-Host "Docker CLI added to PATH"
} else {
    Write-Host "Docker CLI already in PATH"
}
```

**Close and reopen PowerShell** after this step — PATH changes only take effect in new windows.

---

## 8. Fix Docker Context

### Launch Docker Desktop first

1. Start Docker Desktop from the Start menu
2. Wait for the **whale icon** in the taskbar system tray to show **"Docker Desktop is running"** (can take 2–3 minutes on first launch)
3. Accept the license and skip any tutorial prompts
4. Once fully running — right-click the whale icon → **Quit Docker Desktop**
5. Then shut down WSL:

```powershell
wsl --shutdown
```

### Copy Docker config from C to D drive

Docker Desktop creates its context files in `C:\Users\YourUsername\.docker\` by default. Since we pointed `DOCKER_CONFIG` to D drive, the Docker CLI cannot find them. Copy them over:

```powershell
# Replace 'YourUsername' with your actual Windows username
Copy-Item -Path "C:\Users\YourUsername\.docker" `
          -Destination "D:\ProgramFiles\Docker\config" `
          -Recurse -Force

Write-Host "Docker config copied to D drive"
```

Verify the context files are present:

```powershell
Get-ChildItem "D:\ProgramFiles\Docker\config" -Recurse -Depth 3
```

You should see a `contexts\meta\...\meta.json` file in the output.

**Close and reopen PowerShell**, then verify Docker works:

```powershell
docker context list
docker --version
```

---

## 9. Move Docker WSL Disk to D Drive

Docker Desktop 4.20+ uses a **single WSL distro** called `docker-desktop` (older versions also had `docker-desktop-data` — that has been merged). This distro holds all your images, containers, and volumes.

### Confirm WSL distros exist

```powershell
wsl --list --verbose
```

Expected output:
```
  NAME              STATE    VERSION
* Ubuntu-22.04      Stopped  2
  docker-desktop    Stopped  2
```

### Export, unregister, and re-import docker-desktop to D drive

```powershell
# Step 1: Export to D drive tar file
wsl --export docker-desktop "D:\WORK\Docker_Images\wsl\docker-desktop.tar"

# Step 2: Unregister from its default C drive location
wsl --unregister docker-desktop

# Step 3: Re-import into D drive
wsl --import docker-desktop `
    "D:\WORK\Docker_Images\wsl" `
    "D:\WORK\Docker_Images\wsl\docker-desktop.tar" `
    --version 2

# Step 4: Remove the tar file to save space
Remove-Item "D:\WORK\Docker_Images\wsl\docker-desktop.tar"

# Step 5: Verify
wsl --list --verbose
```

### Update Docker Desktop disk image location

1. Launch Docker Desktop
2. Go to **Settings (gear icon) → Resources → Advanced**
3. Set **Disk image location** to: `D:\WORK\Docker_Images\wsl\disk`
4. Docker Desktop will automatically append `\DockerDesktopWSL` to this path — this is **expected and correct**. The final shown path will be `D:\WORK\Docker_Images\wsl\disk\DockerDesktopWSL`
5. Click **Apply & Restart**

Your actual disk structure will look like this:

```
D:\WORK\Docker_Images\wsl\
├── disk\
│   └── DockerDesktopWSL\
│       ├── disk\
│       │   └── docker_data.vhdx   ← all images and containers live here
│       └── main\
│           └── ext4.vhdx          ← Docker engine files
└── main\
```

---

## 10. Verify Installation

```powershell
# 1. Check Docker version
docker --version
docker compose --version

# 2. Run a test container
docker run hello-world

# 3. Confirm images are on D drive (docker_data.vhdx should exist and be non-zero)
Get-ChildItem "D:\WORK\Docker_Images\wsl\disk\DockerDesktopWSL" -Recurse

# 4. Confirm WSL distros are healthy
wsl --list --verbose

# 5. Confirm no vhdx files landed on C drive
Get-ChildItem "C:\Users\$env:USERNAME\AppData\Local\Docker" -Recurse -ErrorAction SilentlyContinue
```

### What to expect

- `docker run hello-world` prints **"Hello from Docker!"**
- `docker_data.vhdx` exists under `D:\WORK\Docker_Images\wsl\disk\DockerDesktopWSL\disk\` and grows as you pull images
- `docker info | findstr "Docker Root Dir"` returns `/var/lib/docker` — this is the Linux path **inside** WSL2 and is correct. The physical file is the `.vhdx` on D drive
- `C:\Users\...\AppData\Local\Docker\` contains only small log files and lock files — **no `.vhdx` files** — this is correct

> **Note on C drive:** A few KB of log files, lock files, and sockets will always exist in `AppData\Local\Docker\` on C drive. This is unavoidable Windows application behavior and takes up negligible space. All actual data stays on D drive.

---

## 11. Final Path Summary

### Installation paths

| Component | Path |
|---|---|
| Docker Desktop app | `D:\ProgramFiles\Docker\App` |
| Docker CLI (`docker.exe`) | `D:\ProgramFiles\Docker\App\resources\bin` |
| Docker config & context | `D:\ProgramFiles\Docker\config` |
| Docker Desktop installer | `D:\ProgramFiles\Docker\DockerDesktopInstaller.exe` |
| Ubuntu WSL2 distro data | `D:\ProgramFiles\WSL\Ubuntu2204-Data` |
| Ubuntu WSL2 installer files | `D:\ProgramFiles\WSL\` |

### Docker data paths

| Component | Path |
|---|---|
| Docker images & containers (vhdx) | `D:\WORK\Docker_Images\wsl\disk\DockerDesktopWSL\disk\docker_data.vhdx` |
| Docker engine files (vhdx) | `D:\WORK\Docker_Images\wsl\disk\DockerDesktopWSL\main\ext4.vhdx` |

### Dependencies

| Dependency | Source | Required |
|---|---|---|
| WSL (Windows Subsystem for Linux) | Built into Windows, enabled via `dism` | ✅ Required |
| Virtual Machine Platform | Built into Windows, enabled via `dism` | ✅ Required |
| Hyper-V | Built into Windows (Pro/Enterprise only), enabled via `dism` | ✅ Required (skip on Home) |
| WSL2 Linux kernel | Auto-installed by `wsl --update` | ✅ Required |
| Ubuntu 22.04 LTS | Downloaded from `aka.ms/wslubuntu2204` | ✅ Required |
| Docker Desktop | Downloaded from `docker.com` | ✅ Required |
| BIOS Virtualization (VT-x / AMD-V) | Enabled in motherboard BIOS/UEFI | ✅ Required |

---

## Troubleshooting

**`WSL2 requires an update to its kernel component`**
Run `wsl --update` in Administrator PowerShell, then restart your PC.

**`docker : The term 'docker' is not recognized`**
The Docker CLI is not in your PATH. Follow [Step 7](#7-add-docker-cli-to-system-path) to add it.

**`context "desktop-linux": context not found`**
The Docker context files are missing from D drive. Follow [Step 8](#8-fix-docker-context) to copy them from C drive.

**`There is no distribution with the supplied name`** when exporting `docker-desktop`
Docker Desktop has not been launched yet. Start it, wait for the whale icon to show running, quit it, run `wsl --shutdown`, then retry.

**Docker Desktop starts but shows "engine not running"**
Run `wsl --list --verbose` and confirm `docker-desktop` appears. If it is missing, redo Step 9.

**Virtualization error in Docker Desktop**
Open Task Manager → Performance → CPU and verify Virtualization shows **Enabled**. If not, return to BIOS and enable VT-x (Intel) or SVM/AMD-V (AMD).

**Docker Root Dir still shows C: path**
Open Docker Desktop → Settings → Docker Engine, verify the `daemon.json` has `"data-root": "D:\\WORK\\Docker_Images"`, then click **Apply & Restart**.

**`_ARM64.appx` instead of `_x64.appx`**
Use the `_x64.appx` file. The ARM64 package is only for ARM-based Windows devices (Surface Pro X, Snapdragon PCs). A standard desktop or laptop uses x64.
