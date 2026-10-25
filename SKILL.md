---
name: wx-cli
description: Operate the wx-cli command-line tool for a user's local WeChat 4.x data, including diagnostics, message search, exports, attachments, and offline voice transcription. Use when the user asks to query or manage their local wx-cli data; do not use for sending WeChat messages.
---

# wx-cli

Use this skill to work with the local `wx` executable and only the authenticated user's local WeChat data. Treat all returned chats, contacts, attachments, and transcripts as sensitive content; show only what the user asks for.

## Locate and validate the executable

Prefer `wx` when it is on `PATH`. On this Windows workstation, the built executable is at:

```text
C:\Users\yongc\Downloads\dev-pci\wx-cli-again\target\release\wx.exe
```

If `wx` is unavailable, run that executable directly or ask for the project/executable location. Before diagnosing a feature failure, run `wx doctor`; use `wx --help` or the command's `--help` to confirm options for an installed version.

## First-time setup and recovery

`wx init` creates the local configuration and collects available database keys. On Windows, run it from an elevated PowerShell while WeChat is running and logged in. Do not use `sudo` on Windows.

If `wx doctor` reports missing keys or unknown shards, use:

```powershell
wx key extract --hook-seconds 90
```

The user must keep WeChat open and open the relevant conversation during the hook period so cold database shards can load. Do not run key extraction unless the user requests setup or recovery.

## Querying local data

Use `--json` when results must be inspected or passed to another command. The normal query flow is:

```powershell
wx sessions --json
wx history "<chat name>" --json
wx search "<keywords>" --json
```

For larger result sets, add `-n <limit>` and restrict by conversation or date where supported. Query results can include a `meta` object; if its status is stale or `unknown_shards` is non-empty, explain that results may be incomplete and offer the recovery flow above.

Useful read-only commands include `unread`, `new-messages`, `contacts`, `members`, `favorites`, `stats`, `attachments`, `sns-feed`, `sns-search`, and `biz-articles`. Use `export` only when the user requests an output file, and use a user-provided or clearly stated destination.

## Attachments

List attachments first to obtain an `attachment_id`, then extract only the record requested by the user:

```powershell
wx attachments "<chat name>" --kind image --json
wx extract <attachment_id> -o <output-path>
```

Do not overwrite an existing file unless the user asks for it.

## Offline voice transcription

The CLI's voice transcription is local: it reads an encrypted voice record, decodes its Silk audio, and runs a local ggml Whisper model. It does not use WeChat's built-in speech-recognition service and does not upload audio.

First list voice records to obtain `svr_id`:

```powershell
wx media voices --json
```

Then transcribe the selected item:

```powershell
wx media transcribe <svr_id> --language zh --json
```

The default model location is `~/.wx-cli/models/ggml-base.bin` (on this Windows machine: `C:\Users\yongc\.wx-cli\models\ggml-base.bin`). If the model is absent, ask the user to provide a local ggml Whisper model or approve obtaining one. To use another model, pass `--model <path>`.

Voice records may have an empty `chat` field. In that case, list all voices first; do not rely on `--chat` filtering. Present the resulting `text` as a transcription that can contain recognition errors.

## Daemon

The daemon starts automatically for most database queries. Use `wx daemon status` for diagnosis and `wx daemon stop` only when the user requests stopping it or when a restart is necessary to recover a failed query. A subsequent query starts it again.
