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

1. **Utility functions** (lines 1–110): `err`, `warn`, `download`, `in_target`, `configure_sshd`, `prompt_password`, plus version/suite resolution helpers.
2. **Variable defaults** (lines 228–279): All ~50 configuration variables initialized with defaults. `set_debian_version 13` is called inline during init to set `$suite`, `$daily_d_i`, and `$security_archive`.
3. **Argument parsing** (lines 281–547): Single `while/case` loop. Each `--flag` directly mutates the corresponding variable. Flags that take a value do `shift` to consume the argument.
4. **Post-parse validation** (lines 549–598): Architecture detection, kernel selection, SSH key validation, APT component assembly.
5. **Preseed generation** (lines 600–935): Builds `preseed.cfg` by piping heredocs and echo statements through `$save_preseed` (either `cat` for dry-run or `tee -a preseed.cfg` for real runs). This is the bulk of the script.
6. **Installer preparation** (lines 940–994): Downloads `linux` + `initrd.gz` from the mirror, injects preseed into initrd via cpio, writes GRUB config, runs `update-grub`.
7. **GRUB menu entry** (lines 1005–1022): Writes the final `menuentry 'Debian Installer'` block.

### Key patterns

- **`in_target()` accumulates commands** into `$in_target_script` (semicolon-separated), which gets executed as a single `preseed/late_command` at the end of installation. Order matters — calls to `in_target` throughout the script build up this command string.
- **`$save_preseed` / `$save_grub_cfg`** are either `cat` (dry-run, output to stdout) or `tee -a <file>` (real run). This dual-mode pattern means the same heredocs serve both paths.
- **Version → suite resolution** is a two-step chain: `set_debian_version()` → `set_suite()` → `set_daily_d_i()` + `set_security_archive()`. All three must stay in sync when adding/removing Debian releases.
- **`download()`** tries wget → curl → busybox wget in order. Proxy env vars are set from `$mirror_proxy` only if no proxy env vars were already set.
- **Password hashing** tries mkpasswd → openssl → busybox mkpasswd → python (3/2) in a fallback chain.

## Development Guidelines

- **POSIX sh only** — the shebang is `#!/bin/sh` with `# shellcheck shell=dash`. No bashisms (no arrays, no `[[ ]]`, no `$(( ))` arithmetic beyond what POSIX allows, no `local` beyond simple variables).
- **When adding/removing a Debian release**: update every `case` statement that matches suite names — there are ~7 of them (`set_debian_version`, `set_suite`/`set_security_archive`, `set_daily_d_i`, `has_cloud_kernel`, `has_backports`, `non_free_firmware_available`). Grep for the suite name to find them all.
- **When adding a new flag**: add it to the `while/case` argument parser, set a default variable near line 228, and update README.md.
- **The script modifies the host system** (GRUB config, `/boot/debian-*/`). Changes are reversible before reboot. Always test with `--dry-run` first.
