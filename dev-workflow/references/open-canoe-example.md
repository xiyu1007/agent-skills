# Open-Canoe: Reference Example

This project demonstrates the full `dev-workflow` workflow in practice. It has two layers (Python App + C Firmware), a binary communication protocol, real-hardware testing, GUI simulation tests, and 30+ tracked bugs.

## How the workflow was applied

### 1. README.md
Created first, rewritten 3 times as features stabilized. Now concise: features → pin connections → prerequisites → quick start → project structure.

### 2. doc/SPECIFICATION.md
1300+ lines. System architecture with ASCII diagrams of App↔Firmware layers. Complete binary protocol definition (frame format, CRC, command codes). Data flow state machines. Implementation plan. **Timeout estimates per module** (serial tests: 30s, firmware flash: 120s, GUI interaction: 30s).

### 3. doc/REQUIREMENTS.md
300+ lines. 26 firmware bugs + 10 app bugs tracked. Each bug: symptom → root cause → fix → status (✅/❌/⚠️). Core behavior rules (NORMAL=no RX, LOOPBACK=RX matches TX). Complete UI control checklist (every button/dropdown/checkbox). Cross-module testing rule. Document sync rule. **Test flows with explicit timeouts** — each module's test procedure written before implementation, defining "done" as a contract.

### 4. CLAUDE.md
Firmware layer rules (SPL-style CAN, poll-driven, never HAL_CAN_Init). Path rules (relative internal, env vars for external). Living documentation checklist (review 3 docs after every conversation). **Test timeout rules** (every subprocess.run must have timeout=, spawned-process tests must have cleanup in tearDown, timeout is a bug before it's a number to adjust).

### 5. Directory structure
`firmware/` + `open-canoe/` layers. `test/` with 5 test scripts, **all with timeouts matching module type**. `tools/` with deploy + clean scripts.

### 6. Test philosophy
`test_gui_full.py` does 27-step CAN flow test using `MainWindow()` + `btn.invoke()` + `app.root.update()` to drive event loop. Verifies by reading `_tbl._tree.get_children()` and `_log._text.get()`. `test_ui_controls.py` tests 40 UI items. **All tests have timeouts and cleanup in tearDown** — spawned app processes are killed in tearDown, serial ports closed in finally, preventing resource leaks.

### 7. Config
Single `config/defaults.yaml` for all user settings. `gui/config.py` for non-configurable constants (colors, fonts) — not user settings, don't belong in config file.

## Key lessons

- **Bug tracker prevented repeated debugging**: The same issues (ISR storm, HAL_CAN_Init unreliability, baudrate timing errors) were documented with root causes, so future debugging started from known fixes instead of rediscovering them.
- **Living documentation caught stale docs**: After every session, the doc sync checklist forced review of all three docs. This caught outdated architecture descriptions, missing test criteria, and stale README features.
- **Real GUI clicks caught bugs that callback-only tests missed**: ISR storm during loopback only triggered through the real event path (`btn.invoke()` → `app.root.update()`), not through direct callback calls. Disconnect/reconnect state corruption also only surfaced through real click sequences.
- **Test timeouts prevented hung test suites**: When a serial test hung waiting for bytes that never arrived, the 30s timeout killed it and tearDown closed the port. Without timeout, this would have blocked all subsequent tests and leaked the serial port handle. The hang was tracked as bug #27 in REQUIREMENTS.md.
