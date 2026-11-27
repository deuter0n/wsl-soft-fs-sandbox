# WSL Soft Filesystem Sandboxing

Keep a WSL2 Ubuntu distro from seeing your whole Windows drive by default, and expose
only the exact Windows folders you actually need — read-only unless you say otherwise.

This is done entirely with two files:

- `/etc/wsl.conf` — turns off the blanket auto-mount of every Windows drive, and turns off Windows interop
- `/etc/fstab` — mounts back the one path that has to stay reachable, read-only

## What it does

- A stray `rm -rf`, a runaway build script, or an agentic tool can't wander into `C:\Users\...`, because the path simply isn't mounted.
- The one mount that remains is `ro`, so accidental writes fail with `EROFS` instead of corrupting Windows-side files.
- No Windows `PATH` entries leak into your Linux shell, so `python`, `node` and friends always resolve to the Linux binaries.
- Windows still reaches your Linux files through `\\wsl.localhost\<distro>`. That is intended — the isolation is deliberately one-directional, so you keep editing, browsing and backing up the Linux side from Windows.

## Why this instead of a built-in or OS-level sandbox

The threat model is narrow and specific: **stop tooling inside the distro from touching unrooted
Windows files.** Nothing else needs isolating. Network access in particular should stay exactly as
it was — no allowlist, no interception, no per-domain decisions.

Most sandboxing options fail that requirement by bundling concerns together:

| Option | Why it doesn't fit |
|---|---|
| Agent/CLI built-in sandboxes (Claude Code's included) | The two halves only separate one way. Claude Code has `sandbox.filesystem.disabled` to drop filesystem isolation and keep the network sandbox, but no inverse — enabling the sandbox for its filesystem restrictions also routes traffic through its proxy, with an allowlist that starts empty. |
| Container per project | Re-plumbing toolchains, caches, credentials and editor integration for every project, to solve a problem that only exists at the Windows/Linux edge. |
| Second WSL distro, or a VM | Works, but duplicates the entire environment and still needs this same config to keep *its* `/mnt` clean. |
| Windows-side ACLs | Per-folder permission management that fights every Windows update and tool installer. |

The specific cost being avoided is proxy plumbing. Once an agent's sandbox routes traffic through a
local proxy, every tool it shells out to has to agree: `HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY` are
honoured inconsistently, and anything doing its own TLS verification needs the proxy's CA added to
its own trust store — `pip`, `npm`, `cargo`, `go`, `git`, `curl`, `apt`, `uv`, Docker and every JVM
keep separate ones. While vibecoding, that turns into a steady stream of TLS and connectivity
failures that have nothing to do with the code being written.

Doing it at the mount layer sidesteps all of that:

- The restriction lives **below every tool**, in the distro's mount table. Nothing needs per-tool configuration, and nothing can be misconfigured into bypassing it.
- It is **invisible to the network stack**. Agents, package managers and language servers see a completely ordinary internet connection.
- It is **two files**, readable in full, with an obvious revert path.
- The protection is **structural, not advisory**: the path isn't denied, it doesn't exist. Tools that ignore allowlists, sandboxing flags or working-directory restrictions still can't reach `C:\Users\...`, because there is nothing there to reach.

## What this cannot prevent

> [!WARNING]
> **This is a guardrail against accidents, not a security boundary.** It raises the cost of a
> mistake. It stops nothing that is actively trying to get out.

**`sudo` undoes the whole thing in one command.** Your default user has `sudo`, so anything running
as you can put the entire drive back:

```bash
sudo mkdir -p /mnt/win && sudo mount -t drvfs C: /mnt/win
```

Nothing here removes that ability, and nothing notices when it is used.

**Your Linux files have no protection whatsoever.** SSH keys, cloud tokens, `~/.claude`, git
credentials and all your source are fully readable and writable by anything running in the distro.
This config limits what the distro sees of Windows; it does nothing about what code does once it is
inside the distro.

**The network is wide open.** No filtering, no proxy, no allowlist — deliberately, since that is the
whole reason for the approach. Anything in the distro can reach the internet and send whatever it
can read.

**The Windows side is untouched.** Processes running as your Windows user are entirely unaffected.
Worse, `wsl.exe -u root` from Windows gets root inside the distro, so a compromised Windows account
owns the distro too.

**The mount you keep is still a live window into Windows.** `.vscode\extensions` is read-only, but
anything in the distro can read it. A path you mount is a path you have exposed.

**It is per-distro, not per-project.** Every process, container and agent in the distro gets the
same view. There is no separation between two projects running side by side.

If your threat model is hostile code rather than careless code, this is the wrong tool — use a
disposable distro or a VM.

## Setup

Both files live inside the distro and are read at boot.

### 1. `/etc/wsl.conf`

Create it if it doesn't exist:

```ini
[automount]
enabled = false
mountFsTab = true

[interop]
enabled = false
appendWindowsPath = false
```

| Line | What it does |
|---|---|
| `[automount] enabled = false` | Stops WSL from mounting every Windows drive under `/mnt`. The distro no longer sees `C:` at all. |
| `[automount] mountFsTab = true` | Keeps `/etc/fstab` processed at boot, so the entry below still mounts. Defaults to `true`; set explicitly because the whole setup depends on it. |
| `[interop] enabled = false` | No launching Windows `.exe` files from inside the distro. `explorer.exe`, `clip.exe` and `wslpath` stop working, and so does `code .` — open the folder from Windows VS Code instead and let Remote-WSL connect in. |
| `[interop] appendWindowsPath = false` | Keeps the Windows `PATH` out of the Linux `PATH`, so `python`, `node` and friends always resolve to Linux binaries. |

### 2. `/etc/fstab`

With automount off, no Windows path is mounted unless it is listed here. Add one read-only entry,
so VS Code on Windows can still read its extension directory and drive the distro over Remote-WSL:

```fstab
C:\Users\<your-windows-user>\.vscode\extensions  /mnt/c/Users/<your-windows-user>/.vscode/extensions  drvfs  defaults,ro  0 0
```

| Field | What it does |
|---|---|
| `C:\Users\<your-windows-user>\.vscode\extensions` | The exact Windows folder to expose. A bare `C:` here would mount the whole drive. |
| `/mnt/c/Users/<your-windows-user>/.vscode/extensions` | Where it appears in Linux. WSL creates it when it processes fstab at boot; if you get `mount point does not exist`, `sudo mkdir -p` it once. |
| `drvfs` | The WSL driver for Windows filesystems. |
| `defaults,ro` | Read-only. These files are only ever read from the Linux side. |
| `0 0` | No dump, no fsck. Always this for drvfs. |

### 3. Apply

`wsl.conf` and `fstab` are read at distro boot, not on login. From **Windows** (PowerShell or CMD):

```powershell
wsl --shutdown
```

Then start the distro again. `wsl --shutdown` stops every distro and the WSL VM, so close anything
you care about first.

## Verifying

```bash
# Which Windows paths are actually mounted?
mount | grep drvfs

# Expect exactly your fstab entries, each marked (ro,...):
#   C:\Users\<your-windows-user>\.vscode\extensions on /mnt/c/Users/<your-windows-user>/.vscode/extensions type 9p (ro,relatime,aname=drvfs;...)

# /mnt should be nearly empty — your mountpoints, plus WSL's own wsl/ and wslg/
ls /mnt

# Interop off: this should fail
explorer.exe .          # => command not found

# PATH should contain no /mnt/c/... entries
echo "$PATH" | tr ':' '\n' | grep '/mnt/' || echo "clean"

# Read-only enforcement
touch /mnt/c/Users/<your-windows-user>/.vscode/extensions/test   # => Read-only file system
```

## Reference

- [WSL advanced settings (`wsl.conf` / `.wslconfig`)](https://learn.microsoft.com/windows/wsl/wsl-config)
- [Working across Windows and Linux file systems](https://learn.microsoft.com/windows/wsl/filesystems)
- [Claude Code: configure the sandboxed Bash tool](https://code.claude.com/docs/en/sandboxing)

---

Tested on WSL2 with Ubuntu 26.04 LTS. `<your-windows-user>` is a placeholder — substitute your actual Windows username.
