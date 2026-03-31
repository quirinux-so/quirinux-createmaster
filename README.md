# CreateMaster

**CreateMaster** is a collection of Bash scripts designed to automate the full lifecycle of a custom GNU/Linux live distribution: from building a master installation, through cleaning it up and packaging it as an ISO, to post-processing the resulting image for maximum compression.

It was originally developed for [Quirinux](https://www.quirinux.org), a Devuan-based distribution, but its architecture is intentionally modular and can be adapted to other Debian/Devuan derivatives.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Directory Structure](#directory-structure)
- [Commands](#commands)
  - [createmaster](#createmaster)
  - [cleanmaster](#cleanmaster)
  - [createiso](#createiso)
  - [recomp](#recomp)
- [Supporting Infrastructure](#supporting-infrastructure)
  - [functions](#functions)
  - [var\_distro](#var_distro)
  - [Language system](#language-system)
  - [Modules](#modules)
- [Typical Workflow](#typical-workflow)
- [Requirements](#requirements)
- [License](#license)

---

## Project Overview

The four commands work as a pipeline. A developer starts from a minimal base system and ends up with a distributable, optimally compressed hybrid ISO:

```
createmaster  →  cleanmaster  →  createiso  →  recomp
     │                │               │             │
Build the         Strip bloat     Package the   Re-squeeze
master system     and temp files  ISO with eggs the squashfs
```

Each command is a standalone executable placed in `/usr/local/bin/`. They share a common library at `/opt/createmaster/` that holds global functions, distribution variables, language files, and software modules.

---

## Directory Structure

```
/usr/local/bin/
├── createmaster        # Interactive TUI to build the master system
├── cleanmaster         # Deep cleanup before generating the ISO
├── createiso           # ISO generation wrapper around Penguin's Eggs
└── recomp              # Post-generation ISO recompressor

/opt/createmaster/
├── var_distro          # Distribution-specific variables (name, version, paths…)
├── functions           # Shared utility functions (root check, update, etc.)
├── lang/
│   ├── lang_createmaster   # UI strings for createmaster
│   ├── lang_createiso      # UI strings for createiso
│   ├── lang_cleanmaster    # UI strings for cleanmaster
│   └── lang_recomp         # UI strings for recomp
├── mods/
│   ├── accessories/        # Module files (e.g. kruler_mod, galculator_mod)
│   ├── audio/
│   └── …                   # One subdirectory per category
├── requirements/           # Marker files that track completed steps
│   ├── ok-depends          # Created after dependencies are installed
│   ├── ok-standar          # Created after standard modules are installed
│   └── ok-specific         # Created after specific modules are installed
└── verif/
    ├── ok-lang             # Created after locale configuration
    └── ok-dependencias     # Created after base dependency installation
```

---

## Commands

### createmaster

**Purpose:** Interactive TUI (text user interface) that turns a bare Devuan/Debian installation into a fully configured master distribution ready to produce live ISOs.

**Usage:**
```bash
sudo createmaster
```

**How it works:**

The script uses `dialog` to present a menu-driven interface. On first launch it checks for internet connectivity and installs any missing dependencies before showing the main menu. A splash screen and copyright line are generated dynamically, adapting to the terminal width.

The main menu offers nine options:

| Option | Action |
|--------|--------|
| 1 | Remove components listed in `REMOVE_MODS` |
| 2 | Install all standard modules (`ESTANDAR_MODS`) in one go |
| 3 | Install all specific modules (`ESPECIFIC_MODS`) in one go |
| 4 | Pick and install individual standard modules (checklist) |
| 5 | Pick and install individual specific modules (checklist) |
| 6 | Full installation without removing anything first |
| 7 | Full installation — removes unwanted components first |
| 8 | Show help |
| 9 | Exit |

**Module system:**

Software is grouped into *modules*. A module is a small variable file that declares three things:

```bash
MOD_NAME="galculator_mod"
MOD_PACKS="galculator"
MOD_MENU="Calculator (Galculator)"
```

- `MOD_NAME` — internal identifier.
- `MOD_PACKS` — space-separated list of packages to install or remove.
- `MOD_MENU` — human-readable label shown in the checklist dialogs.

Modules are stored under `/opt/createmaster/mods/<category>/`. The script searches all subdirectories recursively, so adding a new category is just a matter of creating a new folder and dropping module files into it.

Three module lists are defined at the top of the script:

```bash
ESTANDAR_MODS=("audio_mod" "printers_mod")
ESPECIFIC_MODS=("kruler_mod" "galculator_mod" "kdenlive_mod")
REMOVE_MODS=("libreoffice_mod")
```

Edit these arrays to customise what your derivative distribution installs or removes.

**Marker files:**

To avoid reinstalling packages on subsequent runs, the script creates empty marker files under `/opt/createmaster/requirements/`:

- `ok-standar` — standard modules have been installed.
- `ok-specific` — specific modules have been installed.
- `ok-depends` — base dependencies are present.

---

### cleanmaster

**Purpose:** Deep unattended cleanup of the master system to reduce the size of the resulting live ISO.

**Usage:**
```bash
sudo cleanmaster
```

**How it works:**

The script records disk usage before and after so it can report how much space was recovered. It then runs the following steps in order:

| Step | Function | What it does |
|------|----------|--------------|
| 1 | `_package_cleanup` | Runs the appropriate package manager's clean/autoremove commands and wipes `/var/lib/apt/lists/` |
| 2 | `_bash_history` | Deletes `~/.bash_history` |
| 3 | `_trash_cleanup` | Empties user caches, recent-files lists and Trash folders for every account under `/home/` and for root |
| 4 | `_log_cleanup` | Removes system log files and clears the kernel ring buffer |
| 5 | `_deep_cleanup` | Deletes `*.bak`, `*~` and VIM swap files from all home directories |
| 6 | `_lang` | Removes locale data for languages other than: en, de, es, fr, it, gl, pt |
| 7 | `_doc` | Removes `/usr/share/doc` (except `quirinux*` entries) and `/usr/share/info` |
| 8 | `_man` | Removes all manual pages from `/usr/share/man` |

> **Note:** Steps 6, 7 and 8 are destructive by design — they are intended for a *master system whose only purpose is producing ISOs*, not for a daily-use machine. Any of these three steps can be disabled by commenting out the corresponding call in the "Function Execution" section at the bottom of the script.

Multi-distro package manager support: the script automatically detects `apt-get`, `dnf`, `pacman` or `zypper` and invokes the right commands.

---

### createiso

**Purpose:** Automates the generation of a live ISO from the master system using [Penguin's Eggs](https://penguins-eggs.net/).

**Usage:**
```bash
sudo createiso           # Standard (pendrive-bootable) ISO
sudo createiso -f        # Forced/flat ISO (no pendrive mode)
sudo createiso --test    # Dry run — no files are modified
```

**How it works:**

The script runs the following functions in sequence:

1. **`_rootcheck`** — Aborts if not run as root.
2. **`_internet`** — Aborts if there is no internet access.
3. **`_clean`** — Calls `cleanmaster` to strip the system before packaging.
4. **`_rev`** — Prompts the operator to keep or update the revision number embedded in the Calamares branding file (`show.qml`). Revision 0 means no revision tag is appended.
5. **`_calamares`** — Updates the version string and optional revision label inside the two copies of `show.qml` (wardrobe copy and live system copy). Uses `sed` to rewrite the `<h1>` title and the `<b>Version:</b>` line in place.
6. **`_eggsyaml`** — Detects the latest installed kernel under `/boot/vmlinuz-*` and rewrites the `initrd_img` and `vmlinuz` keys in `/etc/penguins-eggs.d/eggs.yaml`.
7. **`_createiso`** — Calls `eggs produce` with the distribution's wardrobe theme. Passes `--pendrive` by default or `-f` if requested.
8. **`_isoname`** — Renames the generated ISO file to the project naming convention: `<name>-<version>-<release>_x64[_Rev<N>].iso` and moves it to the user's desktop.
9. **`_md5sum`** — Generates a `.md5` checksum file alongside the ISO.
10. **`_bye`** — Prints a success message.

**Test mode:**

Passing `--test` as the first argument activates a dry run. Every destructive step prints what it *would* do without making any changes. This is useful for verifying the configuration before a real build.

---

### recomp

**Purpose:** Takes an existing hybrid live ISO and recompresses its internal squashfs filesystem using the `xz + bcj x86` algorithm, producing a significantly smaller image while preserving full hybrid (BIOS + UEFI) boot capability.

**Usage:**
```bash
sudo recomp <file.iso>
```

The output file is written to the same directory as the input, with `-xz` appended to the base name:
```
quirinux-2.0-stable_x64.iso  →  quirinux-2.0-stable_x64-xz.iso
```

**How it works — step by step:**

```
_preflight      Validate arguments, file extension and required commands
     │
_detect_workbase  Auto-select the partition with most free space for temp work
     │
_set_variables  Calculate paths and verify ~3× the ISO size is available
     │
_step1_mbr      Extract the first 432 bytes of the ISO (hybrid MBR)
     │
_step2_copy     Mount the ISO loop and copy its full tree to a work directory;
                locate and remove the original filesystem.squashfs
     │
_step3_decompress  Unpack the squashfs with unsquashfs, then unmount the ISO
     │
_menu           Interactive pause: enter chroot to edit the system,
                proceed to recompression, or abort and clean up
     │
_step4_compress  Repack with mksquashfs using xz + bcj x86, 1 MB blocks
     │
_step5_metadata  Update filesystem.size and regenerate md5sum.txt
     │
_step6_build    Rebuild the ISO with xorriso reusing the original boot
                parameters and volume ID; inject the saved MBR back
     │
_summary        Delete temp files and print original vs. new sizes
```

**Chroot menu:**

After decompressing the squashfs but before recompressing it, the script pauses and shows an interactive menu with three options:

- **Option 1 — Enter chroot:** Opens a shell inside the unpacked squashfs root with all necessary bind-mounts (`/dev`, `/dev/pts`, `/proc`, `/sys`, `/run`) and optional X11 forwarding. The prompt is relabelled to `root@ISO-QUIRINUX:/#` to make it visually clear the operator is inside the chroot. Typing `exit` returns to the menu without stopping the script.
- **Option 2 — Continue:** Unmounts the chroot binds and proceeds to recompression.
- **Option 3 — Exit:** Cleans up all temporary files and exits without producing a new ISO.

**Work directory selection:**

Instead of hardcoding a temp path, `recomp` inspects all mounted real filesystems and picks the one with the most available space. This avoids failures on systems where `/` is small but a secondary disk has plenty of room.

**Required tools:** `xorriso`, `unsquashfs`, `mksquashfs`, `mount`, `umount`, `dd`, `chroot`. The preflight check verifies all of them and tells the operator which package to install if any is missing.

---

## Supporting Infrastructure

### functions

`/opt/createmaster/functions` is sourced by all four commands. It provides:

| Function | Purpose |
|----------|---------|
| `_rootcheck` | Exits with an explanatory message if `$EUID` is not 0 |
| `_internet` | Pings google.com and exits if there is no connectivity |
| `_update` | Detects `apt-get`, `dnf` or `yum` and runs a system update |
| `_depends` | Installs base packages (`wget`, `git`, `dialog`, etc.) and any additional `.deb`/`.rpm` packages listed in `$DISTRO_DEPENDS_LOCATION`; creates a marker file when done |
| `_lang` | Generates and activates the required system locales; creates a marker file when done |
| `_verif_path` | Confirms the script is running on the expected master system path (`$DISTRO_PATH`) |
| `_quit` | Clears the screen and exits cleanly |

The file also sources `/opt/createmaster/var_distro` to make the distribution variables available to every consumer.

### var_distro

`/opt/createmaster/var_distro` is a plain variable file (not shown in this repository) that defines the identity of the target distribution. At minimum it is expected to export:

| Variable | Example | Purpose |
|----------|---------|---------|
| `DISTRO_NAME` | `quirinux` | Short lowercase name used in filenames and wardrobe paths |
| `DISTRO_VERSION` | `2.0` | Version string embedded in the ISO name and Calamares branding |
| `DISTRO_RELEASE` | `stable` | Release tag appended to the ISO name |
| `DISTRO_DATE_RELEASE` | `2025` | Year or date shown in the Calamares installer |
| `DISTRO_PATH` | `/home/user/quirinux` | Path to the master distribution's working tree |
| `DISTRO_SPLASH` | `"…"` | ASCII art shown on the createmaster welcome screen |
| `DISTRO_DEPENDS_LOCATION` | `(…)` | Array of local or remote paths where extra `.deb`/`.rpm` packages are fetched from |

### Language system

Every script exposes user-facing strings through a `_msg KEY` function defined in its corresponding `lang_*` file. The function holds an associative array keyed by `<language-code>_<KEY>`:

```bash
[es_PKG_CLEANUP]="Limpiando cachés de paquetes..."
[en_PKG_CLEANUP]="Cleaning package caches..."
[fr_PKG_CLEANUP]="Nettoyage des caches de paquets..."
```

At runtime, `_msg` reads the first two characters of `$LANG` and looks up the matching entry, falling back to `en_` if the current locale is not covered. Adding a new language requires adding one line per key.

Supported languages: **es** (Spanish), **en** (English), **it** (Italian), **de** (German), **fr** (French), **gl** (Galician), **pt** (Portuguese).

### Modules

A module is a text file that lives anywhere under `/opt/createmaster/mods/`. It must define exactly three variables:

```bash
MOD_NAME="kruler_mod"
MOD_PACKS="kruler"
MOD_MENU="Screen ruler (KRuler)"
```

`createmaster` discovers modules dynamically by searching all subdirectories, so the category folder structure is purely organisational — it has no effect on how modules are loaded. To add a new piece of software to the distribution, create a new module file and add its name to the appropriate array (`ESTANDAR_MODS` or `ESPECIFIC_MODS`) in `createmaster`.

---

## Typical Workflow

Below is the sequence a maintainer follows to produce a new release:

```
1.  Boot into the master system (a Devuan/Debian installation).

2.  sudo createmaster
        → Use the TUI to install or update software modules.
        → Option 7 performs a full build (removes old components, installs new ones).

3.  sudo createiso
        → Enter the revision number when prompted.
        → The script cleans the system, updates Calamares branding,
          runs eggs produce, renames the ISO and generates an MD5.

4.  sudo recomp /path/to/generated.iso # Optional, to modify details and/or achieve maximum compression.
        → The script decompresses, optionally lets you enter a chroot
          to make last-minute fixes, then recompresses with xz + bcj x86.
        → The result is a smaller ISO in the same directory.
```

Steps 3 and 4 can be repeated independently without going through `createmaster` again, as long as the master system has not changed.

---

## Requirements
| Dependency | Used by | Notes |
|------------|---------|-------|
| `bash` ≥ 4.0 | all | Associative arrays require Bash 4+ |
| `dialog` | createmaster | TUI menus |
| `penguins-eggs` or `eggs` | createiso | ISO generation engine — download at [repo.quirinux.org](https://repo.quirinux.org/pool/main/p/penguins-eggs/) |
| `eggs-quirinux-config` | createiso | Quirinux-specific eggs configuration (recommended) — download at [repo.quirinux.org](https://repo.quirinux.org/pool/main/e/eggs-quirinux-config/) |
| `xorriso` | recomp | ISO inspection and rebuild |
| `squashfs-tools` (`unsquashfs`, `mksquashfs`) | recomp | squashfs pack/unpack |
| `coreutils` (`dd`, `df`, `du`, `md5sum`) | recomp, cleanmaster | Standard utilities |
| `apt-get` / `dnf` / `pacman` / `zypper` | cleanmaster, functions | Package management (one required) |


---

## License

Copyright © 2019–2025 Charlie Martínez - Quirinux GNU/Linux. All rights reserved.  
Licensed under the [GNU General Public License v2.0](https://www.gnu.org/licenses/gpl-2.0.txt).  
For permitted and unauthorized uses of the Quirinux trademark, see [https://www.quirinux.org/aviso-legal](https://www.quirinux.org/aviso-legal).
