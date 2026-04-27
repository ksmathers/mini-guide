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

Building a non-trivial application with an LLM agent is not just a matter of typing "write me an app." Unstructured prompts produce unstructured output — the agent invents scope, skips design, and produces code that is hard to verify. Amazon's **AI-DLC (AI-Driven Development Life Cycle)** solves this by giving the agent a structured methodology to follow: it asks you the right questions, produces a design document you approve, then generates code against that approved design. This tutorial shows you how to wire AI-DLC into GitHub Copilot running inside VS Code, and how to use the `gh copilot` CLI to complement that workflow from the terminal.

By the end of this tutorial you will understand:

- What AI-DLC is, what the three phases mean, and why that structure matters
- How GitHub Copilot custom instructions work and why they are the right integration point
- How to install the `gh copilot` CLI extension and what it adds to the workflow
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
- `gh` CLI ≥ 2.x — the GitHub CLI (`brew install gh` on macOS)
- `git`

**Optional but useful:**

- `unzip` (pre-installed on macOS/Linux)
- `jq` (for parsing the release API response cleanly)

Verify the essentials:

```bash
gh --version && git --version && code --version
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

## 🔧 Part 2: Installing the GitHub Copilot CLI Extension

The `gh copilot` CLI extension brings Copilot assistance to the terminal. It is distinct from the VS Code extension — it is useful for quick questions during development, for explaining shell commands before you run them, and for suggesting commands you do not know the syntax of.

### Install the extension

```bash
gh auth login   # if not already authenticated
gh extension install github/gh-copilot
```

Verify:

```bash
gh copilot --version
```

### What it provides

The extension adds two subcommands:

| Command | What it does |
|---|---|
| `gh copilot suggest` | Suggests a shell command that accomplishes what you describe in plain English |
| `gh copilot explain` | Explains what a given shell command does |

Example — you want to find all files modified in the last 24 hours but cannot remember the `find` flags:

```bash
gh copilot suggest "find all files modified in the last 24 hours, excluding .git"
```

The agent asks whether you want a `git` command, a `gh` command, or a generic shell command, then outputs a runnable suggestion. You can copy it, run it directly, or ask for a revision.

> **Concept:** `gh copilot suggest` is not AI-DLC-aware — it is a standalone terminal assistant. The two tools complement each other: use AI-DLC in the VS Code chat for design and implementation sessions; use `gh copilot suggest` in the terminal for one-off command lookups during those sessions.

### Optional: set up a shell alias

Many users add an alias so `?` calls the suggest subcommand:

```bash
# Add to ~/.zshrc or ~/.bashrc
eval "$(gh copilot alias -- zsh)"   # or bash
```

After reloading your shell, `ghcs "your question"` invokes suggest and `ghce "command"` invokes explain.

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

The `gh copilot` CLI integrates naturally with an AI-DLC session. While Copilot Chat in VS Code handles design and code generation, the terminal extension handles operational questions that arise during implementation.

### Common patterns

**Verifying the filesystem watcher works:**

```bash
gh copilot explain "inotifywait -m -e close_write --format '%w%f' /tmp/watch"
```

The agent explains each flag and what the output format means — useful when you are wiring the watcher output to your application and need to understand the exact event format.

**Finding a flag you cannot remember:**

```bash
gh copilot suggest "run pytest and show only failed tests with full output, no capture"
```

Output:
```
pytest -v --no-header -rN --tb=short --no-cov -x 2>&1 | grep -E "FAILED|ERROR|assert"
```

The agent suggests a command; you review it before running.

**Understanding a cryptic error:**

```bash
gh copilot explain "OSError: [Errno 28] No space left on device when writing to /tmp"
```

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

**Symptom:** `gh copilot suggest` returns "command not found" after installing the extension.

**Cause:** The shell alias has not been configured, or the extension installation failed silently.

**Resolution:**
```bash
gh extension list              # confirm github/gh-copilot is listed
gh extension upgrade copilot   # upgrade if installed but stale
gh copilot suggest "test"      # invoke directly to verify
```

---

## 📚 Further Reading

- [AI-DLC GitHub repository](https://github.com/awslabs/aidlc-workflows) — source for the rules and latest releases
- [AI-DLC Method Definition Paper](https://prod.d13rzhkk8cj2z0.amplifyapp.com/) — the methodology's theoretical foundation
- [AWS Blog: AI-Driven Development Life Cycle](https://aws.amazon.com/blogs/devops/ai-driven-development-life-cycle/) — introductory overview from the AWS DevOps blog
- [GitHub Copilot custom instructions docs](https://code.visualstudio.com/docs/copilot/customization/custom-instructions) — VS Code documentation on the `copilot-instructions.md` mechanism
- [gh copilot CLI documentation](https://docs.github.com/en/copilot/github-copilot-in-the-cli/about-github-copilot-in-the-cli) — official reference for the terminal extension
