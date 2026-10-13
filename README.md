# delete-locked-file

**Delete a locked file — or the currently running executable — from disk on Windows.**

A tiny, dependency-free C utility that removes files that Windows refuses to delete: files held open by another process, executables that are still running, and DLLs/artifacts that trigger *"The action can't be completed because the file is open in another program"*, `Access is denied`, or a sharing violation.

Unlike `MoveFileEx(..., MOVEFILE_DELAY_UNTIL_REBOOT)` this tool does **not** wait for a reboot, and unlike `handle.exe`/`Process Explorer` it needs **no** prior knowledge of which process holds the lock.

![Platform: Windows](https://img.shields.io/badge/platform-Windows%2010%20%7C%2011-0078D6?logo=windows&logoColor=white)
![Language: C](https://img.shields.io/badge/language-C-555555?logo=c&logoColor=white)
![Build: MSVC](https://img.shields.io/badge/build-MSVC%20cl.exe-5C2D91?logo=visualstudio&logoColor=white)
![License: MIT](https://img.shields.io/badge/license-MIT-green)

> Forked from the original research PoC. The technique was discovered by **Jonas Lykkegaard**; this repository keeps the minimal single-file implementation, tracks the Windows 11 24H2 behaviour change, and ships a ready-to-run static binary.

---

## Table of contents

- [What problem does it solve?](#what-problem-does-it-solve)
- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Build](#build)
- [Usage](#usage)
- [Verification](#verification)
- [How this compares to other unlock tricks](#how-this-compares-to-other-unlock-tricks)
- [Limitations and known behaviour](#limitations-and-known-behaviour)
- [FAQ](#faq)
- [Credits and acknowledgements](#credits-and-acknowledgements)
- [中文说明](#中文说明)
- [Disclaimer](#disclaimer)

---

## What problem does it solve?

A Windows file can be impossible to delete even when you are an administrator:

| Symptom | Typical cause |
| --- | --- |
| `The action can't be completed because the file is open in another program` | Another process holds a handle with a restrictive share mode |
| `Access is denied` while deleting a running `.exe` | The image is mapped into a live process |
| Locked `.dll`, `.sys`, log or cache file | Loader/AV/backup agent keeps a handle open |
| Malware or an installer that "self-deletes" | The payload needs to remove its own image |

`delete-locked-file.exe <path>` renames the file's primary data stream and then marks it for POSIX-semantics deletion, so the directory entry disappears immediately — even though the file is still open and executing.

## How it works

The trick relies on the fact that `SetFileInformationByHandle` only needs `DELETE` access, and that a lock is normally attached to a *file name*, not to the file object itself.

1. **Open a handle with `DELETE` access only.**
   `CreateFileW(path, DELETE, 0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL)` — no `GENERIC_READ`/`GENERIC_WRITE` required, so an already-locked file can still be opened.
2. **Rename the primary file stream.**
   `SetFileInformationByHandle(h, FileRenameInfo, ...)` renames `:$DATA` to a private stream name (`:dlf` in this implementation). This is the pivotal step: the original name is freed while the open handle keeps working.
3. **Close the handle.**
4. **Reopen the file and set the delete disposition.**
   `SetFileInformationByHandle(h, FileDispositionInfoEx, ...)` with `FILE_DISPOSITION_FLAG_DELETE | FILE_DISPOSITION_FLAG_POSIX_SEMANTICS`.
5. **Close the handle to trigger deletion.**
   With POSIX semantics the link is removed immediately and the storage is reclaimed once the last open handle goes away.
6. **Verify.**
   `PathFileExistsW` confirms the file is gone from the namespace.

The whole flow lives in [`delete-locked-file.c`](delete-locked-file.c) and is only a few dozen lines of readable Win32 — no shellcode, no drivers, no kernel tricks.

## Requirements

- Windows 10 / Windows 11 (x64 releases are provided; the source builds for x86 and ARM64 too)
- Permission to open the target with `DELETE` access — i.e. administrator for files owned by other users or protected services
- To build: MSVC (`cl.exe`), any recent Visual Studio or the Build Tools
- No runtime dependencies: the release binary is statically linked (`/MT`) and only needs `Shlwapi.lib` at link time

## Build

Using a Developer Command Prompt:

```bat
cl /nologo /O2 /MT /W4 delete-locked-file.c /link Shlwapi.lib /out:delete-locked-file.exe
```

The `#pragma comment(lib, "Shlwapi.lib")` at the top of the source means the link flag is optional when compiling with MSVC. A subsystem/`wmain` entry point is provided, so the tool handles Unicode paths and long file names correctly.

Prefer not to compile? Grab the prebuilt static binary from this repository: [`delete-locked-file.exe`](delete-locked-file.exe).

## Usage

```bat
delete-locked-file.exe <path_to_file>
```

Example — remove an executable that is currently running, and a DLL held by a service:

```bat
:: note: quote paths that contain spaces
delete-locked-file.exe "C:\Windows\Temp\payload.exe"
delete-locked-file.exe "C:\ProgramData\Vendor\Agent\locked_module.dll"

:: check the exit code in a script (1 = deleted, 0 = failure)
delete-locked-file.exe "C:\temp\locked.bin" && echo DELETED || echo FAILED
```

Sample output:

```
[LOG] - attempting to rename file name
[LOG] - successfully renamed file primary :$DATA ADS to specified stream, closing initial handle
[LOG] - closing handle to trigger deletion deposition
[LOG] - successfully deleted target file from disk
```

Run it from an **elevated** prompt when the target is owned by another account, a service, or TrustedInstaller-managed infrastructure.

## Verification

After the run the file is gone from Explorer, `dir`, and `Get-ChildItem`. To confirm it really left the filesystem, check that the path no longer resolves and that no residual alternate data streams remain:

```bat
dir /a "C:\path\to\target.exe"
dir /r "C:\path\to"
```

## How this compares to other unlock tricks

| Approach | Needs the locking process? | Reboot? | Works on a running `.exe`? |
| --- | --- | --- | --- |
| **`delete-locked-file` (this repo)** | No | No | Yes |
| `MoveFileEx` + `MOVEFILE_DELAY_UNTIL_REBOOT` | No | **Yes** | Only after reboot |
| `handle.exe` / Process Explorer → close handle | Yes | No | Sometimes |
| Restart Manager API (`RmShutdown`) | Yes (identifies it) | No | Yes, but may terminate the app |
| Kernel driver / filter | No | No | Yes, higher risk |

## Limitations and known behaviour

- **Windows 11 24H2 and later changed self-deletion behaviour.** This build uses `FileDispositionInfoEx` with `FILE_DISPOSITION_FLAG_POSIX_SEMANTICS` to stay compatible; validate against your target OS build before relying on it in automation.
- Requires `DELETE` access on the target. If the ACL denies it, the tool fails and logs `failed to acquire handle to target file` — elevate or take ownership first.
- The storage is only reclaimed after the last remaining handle is closed; the *name* disappears immediately.
- Files on network shares, ReFS with active snapshots, or protected by an EDR/AV kernel filter may behave differently. Expect `failed to set delete deposition` in those cases.
- Antivirus and EDR products frequently flag this pattern. Test in an isolated environment.
- Memory-mapped images can be deleted by name while remaining resident until unmapped — the classic "deleted but still running" state.

## FAQ

**Do I need to know which process holds the lock?**
No. That is the whole point — the technique operates purely on the file name and does not enumerate handles.

**Does it force-close the other process's handle?**
No. It never touches the other process. It removes the directory entry; the owner keeps its handle to the now-unnamed file until it exits.

**Does it work for folders?**
No, this targets regular files/streams.

**Why rename to `:dlf`?**
Any name works. Renaming the `:$DATA` stream is what detaches the locked name from the handle.

**Is this a Windows vulnerability?**
It is documented Win32 behaviour (`FILE_DISPOSITION_FLAG_POSIX_SEMANTICS`), not a memory-corruption bug. It is nonetheless a well-known defence-evasion pattern and is treated as such by security vendors.

## Credits and acknowledgements

- **Jonas Lykkegaard** — original discovery of the delete-while-locked technique.
- **Lloyd** / [LloydLabs](https://github.com/LloydLabs/delete-self-poc) — the original proof-of-concept this repository is forked from (MIT, 2021).
- This fork: maintained independently, with a static release binary and notes on modern Windows builds.

Licensed under the [MIT License](LICENSE). The original copyright notice is preserved.

## 中文说明

**用途：** 在 Windows 上删除正在被占用、正在运行、或提示「文件已在另一个程序中打开 / 拒绝访问」的文件或可执行程序。

**原理简述：** 先用 `DELETE` 权限打开文件句柄（不需要读写权限，所以即使被占用也能打开），通过 `SetFileInformationByHandle(FileRenameInfo)` 把主数据流 `:$DATA` 改名，再以 `FileDispositionInfoEx`（`FILE_DISPOSITION_FLAG_DELETE` + `FILE_DISPOSITION_FLAG_POSIX_SEMANTICS`）设置删除标记并关闭句柄，文件名立即消失，无需重启。

**编译：**

```bat
cl /nologo /O2 /MT /W4 delete-locked-file.c /link Shlwapi.lib /out:delete-locked-file.exe
```

**使用（建议以管理员身份运行）：**

```bat
delete-locked-file.exe <文件路径>
delete-locked-file.exe "C:\temp\locked.exe"
```

**注意：** 需要对目标文件具备 `DELETE` 权限；Windows 11 24H2 及以上版本已调整自我删除相关行为，本实现使用 `FileDispositionInfoEx` 以保持兼容；EDR / 杀毒软件可能会拦截此类行为，请在隔离环境中测试。文件仅剩最后一个句柄未关闭时才会真正释放磁盘空间，但文件名会立刻消失。

## Disclaimer

This project is published for **defensive research, incident response, malware analysis, and general file-management troubleshooting**. Only run it against files you own or are explicitly authorised to remove — deleting files out of a running system can cause data loss, service outages, or an unbootable machine. The authors accept no liability for misuse or damage.
