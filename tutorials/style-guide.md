---
title: style-guide
description: 
published: true
date: 2026-04-27T00:00:00.000Z
tags: 
editor: markdown
dateCreated: 2026-04-27T00:00:00.000Z
---

# Tutorials Style Guide

This document defines the structure, tone, and formatting conventions for guides in the `tutorials/` directory.

---

## What Belongs Here vs. in Mini-Guides

| | Mini-Guide (`Services/`) | Tutorial (`tutorials/`) |
|---|---|---|
| **Goal** | Get a service running fast | Understand how and why it works |
| **Length** | Short — steps only | Long — concepts woven throughout |
| **Audience** | Operator who just needs it deployed | Reader who wants to learn the technology |
| **Explanations** | Minimal; just enough to proceed | After every significant step or decision point |
| **Alternatives** | Not discussed | Actively called out with guidance on when to choose them |
| **Prerequisite depth** | Listed at the top | Explained or linked with context |

If a guide can be followed without reading it — just copying commands — it belongs in Mini-Guides. If skipping the text would leave the reader unable to adapt the setup to their situation, it belongs here.

---

## Required Frontmatter

Every tutorial must begin with this YAML block (WikiJS format):

```yaml
---
title: <filename-without-extension>
description: 
published: true
date: <ISO-8601 datetime>Z
tags: 
editor: markdown
dateCreated: <ISO-8601 datetime>Z
---
```

Dates use the format `2026-04-27T00:00:00.000Z`. All fields are required.

---

## File Naming

Tutorials use the prefix `tut-` followed by a short kebab-case topic name:

```
tutorials/tut-kubernetes-networking.md
tutorials/tut-tls-with-step-ca.md
tutorials/tut-proxmox-storage-types.md
```

---

## Document Structure

Every tutorial must contain these sections, in order:

### 1. Title and Introduction

The H1 title should state the subject and signal that this is a tutorial (not a reference page or quick-start):

```markdown
# Kubernetes Networking: How Services, DNS, and MetalLB Work Together
```

The introduction (no heading) follows immediately. It should answer:
- What problem or concept this tutorial covers
- What the reader will have built or understand by the end
- What background knowledge is assumed

Keep the introduction to 2–4 paragraphs.

### 2. 📋 Overview (optional but encouraged for complex topics)

A brief conceptual map of the territory before diving in. Use a diagram, table, or short prose to orient the reader. Label this section:

```markdown
## 📋 Overview
```

### 3. ⚙️ Prerequisites and Configuration Variables

List tools and access required before starting. For tutorials that involve shell commands, export configuration variables at the top so every command block can be copy-pasted without substitution:

```bash
export NAMESPACE="my-namespace"
export DOMAIN="myservice.local"
```

Explain *why* each variable matters — not just what it is.

### 4. Tutorial Body: numbered parts with concept explanations

Divide the body into **Parts** (not "Steps" — tutorials progress through concepts, not just tasks). Each Part groups related commands and the explanation that makes them meaningful.

```markdown
## 🔧 Part 1: Title of the Concept Being Explored
```

Within each Part:

1. **Explain the concept first.** Before showing a command or manifest, describe what it does and why it is done this way. One or two sentences is often enough; use a paragraph when the concept is non-obvious.
2. **Show the command or config block.**
3. **Verify the result.** Include a verification command and explain what the expected output means.
4. **Explain alternatives.** After each significant decision, note what other approaches exist and when to prefer them. Use a blockquote or callout:

```markdown
> **Alternative:** You could use `NodePort` instead of `LoadBalancer` here if MetalLB is not available.
> `NodePort` assigns a random high port on every node and is appropriate for single-node clusters or when you control iptables rules directly.
> `LoadBalancer` is preferred in this homelab setup because MetalLB assigns a stable IP that Avahi can advertise as a `.local` name.
```

### 5. 🔧 Troubleshooting

Required. Cover the failure modes most likely to be hit when following this tutorial. For each:
- **Symptom** — what the reader observes
- **Cause** — why it happens
- **Resolution** — the fix

### 6. 📚 Further Reading (optional)

Links to upstream documentation, RFCs, or related guides in this repo. Keep this section short — 3–6 links maximum.

---

## Writing Tone

- **Address the reader directly** using "you". Avoid passive voice and third-person constructions ("the user should…").
- **Explain decisions, not just actions.** "Run this command" is insufficient. "Run this command to confirm that the controller has registered the CRD before proceeding — if it has not, the next `apply` will fail silently" is the target.
- **Treat alternatives as first-class content.** A reader who understands why one choice was made over another can adapt the tutorial to their situation. Do not bury alternatives in footnotes.
- **Be precise about scope.** State explicitly when a technique is specific to this homelab's hardware or configuration vs. when it applies generally. Use callout blocks:

```markdown
> **Homelab-specific:** This step sets `securityContext.privileged: true` because the cluster runs on bare-metal nodes with direct device access.
> In a managed cloud cluster this is typically disallowed by policy; use a device plugin instead.
```

---

## Callout Block Conventions

| Purpose | Format |
|---|---|
| Homelab-specific behavior | `> **Homelab-specific:**` |
| Alternative approach | `> **Alternative:**` |
| Warning / destructive action | `> ⚠️ **Warning:**` |
| Key concept summary | `> **Concept:**` |
| Tip or shortcut | `> **Tip:**` |

---

## Code Block Conventions

Inherited from the Mini-Guides conventions, with one addition:

- Annotate non-obvious flags or fields inline:

```bash
kubectl apply -f - <<EOF
# ReadWriteOnce: only one node can mount this volume at a time.
# Use ReadWriteMany if multiple pods across nodes need simultaneous access.
accessModes:
  - ReadWriteOnce
EOF
```

- For multi-step sequences where order matters, number the commands in comments:
```bash
# 1. Create the namespace first — resources below depend on it existing
kubectl create namespace example
# 2. Apply the deployment into that namespace
kubectl apply -f deployment.yaml -n example
```

---

## Length and Scope

- A tutorial should cover **one concept or workflow end-to-end**. If the scope grows to cover two distinct concepts (e.g., both storage provisioning and TLS issuance), split it into two tutorials and link them.
- There is no target length, but every sentence should earn its place. Explanations that do not help the reader understand a decision or adapt the tutorial can be cut.
- Tutorials are not reference documentation. Avoid exhaustively listing every flag or option; link to upstream docs instead and explain only the choices made here.

---

## Relationship to Mini-Guides

A tutorial and a mini-guide can cover the same service. The tutorial explains the concepts; the mini-guide is the quick-reference operators use after they have already worked through (or do not need) the tutorial. Cross-link them:

In the tutorial:
```markdown
> **Quick-start:** If you only need this service deployed without the background, see the [WikiJS mini-guide](../Services/kube/mgk-wikijs.md).
```

In the mini-guide (optional):
```markdown
> **Deep dive:** For a full explanation of how each component works, see the [WikiJS tutorial](../tutorials/tut-wikijs-on-kubernetes.md).
```
