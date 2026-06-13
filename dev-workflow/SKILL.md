---
name: dev-workflow
description: Software development workflow — MUST trigger on EVERY conversation. First checks whether this is a new project requiring bootstrap (README, SPECIFICATION, REQUIREMENTS, CLAUDE.md, directory structure). If the three core docs already exist and development is ongoing, evaluates the current user input against them and updates accordingly. Enforces test timeouts with cleanup, real I/O verification, test-flow-first development, and systematic bug tracking.
---

# Development Workflow

## ⚠️ FIRST — Determine Project Phase (EVERY conversation)

At the start of **every** conversation, check the current directory:

```
doc/SPECIFICATION.md + doc/REQUIREMENTS.md + CLAUDE.md exist?
  ├─ NO  → Phase A: Bootstrap new project
  └─ YES → Is this a development conversation (adding features, fixing bugs, changing code)?
              ├─ NO  → No doc changes needed, proceed with the user's request
              └─ YES → Phase B: Evaluate current input → update docs → implement
```

Do not skip this check. Phase B only applies when all three conditions are met: docs exist, project is in active development, and the conversation involves code/feature/bug changes.

---

## Phase A: Bootstrap New Project

When the project lacks the three core docs, create them in order.

### A.1 Language

Before writing any document, ask: "项目文档使用什么语言？中文 / English / 中英双语？"

Bilingual format uses anchor navigation — each language is a complete independent section, never mixed inline:

```markdown
# Project Name
**EN** · Brief description.  **CN** · 简短描述。
**[English](#english) | [中文](#中文)**
<a name="english"></a>## English
<a name="中文"></a>## 中文
```

### A.2 Create doc/SPECIFICATION.md

The authoritative development specification. Required sections:
- **Project Overview**: What and why
- **System Architecture**: ASCII diagrams, layers, communication paths
- **Communication/Interface Contract**: Wire format, API signatures, protocol definition
- **Component Breakdown**: Each module, its responsibility, its interface
- **Data Flow**: State machines, how data moves
- **Error Handling Strategy**: Propagation and reporting
- **Implementation Plan**: Order of work, dependencies
- **Testing Strategy**: Methodology. Must include timeout estimates per module.

### A.3 Create doc/REQUIREMENTS.md

Test-oriented document. Required sections:
- **Project Goals**: 1-2 sentences
- **Feature Modules**: Table — module, function, test criteria, expected max test runtime
- **Test Flows**: Step-by-step procedures with explicit timeout values. This is the contract — code is not "done" until these pass.
- **Core Behavior Rules**: What MUST / MUST NOT happen
- **Test Methodology**: Real I/O patterns (see Common Rules below)
- **Historical Bug Tracker**: #, symptom, root cause, fix, status (✅/❌/⚠️)
- **UI Control Checklist** (if GUI): Every widget with expected behavior and verification
- **Test Criteria**: Specific pass/fail steps

### A.4 Create README.md

Minimal first version — grows as features stabilize:
```markdown
# Project Name
Brief one-line description.
## Quick Start
## Features
```

### A.5 Create Directory Structure

Work directly in the current directory — it IS the project root. Never create a subdirectory for the project name.

**Single-layer:**
```
├── README.md
├── CLAUDE.md
├── doc/SPECIFICATION.md, REQUIREMENTS.md
├── config/defaults.yaml
├── data/          # runtime-generated, not committed
├── src/
├── test/          # all tests, every test with timeout
└── tools/         # deploy/clean scripts
```

**Multi-layer** (e.g., firmware + app):
```
├── doc/           # 3 docs cover ALL layers — never per-layer
├── firmware/      # layer with optional config/, data/
├── app/           # layer with optional config/, data/
├── test/          # shared, all with timeouts
└── tools/         # shared
```

**Rules**: Exactly 3 docs (README + SPECIFICATION + REQUIREMENTS), never per-layer. `config/` = committed settings. `data/` = uncommitted runtime files.

### A.6 Create CLAUDE.md

Use the template from the end of this skill. The doc-sync block and timeout rules go at the VERY TOP.

---

## Phase B: Ongoing Development — Evaluate & Update Docs

Only proceed when **all three conditions** are met: (1) three core docs exist, (2) the project is in active development, (3) the conversation involves code/feature/bug changes. When these hold, **before writing any code**, evaluate the user's input against each doc:

| Doc | Ask |
|---|---|
| **SPECIFICATION.md** | Does this change architecture? Add/remove modules? Change interfaces? Update timeout estimates? |
| **REQUIREMENTS.md** | New behavior rule? New test flow needed? Bug to track? Missing timeout on an existing test? |
| **README.md** | New feature stable enough to document? Quick start changed? |

Extract implicit requirements: if the user says "I want X to do Y" or "X should work like Y", that goes into REQUIREMENTS.md immediately.

**If code changes are made, doc review is mandatory. This is not optional.**

### B.1 Adding a Feature / Module — Test Flow FIRST

**No code before test flow.** Sequence:

1. Update SPECIFICATION.md — module description, architecture, interfaces, timeout estimates
2. Update REQUIREMENTS.md — add test flow with step-by-step procedure and timeout values. Add behavior rules.
3. Confirm the test flow with the user
4. THEN implement the feature
5. Run existing tests (regression check, all with timeouts)
6. Execute the test flow from REQUIREMENTS.md — verify real output
7. Fix bugs, update bug tracker

The test flow is the contract — it defines "done" before implementation starts.

### B.2 Bug Fix Workflow

1. Reproduce with a test (with timeout)
2. Fix the code
3. Verify with the test
4. Document in REQUIREMENTS.md bug tracker: symptom → root cause → fix → status ✅

---

## Common Rules (Both Phases)

### ⚠️ Test Timeouts — CRITICAL

**Every test MUST have a timeout. No exceptions.** A hung test is a SEVERE BUG — it blocks development and leaks resources.

**Timeout by module type:**

| Module Type | Max Timeout | Rationale |
|---|---|---|
| Pure function / unit test | 5s | Near-instant |
| File I/O | 10s | Fast disk ops |
| CLI subprocess | 15s | Spawn + execute |
| GUI interaction | 30s | Event loop + render |
| Serial / hardware | 30s | Physical device |
| Network / API | 20s | HTTP round-trip |
| App full flow | 60s | Full lifecycle |
| Firmware flash | 120s | Hardware programming |

**Implementation:**

```python
# pytest
@pytest.mark.timeout(30)
def test_serial(): ...

# subprocess — ALWAYS set timeout
subprocess.run(['python', 'app.py'], capture_output=True, text=True, timeout=30)

# serial — per-read timeout
ser = serial.Serial('COM7', 115200, timeout=2)
```

**Cleanup — kill orphaned processes on timeout:**

```python
class AppTester:
    def setUp(self):
        self.proc = subprocess.Popen(['python', 'app.py'], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        time.sleep(2)

    def tearDown(self):
        """Always kill spawned processes, even on timeout or error."""
        if self.proc and self.proc.poll() is None:
            self.proc.terminate()
            try:
                self.proc.wait(timeout=5)
            except subprocess.TimeoutExpired:
                self.proc.kill()
                self.proc.wait()
```

For serial: always close in `finally` — `if ser and ser.is_open: ser.close()`

**If a test times out, treat it as a bug first.** Investigate deadlocks, unreleased resources, unresponsive dependencies. Only increase timeout if legitimately needed — document the reason.

**Pre-commit timeout checklist:**
- [ ] Every `subprocess.run()` has `timeout=`
- [ ] Every `requests.get/post()` has `timeout=`
- [ ] Every `serial.Serial()` has `timeout=`
- [ ] Spawned-process tests have cleanup in `tearDown` / `finally`
- [ ] Timeout values match module type

### ❌ FORBIDDEN: Fake Testing

| Fake pattern | Why it fails |
|---|---|
| "I reviewed the code, the logic looks correct" | Code can be perfect but fail at runtime |
| "The function returns the right value when called" | Testing callback directly skips the real trigger path |
| "The response should contain..." (guesswork) | Assuming output instead of reading it |
| "If we set X, then Y should happen" (hypothetical) | Theory vs reality — hardware, timing, race conditions differ |

### ✅ REQUIRED: Real I/O Verification

Every test MUST interact with the real system boundary:

- **GUI**: `btn.invoke()` → `app.root.update()` → read widget state (`tree.get_children()`, `text.get()`)
- **CLI**: `subprocess.run()` with `timeout=`, check `returncode` + `stdout` + `stderr`
- **API**: Real HTTP call with `requests`, `timeout=`, check `status_code` + response body
- **Serial/Hardware**: `serial.Serial()` with `timeout=`, read actual bytes, verify hex content
- **Assert with evidence**: print expected vs actual on every failure

---

## Example: Phase B in Practice

> User: "the app should work in loopback mode where sent messages come back as RX"

→ SPECIFICATION.md: Add loopback mode to architecture
→ REQUIREMENTS.md: Add behavior rules (LOOPBACK: TX+RX appear, RX matches TX; NORMAL: only TX, no RX; mode switch must not produce stale frames)

> User: "add a serial monitor module for logging incoming bytes"

→ SPECIFICATION.md: Add module description, interface contract
→ REQUIREMENTS.md: Add test flow with 30s timeout — open port, send known bytes, verify monitor captures them, close port in finally
→ THEN implement

> User: "the delete button should clear the selection after removing a row"

→ REQUIREMENTS.md: Add test flow with timeout, add to UI control checklist
→ SPECIFICATION.md: Only if architectural (UI behavior changes usually aren't)

---

## CLAUDE.md Template

```markdown
# CLAUDE.md — Project Name

## ⚠️ READ FIRST — Documentation Sync (DO NOT SKIP)

After EVERY conversation, before responding:

1. Review `doc/SPECIFICATION.md` — architecture changes? New modules? Changed interfaces?
2. Review `doc/REQUIREMENTS.md` — new bugs? Changed behaviors? New test flows? Missing timeouts?
3. Review `README.md` — new features stable enough to document?
4. If the user said "I want X to do Y" or "add a module for Z", extract into REQUIREMENTS.md as a test flow FIRST

**If you made code changes, you MUST check the docs. This is not optional.**

## ⚠️ Test Timeout Rules (DO NOT SKIP)

- Every test MUST have a timeout matching its module type
- Spawned-process tests MUST clean up in tearDown/finally
- If a test times out: investigate as a bug BEFORE increasing the timeout
- Timeout without cleanup = orphaned process = resource leak = SEVERE BUG
- Pre-commit: verify every subprocess.run/requests/serial.Serial call has timeout=
```
