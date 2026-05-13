# docker_installation_win
Docker installation in Windows 11 in D drive instead of C drive

# Docker on Windows — D Drive Installation Guide

> 🐳 Docker · Windows · Full D: Drive Setup

Complete step-by-step guide to install Docker Desktop and all its dependencies entirely on your D drive. Programs go to `D:\ProgramFiles\` and all Docker images, containers, and volumes go to `D:\WORK\Docker_Images\`

**Target Paths:**
- **Programs installed to:** `D:\ProgramFiles\`
- **Docker images & data:** `D:\WORK\Docker_Images\`

---

## Table of Contents

- [00. Prerequisites & BIOS Check](#phase-prereq)
- [01. Enable WSL2 & Hyper-V](#phase-wsl)
- [02. Install WSL2 to D Drive](#phase-wsl-install)
- [03. Install Docker Desktop to D Drive](#phase-docker)
- [04. Move Docker Data to D Drive](#phase-move)
- [05. Configure Docker Engine](#phase-config)
- [06. Verify Installation](#phase-verify)
- [07. Path Summary](#phase-summary)

---

## PHASE 00: Prerequisites & BIOS Virtualization {#phase-prereq}

### Step 01: Check Windows Version & System Requirements

Docker Desktop requires Windows 10 64-bit (Build 19041+) or Windows 11. Run the following in PowerShell to verify.

```powershell
# PowerShell (Run as Administrator)

# Check Windows version (must be 19041+ for Win10 or any Win11)
winver

# Or check via command
Get-ComputerInfo -Property OsName, OsVersion, OsBuildNumber
```

> **ℹ️ Minimum Requirements**
> Windows 10 Build 19041 (20H1) or Windows 11 · 64-bit processor with SLAT · 4 GB RAM minimum (8 GB+ recommended) · BIOS-level hardware virtualization enabled

---

### Step 02: Enable Virtualization in BIOS/UEFI

Virtualization must be enabled at the BIOS level before WSL2 or Docker will work. This step is done before booting Windows.

1. Restart your PC and press the BIOS key during POST — usually `DEL`, `F2`, `F10`, or `F12` depending on your motherboard brand (shown briefly on screen).
2. Navigate to **Advanced** → **CPU Configuration** (Intel) or **Advanced** → **SVM Mode** (AMD).
3. **Intel CPUs:** Enable `Intel Virtualization Technology (VT-x)` and `Intel VT-d`.
4. **AMD CPUs:** Enable `AMD-V (SVM)`.
5. Save and Exit BIOS. Your PC will reboot normally.
6. Verify in Windows — open Task Manager → Performance → CPU → confirm **Virtualization: Enabled**

> **⚠️ New PC heads-up**
> Most new builds have virtualization OFF by default. If you skip this step, WSL2 and Docker will fail to start with a cryptic error.

---

### Step 03: Create Required Directories on D Drive

Create all target folders before installing anything so paths are ready.

```powershell
# PowerShell (Run as Administrator)

# Create all required directories on D drive
New-Item -ItemType Directory -Force -Path "D:\ProgramFiles"
New-Item -ItemType Directory -Force -Path "D:\ProgramFiles\Docker"
New-Item -ItemType Directory -Force -Path "D:\ProgramFiles\WSL"
New-Item -ItemType Directory -Force -Path "D:\WORK\Docker_Images"
New-Item -ItemType Directory -Force -Path "D:\WORK\Docker_Images\wsl"

# Verify they exist
Get-ChildItem D:\ -Directory
```

---

## PHASE 01: Enable WSL2 & Hyper-V Windows Features {#phase-wsl}

> **⚠️ Why WSL2?**
> Docker Desktop on Windows uses WSL2 (Windows Subsystem for Linux 2) as its backend engine. This gives much better performance than the older Hyper-V backend and is required by default in recent Docker Desktop versions.

---

### Step 04: Enable Required Windows Features

Enable WSL, Virtual Machine Platform, and Hyper-V via PowerShell. A restart is required after this step.

```powershell
# PowerShell (Run as Administrator)

# Enable Windows Subsystem for Linux
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# Enable Virtual Machine Platform (required for WSL2)
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Enable Hyper-V (optional but recommended)
dism.exe /online /enable-feature /featurename:Microsoft-Hyper-V-All /all /norestart

# RESTART your PC before proceeding!
Restart-Computer
```

> **🔴 You MUST restart here**
> before continuing. The WSL kernel won't be available until after reboot.

---

## PHASE 02: Install & Configure WSL2 on D Drive {#phase-wsl-install}

### Step 05: Set WSL Default Version to 2

After reboot, open PowerShell as Administrator and set WSL2 as default.

```powershell
# PowerShell (Run as Administrator)

# Set WSL2 as default version
wsl --set-default-version 2

# Update WSL to latest version
wsl --update

# Confirm version
wsl --version
```

---

### Step 06: Configure WSL Global Config to Use D Drive

Edit the global WSL config file to set the default install location to D drive before installing any Linux distro.

> **ℹ️**
> The file `.wslconfig` lives in your user home at `C:\Users\YourName\.wslconfig` — this is a WSL config file, not an application. It only controls WSL's behavior, not its install location. We will point the actual distro VHD files to D drive in the next step.

```powershell
# PowerShell — create .wslconfig

# Create or overwrite .wslconfig in your user home
$config = @"
[wsl2]
memory=4GB
processors=2
swap=2GB
"@

Set-Content -Path "$env:USERPROFILE\.wslconfig" -Value $config
Write-Host "WSL config written to $env:USERPROFILE\.wslconfig"
```

---

### Step 07: Install Ubuntu (WSL2 Distro) Directly to D Drive

Docker Desktop needs a Linux distro. We install Ubuntu using WSL's `--install` with a custom location so the VHD lives on D drive.

```powershell
# PowerShell (Run as Administrator)

# Download Ubuntu WSL2 distro package
Invoke-WebRequest -Uri "https://aka.ms/wslubuntu2204" -OutFile "D:\ProgramFiles\WSL\Ubuntu2204.appx" -UseBasicParsing

# Rename to zip and extract
Copy-Item "D:\ProgramFiles\WSL\Ubuntu2204.appx" "D:\ProgramFiles\WSL\Ubuntu2204.zip"
Expand-Archive "D:\ProgramFiles\WSL\Ubuntu2204.zip" "D:\ProgramFiles\WSL\Ubuntu2204" -Force

# Import as WSL2 distro directly into D drive
wsl --import Ubuntu-22.04 "D:\ProgramFiles\WSL\Ubuntu2204-Data" "D:\ProgramFiles\WSL\Ubuntu2204\install.tar.gz" --version 2

# Verify it's registered and on WSL2
wsl --list --verbose
```

> **✅**
> The Ubuntu VHD (`ext4.vhdx`) will be stored at `D:\ProgramFiles\WSL\Ubuntu2204-Data\` — not on C drive.

---

## PHASE 03: Install Docker Desktop to D:\ProgramFiles\Docker {#phase-docker}

### Step 08: Download Docker Desktop Installer

Download the official Docker Desktop installer directly to D drive.

```powershell
# PowerShell (Run as Administrator)

# Download Docker Desktop installer to D drive
Invoke-WebRequest -Uri "https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe" `
  -OutFile "D:\ProgramFiles\Docker\DockerDesktopInstaller.exe" `
  -UseBasicParsing

Write-Host "Download complete: D:\ProgramFiles\Docker\DockerDesktopInstaller.exe"
```

---

### Step 09: Install Docker Desktop with Custom Path Flags

Run the installer in silent mode with command-line flags to set the installation directory to D:\ProgramFiles\Docker. This prevents any files going to C:\Program Files.

```powershell
# PowerShell (Run as Administrator)

# Install Docker Desktop silently to D:\ProgramFiles\Docker
Start-Process "D:\ProgramFiles\Docker\DockerDesktopInstaller.exe" -ArgumentList `
  "install",
  "--quiet",
  "--accept-license",
  "--installation-dir=D:\ProgramFiles\Docker\App",
  "--wsl-default-data-root=D:\WORK\Docker_Images\wsl",
  "--backend=wsl-2",
  "--no-windows-containers" `
  -Wait -NoNewWindow

Write-Host "Docker Desktop installation complete."
```

> **ℹ️ Flag breakdown:**
> - `--installation-dir` → Where Docker Desktop's app files go
> - `--wsl-default-data-root` → Where Docker's WSL2 virtual disks go (images, containers)
> - `--backend=wsl-2` → Forces WSL2 engine (not Hyper-V)
> - `--quiet` → Silent install, no UI wizard

> **⚠️ Note:**
> Docker Desktop still writes a small shortcut and registry entry to Windows default locations, but all actual application data and images will be on D drive. This is expected behavior for Windows applications.

---

### Step 10: Set Docker Desktop Environment Variable

Set the `DOCKER_DATA_ROOT` and `DOCKER_CONFIG` system environment variables so Docker CLI always points to D drive.

```powershell
# PowerShell (Run as Administrator)

# Set system-wide environment variables for Docker
[System.Environment]::SetEnvironmentVariable(
  "DOCKER_DATA_ROOT",
  "D:\WORK\Docker_Images",
  "Machine"
)

[System.Environment]::SetEnvironmentVariable(
  "DOCKER_CONFIG",
  "D:\ProgramFiles\Docker\config",
  "Machine"
)

# Verify
Write-Host "DOCKER_DATA_ROOT = $([System.Environment]::GetEnvironmentVariable('DOCKER_DATA_ROOT','Machine'))"
```

---

## PHASE 04: Move Docker WSL2 Disk Images to D:\WORK\Docker_Images {#phase-move}

> **ℹ️ Why this phase?**
> Docker Desktop creates two internal WSL2 distros: `docker-desktop` and `docker-desktop-data`. These hold all your pulled images, running containers, and volumes. By default they land in your C: user profile. These steps move them fully to `D:\WORK\Docker_Images\wsl\`

---

### Step 11: Start Docker Desktop Once to Initialize

Docker Desktop must run once to create its internal WSL distros before you can move them.

1. Launch Docker Desktop from the Start menu or desktop shortcut.
2. Complete the onboarding (accept terms, skip tutorial).
3. Wait until the whale icon in the taskbar shows **Docker Desktop is running**.
4. Then completely close Docker Desktop — right-click the whale icon → **Quit Docker Desktop**.

```powershell
# PowerShell — shut down WSL after quitting Docker Desktop

# Shut down all WSL after Docker Desktop is closed
wsl --shutdown

# Confirm Docker WSL distros exist
wsl --list --verbose
# You should see: docker-desktop  and  docker-desktop-data
```

---

### Step 12: Export and Re-import `docker-desktop-data` to D Drive

This is the critical distro — it holds all your Docker images and container data. Export it, unregister from C drive, import to D drive.

```powershell
# PowerShell (Run as Administrator)

# Step 1: Export docker-desktop-data to a tar on D drive
wsl --export docker-desktop-data "D:\WORK\Docker_Images\wsl\docker-desktop-data.tar"

# Step 2: Unregister the distro from its default C: location
wsl --unregister docker-desktop-data

# Step 3: Import the distro into D:\WORK\Docker_Images\wsl\
wsl --import docker-desktop-data `
  "D:\WORK\Docker_Images\wsl\docker-desktop-data" `
  "D:\WORK\Docker_Images\wsl\docker-desktop-data.tar" `
  --version 2

# Step 4: Verify the new location
wsl --list --verbose
```

---

### Step 13: Export and Re-import `docker-desktop` to D Drive

The `docker-desktop` distro holds Docker's engine binaries. Move it to D drive as well.

```powershell
# PowerShell (Run as Administrator)

# Step 1: Export docker-desktop distro
wsl --export docker-desktop "D:\WORK\Docker_Images\wsl\docker-desktop.tar"

# Step 2: Unregister from C: drive
wsl --unregister docker-desktop

# Step 3: Import into D:\WORK\Docker_Images\wsl\
wsl --import docker-desktop `
  "D:\WORK\Docker_Images\wsl\docker-desktop" `
  "D:\WORK\Docker_Images\wsl\docker-desktop.tar" `
  --version 2

# Step 4: Clean up the tar files (optional, saves space)
Remove-Item "D:\WORK\Docker_Images\wsl\docker-desktop.tar"
Remove-Item "D:\WORK\Docker_Images\wsl\docker-desktop-data.tar"

# Final check — all distros should show D: paths
wsl --list --verbose
```

> **✅**
> Both Docker WSL2 distros are now stored under `D:\WORK\Docker_Images\wsl\`. The actual VHD files (`ext4.vhdx`) inside those folders hold all your images and containers.

---

## PHASE 05: Configure Docker Engine & Settings {#phase-config}

### Step 14: Configure Docker Engine daemon.json

Edit the Docker daemon configuration to explicitly set the data root and log location to D drive. This ensures any future images and containers always land on D drive.

```powershell
# PowerShell — write daemon.json

# Create the Docker config directory on D drive
New-Item -ItemType Directory -Force -Path "D:\ProgramFiles\Docker\config"

# Write daemon.json with D drive data root
$daemonJson = @"
{
  "data-root": "D:\\WORK\\Docker_Images",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "features": {
    "buildkit": true
  }
}
"@

Set-Content -Path "D:\ProgramFiles\Docker\config\daemon.json" -Value $daemonJson
Write-Host "daemon.json written successfully"
```

> **⚠️ After writing daemon.json**
> — Open Docker Desktop → Settings → Docker Engine → paste and merge this JSON into the editor there as well, then click **Apply & Restart**. Docker Desktop's UI editor is the authoritative source.

---

### Step 15: Configure Docker Desktop UI Settings

Open Docker Desktop and configure settings via the graphical interface to ensure the WSL integration and disk path are correctly set.

1. Launch Docker Desktop. Go to ⚙️ **Settings** (top right gear icon).
2. **General tab:** Ensure *"Use the WSL 2 based engine"* is checked.
3. **Resources → Advanced:** Set *Disk image location* to `D:\WORK\Docker_Images\wsl\docker-desktop-data`
4. **Resources → WSL Integration:** Enable integration for your `Ubuntu-22.04` distro.
5. **Docker Engine tab:** Paste and save the `daemon.json` from Step 14.
6. Click **Apply & Restart** and wait for Docker to fully restart.

> **ℹ️**
> The **Disk image location** field in Docker Desktop UI controls where the WSL virtual disk lives. Changing it here is equivalent to the import you did in Steps 12–13.

---

## PHASE 06: Verify Everything Is Working {#phase-verify}

### Step 16: Run Verification Commands

Confirm Docker is healthy, images go to D drive, and no Docker data lives on C drive.

```powershell
# PowerShell or Command Prompt

# 1. Check Docker version
docker --version
docker compose --version

# 2. Check Docker info — look for "Docker Root Dir: D:\WORK\Docker_Images"
docker info | findstr "Docker Root Dir"

# 3. Pull and run hello-world — confirms Docker engine works
docker run hello-world

# 4. Verify the image was downloaded to D drive
docker image ls
Get-ChildItem "D:\WORK\Docker_Images" -Recurse -Depth 2

# 5. Confirm WSL distros are on D drive
wsl --list --verbose

# 6. Double check no docker data landed on C drive
Get-ChildItem "C:\Users\$env:USERNAME\AppData\Local\Docker" -ErrorAction SilentlyContinue
```

> **✅ Expected results:**
> - `Docker Root Dir: D:\WORK\Docker_Images` in docker info
> - `hello-world` image pulls and runs successfully
> - WSL distros list shows `docker-desktop` and `docker-desktop-data` both at D: paths
> - `C:\Users\...\AppData\Local\Docker\wsl\` should be empty or not exist

---

## PHASE 07: Complete Path Summary {#phase-summary}

### Component Paths

| Component | D Drive Path | Type |
|-----------|--------------|------|
| Docker Desktop App | `D:\ProgramFiles\Docker\App` | Required |
| Docker CLI Config | `D:\ProgramFiles\Docker\config` | Required |
| Docker Data Root (images/containers) | `D:\WORK\Docker_Images` | Required |
| docker-desktop WSL2 distro | `D:\WORK\Docker_Images\wsl\docker-desktop` | Required |
| docker-desktop-data WSL2 distro | `D:\WORK\Docker_Images\wsl\docker-desktop-data` | Required |
| Ubuntu WSL2 Distro (Linux env) | `D:\ProgramFiles\WSL\Ubuntu2204-Data` | Required |
| Ubuntu WSL2 Installer Files | `D:\ProgramFiles\WSL\` | Keep or delete |
| Docker Desktop Installer .exe | `D:\ProgramFiles\Docker\DockerDesktopInstaller.exe` | Can delete |

### Dependency Reference

| Dependency | Where it comes from | Required? |
|------------|---------------------|-----------|
| WSL (Windows Subsystem for Linux) | Built into Windows — enabled via `dism` | Required |
| Virtual Machine Platform | Built into Windows — enabled via `dism` | Required |
| Hyper-V | Built into Windows — enabled via `dism` | Required |
| WSL2 Linux kernel update | Auto-installed by `wsl --update` | Required |
| Ubuntu 22.04 LTS (WSL2 distro) | Downloaded via `aka.ms/wslubuntu2204` | Required |
| Docker Desktop | Downloaded from docker.com | Required |
| BIOS Virtualization (VT-x / AMD-V) | Enabled in motherboard BIOS/UEFI | Required |

---

## 🛠️ Common Issues & Fixes

> **⚠️ "WSL2 requires an update to its kernel component"**
> — Run `wsl --update` in Administrator PowerShell, then restart.

> **⚠️ Docker Desktop starts but shows "engine not running"**
> — Run `wsl --list --verbose` and confirm both `docker-desktop` and `docker-desktop-data` are in *Stopped* state (not missing). If missing, redo Steps 12–13.

> **⚠️ Virtualization error in Docker Desktop**
> — Go to Task Manager → Performance → CPU and verify Virtualization says *Enabled*. If not, go back to BIOS and enable VT-x/AMD-V.

> **⚠️ Docker Root Dir still shows C: after config**
> — Open Docker Desktop → Settings → Docker Engine, paste the daemon.json manually, click Apply & Restart. The UI overrides the file.

> **⚠️ Hyper-V not available on Windows Home**
> — Hyper-V is only included in Windows 10/11 Pro, Enterprise, and Education. On Home edition, WSL2 backend works without Hyper-V — skip that `dism` command.

---

**Docker Desktop · Windows · D Drive Installation Guide**

All installations under `D:\ProgramFiles\` · All images under `D:\WORK\Docker_Images\`
