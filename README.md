# zdroid-spawnd

Privileged spawn daemon + Magisk module that backs the **chroot runtime adapter** in the [Zdroid Android port of Zed](https://github.com/Dylanmurzello/zed-android-port).

When the user picks **Kali chroot** in the runtime picker, every subprocess Zed wants to spawn (`bash`, `apt`, language servers, etc.) routes through a Unix socket connection to this daemon. The daemon does the `fork` + `chroot` + `setuid` + `execve` on behalf of the editor, with the right `SCM_RIGHTS` ancillary data so stdio FDs flow back cleanly.

Per-spawn cost is roughly **5 ms** (connect + sendmsg + fork/chroot/exec) vs. ~200 ms for the Magisk-`su`-mediated path the daemon replaces.

## Layout

```
zd-spawnd.c              C source, ~700 LOC of daemon (Unix socket server,
                         SCM_RIGHTS protocol, chroot/setuid/exec dispatch)
Makefile                 aarch64-linux-android cross-compile via NDK clang
PROTOCOL.md              wire format for the editor's chroot adapter client
QUEUED_RELEASE.md        notes for the next release cycle

magisk-module/           packaged Magisk module
  module.prop              version + name + author metadata
  customize.sh             Magisk install hook (sets uid, copies daemon)
  service.sh               late-boot daemon launcher (reactive to pm + ce_available)
  action.sh                Magisk Action-button shim (opens WebUI)
  uninstall.sh             clean rollback (restores chroot originals)
  chroot-init.sh           one-shot chroot setup (mounts /dev /proc /sys, etc.)
  webroot/index.html       module WebUI (status, logs, controls)
  update.json              points at this repo's releases for OTA updates
  CHANGELOG.md             per-version release notes
  README.md                module-side README
  META-INF/...             Magisk module signing scaffold
```

## Build

```sh
make ANDROID_NDK_HOME=/path/to/ndk    # produces ./zd-spawnd (aarch64 ELF)
cd magisk-module && cp ../zd-spawnd . && zip -r ../zdroid-spawnd-vX.Y.Z.zip .
```

The zip is what users flash via the Magisk app's **Install From Storage** flow, or what `update.json` ships via Magisk's built-in module updater.

## Releases

Tag scheme going forward is `vX.Y.Z` (no `zdroid-spawnd-` prefix; the repo name carries it). Pre-split releases under `zdroid-spawnd-v1.1.1` through `zdroid-spawnd-v1.1.5` lived on [zed-android-port](https://github.com/Dylanmurzello/zed-android-port/releases) and are mirrored as tags in this repo's history.

## Editor-side client

The chroot adapter that talks to this daemon lives at [`crates/zdroid_runtime/src/adapters/chroot.rs`](https://github.com/Dylanmurzello/zed-android-port/blob/main/crates/zdroid_runtime/src/adapters/chroot.rs) in the editor repo. It speaks the protocol documented in `PROTOCOL.md` here. Changes to that protocol need coordinated commits in both repos; everything else can iterate independently.
