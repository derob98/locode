# locode

A local coding assistant that configures itself.

One binary measures your machine, picks and downloads the open-weight models it can actually run, and
gives you a terminal coding agent on top of them. Everything runs on your computer: no cloud, no
account, no telemetry.

## Install

macOS and Linux:

```sh
curl -fsSL https://github.com/derob98/locode/releases/latest/download/install.sh | sh
```

Windows (PowerShell):

```powershell
irm https://github.com/derob98/locode/releases/latest/download/install.ps1 | iex
```

No administrator rights and no prerequisites. The script downloads one binary for your platform,
checks its SHA-256 against the release's `checksums.txt` and puts it in `~/.local/bin` (macOS, Linux)
or `%LOCALAPPDATA%\Programs\locode` (Windows, added to your user `PATH`).

| Variable | Effect |
| --- | --- |
| `LOCODE_VERSION` | install a given release, for example `v0.1.0`, instead of the latest |
| `LOCODE_INSTALL_DIR` | put the binary somewhere else |
| `LOCODE_NO_MODIFY_PATH` | Windows: leave the user `PATH` alone |

## First run

```sh
locode setup
```

`setup` does the whole job, asking once before it downloads anything:

1. **Profiles the hardware**: CPU and its instruction sets, total and free RAM, GPUs and VRAM, disk
   space, plus short memory and disk benchmarks.
2. **Installs the runtime**: a pinned [llama.cpp](https://github.com/ggml-org/llama.cpp) build for the
   backend it found (CUDA, Vulkan, Metal or CPU), checked against known hashes.
3. **Picks four tiers** (`max`, `complex`, `daily`, `fast`) among the models of its catalog that fit
   the memory that is free right now, and shows what it would download: sizes, licences, disk space.
4. **Downloads** them (resumable, SHA-256 verified), **tunes** each one (GPU layers, context size,
   KV cache, speculative decoding) and prints a table with the tokens per second it measured.

Models take from a few to tens of gigabytes. `locode setup -yes` runs unattended; `locode doctor`
does only steps 1 and 2 and tells you what the machine can do.

## Use

```sh
cd your-project
locode                                   # the interactive agent
locode "fix the failing test in parser_test.go"
locode -p "list the Go files" -output-format json   # headless: one answer, then exit
```

What the agent does, with local models in mind:

- **Routes each prompt to a tier**: a question goes to `fast`, everyday edits to `daily`, refactoring
  and debugging to `complex` or `max`. It moves down only when the time saved beats the measured
  load time, and moves up by itself when a model gets stuck. `-model` or `/model` pin one.
- **Keeps tool calls working**: malformed calls are repaired and validated against the tool's schema;
  after repeated failures the output is constrained by a grammar.
- **Checks its own edits**: after writing files it runs the project's formatter, linter and tests
  (Go, Node, Python, Rust, or the commands you configure) and feeds the errors back to the model.
- **Spends context carefully**: a map of the repository in the prompt, long files read in chunks,
  a token budget per tool, old tool output pruned first, summaries written by the `fast` model.
- **Shows what is going on**: a live panel with the active model, tokens per second, context used,
  VRAM and RAM.

It also has the things you would expect from a terminal agent: file, search, shell and web-fetch
tools; permission modes including plan mode; checkpoints and `/rewind`; sessions you can resume and
fork; manual and automatic compaction; `AGENTS.md` and `CLAUDE.md` instructions; MCP servers
(stdio, HTTP, SSE); skills in the `SKILL.md` format; hooks; subagents, also in the background;
memory across sessions; goals and loops; git worktrees; a status line. The interface speaks English
and Italian and follows the system language (`LOCODE_LANG=en|it` to force one).
Type `/help` inside the agent and `locode -h` for its flags.

## Commands

| Command | What it does |
| --- | --- |
| `locode [flags] [prompt]` | the coding agent in the current folder |
| `locode -p "prompt"` | headless mode (`-output-format text`, `json` or `stream-json`) |
| `locode setup` | profile, runtime, the four model tiers, tuning and the results table |
| `locode doctor` | detect the hardware, run the benchmarks, save the profile |
| `locode bench` | measure the installed models again |
| `locode models` | list the catalog: sizes, licences, what is installed, tiers and speeds |
| `locode update` | download the current model catalog (SHA-256 checked) |
| `locode serve` | an OpenAI-compatible endpoint on `127.0.0.1` for other tools |
| `locode uninstall` | remove everything locode put on the machine |
| `locode version` | print the version |

## Safety and privacy

- By default the agent asks before it writes a file, runs a command or fetches a URL. Writes are
  limited to the working folder, and destructive commands are always confirmed, in every mode.
- Settings found inside a project can only restrict what the agent may do until you say you trust
  that folder: a cloned repository cannot approve its own commands.
- The network is used for: the llama.cpp runtime and this project's catalog (GitHub), the models
  (Hugging Face), and whatever you ask the agent to fetch or the MCP servers you configure. Nothing
  about you, your code or your prompts is sent anywhere.

## Update

Run the install command again to get a newer binary: your models and settings stay.
`locode update` refreshes only the model catalog.

## Uninstall

```sh
locode uninstall
```

It lists what it is about to delete and asks (`-yes` to skip the question):

- the data folder with the runtime, models, profile, sessions and settings. If you moved it to
  another disk behind a symlink or a junction, the real folder is deleted too, not just the link;
- the `locode` binary itself;
- on Windows, the entry the installer added to your user `PATH`.

The data folder is `%LOCALAPPDATA%\locode` on Windows, `~/Library/Application Support/locode` on
macOS and `~/.local/share/locode` (or `$XDG_DATA_HOME/locode`) on Linux.

What stays, because it lives in your projects: `.locode/` folders (project settings, worktrees) and
`worktree-*` git branches created with `-worktree`. On macOS and Linux, also a `PATH` line you may
have added to your shell profile by hand.

## Platforms

| System | Processors | GPU backends |
| --- | --- | --- |
| Windows 10 and 11 | x64, ARM64 | CUDA (x64), Vulkan, CPU |
| Linux (glibc) | x64, ARM64 | CUDA (x64), Vulkan, CPU |
| macOS 13 or later | Apple Silicon, Intel | Metal (Apple Silicon), CPU |

`locode` itself is a static binary; the llama.cpp runtime it downloads is the upstream prebuilt one.

## Known limits

- **Local models are weaker than the large hosted ones.** Routing, repair and verification narrow
  the gap on everyday work; they do not close it on hard problems. There are no public benchmark
  numbers for locode.
- **Windows is where it is developed and tested with real models.** The Linux and macOS builds are
  covered by automated tests and by an automated install, `doctor` and uninstall run, not by daily use.
- **The binaries are not code-signed.** Installed with the commands above they run without prompts,
  but antivirus software or company policy may still object, and Windows 11 Smart App Control, where
  it is turned on, blocks unsigned programs with no per-app exception. If you download a macOS binary
  with a browser instead, remove the quarantine flag: `xattr -d com.apple.quarantine locode-darwin-arm64`.
- **One large model in memory at a time.** Changing tier means loading another model, which takes
  from seconds to minutes depending on the disk.
- **Models are chosen by memory fit and measured speed**, with quality scores from the catalog; how
  well each one handles tools is not measured on your machine.
- Model files you already have elsewhere are not reused. There is no self-update: run the installer
  again. No plugins, themes, vim mode or editor integration.

## Licence

The binaries are free to use, for personal and commercial work. Copyright © 2026 Gianluca De
Robertis, all rights reserved; the source code is not published. See [LICENSE](LICENSE).
The open-source components inside the binary are listed with their licences in
`THIRD_PARTY_LICENSES.txt`, attached to every release. llama.cpp and the models are separate works,
downloaded under their own licences, which `setup` shows before each download.
