# Open-Canoe: Reference Example

This project demonstrates the full `project-init` workflow in practice. It has two layers (Python App + C Firmware), a binary communication protocol, real-hardware testing, GUI simulation tests, and 30+ tracked bugs.

## How the workflow was applied

1. **README.md** — Created first, rewritten 3 times as features stabilized. Now concise: features → pin connections → prerequisites → quick start → project structure
2. **doc/SPECIFICATION.md** — 1300+ lines. System architecture with ASCII diagrams of App↔Firmware layers. Complete binary protocol definition (frame format, CRC, command codes). Data flow state machines. Implementation plan.
3. **doc/REQUIREMENTS.md** — 300+ lines. 26 firmware bugs + 10 app bugs tracked. Each bug: symptom → root cause → fix → status (✅/❌/⚠️). Core behavior rules (NORMAL=no RX, LOOPBACK=RX matches TX). Complete UI control checklist (every button/dropdown/checkbox). Cross-module testing rule. Document sync rule.
4. **CLAUDE.md** — Firmware layer rules (SPL-style CAN, poll-driven, never HAL_CAN_Init). Path rules (relative internal, env vars for external). Living documentation checklist.
5. **Directory structure** — `firmware/` + `open-canoe/` layers. `test/` with 5 test scripts. `tools/` with deploy + clean.
6. **Test philosophy** — `test_gui_full.py` does 27-step CAN flow test using `MainWindow()` + `btn.invoke()` + `app.root.update()` to drive event loop. Verifies by reading `_tbl._tree.get_children()` and `_log._text.get()`. `test_ui_controls.py` tests 40 UI items.
7. **Config** — Single `config/defaults.yaml` for all user settings. `gui/config.py` for non-configurable constants (colors, fonts).

## Key lessons

- The bug tracker in REQUIREMENTS.md prevented repeated debugging of the same issues (ISR storm, HAL_CAN_Init unreliability, baudrate timing errors)
- The living documentation rule caught stale docs after every session
- Simulating real GUI clicks caught bugs that callback-only tests missed (ISR storm during loopback, disconnect/reconnect state corruption)
