# Access a Remote Repository in VS Code with SSHFS

This guide explains how to mount a repository from an SSH server on a Windows or Linux computer, open it in Visual Studio Code, and make an SSH shell the default integrated terminal.

## How the setup works

SSHFS exposes a remote directory as either a Windows drive such as `X:` or a Linux mount point such as `~/remote-repo`. VS Code edits files through that mount, while its terminal connects directly to the server over SSH.

> Files are not copied locally. Changes made through the mounted drive modify files on the remote server.

## Values you must provide

| Placeholder | Meaning | Example |
| --- | --- | --- |
| `<username>` | SSH account | `assale02` |
| `<hostname>` | Server address | `zurada.rc.louisville.edu` |
| `<remote-path>` | Absolute remote directory | `/work/assale02` |
| `<repository>` | Repository directory | `my-project` |
| `<local-mount>` | Linux mount directory | `~/remote-repo` |
| `X:` | Windows drive letter | `X:` |

## Common prerequisites

- [Visual Studio Code](https://code.visualstudio.com/)
- An SSH client
- An SSH account on the remote server

Test the connection:

```powershell
ssh <username>@<hostname>
```

For example:

```powershell
ssh assale02@zurada.rc.louisville.edu
```

SSH key authentication is recommended to avoid repeated password prompts.

## Windows client setup

On the Windows computer where VS Code runs, install:

- [WinFsp](https://winfsp.dev/)
- [SSHFS-Win](https://github.com/winfsp/sshfs-win)
- Windows OpenSSH Client

### 1. Add a generic mounting function to PowerShell

Open your PowerShell profile:

```powershell
if (-not (Test-Path -LiteralPath $PROFILE)) {
    New-Item -ItemType File -Path $PROFILE -Force
}
notepad $PROFILE
```

Add this reusable function:

```powershell
function Mount-RemoteDirectory {
    param(
        [Parameter(Mandatory, Position = 0)]
        [ValidateNotNullOrEmpty()]
        [string]$HostName,

        [Parameter(Mandatory, Position = 1)]
        [ValidateNotNullOrEmpty()]
        [string]$UserName,

        [Parameter(Position = 2)]
        [ValidateNotNullOrEmpty()]
        [string]$RemotePath = '/',

        [Parameter(Position = 3)]
        [ValidatePattern('^[A-Za-z]:$')]
        [string]$DriveLetter = 'X:'
    )

    if (-not $RemotePath.StartsWith('/')) {
        throw 'Use an absolute remote path, such as /work/username.'
    }

    $sshfs = "$env:ProgramFiles\SSHFS-Win\bin\sshfs-win.exe"
    if (-not (Test-Path -LiteralPath $sshfs)) {
        throw "SSHFS-Win was not found at: $sshfs"
    }

    $remote = "${UserName}@${HostName}:$RemotePath"
    & $sshfs cmd -f -o 'uid=-1,gid=-1' $remote $DriveLetter
}
```

Save and reload the profile:

```powershell
. $PROFILE
```

### 2. Mount the remote directory on Windows

Generic command:

```powershell
Mount-RemoteDirectory '<hostname>' '<username>' '<remote-path>' 'X:'
```

Example using the Zurada server:

```powershell
Mount-RemoteDirectory `
    'zurada.rc.louisville.edu' `
    'assale02' `
    '/work/assale02' `
    'X:'
```

Check the mount from another PowerShell window:

```powershell
Get-ChildItem 'X:\'
```

Keep the mounting PowerShell window open. The mount runs in the foreground and disconnects when that process ends.

### 3. Open the mounted repository on Windows

If the mounted directory is the repository:

```powershell
code 'X:\'
```

If the repository is inside the mounted directory:

```powershell
code 'X:\<repository>'
```

You can also use **File > Open Folder** in VS Code.

## Linux client setup

The package name is normally `sshfs`. Install it with the package manager for your distribution.

Ubuntu or Debian:

```bash
sudo apt update
sudo apt install sshfs
```

Fedora:

```bash
sudo dnf install fuse-sshfs
```

Arch Linux:

```bash
sudo pacman -S sshfs
```

### 1. Create a local mount point

```bash
mkdir -p "$HOME/remote-repo"
```

### 2. Mount the remote directory on Linux

```bash
sshfs <username>@<hostname>:<remote-path> "$HOME/remote-repo"
```

For example:

```bash
sshfs assale02@zurada.rc.louisville.edu:/work/assale02 "$HOME/remote-repo"
```

Confirm that the files are available:

```bash
ls -la "$HOME/remote-repo"
```

Useful optional settings include reconnecting after a temporary connection loss and periodically checking the connection:

```bash
sshfs \
  -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3 \
  <username>@<hostname>:<remote-path> \
  "$HOME/remote-repo"
```

### 3. Open the mounted repository on Linux

If the mount is the repository itself:

```bash
code "$HOME/remote-repo"
```

If the repository is inside the mounted directory:

```bash
code "$HOME/remote-repo/<repository>"
```

## Make SSH the default VS Code terminal

### Windows VS Code profile

Press `Ctrl+Shift+P`, run **Preferences: Open User Settings (JSON)**, and add:

```json
{
  "terminal.integrated.profiles.windows": {
    "Remote SSH": {
      "path": "C:\\Windows\\System32\\OpenSSH\\ssh.exe",
      "args": ["-t", "<username>@<hostname>"]
    }
  },
  "terminal.integrated.defaultProfile.windows": "Remote SSH"
}
```

Replace the placeholders. For the Zurada example, the `args` entry is:

```json
"args": ["-t", "assale02@zurada.rc.louisville.edu"]
```

If the settings file already contains entries, add these two properties to its existing outer object and separate properties with commas.

Open a new terminal with `` Ctrl+` ``. The `-t` option requests an interactive SSH terminal.

### Linux VS Code profile

Open **Preferences: Open User Settings (JSON)** and add:

```json
{
  "terminal.integrated.profiles.linux": {
    "Remote SSH": {
      "path": "/usr/bin/ssh",
      "args": ["-t", "<username>@<hostname>"]
    }
  },
  "terminal.integrated.defaultProfile.linux": "Remote SSH"
}
```

The SSH executable may be in a different location. Check it with:

```bash
command -v ssh
```

Use the returned path as the profile's `path` value.

## Start the terminal in the repository

SSH normally starts in the remote home directory. Change to the repository after connecting:

```bash
cd <remote-path>/<repository>
```

To make it the automatic starting directory, change the profile's `args` value:

```json
"args": [
  "-t",
  "<username>@<hostname>",
  "cd <remote-path>/<repository> && exec $SHELL -l"
]
```

Example:

```json
"args": [
  "-t",
  "assale02@zurada.rc.louisville.edu",
  "cd /work/assale02/my-project && exec $SHELL -l"
]
```

## Connecting to Linux and Windows SSH hosts

The mounting computer and remote server do not need to use the same operating system. A Windows or Linux client can connect to either kind of SSH host, provided the host runs an SSH server and permits SFTP access.

### Remote Linux host

Use a POSIX-style absolute path:

```text
/home/username/project
/work/username/project
```

The terminal startup command from the earlier examples works with common Linux shells:

```text
cd /remote/project/path && exec $SHELL -l
```

### Remote Windows host

The Windows host must have **OpenSSH Server** installed and running. Test normal SSH and SFTP access before using SSHFS:

```powershell
ssh <username>@<windows-hostname>
sftp <username>@<windows-hostname>
```

OpenSSH for Windows exposes paths to SFTP using forward slashes. A drive path is commonly written as:

```text
/C:/Users/username/project
```

The exact root and accessible paths depend on the Windows OpenSSH/SFTP configuration. Verify the path interactively with `sftp` and its `ls` and `cd` commands before mounting it.

If the remote Windows SSH session starts PowerShell, use a PowerShell startup command in the VS Code profile instead of the Linux shell command:

```json
"args": [
  "-t",
  "<username>@<windows-hostname>",
  "Set-Location 'C:\\Users\\<username>\\<repository>'; powershell -NoExit"
]
```

## Daily workflow

1. Mount the remote directory using the Windows PowerShell function or the Linux `sshfs` command.
2. Open the mounted repository in VS Code.
3. On Windows, keep the mounting PowerShell window open.
4. Open a new VS Code terminal to connect over SSH.
5. Run Git, compilers, package managers, and development commands in the SSH terminal so they execute remotely.

## Disconnect on Windows

Close VS Code and press `Ctrl+C` in the PowerShell window running the mount. If the drive remains connected:

```powershell
net use X: /delete
```

## Unmount on Linux

Use `fusermount` on most Linux distributions:

```bash
fusermount -u "$HOME/remote-repo"
```

On systems that provide `fusermount3`, use:

```bash
fusermount3 -u "$HOME/remote-repo"
```

If neither command is available, try:

```bash
umount "$HOME/remote-repo"
```

## Troubleshooting

### SSHFS-Win is not found

The default executable path is:

```text
C:\Program Files\SSHFS-Win\bin\sshfs-win.exe
```

If installed elsewhere, update `$sshfs` in the function.

### The drive letter is already in use

Disconnect the existing mapping or pass another unused drive letter.

### The Linux mount point is busy

Close VS Code and any terminal whose current directory is inside the mount, then run the unmount command again.

### Authentication is requested repeatedly

Configure SSH key authentication and verify that `ssh <username>@<hostname>` works before mounting.

### The terminal opens locally

Run **Terminal: Select Default Profile**, select **Remote SSH**, close the old terminal, and create a new one.

### Development commands behave unexpectedly

Run development commands inside the SSH terminal. Use the mounted drive for file access and the SSH shell for execution in the remote environment.
