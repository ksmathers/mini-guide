---
title: tut-ai-dlc-copilot-vscode
description: 
published: true
date: 2026-04-27T00:00:00.000Z
tags: 
editor: markdown
dateCreated: 2026-04-27T00:00:00.000Z
---

# AI-Assisted Application Development with GitHub Copilot, VS Code, copilot-cli, and Amazon AI-DLC

Building a non-trivial application with an LLM agent is not just a matter of typing "write me an app." Unstructured prompts produce unstructured output — the agent invents scope, skips design, and produces code that is hard to verify. Amazon's **AI-DLC (AI-Driven Development Life Cycle)** solves this by giving the agent a structured methodology to follow: it asks you the right questions, produces a design document you approve, then generates code against that approved design. This tutorial shows you how to wire AI-DLC into GitHub Copilot running inside VS Code, and how to use the standalone **GitHub Copilot CLI** (`copilot-cli`) — a full agentic terminal tool analogous to Claude Code — to complement that workflow from the command line.

By the end of this tutorial you will understand:

- What AI-DLC is, what the three phases mean, and why that structure matters
- How GitHub Copilot custom instructions work and why they are the right integration point
- How to install and use `copilot-cli`, the standalone Copilot terminal agent, and how it complements the VS Code workflow
- How to bootstrap a new project so that AI-DLC is active from the first prompt
- How to use the Inception → Construction → Operations phases to take a feature from idea to working code
- How to add extension rules for security and testing on top of the core workflow

This tutorial assumes you are comfortable with the command line, git, and have basic familiarity with VS Code. No specific programming language is assumed — AI-DLC is language-agnostic.

---

## 📋 Overview

AI-DLC is a set of markdown files published by AWS Labs at [awslabs/aidlc-workflows](https://github.com/awslabs/aidlc-workflows). There is nothing to install in the traditional sense — the rules are plain text that your coding agent reads as project context. The agent then uses those rules to govern its own behavior.

The workflow has three phases:

| Phase | Color | Purpose | Output |
|---|---|---|---|
| **Inception** | 🔵 | Determines *what* to build and *why* | Requirements doc, user stories, design units |
| **Construction** | 🟢 | Determines *how* to build it | Implementation plan, generated code, tests |
| **Operations** | 🟡 | Deployment and observability (future) | Deployment config, monitoring setup |

The agent adapts automatically: simple changes skip stages that add no value; complex features get the full treatment. You control every transition — the agent proposes, you approve, then it proceeds.

For GitHub Copilot, the rules are delivered via the `.github/copilot-instructions.md` file. VS Code's Copilot Chat picks this up automatically and applies it to every chat request in the workspace. No extension configuration, no settings JSON — just a file in the right place.

---

## ⚙️ Prerequisites and Configuration Variables

**Required tools:**

- VS Code (any recent release) with the **GitHub Copilot** and **GitHub Copilot Chat** extensions installed
- `copilot-cli` — the standalone GitHub Copilot terminal agent (`brew install --cask copilot-cli` on macOS)
- A GitHub account with an active **GitHub Copilot** plan (Free, Pro, or Business)
- `git`

**Optional but useful:**

- `gh` CLI ≥ 2.x — the GitHub CLI; `copilot-cli` uses it for GitHub.com operations such as creating pull requests
- `unzip` (pre-installed on macOS/Linux)

Verify the essentials:

```bash
copilot --version && git --version && code --version
```

**Configuration variables** — set these once per project:

```bash
# Where you want the new project to live
export PROJECT_DIR="$HOME/dev/my-app"

# A short name used in some commands
export PROJECT_NAME="my-app"
```

> **Tip:** AI-DLC itself is version-controlled. Pin the release you use in your project's README so teammates get the same behavior.

---

## 🔧 Part 1: What AI-DLC Is and Why Structure Matters

### The problem with open-ended prompts

When you tell a coding agent "build me a REST API for managing inventory," it will produce *something* — but you have no way to verify that what it built matches what you actually needed before you have already committed to running the code. The agent made decisions about data models, authentication, error handling, and API surface that were never discussed. Some will be wrong.

AI-DLC reframes the interaction: instead of asking the agent to build, you ask it to *plan*, review the plan, then ask it to build against the plan. The resulting artifacts go into an `aidlc-docs/` directory in your project:

```
aidlc-docs/
├── requirements.md          ← what you are building and why
├── user-stories.md          ← who uses it and for what
├── design.md                ← component breakdown and interfaces
├── construction-plan.md     ← implementation order and test strategy
└── ...
```

You review and edit these files before the agent writes a single line of production code. This is the "human in the loop" tenet: the agent proposes, you approve.

### Why this beats a simple system prompt

A one-line system prompt like "follow best practices" gives the agent no procedural guidance. AI-DLC provides:

- **Structured multiple-choice questions** written to files (not chat), so the record is preserved and you can revisit your answers
- **Stage gates** — the agent cannot proceed to Construction until the Inception artifacts are approved
- **Adaptive execution** — the agent evaluates which stages are actually needed for your request and skips the ones that are not
- **Extensibility** — you can layer in security or testing rules on top without modifying the core workflow

---

## 🔧 Part 2: Installing GitHub Copilot CLI

GitHub Copilot CLI (`copilot-cli`) is a standalone agentic terminal application — analogous to Claude Code or Amazon Q Developer CLI. You run it from your project directory and hold a full conversation with it: it can read and modify files, execute shell commands, interact with GitHub.com, create pull requests, and run tests, all from a single terminal session. It is entirely separate from the VS Code extension.

### Install copilot-cli

On macOS, install via Homebrew Cask:

```bash
brew install --cask copilot-cli
```

On Linux, download the binary for your architecture from the [GitHub releases page](https://github.com/github/copilot-cli/releases) and place it on your `$PATH`.

Verify:

```bash
copilot --version
```

### Authenticate

On first launch, `copilot` will open a browser window to authenticate with your GitHub account. Your Copilot plan must be active — the CLI draws from the same subscription as the VS Code extension.

### Modes of use

`copilot-cli` has two interfaces:

| Interface | How to start | Best for |
|---|---|---|
| **Interactive** | `copilot` (no arguments) | Iterative development sessions, multi-step tasks |
| **Programmatic** | `copilot -p "prompt"` | Scripted or CI tasks with a single well-defined goal |

In the interactive interface, you hold a running conversation. The agent can enqueue follow-up instructions mid-response and you can steer it as it works.

**Plan mode** is a second mode within the interactive interface. Press Shift+Tab to toggle into it. In plan mode the agent analyzes your request, asks clarifying questions, and builds a structured implementation plan before writing any code — mirroring the intent of AI-DLC's Inception phase. Use plan mode for larger tasks directly in the terminal.

> **Concept:** `copilot-cli` and AI-DLC's rules in Copilot Chat are complementary, not redundant. AI-DLC in VS Code Chat produces *durable, committed artifacts* (`aidlc-docs/`) that capture design decisions in version control. `copilot-cli` in the terminal is better suited for *execution tasks* — running the code, fixing a failing test, creating a pull request — where you need shell access and do not need a permanent design record.

### Tool approval and security

The first time the agent needs to execute or modify a file it will prompt you:

```
1. Yes
2. Yes, and approve TOOL for the rest of this session
3. No, and tell Copilot what to do differently (Esc)
```

Option 1 approves this single invocation. Option 2 approves the tool for the session. Option 3 cancels and lets you redirect. For CI or scripted use you can pre-approve with flags:

```bash
# Allow all tools — use only in a sandboxed environment
copilot -p "Run the test suite and fix any failures" --allow-all-tools

# Allow specific tools, deny destructive ones
copilot --allow-all-tools --deny-tool='shell(rm)' --deny-tool='shell(git push)'
```

> ⚠️ **Warning:** `--allow-all-tools` grants the agent the same file and shell access you have. Only use it in a container or VM with limited blast radius. Never run it from your home directory.

Always launch `copilot` from your project directory, not your home directory. On first launch from a new directory, the agent will ask you to confirm you trust the files there.

---

## 🔧 Part 3: Bootstrapping a Project with AI-DLC

### Step 1: Create your project directory

```bash
mkdir -p "$PROJECT_DIR" && cd "$PROJECT_DIR"
git init
```

### Step 2: Download the latest AI-DLC release

```bash
# Fetch the download URL from the GitHub releases API
AIDLC_URL=$(curl -sL https://api.github.com/repos/awslabs/aidlc-workflows/releases/latest \
  | grep -o '"browser_download_url": *"[^"]*"' \
  | head -1 \
  | cut -d'"' -f4)

echo "Downloading: $AIDLC_URL"
curl -L "$AIDLC_URL" -o /tmp/aidlc-rules.zip
unzip -o /tmp/aidlc-rules.zip -d /tmp/aidlc-release
```

### Step 3: Install the rules into your project

For GitHub Copilot, the core workflow rule goes into `.github/copilot-instructions.md`. The detailed rule files (conditionally referenced by the core) go into `.aidlc-rule-details/`:

```bash
mkdir -p .github
cp /tmp/aidlc-release/aidlc-rules/aws-aidlc-rules/core-workflow.md \
   .github/copilot-instructions.md

mkdir -p .aidlc-rule-details
cp -R /tmp/aidlc-release/aidlc-rules/aws-aidlc-rule-details/* \
   .aidlc-rule-details/

# Clean up
rm -rf /tmp/aidlc-rules.zip /tmp/aidlc-release
```

Your project structure should now look like:

```
my-app/
├── .github/
│   └── copilot-instructions.md     ← AI-DLC core workflow rules
└── .aidlc-rule-details/
    ├── common/
    ├── inception/
    ├── construction/
    ├── extensions/
    └── operations/
```

### Step 4: Commit the rules to version control

The rules should be committed so every collaborator and every CI agent gets the same workflow behavior:

```bash
git add .github/copilot-instructions.md .aidlc-rule-details/
git commit -m "chore: add AI-DLC workflow rules"
```

### Step 5: Verify the rules are loaded in VS Code

1. Open the project in VS Code: `code .`
2. Open Copilot Chat (⌘⇧I on macOS, Ctrl+Shift+I on Windows/Linux)
3. Click the gear icon → **Chat Instructions** and verify that `copilot-instructions` appears in the list
4. Alternatively, type `/instructions` in the chat input to view active instruction files

> **Concept:** VS Code's Copilot Chat automatically discovers `.github/copilot-instructions.md` in the workspace root and prepends it to every chat request. The file does not need to be referenced explicitly — presence in the right location is enough.

---

## 🔧 Part 4: The Inception Phase — Designing Before Building

### Starting a session

With AI-DLC rules loaded, every session starts the same way: state your intent prefixed with "Using AI-DLC":

```
Using AI-DLC, I want to build a command-line tool that monitors a directory
for new JSON files, validates them against a schema, and moves them to either
a processed/ or errors/ subdirectory.
```

The agent does **not** start writing code. Instead, it opens the Inception phase.

### What Inception produces

The agent asks structured questions about your requirements — not in free-form chat, but by writing question files to `aidlc-docs/` that you answer by editing them. This produces a durable record of the decisions made.

Typical Inception outputs:

**`aidlc-docs/requirements.md`** — a structured requirements document:
```markdown
## Functional Requirements
- FR-01: Monitor a specified directory path for new files matching *.json
- FR-02: Validate each file against a JSON Schema provided at startup
- FR-03: Move valid files to processed/ and invalid files to errors/
- FR-04: Log each event with timestamp, filename, and validation result

## Non-Functional Requirements
- NFR-01: Must not block on file I/O; use async processing
- NFR-02: Schema path configurable via CLI argument, not hardcoded
```

**`aidlc-docs/design.md`** — component breakdown with interfaces defined before implementation.

Review these files carefully. Edit them. The agent will not proceed to Construction until you explicitly approve.

### Approving Inception

After reviewing the generated documents, tell the agent in chat:

```
The requirements and design look correct. Approved — proceed to Construction.
```

> **Alternative:** If your request is small — "add a `--dry-run` flag to an existing tool" — AI-DLC will evaluate that Inception adds little value and may propose skipping directly to Construction. This is by design: the workflow is adaptive, not bureaucratic.

---

## 🔧 Part 5: The Construction Phase — Generating Code Against the Design

### What Construction does

Once Inception artifacts are approved, the agent enters Construction. It:

1. Produces a `aidlc-docs/construction-plan.md` listing implementation units in dependency order
2. Asks you to approve the plan before writing code
3. Implements each unit against the approved design
4. Writes or proposes tests alongside each unit

You remain in the driver's seat: each unit is a checkpoint. The agent tells you what it is about to do, waits for your approval, then does it.

### Reviewing the construction plan

The plan will look something like:

```markdown
## Implementation Units

### Unit 1: Schema Loader (no dependencies)
- Load and parse JSON Schema from file path
- Validate schema is well-formed on startup
- Raise ConfigError if schema file is missing or malformed

### Unit 2: File Watcher (depends on: none)
- Use OS-level filesystem events (inotify/FSEvents/kqueue)
- Emit events for new files matching *.json pattern only

### Unit 3: Validator (depends on: Unit 1)
- Accept a file path and schema object
- Return (valid: bool, errors: list[str])

### Unit 4: File Router (depends on: Units 2, 3)
- On new file event: validate, then move to processed/ or errors/
- Log each decision

### Unit 5: CLI Entry Point (depends on: Units 2, 4)
- Parse --directory and --schema arguments
- Wire components together
- Handle SIGINT for clean shutdown
```

This plan makes explicit what the agent intends to build and in what order. If a dependency is wrong or a unit is missing, correct it now — not after half the code is written.

### Running Construction

After approving the plan:

```
Approved. Implement Unit 1: Schema Loader.
```

The agent generates the code, explains each significant decision inline, and proposes a test. Review both. When satisfied:

```
Looks good. Implement Unit 2: File Watcher.
```

Continue unit by unit. This sequential approval pattern may feel slower than asking for everything at once, but it gives you meaningful review points and prevents the agent from making cascading mistakes that compound through the codebase.

> **Alternative:** For small features or single-file changes, you can approve multiple units at once: "Approved. Implement Units 3 and 4." AI-DLC does not enforce one-unit-at-a-time; the granularity is yours to choose.

---

## 🔧 Part 6: Extensions — Layering Security and Testing Rules

### What extensions do

Extensions are additional rule files that layer on top of the core workflow. They are opt-in by default: during the Inception phase, the agent presents each extension's opt-in question, and you choose whether to activate it.

Two extensions ship with AI-DLC out of the box:

| Extension | What it enforces |
|---|---|
| `security/baseline` | OWASP-aligned security checks at each Construction stage |
| `testing/property-based` | Requires property-based tests alongside unit tests for applicable components |

### Activating the security extension

When Inception asks about security, answer yes. The agent will then, at each Construction stage, verify that the generated code satisfies the security baseline rules before marking the unit complete.

For a file-processing tool like the example above, the security extension would flag:
- Path traversal: the tool must validate that resolved file paths remain within the monitored directory
- Schema injection: user-supplied schema files must be validated before use
- Symlink following: the file watcher must not blindly follow symlinks outside the directory

These are checks the agent performs on its own output — you are not writing them manually.

### Writing a custom extension

You can add project-specific rules. Create a new directory under `.aidlc-rule-details/extensions/`:

```bash
mkdir -p .aidlc-rule-details/extensions/compliance/hipaa
```

Create two files:

**`.aidlc-rule-details/extensions/compliance/hipaa/hipaa.md`** — the rules:

```markdown
## Rule HIPAA-01: No PHI in Log Output
**Rule:** Logging statements must not include patient identifiers, file contents,
or any field that could contain PHI.
**Verification:** Scan generated logging calls; flag any that log request bodies,
file contents, or user-supplied strings directly.

## Rule HIPAA-02: Files at Rest Must Be Encrypted
**Rule:** Any file written to disk that may contain PHI must use OS-level
encryption or an explicit encryption wrapper.
**Verification:** Confirm no plain `open(path, 'w')` writes for files in the
processed/ or errors/ directories.
```

**`.aidlc-rule-details/extensions/compliance/hipaa/hipaa.opt-in.md`** — the opt-in prompt:

```markdown
# HIPAA Compliance Extension

This extension enforces HIPAA-aligned rules for applications that handle
Protected Health Information (PHI).

Apply this extension? (yes/no)
```

On the next AI-DLC session, the agent will discover the new `.opt-in.md` file and present the question during Inception.

> **Concept:** Extensions without a matching `.opt-in.md` file are **always enforced** — there is no opt-out. Use this for rules that must apply to every project in your organization (e.g., a `no-credentials-in-code` rule checked into a shared template repository).

---

## 🔧 Part 7: Using copilot-cli During a Development Session

Once AI-DLC in VS Code Chat has produced approved design artifacts and Construction is underway, `copilot-cli` becomes the right tool for implementation-time tasks that require shell access.

### Starting a session in your project

```bash
cd "$PROJECT_DIR"
copilot
```

The agent loads the workspace context, prompts you to confirm you trust the directory, and presents a welcome interface. From here you type instructions in plain English.

### Common patterns during a Construction phase

**Running the test suite and fixing failures:**

```
Run the tests. For each failure, show me the failing assertion and propose a fix. Wait for my approval before changing any file.
```

The agent executes the tests, reads the output, identifies root causes, proposes patches, and applies them only after you approve each tool use.

**Implementing a unit from the AI-DLC construction plan:**

After the design artifacts are approved in VS Code, you can hand off a specific unit to `copilot-cli` for implementation:

```
Using aidlc-docs/design.md as the specification, implement Unit 3: Validator.
The interface is defined in section "Component Interfaces". Write the implementation
and a corresponding test file. Do not proceed beyond Unit 3.
```

This keeps the agent scoped — it reads the committed design document and stays within the unit boundary you specify.

**Creating a pull request after a Construction unit is complete:**

```
The changes for Unit 3 are ready. Commit them with a message that references
the construction plan, then create a pull request against main titled
"feat: add JSON schema validator (AI-DLC Unit 3)".
```

The agent stages the files, writes the commit, pushes the branch, and creates the PR on GitHub.com. You are listed as the author.

**Investigating a runtime error:**

```
The file watcher is emitting events for files that were moved into the directory
by another process, not just newly created ones. Show me the relevant code and
explain why this is happening.
```

The agent reads the relevant source file, inspects the filesystem event type handling, and explains the cause — without modifying anything until you ask it to.

### Using plan mode for unplanned tasks

If a task arises during Construction that was not covered in the AI-DLC design — for example, a dependency conflict that requires refactoring a shared utility — switch to plan mode in `copilot-cli` by pressing Shift+Tab before describing the task. The agent will ask clarifying questions and produce an implementation plan for your review before touching any files.

> **Tip:** Use `/compact` in the `copilot-cli` session if it approaches its context limit during a long Construction phase. The agent compresses its history without losing the thread of the conversation.

---

## 🔧 Troubleshooting

**Symptom:** Copilot Chat does not appear to be following AI-DLC rules; it skips directly to writing code.

**Cause:** The `.github/copilot-instructions.md` file is not being picked up, or the session started before the file existed.

**Resolution:**
1. Confirm the file exists at `<project-root>/.github/copilot-instructions.md`
2. In Copilot Chat, click gear → Chat Instructions and verify it is listed
3. Start a new chat session (existing sessions may not reload the file)
4. Type `/instructions` to confirm active instructions

---

**Symptom:** The agent presents the Inception questions but then proceeds to Construction without waiting for approval.

**Cause:** Your response was interpreted as approval. AI-DLC watches for affirmative language.

**Resolution:** Be explicit in your non-approval responses. Instead of "that looks fine but I want to change the error handling," say: "**Not approved.** Revise requirement FR-03 to use a quarantine/ directory instead of errors/."

---

**Symptom:** `.aidlc-rule-details/` exists but the agent says it cannot find rule detail files.

**Cause:** The rule detail files use relative paths based on where `copilot-instructions.md` expects them. If you placed the details in a subdirectory or renamed the folder, the paths break.

**Resolution:** The folder must be named `.aidlc-rule-details/` and must be at the project root (the same level as `.github/`). Do not rename it.

---

**Symptom:** `copilot` returns "command not found" after installing via Homebrew.

**Cause:** The Homebrew Cask installs the binary but it may not be on your `$PATH` immediately, or the cask installation failed silently.

**Resolution:**
```bash
brew list --cask copilot-cli   # confirm the cask is installed
brew reinstall --cask copilot-cli
which copilot                  # verify the binary is on PATH
copilot --version              # confirm it launches
```

If `which copilot` returns nothing, add the Homebrew bin directory to your PATH:
```bash
echo 'export PATH="/opt/homebrew/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

---

**Symptom:** `copilot-cli` modifies a file you did not intend to change.

**Cause:** You approved a tool (option 2 — "for the rest of this session") that was broader than intended, or used `--allow-all-tools`.

**Resolution:** Use `git diff` to inspect all changes before committing. Revert unintended changes with `git checkout -- <file>`. In future sessions, prefer option 1 (per-invocation approval) for any tool involving writes until you are confident in the agent's behavior.

---

## 📚 Further Reading

- [AI-DLC GitHub repository](https://github.com/awslabs/aidlc-workflows) — source for the rules and latest releases
- [AI-DLC Method Definition Paper](https://prod.d13rzhkk8cj2z0.amplifyapp.com/) — the methodology's theoretical foundation
- [AWS Blog: AI-Driven Development Life Cycle](https://aws.amazon.com/blogs/devops/ai-driven-development-life-cycle/) — introductory overview from the AWS DevOps blog
- [GitHub Copilot custom instructions docs](https://code.visualstudio.com/docs/copilot/customization/custom-instructions) — VS Code documentation on the `copilot-instructions.md` mechanism
- [About GitHub Copilot CLI](https://docs.github.com/en/copilot/concepts/agents/copilot-cli/about-copilot-cli) — official reference for the standalone terminal agent
- [Installing GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/set-up/install-copilot-cli) — installation instructions for all platforms
- [Homebrew Cask: copilot-cli](https://formulae.brew.sh/cask/copilot-cli) — Homebrew formula for macOS installation
