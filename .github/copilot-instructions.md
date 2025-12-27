# Copilot / AI Agent Instructions for oracle-rman_dup_refresh 🔧

Purpose
- Help an AI agent become productive quickly: explain the project's goal, core files, runtime expectations, and common workflows.

Quick summary
- This repo contains a shell-script based Oracle RMAN "duplicate from active" database refresh utility.
- Main entry: `rman_dup_refresh.sh` (orchestrates steps). It loads `rman_dup_refresh.env` (config) and `rman_dup_refresh.func` (operation functions).

Key files & where to look ✅
- `rman_dup_refresh.sh` – entrypoint, CLI parsing, high-level sequence and safety checks.
- `rman_dup_refresh.env` – environment defaults, ORATAB lookup logic, derived variables (`SOURCE_SID`, `DEST_SID`, `ORAVER`, `RMAN_*`, log paths).
- `rman_dup_refresh.func` – the bulk of actionable steps (e.g., `do_rman_dup_from_active_to_DEST_DB`, `tnsping_DEST_DB`, `execute_post_DB_refresh_script`). Use `grep ^function` to list functions.
- `<SYSTEM>/post_DB_refresh_*.sql` – system- and env-specific SQL scripts executed post-duplication (e.g., `JOSH/post_DB_refresh_JOSH.sql`).
- `JOSH/tns_admin/*` – example TNS configuration used by scripts.

Runtime assumptions & external dependencies ⚠️
- Runs as the Oracle OS user with access to ORATAB and ORACLE_HOME.
- Requires Oracle CLI tools in PATH: `sqlplus`, `rman`, `tnsping`, `lsnrctl`, `srvctl`, `adrci` and a working `/usr/local/bin/oraenv`.
- Expects `$ORATAB` to contain an entry for the destination SID (script exits otherwise).
- Passwords are read from files in `$SYS_DIR` (e.g., `.orapwd_sysdba_${SOURCE_SID}`, `.orapwd_sysbackup_${SOURCE_SID}`) — treat as secrets. Do not commit.

How to run / debug (examples) 🛠
- Basic usage (from repo root):
  ```bash
  ./rman_dup_refresh.sh -system=JOSH -source_env=C1SB -dest_env=C2 -noPrompt
  ```
- Run a single step to test behavior (safe for iteration):
  ```bash
  ./rman_dup_refresh.sh -system=JOSH -source_env=C1SB -dest_env=C2 -func=tnsping_DEST_DB -noPrompt
  ```
- Enable more logging / non-interactive: `-debug` and `-noPrompt` flags.

Project-specific patterns & conventions 🔎
- Naming: SIDs are built from `${SYSTEM}${ENV}` (e.g., `JOSH` + `C1SB`).
- File naming: script basename determines env/func files (e.g., `rman_dup_refresh.env`, `rman_dup_refresh.func`).
- Logging: log files are created in `$SYS_DIR/logs` with timestamped names (`${SCRIPT_NAME}_${SOURCE_SID}_${DEST_SID}_${DATETIME}.log`).
- Functions follow `function name(){ ... }` style; the orchestrator checks for existence via `grep -c ^'function '${FUNCTION_TO_EXEC}`.
- Post refresh SQL: `execute_post_DB_refresh_script` runs `$POST_DB_REFRESH_SCRIPT_DEF_FILE` then `$POST_DB_REFRESH_SCRIPT_SQL_FILE` — system-specific behavior is driven by these SQL files.

Safety & suggested precautions 🧯
- Many functions perform destructive ops (drop database, delete archivelogs). When changing behavior:
  - Add a `--dry-run` or `DRY_RUN=1` environment check and skip destructive commands as a safety enhancement.
  - Add unit-style smoke checks (e.g., `tnsping` and `status_DEST_DB_LSNR`) before destructive steps.
- Never hard-code or commit passwords. Use existing password files or document required secret inputs in the repo's README.

Editing guidance for AI PRs ✍️
- Keep changes small and focused; include a manual testing checklist in the PR description with exact example invocations.
- For modifications that touch runtime behavior, add a non-destructive test mode and at least one example `-func=` invocation that verifies the change.
- Update `README.md` whenever CLI flags or key environment variables change.

Useful commands for exploration
- List functions: `grep ^function rman_dup_refresh.func | awk '{print $2}' | cut -d\( -f1 | sort`
- See env-derived values: read `rman_dup_refresh.env` for ORATAB parsing and `ORAVER` detection logic.

Security & compliance notes 🔐
- Secret files: `$SYS_DIR/.orapwd_*` and `orapw${DEST_SID}` are sensitive — ensure .gitignore excludes such files and never print passwords to logs.
- The repo sets `umask 0066` in `rman_dup_refresh.env` — keep secure file modes if adding files.

If unsure, ask the maintainer
- If a change requires environment-level testing (Oracle home, oratab), request guidance or an ephemeral test environment from the maintainer.

---

If you'd like, I can:
- Add a `--dry-run` flag and a minimal smoke-test suite, or
- Create test helpers (mock wrappers for `sqlplus`/`rman`) so CI can run script checks without an Oracle instance.

Please tell me which next step you'd prefer or what part you'd like clarified. ✅