# Forgenity IDE

Forgenity is a desktop code editor for Windows that I built on Tauri and Rust. It has an AI assistant with four modes, and the assistant can write to a project only through a controlled path: diffs, lint, review and an undo step.

This repository holds only this README and screenshots. The source code is private.

## Status

- Open beta, current version 0.21.3.
- Website: [forgenity.com](https://forgenity.com)
- Windows only.
- Source code: private.

## How it works

The assistant has four modes. The first three are split by what the model is allowed to do to your files, not by the type of task. The fourth, AGI, changes who is in charge of the run. ("AGI" is my internal name for a mode with a fixed pipeline in which the model only writes; it is not a claim about the model. It is newer than the other three and is not yet described on the website.)

### Chat

Chat is read-only by construction: the model gets no tools in this mode. Before each question, the app searches the open project and adds the most relevant files and symbols to the message. The search combines symbol lookup, text search and a BM25 ranking, and merges the three result lists. Conversations are saved locally. Code blocks in an answer have an "Insert at cursor" button. Inserting is a click by the user, not an action of the model, so the read-only rule still holds.

### Plan

Plan is also read-only. The model investigates the project with a small set of read-only tools (list a folder, read a file, find a symbol, search text, semantic search), limited to eight steps. It then answers with a plan as structured data. The Rust side parses the plan and stores it in a local database with a status, and the plan is also written as a `PLAN.md` file that I can edit in the editor. "Implement plan" runs the text as I left it, and only after I approve the list of steps in a dialog, where a step can be changed or emptied. After approval the work is handed to Agent mode, either whole or one step at a time. A single-step run is restricted to the files of that step.

### Agent

Agent is the only one of the first three modes that changes files. It is a loop in which the model chooses tools and the app executes them.

- **Context.** A deterministic discovery phase runs first: it lists the project and searches for keywords from the request. Then a planning call, then the tool loop.
- **Tools.** Reading and searching (folder listing, file read, symbol and text search, semantic search, project map), editing (edit, create, replace line, write, delete, move), shell commands (foreground and background, with output and kill), `git status` and `git diff`, diagnostics, a question to the user, and on-demand loading of project "skills" (markdown files in `.forgenity/skills`).
- **Approval of changes.** There are two settings. The default writes the change first, then runs lint and a review, and reverts the change if either fails. The second setting, "preview first", shows a diff and writes nothing until I accept it. Every turn has a "Revert this turn" button.
- **Approval of commands.** Every shell command is shown to me before it runs. Command prefixes I list in settings are run without asking; with an empty list, the agent always asks.
- **Review.** After an edit, the file is re-read and linted, and a reviewer call checks the change. The reviewer's verdict is read as a probability instead of free text (more on that in the architecture section). Security problems that the agent's own edits introduce are collected from the Problems panel scanner.
- **Stopping.** A Stop button cancels the request that is in flight. The loop also has a guard against the model repeating the same failing change over and over.

### AGI

AGI inverts the split used in Agent mode. Here the code runs a fixed pipeline, and the model only writes text or code. It never calls tools and never decides what is true. The phases are: clarify the request, find the files, derive what "done" means as checks the code can run, plan one-file steps, execute each step (lint, review, security scan, revert on failure), verify (tests against a baseline, open security problems, each check closed), and report. The report is prose written by the model, and the code appends the verified facts so the prose cannot misstate them. For web projects there is an extra step that starts the project on a dev server and walks its pages. An empty project first gets a blueprint to approve before anything is built. The pipeline code is split into a language-agnostic core, a web part, and per-stack packs; PHP is the only pack so far, and the PHP assumptions that remain in the web part are listed as known limits in my own notes.

## Architecture

Forgenity is a Tauri 2 desktop application. The backend is Rust and the interface is HTML, CSS and JavaScript running in a webview. The JavaScript side holds the orchestration of the modes (the Agent loop, the Plan investigation and the AGI pipeline). The Rust side does the work that needs the operating system.

**Main Rust modules**

- Path confinement: the commands that read, write, create, move and list files inside a project resolve their paths through one function that keeps the result inside the project root. Saving a file I opened through the file dialog is a separate command and uses the path as given.
- Model client: the HTTP client for the model providers, with streaming, retries and cancellation.
- Code index: tree-sitter parsing for JavaScript, TypeScript, PHP, HTML and CSS, stored in SQLite, plus a ranking index used for semantic search.
- File watcher, so the editor notices changes made outside it.
- Integrated terminal (a pseudo-terminal), Git panel (calls the system `git`), and language server client.
- Dev server for previews, and deploy over FTP, FTPS or SFTP with a diff before upload.
- Security scan: runs Opengrep with rule packs whose signatures are checked before use. Scanning is local.
- Key storage in the operating system keychain. The interface can save, check and delete a key but never read it.
- Local model process manager, for inline completion and background helpers (see below).

**How it talks to the language model.** The JavaScript side builds the request; the Rust side sends it. The webview's content security policy allows connections only to the app itself and its IPC channel, so model traffic goes through Rust. The default provider is BgGPT. Any OpenAI-compatible endpoint works, including OpenRouter, and Claude is supported through the Anthropic API with an adapter that translates requests and responses to the same shape. Each role (discovery, planner, editor, reviewer, and the two AGI roles) can use its own model; a role without an override uses the main one. Keys are the user's own.

**Probabilities instead of free text.** For small yes/no or multiple-choice decisions (such as whether a review should block a change), the app asks a closed question with lettered options, generates a single token, and reads the probability of each option. That lets "yes at 74%" and "yes at 99%" be handled differently. It needs a provider that returns token probabilities. The Anthropic API does not, so for it the call returns nothing and the caller falls back: the Agent's review uses the written review instead, or skips the review if no separate reviewer model is configured.

**Context management.** Before a run, the project is searched (symbols, text and a ranked index, merged). During a run, long histories are trimmed: the system prompt and the most recent messages stay intact, and older ones are compressed into a summary. Very large tool results are cut for the model and the full text is saved to a file in the project, which the model can read back.

**How changes reach files.** Edits are made by exact replacement: the model states the old text and the new text, and the app checks the old text against the file as it is on disk. A change passes through a diff view in the editor, lint, and a review. A failed change is reverted by writing back the content captured before the edit, and a rejected new file is removed. Each turn keeps a checkpoint so the whole turn can be undone.

**Local models.** Inline completion can run on the computer through llama.cpp with Qwen2.5-Coder-1.5B-Instruct, which is downloaded once with a checksum check. Background helpers ("minions") use the same model to draft doc comments, which are written only after I press Apply.

## Stack

- **Backend:** Rust, Tauri 2, tokio, reqwest, SQLite (rusqlite), tree-sitter, warp (dev server), notify (file watching), portable-pty (terminal), keyring, russh and suppaftp (deploy).
- **Frontend:** JavaScript, Alpine.js, Monaco Editor, xterm.js, Vite.
- **Local inference:** llama.cpp.
- **Tests and checks:** Vitest, ESLint, `cargo clippy`, `cargo test`, and end-to-end harnesses for the agent modes.

## Problems I ran into

**1. Streamed responses that arrive broken.**
A provider's streaming endpoint sometimes returns broken responses for a stretch of time. The Agent loop counts those failures per run. After two, it stops streaming for the rest of that run and uses ordinary calls with a 180-second timeout, and it tells the user why.

**2. Stopping a task that is waiting on the network.**
A cancel flag alone does nothing while a request is in flight. The Rust side races each model request against that flag, checking it every 150 ms, and dropping the request closes the connection. On the JavaScript side, a cancellation is treated as the user's choice and not as a provider failure, so the retry and fallback logic does not kick in right after I press Stop.

**3. Long runs and large tool output.**
Context grows with every tool call. I trim older messages into a summary and keep the recent ones, with one detail that mattered: a cut can leave a tool result without the assistant message that requested it, and strict gateways reject that with an HTTP 400, so the kept window is widened back to the parent message. Oversized tool results are shortened for the model and saved to a file it can read in full if needed; leftovers from a previous run are deleted at the start of the next one.

**4. Writing to files safely.**
Several layers exist because each one fixed a real failure. Paths are resolved against the project root, with `..` rejected and symlinks followed before the check. An edit's "old text" is verified against the disk, not against an editor tab: an earlier version compared with a cached tab after a shell command had changed the file, which produced repeated mismatch errors that the model could not explain. A change that fails lint or review is reverted, and "preview first" mode writes nothing until accepted. A single plan step can be run with a file allowlist so it cannot touch other files. A guard also stops the model from repeating an identical failing change.

**5. Network and provider reliability.**
On one network, a provider's IPv6 addresses were unreachable while IPv4 worked, so every call waited for the full connection timeout. The HTTP client now tries IPv4 addresses first and keeps IPv6 as a fallback. Transport-level failures are retried twice with short delays. When the editor role's provider keeps failing (rate limit, server error, timeout), the same step is retried on the next configured model. To avoid depending on one vendor, every provider is called through the same request shape, and an adapter translates the Anthropic format into it.

**6. Models that invent files.**
Early versions of the planning and orchestration code produced plans with file paths that did not exist. Plan mode now investigates the project with read-only tools before it writes the plan. AGI mode went further: the earlier free-form orchestrator was replaced by a fixed pipeline in which findings are facts with a file and a line, step plans that name files missing from the disk are rejected and requested again, and a finding the code can refute is marked as refuted.

## Screenshots

**Welcome screen.** The editor with no project open: the activity bar on the left, the open-file, open-folder, remote-folder and clone-repository actions, and keyboard shortcut hints. The status bar shows the llama.cpp indicator, the active model and the current mode.

![Forgenity welcome screen](images/Forgenity%20IDE.png)

**Chat mode.** A project open in the file explorer, with the AI panel on the right in CHAT mode and a question about the project answered from its files. The status bar also shows the GRAPH and MINION indicators.

![Forgenity in Chat mode](images/Forgenity%20IDE%20chat%20mode.png)

## Author

Dean Bouzekow

- LinkedIn: [linkedin.com/in/dean-b-513b24162](https://www.linkedin.com/in/dean-b-513b24162/)
- GitHub: [github.com/deandbdev](https://github.com/deandbdev)
