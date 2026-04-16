---
name: install-cli
description: Install or uninstall the clc CLI end-to-end, including auto-installing its prerequisites (uv, Node.js) on macOS and Linux. USE WHEN user says 'install clc', 'set up clc', 'install this', 'install the tool', 'uninstall clc', 'uninstall this', OR any variant. Bundled with the dev.clc repo. This skill orchestrates prerequisite checks, OS-aware package installs, and then delegates to `clc install` / `clc uninstall`.
---

## Your job

Walk the user all the way from "missing prerequisites on a fresh machine" to "`clc` on PATH and ready to launch Claude Code." Do NOT stop on warnings. If a prerequisite is missing, install it (using the OS's native mechanism). Then run the installer.

Announce each step before running it so the user sees what's happening.

## 0. Locate the repo root

Walk up from `$PWD` until you find a directory containing `clc` + `params.json`.

```bash
dir="$(realpath "$PWD")"
while [ "$dir" != "/" ]; do
  if [ -f "$dir/clc" ] && [ -f "$dir/params.json" ]; then
    break
  fi
  dir="$(dirname "$dir")"
done

if [ "$dir" = "/" ]; then
  echo "ERROR: Not inside the dev.clc repo."
  echo "cd into the repo (or a subdirectory) and retry."
  exit 1
fi
echo "Repo: $dir"
```

## 1. Ensure `uv` is installed

```bash
if ! command -v uv >/dev/null 2>&1; then
  echo "Installing uv (official installer, no sudo)..."
  curl -LsSf https://astral.sh/uv/install.sh | sh
  export PATH="$HOME/.local/bin:$HOME/.cargo/bin:$PATH"
  command -v uv >/dev/null 2>&1 || { echo "uv install failed"; exit 1; }
fi
echo "uv: $(uv --version)"
```

## 2. Ensure `npx` is available (Node.js)

`clc` pins `CLAUDE_CODE_VERSION` by default, which invokes claude via `npx`. Install Node using the native package manager for the OS/distro. This step usually needs sudo — surface the sudo command to the user via the bash tool so they can approve it.

```bash
if ! command -v npx >/dev/null 2>&1 || [ ! -e "$(command -v npx 2>/dev/null)" ]; then
  os=$(uname -s)
  if [ "$os" = "Darwin" ]; then
    if command -v brew >/dev/null 2>&1; then
      echo "Installing Node.js via Homebrew..."
      brew install node
    else
      echo "ERROR: Homebrew not installed. Install it first:"
      echo "  /bin/bash -c \"\$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)\""
      exit 1
    fi
  elif [ "$os" = "Linux" ]; then
    if [ -f /etc/os-release ]; then
      . /etc/os-release
      case "${ID:-}" in
        ubuntu|debian|pop|mint)
          echo "Installing Node.js via NodeSource + apt (requires sudo)..."
          curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
          sudo apt install -y nodejs
          ;;
        fedora|rhel|centos|rocky|almalinux)
          echo "Installing Node.js via dnf (requires sudo)..."
          sudo dnf install -y nodejs
          ;;
        arch|manjaro|endeavouros)
          echo "Installing Node.js via pacman (requires sudo)..."
          sudo pacman -Sy --noconfirm nodejs npm
          ;;
        alpine)
          echo "Installing Node.js via apk (requires sudo)..."
          sudo apk add --no-cache nodejs npm
          ;;
        opensuse*|sles)
          echo "Installing Node.js via zypper (requires sudo)..."
          sudo zypper install -y nodejs npm
          ;;
        *)
          echo "ERROR: Unsupported Linux distro: ${ID:-unknown}"
          echo "Install Node.js manually from https://nodejs.org/, then re-run."
          exit 1
          ;;
      esac
    else
      echo "ERROR: Can't detect Linux distro (no /etc/os-release). Install Node.js manually."
      exit 1
    fi
  else
    echo "ERROR: Unsupported OS: $os"
    exit 1
  fi
  command -v npx >/dev/null 2>&1 || { echo "npx still not found after install"; exit 1; }
fi
echo "node: $(node --version 2>/dev/null || echo '?') | npx: $(command -v npx)"
```

## 3. Run the installer

```bash
"$dir/clc" install
```

If the user passed `--force` or said "force" / "reinstall" / "overwrite", run:

```bash
"$dir/clc" install --force
```

## 4. Verify

```bash
command -v clc && clc --help >/dev/null && echo "clc OK"
```

If `command -v clc` comes up empty, `~/.local/bin/` is probably not on PATH. Tell the user to add it:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc   # or ~/.bashrc
```

## Uninstall

Same repo-root lookup, then:

```bash
"$dir/clc" uninstall
```

With force:

```bash
"$dir/clc" uninstall --force
```

The installer is idempotent (won't prompt in non-interactive contexts); uninstall only removes symlinks that point into this repo unless `--force` is passed.

## Edge cases

| Situation | Action |
|-----------|--------|
| `uv` is installed but not on PATH | Export `$HOME/.local/bin:$HOME/.cargo/bin` to PATH, retry |
| `/snap/bin/npx` exists but is a dead symlink | The `[ ! -e "$(command -v npx)" ]` branch triggers reinstall |
| User has no sudo | Report the failure verbatim — installing Node without sudo on Linux needs nvm/fnm, which is out of scope |
| Symlink at `~/.local/bin/clc` already points at this repo's `clc` | `clc install` exits 0 with "Already installed" — not an error |
| Prompts from `clc install` / `clc uninstall` | The installer is non-interactive-safe. If it ever does prompt, forward it to the user; don't answer on their behalf |

## Report back

Give the user a short summary: uv version, node version, symlink target, whether PATH is good. If anything failed, surface the exit code and the error verbatim — do not hide failures.
