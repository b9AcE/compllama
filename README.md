# compllama
The compllama ("compile llama") script is intended to compile [llama.cpp](https://github.com/ggml-org/llama.cpp) locally from git source,<br />
choosing one backend out of either Vulkan or CUDA and setting up a systemd-service,<br />
while installing its main resulting files in its own date- and backend-tagged subdirectory<br />
of /opt/ so the executables will remain under symlink /opt/llamacpp/bin through version-switches,<br />
as well as a helper "inillama" to scan for GGUF-files to create a models-preset file,<br />
which is not only used by the systemd-service but can then also be invoked through<br />
the same inillama-script to run presets, for example reusing any models in use with LM Studio.<br />
There are also per install-prefix and system-wide config-files.<br />
Everything is of course written so that it can be run repeatedly on the same computer,<br />
e.g. only touching parts of potentially shared config-files that are marked as written by compllama,<br />
and the uninstall-option checks for the same markers,<br />
and inillama only appends to the presets-file unless explicitly user-requested.<br />
<br />
Considering recent and ongoing developments it was deemed of low benefit to also add ROCm-support,<br />
since Vulkan seems to already be usually more performant and improving, so use that for such GPUs.<br />
<br />
It should be able to detect sensible defaults for the initial questions in most situations,<br />
but otherwise there are also many commandline options documented in the script,<br />
visible by running the script with "--help", and by reading the comments in the script.<br />
<br />
Made for Linux, but may work elsewhere.<br />
Prerequisites are same as for compiling llama.cpp.<br />
For using the systemd-service using the cancer systemd is of course required.<br />

The initial upload here, version 1.0.0, was preceded by more than 7 months of leisurly pace<br />
development, testing, and use, but while the preceding versions are archived it would serve<br />
no purpose to retroactively upload the preceding versions here.<br />

---

## What you get

| Component | Path | Purpose |
|-----------|------|---------|
| Build prefix | `/opt/llamacpp-<backend>-YYYYMMDDTHHMM-<ref>` | Binaries, libraries, headers for one backend + git ref |
| Active symlink | `/opt/llamacpp` → latest build in use | Single stable path for everything else to reference |
| Per-build env | `/opt/llamacpp/llamacpp.conf` | GPU pin and other settings generated at build time (regenerated every build) |
| Persistent env | `/etc/localllama.conf` | **Yours.** Created once as a comments-only template; never touched again. Read after the per-build file, so its values win. |
| Systemd unit | `/etc/systemd/system/llama-server.service` | Runs `llama-server` with model presets, auto-restart on failure |
| Model-preset helper | `~/.local/bin/inillama` | Scans folders for `.gguf` models and maintains the preset INI file |
| Shell profile | `/etc/profile.d/llamacpp.sh` | Puts `llama-cli`, `llama-server`, etc. on `PATH` in interactive shells |

---

## Requirements

Run as a **regular user with sudo access** (the script invokes `sudo` only if and when needed).<br />
**Not** intended to be executed as root.

### Common to both backends

- `cmake`, `gcc`, `g++`, `git`, `make`
- `pkg-config` / `pkgconf`

### CUDA backend (nVidia)

- The [CUDA Toolkit](https://developer.nvidia.com/cuda-downloads) with `nvcc` in your `PATH`, i.e. e.g.
  - Add `/usr/local/cuda/bin` to `PATH` (e.g. via a file in `/etc/profile.d/`).
  - If `ldconfig -p | grep libcudart` finds nothing, also add the toolkit's library directory to `/etc/ld.so.conf.d/` and run `sudo ldconfig`.

### Vulkan backend (AMD/any vendor)

- A working Vulkan driver for your GPU
- `glslang` (or `glslangValidator`), `glslc`, `vulkaninfo` (`vulkan-tools`)
- Vulkan development files and SPIR-V headers discoverable via `pkg-config`

> The script checks all of the above before it starts compiling.<br />
Pass `--no-prereq-check` to skip the checks if a false negative obstructs,<br />
then whatever is genuinely missing will surface as a CMake error instead.

---

## Quick start
Commonly used options exist as both short and long options (i.e. both `-d` and `--defaults`).<br />
The long options which take parameters accept space-separator and equals-separattor unless `--help` says otherwise.<br />
Designating proper versions or build-versions can be done with or without `v`/`b` (i.e. accepts `0.7.0`/`v0.7.0` or `11777`/`b11777`).

Some usage examples:
```bash
# Get asked several questions at the start to then compile latest build-release of llama.cpp,
# with or without additional helper-scripts, etc based on your answers.
compllama

# Auto-detect GPU (nVidia → CUDA, AMD/other → Vulkan), compile latest build-release,
# install service + inillama with defaults:
compllama --defaults

# List which GPUs will be detected, with index-numbr usable in the next command.
compllama --gpu=list

# Explicit backend, pin to the first GPU, non-interactive:
compllama --backend=vulkan --gpu 1 --defaults

# Build a specific release tag:
compllama --release 0.7.0 -d

# Offline build from an existing local clone:
git clone https://github.com/ggml-org/llama.cpp llamacpp-clone   # once
compllama --offline -d
# Or to specify the local clone's location
compllama --offline=/path/to/offline-clone -d
```

After the run finishes, `source /etc/profile` (or open a new shell) to pick up the new `PATH`.

---

## Options

| Flag | Description |
|------|-------------|
| `-b, --backend <cuda\|vulkan>` | GPU backend. Omit to auto-detect from sysfs. |
| `-g, --gpu <N\|list>` | Pin the server to one GPU. `--gpu list` prints candidates and exits, while `--gpu N` picks entry *N* without asking. |
| `-a, --activate <backend>` | Switch `/opt/llamacpp` to the most recent build of specified backend **without recompiling**.<br /> Stops the service, repoints the symlink, runs `ldconfig`, restarts the service.<br />All other options are ignored. |
| `-o, --offline[=PATH]` | Build from a local clone instead of cloning from GitHub.<br />Bare `-o` uses the configured default (defined at the start of the script for easy change the default).<br />`--offline=PATH` overrides it. No network needed at all. |
| `-c, --compile-directory <path>` | Scratch build tree (deleted and recreated every run).<br />Must contain a valid llama.cpp source tree before removal is attempted. |
| `-r, --release <ver>` | Choose a specific llama.cpp release tag.<br />Accepts bare number for either style `NNNN` build-tag or for `X.Y.Z` style version,<br />or same with a `v` version prefix or a `b` build-tag prefix.<br />`latest` builds from git HEAD. Default: most recent `bNNNN` build tag. |
| `-d, --defaults` | Use configured defaults for every prompt (no interactive questions).<br />Can be combined with `--offline`.<br />To change the defaults, edit the definitions conveniently placed at the start of the script-file. |
| `-y, --yes` | Auto-accept removal of an existing compile directory without asking. |
| `-t, --tools` | Install or update helper tool(s) (currently `inillama`) only, then exit.<br />No build, no sudo needed. |
| `-u, --uninstall` | Remove system integration (service, profile.d bloc, ld.so.conf.d bloc, `inillama`).<br />Does **not** remove compiled binaries in `/opt`.<br />All other flags are ignored. |
| `-l, --list-tags` | Print the 20 most recent tags and exit. Follows `--offline`: without it queries GitHub's releases API, while with it reads the local clone. |
| `--no-prereq-check` | Skip prerequisite checks entirely. |
| `--no-conf-template` | Do not create `/etc/localllama.conf` if absent. |
| `-h, --help` | Show usage and exit. |

### User-editable defaults (top of the script)

```bash
DEFAULT_SERVICE_USER=""      # Empty → auto-detect ($USER)
DEFAULT_SERVICE_PORT="11435" # Default to not conflict with Ollama's default
DEFAULT_SERVICE_HOST=""      # Empty → auto-detect address of first UP wired/WiFi interface
DEFAULT_CUDA_ARCH=""         # Empty → auto-detect via nvidia-smi
DEFAULT_OFFLINE_CLONE="llamacpp-clone" # Absolute or relative path to existing `got clone`
DEFAULT_COMPILE_DIR="llama.cpp-compile" # Absolute or relative path to directory to build in
```

---

## But how?

1. **Validate** arguments, detect GPU vendor from sysfs, enumerate candidate GPUs (`vulkaninfo` / `nvidia-smi`).
2. **Collect all user input up front** so the rest of the run is unattended: compile-directory removal, service install + parameters, inillama install.
3. **Acquire sudo once**, then proceed without further password prompts (a background keepalive refreshes the timestamp during long compiles).
4. **Clone or copy source**, checkout the requested tag, resolve `git describe` for the build reference.
5. **Configure & compile** with CMake. The selected backend is explicitly `ON`; all other GPU backends are explicitly `OFF` so an unrelated toolkit on the machine cannot be picked up by accident. `$ORIGIN`-relative RPATH is embedded in every binary so each build loads its own libraries regardless of where `/opt/llamacpp` points.
6. **Install** to the versioned prefix, update the symlink, run `ldconfig`.
7. **Resolve GPU pinning**: CUDA uses the device UUID directly; Vulkan asks the freshly built binary which raw position yields the chosen device name (authoritative, same code path as the service).
8. **Write config files**, install the systemd unit and/or inillama, start or restart the service.

An exit trap cleans up: kills background helpers and removes an incomplete install prefix if the run ended before `cmake --install` finished.

---

## The `inillama` helper tool

Installed to `~/.local/bin/inillama`.<br />Manages the model-preset INI-file which `llama-server --models-preset` and the systemd-service can read.

```bash
# Scan configured source folders for .gguf models, append new sections:
inillama

# Also scan an extra folder this run only:
inillama -s ~/path/to/other-models

# Dry-run (show what would be added):
inillama -n

# Remove sections whose model file no longer exists (keeps 1 backup):
inillama --prune

# Tell the running server to re-read the preset file:
inillama --reload

# List all models with their section names, numbered:
inillama ls

# Run llama-cli with a specific model's settings:
inillama run 1            # by number from ls/list
inillama run "publisher/repo:quant"   # by name, valid options shown by ls/list
```

Key behaviours:

- **Append-only.** Sections already in the file (including ones you renamed or tuned) are never changed or removed. Models are recognised by their `model =` path, not by section name.
- **LM Studio layout aware.** For a three-level path `publisher/repo/file.gguf`, the section name becomes `publisher/repo:QUANT` with the quant normalised exactly as llama-server does. Anything else falls back to the file's relative path.
- **Multi-part models** (`-00001-of-00004.gguf`) are handled; only part 1 is added, and a matching `mmproj` projector (if exactly one) is included automatically.
- **Format validation.** Every run checks the file against llama-server's INI grammar and warns about any line that would make it refuse the whole file.
- **Multi-presets per model.** If you for example want one preset for with `mmproj` and another without, or one with higher Context Window traded for more quantized K/V Cache while another not,<br />you can just define more presets pointing at the same `model = ` file,<br />as long as the presets have distinct names (for example `tools publisher/repo:quant` vs `no tools publisher/repo:quant`).

### Preset file location

`~/.config/llama.cpp/inillama.ini` — created by `inillama`, never by `compllama`. The systemd unit points at this path (resolved for the service user's home).

---

## Configuration files

| File | Who writes it | Lifetime |
|------|---------------|----------|
| `/opt/llamacpp/llamacpp.conf` | compllama, every build | Regenerated. NOT a place for persistent settings. |
| `/etc/localllama.conf` | You (compllama creates an empty template once) | Survives every rebuild and backend switch.<br />Read **after** the per-build file, so its values override.<br />To opt out: `sudo truncate -s 0 /etc/localllama.conf` creates an empty file, or `compllama --no-conf-template` every run. |

Both are systemd `EnvironmentFile` format: plain `KEY=value` lines, no `export`, no shell expansion.

Example `/etc/localllama.conf`:

```ini
# Override the GPU pin for this machine:
GGML_VK_VISIBLE_DEVICES=0

# Or for CUDA:
# CUDA_VISIBLE_DEVICES=GPU-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

---

## Switching backends without recompiling

If you have built both `cuda` and `vulkan`:

```bash
compllama --activate cuda    # repoints /opt/llamacpp, restarts service
compllama --activate vulkan  # switch back
```

To activate a build older than the most recent one of that backend, repoint the symlink by hand:

```bash
sudo ln -sfnT /opt/llamacpp-vulkan-20260915T0830-b11042 /opt/llamacpp
sudo ldconfig
sudo systemctl restart llama-server.service
```

---

## Uninstalling

```bash
compllama --uninstall
```

Removes: the systemd unit, managed blocs from `/etc/ld.so.conf.d/llamacpp.conf` and `/etc/profile.d/llamacpp.sh`, and `~/.local/bin/inillama`.

Does **not** remove: compiled binaries in `/opt/llamacpp*`, `/etc/localllama.conf`, or the preset INI-file.<br />
To clean those up manually, CHECK that you actually want to remove what is there first, then:

```bash
sudo rm -rf /opt/llamacpp /opt/llamacpp-*
# and, if desired:
rm ~/.config/llama.cpp/inillama.ini
```

---

## Working directories

Both default to **relative** paths (resolved against the current working directory):

| Directory | Role |
|-----------|------|
| `llamacpp-clone` | The clone `--offline` reads from. Never written to or deleted by this script. |
| `llama.cpp-compile` | Scratch build tree. Deleted and recreated on every run, but kept after a failed compile so you can check for what went wrong in it. |

Set `DEFAULT_OFFLINE_CLONE` and `DEFAULT_COMPILE_DIR` near the top of the script to absolute paths if you want them fixed regardless of where you invoke the script from.

---

## Build directory naming

```
/opt/llamacpp-<backend>-YYYYMMDDTHHMM-<ref>
```

where `<ref>` comes from `git describe --tags --always`:

| Source | Example ref | Resulting dir |
|--------|-------------|---------------|
| Exact build tag | `b11067` | `llamacpp-vulkan-20260921T1200-b11067` |
| Exact proper version tag | `v0.7.0` | `llamacpp-vulkan-20261016T0141-v0.7.0` |
| Commit past a tag | `b11065-3-gce8caa6e6` | `llamacpp-cuda-20260921T1200-b11065-3-gce8caa6e6` |
| No tags at all (bare hash) | `a1b2c3d4` | `llamacpp-vulkan-20260921T1200-a1b2c3d4` |

---

I think that's all I put in this thing.<br />
Enjoy, and be excellent to each other. ;-)
