# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Debi is a single-file POSIX shell script (`debi.sh`, ~1020 lines) that reinstalls any VPS or physical machine to minimal Debian via network boot. It injects the Debian installer into GRUB and automates the installation with preseed configuration.

## Commands

```bash
# Syntax check (fast, always run after edits)
sh -n debi.sh

# Lint (run after syntax check passes)
shellcheck debi.sh

# Dry run — generates preseed config + GRUB entry to stdout without touching the system
sudo ./debi.sh --dry-run
```

There is no test suite. Verification is `sh -n` + `shellcheck` + manual testing in a VM.

## Architecture

The script is a single linear pipeline — no sourced files, no subshells, no background jobs. It runs top-to-bottom in these phases:

1. **Utility functions**: `err`, `warn`, `download`, `in_target`, `configure_sshd`, `prompt_password`, `set_suite`, `set_debian_version`, `has_cloud_kernel`, `has_backports`.
2. **Variable defaults**: All configuration variables initialized with defaults. `set_debian_version 13` is called inline to set `$suite` and its derived properties.
3. **Argument parsing**: Single `while/case` loop. Each `--flag` directly mutates the corresponding variable. Flags that take a value do `shift` to consume the argument. Overrides that must win regardless of flag order (e.g. `--daily-d-i`) are stored separately and applied after the loop.
4. **Post-parse validation**: Architecture detection, kernel selection, SSH key validation, APT component assembly.
5. **Preseed generation**: Builds `preseed.cfg` by piping heredocs and echo statements through `$save_preseed` (either `cat` for dry-run or `tee -a preseed.cfg` for real runs). This is the bulk of the script. An EXIT trap removes `/boot/debian-$suite` and `zz-debi.cfg` if anything fails from here on; it is cleared at the very end.
6. **Installer preparation**: Downloads `linux` + `initrd.gz` from the mirror, injects preseed into initrd via cpio, writes GRUB config, runs `update-grub`.
7. **GRUB menu entry**: Writes the final `menuentry 'Debian Installer'` block.

### Key patterns

- **`in_target()` accumulates commands** into `$in_target_script` (`&&`-chained), which gets executed as a single `preseed/late_command` at the end of installation. Order matters — calls to `in_target` throughout the script build up this command string, and a failure in any command aborts the rest.
- **`$save_preseed` / `$save_grub_cfg`** are either `cat` (dry-run, output to stdout) or `tee -a <file>` (real run). This dual-mode pattern means the same heredocs serve both paths.
- **Version → suite resolution**: `set_debian_version()` maps version numbers/aliases to a suite, then `set_suite()` sets every suite-dependent property (`security_archive`, `daily_d_i`, `_has_backports`, `_has_nonfree_fw`) in one `case`.
- **`prompt_password` installs its own EXIT trap** and clears it, so any other EXIT trap must be set after it runs.
- **`download()`** tries wget → curl → busybox wget in order. Proxy env vars are set from `$mirror_proxy` only if no proxy env vars were already set.
- **Password hashing** tries mkpasswd → openssl → busybox mkpasswd → python (3/2) in a fallback chain.

## Development Guidelines

- **POSIX sh only** — the shebang is `#!/bin/sh` with `# shellcheck shell=dash`. No bashisms (no arrays, no `[[ ]]`, no `$(( ))` arithmetic beyond what POSIX allows, no `local` beyond simple variables).
- **When adding/removing a Debian release**: update `set_suite()` (all suite properties in one place) and `set_debian_version()` (version number → suite mapping). `has_cloud_kernel()` only needs updating if the new suite has different arch support than its neighbors.
- **When adding a new flag**: add it to the `while/case` argument parser, set a default variable near line 228, and update README.md.
- **The script modifies the host system** (GRUB config, `/boot/debian-*/`). Changes are reversible before reboot. Always test with `--dry-run` first.
