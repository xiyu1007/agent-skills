---
name: project-init
description: Init new software projects with README, SPECIFICATION, REQUIREMENTS, directory structure, CLAUDE.md. CRITICAL: use this whenever the user describes building something new, even casually. Triggers on: "new project", "start a project", "initialize", "create a project", "set up a project", "begin development", "scaffold", "I want to build", "开发一个", "创建一个", "帮我搭建", "写一个工具", "做一个项目", "帮我做一个", "I need a tool that", "help me build", "can you help me set up", any description of a new app/tool/service/library they want to create from scratch. Also triggers when user says they want to start coding something and the current directory is empty or lacks CLAUDE.md.
---

# Project Initialization Workflow

This skill encodes the complete development workflow for starting and managing any software project. It emphasizes living documentation, test-driven development with real-environment verification, and systematic bug tracking.

## Phase 1: First Conversation — Bootstrap

When a user describes a project they want to build, immediately create the foundation documents in the **current working directory**. Do NOT create a subdirectory for the project name — the current directory IS the project root.

### 1.0 Ask About Language (BEFORE creating documents)

Before writing any document, ask the user:

> "项目文档使用什么语言？中文 / English / 中英双语？"

Default to user's language (Chinese if they write in Chinese). If unspecified, ask.

**Bilingual format — anchor navigation, NOT inline mixing:**

Documents use anchor links at the top to switch language. Each language is a complete, independent section. Never mix languages within a section.

```markdown
# Project Name

**EN** · Brief English description.
**CN** · 简短中文描述。

---

**[English](#english) | [中文](#中文)**

---

<a name="english"></a>
## English

### Features
(complete English content here)

---

<a name="中文"></a>
## 中文

### 功能特性
(完整中文内容)
```

This applies to README.md and can apply to SPECIFICATION.md if bilingual. REQUIREMENTS.md is typically single-language (user's primary).

### 1.1 README.md

Create `README.md` at the current directory. Keep the first version minimal — it will grow:

```markdown
# Project Name
Brief one-line description.

## Quick Start
(How to install and run)

## Features
(will fill in as features become stable)
```

### 1.2 doc/SPECIFICATION.md

This is the **authoritative development specification**. Create it from the user's requirements. It should be detailed enough that someone unfamiliar with the project could implement it. Required sections:

- **Project Overview**: What and why
- **System Architecture**: ASCII diagrams showing components and their relationships. For layered systems, show each layer and how they communicate
- **Communication/Interface Contract**: If multiple layers (e.g., App↔Firmware), define the exact wire format, API signatures, or protocol
- **Component Breakdown**: Each major module, its responsibility, and its interface
- **Data Flow**: How data moves through the system (diagrams with state machines)
- **Error Handling Strategy**: How errors propagate and are reported
- **Implementation Plan**: Order of work, dependencies between components
- **Testing Strategy**: What testing methodology to use

Use the [open-canoe SPECIFICATION.md](doc/SPECIFICATION.md) as a reference for level of detail and structure.

### 1.3 doc/REQUIREMENTS.md

A test-oriented document derived from the specification. Required sections:

- **Project Goals**: 1-2 sentences
- **Feature Modules**: Table — each module, its function, and test criteria
- **Core Behavior Rules**: Numbered list. What MUST happen. What MUST NOT happen. These are the non-negotiable acceptance criteria
- **Test Methodology**: How tests are executed. Include specific code examples showing the real I/O pattern. The key rules (expand on these, don't just copy):

  > **MUST test with real I/O at the system boundary.**
  > - GUI: `btn.invoke()` → `app.root.update()` → read `tree.get_children()` / `text.get()`
  > - Serial/Hardware: Open port with `serial.Serial()`, read actual bytes, verify hex content
  > - CLI: `subprocess.run()` the actual command, check `returncode` + `stdout` + `stderr`
  > - API: Real HTTP call with `requests`, check `status_code` + response body
  >
  > **FORBIDDEN: "code looks correct" or "should work" — only real output is valid evidence.**

- **Historical Bug Tracker**: Table with columns: #, symptom, root cause, fix, status (✅/❌/⚠️). Every bug found in testing gets an entry. This builds institutional knowledge
- **UI Control Checklist** (if GUI): Every button, checkbox, dropdown, input field — with expected behavior and verification method
- **Test Criteria**: Specific pass/fail steps that must ALL pass

## Phase 2: Project Structure

**CRITICAL: Never create a subdirectory for the project name.** Work directly in the current working directory. All files go at root level. The directory you're in IS the project root.

### 2.1 Directory Layout

**Single-layer projects** (CLI tool, library, web app — most common):
```
(current directory = project root)
├── README.md                     # Project description
├── CLAUDE.md                     # AI guidance
├── doc/                          # Exactly 3 docs, never duplicated across layers
│   ├── SPECIFICATION.md
│   └── REQUIREMENTS.md
├── config/                       # Config files (pyproject.toml, defaults.yaml, etc.)
│   └── defaults.yaml
├── data/                         # Runtime data (.lang, history CSV, logs)
├── src/                          # Source code
├── test/                         # ALL test scripts
└── tools/                        # Deploy/clean scripts (one .py + one .bat per function)
```

**Multi-layer projects** (e.g., firmware + desktop app):
```
(current directory = project root)
├── README.md
├── CLAUDE.md
├── doc/                          # 3 docs cover ALL layers — never per-layer
│   ├── SPECIFICATION.md
│   └── REQUIREMENTS.md
├── firmware/                     # Layer 1: embedded C
│   ├── config/                   # Per-layer config (only if needed)
│   └── ...
├── app/                          # Layer 2: Python GUI
│   ├── config/                   # Per-layer config (only if needed)
│   ├── data/                     # Per-layer runtime data
│   └── ...
├── test/                         # ALL tests (shared)
└── tools/                        # Shared deploy/clean scripts
```

**Three docs rule**: `README.md` + `doc/SPECIFICATION.md` + `doc/REQUIREMENTS.md` — exactly 3, always at project root or `doc/`, never per-layer. They describe the entire project.

**Config vs Data**: `config/` = committed version-controlled settings (yaml/toml). `data/` = runtime-generated files not committed (.lang, csv, logs). For multi-layer, a layer only gets its own `config/` or `data/` if it genuinely needs separate files.

### 2.2 Config Management

**Single configuration file** pattern. Put all user-configurable settings in one YAML/TOML file. Hardcode GUI constants (colors, fonts) in a config module — they're not user settings and don't belong in the config file. This avoids the confusion of splitting settings across multiple files.

## Phase 3: CLAUDE.md Rules

Write a CLAUDE.md at the project root. Use the exact template from the section above (with the ⚠️ READ FIRST block at the top). Then add:

1. **Project overview** — what it does, key files, how to run
2. **Architecture notes** — important patterns, why certain decisions were made
3. **Cross-module testing rule**:

   > If changes to module A affect module B, module B must also be tested. Don't only test the directly modified module.

4. **Path rules**: Internal paths relative, external paths via environment variables
5. **Naming conventions** and editing rules

The documentation sync block MUST be the very first thing in the file, before any project overview. The AI reads files top-to-bottom and gives more weight to content at the top.

## Phase 4: Development Loop

### Adding a Feature

1. Update **SPECIFICATION.md** → add feature description
2. Update **REQUIREMENTS.md** → add test criteria and behavior rules
3. Implement the feature
4. Run existing tests (prevent regression)
5. Write new tests for the feature
6. Fix bugs, update REQUIREMENTS.md bug tracker

### Testing Rules — REAL verification, NEVER fake it

**This is the most important rule in the entire workflow. Violations waste hours of debugging and destroy trust.**

#### ❌ FORBIDDEN: Fake Testing

These are NOT valid testing — they give false confidence:

| Fake pattern | Why it fails | Real example of failure |
|---|---|---|
| "I reviewed the code, the logic looks correct" | Code can look perfect but fail at runtime due to environment, timing, or hardware | AI claimed serial port was sending data. User connected serial monitor — zero bytes received |
| "The function returns the right value when called" | Testing a callback directly skips the actual trigger path (button click, interrupt, network) | Direct callback test passed, but `btn.invoke()` test revealed ISR storm blocking the event loop |
| "The response should contain..." (guesswork) | Assuming what happened instead of reading actual output | Guessed that CAN_FRAME_UP was sent, but actual log showed it wasn't |
| "If we set X, then Y should happen" (hypothetical) | Theory vs reality — hardware, race conditions, and real I/O differ | NART=1 should prevent retries in theory, but F103 still looped back in NORMAL mode |

#### ✅ REQUIRED: Real I/O Verification

Every test MUST interact with the real system boundary and verify actual output:

**For serial/network/hardware:**
```python
# WRONG: "I checked the code, it should send data"
# RIGHT: Actually read the port
import serial
ser = serial.Serial('COM7', 115200, timeout=2)
data = ser.read(100)
assert len(data) > 0, f"No data received! Got: {data}"
print(f"Received: {data.hex()}")  # Show real evidence
```

**For GUI apps:**
```python
# WRONG: Call the callback directly
# self._on_send(msg)  # this skips button click path

# RIGHT: Simulate the real user action, then read real widget state
btn.invoke()                    # real button click
app.root.update()               # drive event loop
time.sleep(0.5)                 # allow processing

# Verify by reading actual widget state — never guess
rows = tree.get_children()
actual_data = [tree.item(r, 'values') for r in rows]
assert any('RX' in str(r[5]) for r in actual_data), f"No RX row found! Rows: {actual_data}"
```

**For CLI tools:**
```python
# WRONG: import and call the function directly
# WRONG: subprocess but don't check exit code / stdout

# RIGHT: Run the actual command, capture and verify everything
result = subprocess.run(['python', 'main.py', '--sync', '/tmp/watch'],
                       capture_output=True, text=True, timeout=10)
assert result.returncode == 0, f"Exit code {result.returncode}: {result.stderr}"
assert 'Synced 3 files' in result.stdout, f"Unexpected output: {result.stdout}"
```

**For APIs:**
```python
# RIGHT: Real HTTP call with timeout, check status + body
resp = requests.post('http://localhost:8080/api/data', json=payload, timeout=5)
assert resp.status_code == 200, f"HTTP {resp.status_code}: {resp.text}"
data = resp.json()
assert data['status'] == 'ok', f"Unexpected response: {data}"
```

#### Testing Workflow

1. **Simulate real user operations**: GUI = `invoke()`/`event_generate()`, CLI = `subprocess.run()`, API = `requests`, Hardware = actual serial read
2. **Read real output**: widget state, stdout/stderr, HTTP response body, serial bytes, file contents
3. **Assert with evidence**: every assertion prints what it expected vs what it got, so failures are debuggable
4. **Tests grow incrementally**: Each new feature adds test cases
5. **Regression prevention**: Run the full test suite after any change — a fix in one place can break another

### Bug Fix Workflow

1. Reproduce with a test
2. Fix the code
3. Verify with the test
4. Document in REQUIREMENTS.md bug tracker: symptom → root cause → fix → status ✅

## Phase 5: Session Close

Before ending any development session:

- [ ] Review SPECIFICATION.md for accuracy
- [ ] Review REQUIREMENTS.md — any new bugs to add? any statuses to update?
- [ ] Update test counts in REQUIREMENTS.md if new tests were added
- [ ] Check README.md for outdated information
- [ ] Verify all tests pass
- [ ] Add any new architectural rules to CLAUDE.md

This checklist ensures the next session starts with accurate context.

## Example: Extracting User Requirements Mid-Conversation

When a user describes a feature or requirement casually during ANY conversation, immediately extract and formalize it into the docs. Don't wait for them to say "update the docs."

> User: "the app should work in loopback mode where sent messages come back as RX"

→ SPECIFICATION.md: Add loopback mode to CAN configuration section  
→ REQUIREMENTS.md: Add behavior rules:
> 1. LOOPBACK mode: TX+RX rows appear, RX ID/data must match TX
> 2. NORMAL mode: Only TX rows, absolutely NO RX rows
> 3. Mode switch must not produce stale frames

> User: "can you make the delete button clear the selection after removing a row"

→ REQUIREMENTS.md: Add new bug/requirement entry, add test criteria  
→ SPECIFICATION.md: Only if it changes architecture (UI behavior changes usually don't)

**Rule**: User says "I want X" or "X should do Y" → it goes into REQUIREMENTS.md as either a feature requirement or a bug to track. SPECIFICATION.md gets updated only for architectural changes.

## CLAUDE.md Template — CRITICAL Rules

When creating CLAUDE.md, put these rules at the VERY TOP where they can't be missed. Use the exact format below:

```markdown
# CLAUDE.md — Project Name

## ⚠️ READ FIRST — Documentation Sync (DO NOT SKIP)

**After EVERY conversation about this project, before responding to the user:**

1. Review `doc/SPECIFICATION.md` — any architecture changes? New modules? Changed interfaces?
2. Review `doc/REQUIREMENTS.md` — any new bugs? Changed behaviors? New test criteria?
3. Review `README.md` — any new features stable enough to document?
4. If the user said "I want X to do Y" or "X should work like Y", extract that into REQUIREMENTS.md

**If you made code changes, you MUST check the docs. This is not optional.**
```

This goes before the project overview, before the file listing, before everything. The urgency markers (⚠️, DO NOT SKIP, MUST) are intentional — they prevent the AI from treating this as a routine suggestion.
