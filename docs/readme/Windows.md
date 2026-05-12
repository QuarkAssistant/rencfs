# Windows support research

Native Windows mounting is not implemented yet. The current code keeps Windows
buildable through `src/mount/dummy.rs`; the CLI exits early from `main.rs` with
a "not yet ready" message.

This note records an implementation path that can be reviewed before code is
added. It intentionally does **not** claim that rencfs mounts on Windows today.

## Recommendation

Use [WinFsp](https://winfsp.dev/) as the native Windows mount backend.

Reasons:

- WinFsp is the maintained Windows equivalent of FUSE: it exposes a user-mode
  filesystem host backed by a Windows kernel driver.
- The Rust [`winfsp`](https://crates.io/crates/winfsp) crate provides safe Rust
  bindings and examples through `winfsp-rs`.
- WinFsp supports the drive-letter/directory-mount workflow expected by Windows
  users, while letting rencfs keep its existing encrypted storage layer.

A WebDAV, Dokan, or WSL-only route is less attractive:

- WebDAV would change filesystem semantics and add a network server surface.
- Dokan is viable, but rencfs already has a FUSE-like design that maps more
  directly to WinFsp.
- WSL is useful for development, but it is not native Windows support.

## Licensing checkpoint

WinFsp is GPLv3 with a FLOSS exception, and commercial licenses are available.
Because rencfs is currently `MIT OR Apache-2.0`, this should be treated as an
explicit maintainer decision before shipping binaries.

If rencfs links to or redistributes WinFsp under the FLOSS exception, user-facing
docs and any UI should include the WinFsp attribution required by the exception:

> WinFsp - Windows File System Proxy, Copyright (C) Bill Zissimopoulos
> https://github.com/winfsp/winfsp

The first implementation PR can avoid redistribution by requiring users and CI
to install WinFsp separately, but dependency licensing should still be reviewed
before release packaging.

## Proposed dependency shape

Add the dependency only for Windows targets:

```toml
[target.'cfg(target_os = "windows")'.dependencies]
winfsp = { version = "0.12", default-features = false, features = ["delayload", "windows-61"] }

[target.'cfg(target_os = "windows")'.build-dependencies]
winfsp = { version = "0.12", default-features = false, features = ["build"] }
```

Add `build.rs` or a Windows-only build script path that emits WinFsp delay-load
link flags via `winfsp::build::winfsp_link_delayload()`. The crate documentation
states this is required for `winfsp-rs`.

Use the `system` feature only if the project wants to link against an installed
WinFsp library during build. For CI and contributor ergonomics, delay-loading is
usually friendlier because the executable can produce a clear runtime error when
WinFsp is missing instead of failing at link time.

## Module layout

Keep the current platform split:

```text
src/mount.rs
src/mount/linux.rs       # existing fuse3 backend
src/mount/windows.rs     # new WinFsp backend
src/mount/dummy.rs       # unsupported non-Linux/non-Windows targets
```

`src/mount.rs` should select backends like this:

```rust
#[cfg(target_os = "linux")]
mod linux;
#[cfg(target_os = "windows")]
mod windows;
#[cfg(not(any(target_os = "linux", target_os = "windows")))]
mod dummy;
```

Do not add a separate Windows CLI parser unless the command line is genuinely
different. The Linux path already goes through `run::run()`, Clap, and
`create_mount_point()`. Reusing that path avoids option drift.

## Runtime architecture

The Windows backend should be a thin adapter around `EncryptedFs`, similar to
`EncryptedFsFuse3`:

1. `MountPointImpl::new(...)` stores mount options.
2. `MountPointImpl::mount()` creates `EncryptedFs::new(...)` with the supplied
   password provider, cipher, and read-only flag.
3. A `WindowsEncryptedFs` context owns `Arc<EncryptedFs>` plus a Tokio runtime or
   runtime handle.
4. WinFsp callbacks synchronously bridge into rencfs async methods with one
   controlled runtime boundary.
5. `MountHandleInnerImpl` owns the WinFsp host/service handle and unmounts it in
   `unmount()`.

Avoid these pitfalls:

- Do not create a Tokio runtime per filesystem callback.
- Do not spawn a thread on every `Future::poll`; that leaks threads and can hang.
- Do not return success for unimplemented callbacks. Return explicit Windows
  errors for unsupported operations.
- Do not parse a second ad-hoc Windows CLI; keep Clap as the single interface.

## Operation mapping

The first useful milestone should support regular files and directories only.
That matches the current `FileType` enum and avoids pretending to support NTFS
features rencfs does not model yet.

- WinFsp `GetSecurityByName` / `GetFileInfo` -> `EncryptedFs::lookup()` and
  `EncryptedFs::get_attr()`.
- WinFsp `Open` -> `EncryptedFs::open()` with read/write flags derived from
  requested access.
- WinFsp `Create` -> `EncryptedFs::create()` using `CreateFileAttr`.
- WinFsp `Read` -> `EncryptedFs::read()`.
- WinFsp `Write` -> `EncryptedFs::write()`.
- WinFsp `Overwrite` / `SetFileSize` -> `EncryptedFs::set_attr()` with
  `SetFileAttr::with_size(...)`.
- WinFsp `ReadDirectory` -> `EncryptedFs::read_dir_plus()` or `read_dir()`.
- WinFsp `Rename` -> `EncryptedFs::rename()`.
- WinFsp `Cleanup` / `Close` -> `EncryptedFs::release()`.
- WinFsp `Unlink` / `RemoveDirectory` -> `EncryptedFs::remove()`.

Unsupported initially:

- reparse points and symlinks;
- alternate data streams;
- extended attributes;
- hard links;
- Windows ACL preservation beyond a simple default descriptor;
- case-insensitive lookup compatibility if encrypted names remain
  case-sensitive.

These should return clear unsupported-operation errors until the storage layer
has matching semantics.

## Windows metadata decisions

rencfs stores Unix-style uid/gid/permission bits in `FileAttr`. Windows needs a
small compatibility layer rather than a lossy one-off conversion in each
callback.

Recommended first pass:

- Treat directories as `FILE_ATTRIBUTE_DIRECTORY`.
- Treat files with no write bit as `FILE_ATTRIBUTE_READONLY`.
- Use `FileAttr::{atime,mtime,ctime,crtime}` for Win32 file times.
- Set a default security descriptor for the mounted volume instead of trying to
  round-trip arbitrary ACLs.
- Document that ACL preservation is not part of the first milestone.

## Build and CI plan

The current GitHub Actions matrix already includes `windows-latest`, but tests
are skipped there. For the Windows implementation branch:

1. Keep `cargo build --all-targets --all-features` on `windows-latest`.
2. Add a Windows-only check that verifies the binary prints a useful error when
   WinFsp is not installed.
3. Add a manual smoke test job or checklist for a runner with WinFsp installed.
   GitHub-hosted Windows runners should not be assumed to have the WinFsp driver.

A local macOS/Linux cross-compile is not enough for this project because native
Windows dependencies such as `ring` and WinFsp linking need MSVC/Windows SDK
behavior. Use it only as an early syntax check, not as release evidence.

## Manual smoke test

On a Windows machine with Rust and WinFsp developer files installed:

```powershell
rustup default nightly
cargo build --release
mkdir data
$env:RENCFS_PASSWORD = "test-password"
cargo run --release -- mount --mount-point X: --data-dir .\data
```

In another PowerShell session:

```powershell
Set-Content X:\hello.txt "hello"
Get-Content X:\hello.txt
New-Item -ItemType Directory X:\dir
Set-Content X:\dir\nested.txt "nested"
Get-ChildItem X:\ -Recurse
Remove-Item X:\hello.txt
Remove-Item X:\dir\nested.txt
Remove-Item X:\dir
```

Then unmount, mount again with the same password, and verify that persisted files
are readable and that `data` does not contain plaintext filenames or contents.

## Milestones

1. **Compile-only adapter:** add Windows cfg wiring, WinFsp dependency, build
   script, and a `windows.rs` that mounts only far enough to return a controlled
   "not implemented" error from each callback. CI must pass on `windows-latest`.
2. **Read-only filesystem:** implement lookup, getattr, open, read, and
   directory listing against an existing encrypted data directory.
3. **Writable regular files:** implement create, write, truncate, rename,
   delete, and release.
4. **Windows semantics hardening:** file times, read-only attribute, case
   behavior, error-code mapping, cleanup/close behavior, and explicit unsupported
   paths.
5. **Packaging:** installer notes, WinFsp attribution, and release validation.

## PR acceptance criteria

A first implementation PR should be considered ready only when it can truthfully
check these boxes:

- The project builds on `windows-latest` with the Windows backend enabled.
- The PR does not add a separate Windows CLI surface unless justified.
- Unsupported operations return explicit errors, not success placeholders.
- The smoke test above has been run on a Windows host with WinFsp installed.
- The PR documents whether WinFsp was linked, delay-loaded, or required at
  runtime.
- AI assistance, if used, is disclosed in the PR body.
