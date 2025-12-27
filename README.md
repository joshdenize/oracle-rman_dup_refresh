# oracle-rman_dup_refresh

date: 2017-10-31
author: Josh Denize
## Purpose
This repository contains a shell-script based Oracle RMAN "duplicate from active" database refresh utility. It automates the process of cloning an Oracle database from a source to a destination, leveraging RMAN's duplicate from active feature.

## Usage
The `rman_dup_refresh.sh` script is the main entry point for the utility. It orchestrates the entire refresh process, loading configuration from `rman_dup_refresh.env` and executing operational functions from `rman_dup_refresh.func`.

```bash
./rman_dup_refresh.sh -system=$SYSTEM -source_env=$SOURCE_ENV -dest_env=$DEST_ENV [-func=$FUNCTION_TO_EXEC] [-help|-h] [-debug] [-noPrompt]
```

### Parameters:
- `-system`          - **Required**: Name of the system that forms the common prefix of the database SID (e.g., `JOSH`).
- `-source_env`      - **Required**: Name of the source environment that forms the suffix of the database SID (e.g., `C1SB`).
- `-dest_env`        - **Required**: Name of the destination environment that forms the suffix of the database SID (e.g., `C2`).
- `-func`            - **Optional**: Name of a specific function to execute from `rman_dup_refresh.func` (e.g., `tnsping_DEST_DB`). If omitted, the script will run through the full refresh sequence.
- `-noPrompt`        - **Optional**: Setting this option will run with no prompts to the user (default is to prompt if an interactive session).
- `-debug`           - **Optional**: Enables debug logging (default is off).
- `-help|-h`         - **Optional**: Show this usage message.

### Examples:
- **Basic usage (from repo root):**
  ```bash
  ./rman_dup_refresh.sh -system=JOSH -source_env=C1SB -dest_env=C2 -noPrompt
  ```
- **Run a single step to test behavior (safe for iteration):**
  ```bash
  ./rman_dup_refresh.sh -system=JOSH -source_env=C1SB -dest_env=C2 -func=tnsping_DEST_DB -noPrompt
  ```
- **Enable debug logging:**
  ```bash
  ./rman_dup_refresh.sh -system=JOSH -source_env=C1SB -dest_env=C2 -debug -noPrompt
  ```

## Project Structure
- `rman_dup_refresh.sh`: The main script for parsing CLI arguments, orchestrating the refresh sequence, and performing safety checks.
- `rman_dup_refresh.env`: Defines environment variables, handles ORATAB lookup, and derives SID-related variables (`SOURCE_SID`, `DEST_SID`, `ORAVER`, `RMAN_*`, log paths).
- `rman_dup_refresh.func`: Contains the core operational functions for RMAN duplication and post-refresh tasks.
- `JOSH/`: Example directory for a system named `JOSH`. This directory contains system-specific configurations and scripts.
  - `JOSH/post_DB_refresh_*.sql`: System and environment-specific SQL scripts executed after the database duplication.
  - `JOSH/tns_admin/*`: Example TNS configuration files (`listener.ora`, `sqlnet.ora`, `tnsnames.ora`).
  - `JOSH/.orapwd_*`: **Sensitive files** containing Oracle passwords. These files are read by the script but should **never be committed to version control**.
  - `JOSH/logs/`: Directory for log files generated during script execution.
  - `JOSH/backups/`: Directory for database backup files.

## Prerequisites & Assumptions
- The script must run as the Oracle OS user with appropriate permissions to access ORATAB and `ORACLE_HOME`.
- **Oracle Client Tools**: `sqlplus`, `rman`, `tnsping`, `lsnrctl`, `srvctl`, `adrci` must be in the system's PATH.
- **`oraenv`**: A functional `/usr/local/bin/oraenv` script is required for setting Oracle environment variables.
- **ORATAB Entry**: The `$ORATAB` file must contain an entry for the destination SID (`DEST_SID`). The script will exit if this entry is not found.
- **Passwords**: Oracle passwords for `SYSDBA` and `SYSBACKUP` users are read from files located in `$SYS_DIR` (e.g., `JOSH/.orapwd_sysdba_C1SB`). These files are not provided in the repository and must be created by the user, ensuring secure permissions.

## Key Concepts
- **SID Naming Convention**: Database SIDs are constructed using `${SYSTEM}${ENV}` (e.g., `JOSHC1SB` from `JOSH` and `C1SB`).
- **Logging**: Detailed logs are generated in `$SYS_DIR/logs` with timestamped names (e.g., `rman_dup_refresh_JOSHC1SB_JOSHC2_20251227_1200.log`).
- **Post-Refresh SQL**: The `execute_post_DB_refresh_script` function runs two main SQL scripts:
  1. `$POST_DB_REFRESH_SCRIPT_DEF_FILE`: Environment-specific SQL for defining variables.
  2. `$POST_DB_REFRESH_SCRIPT_SQL_FILE`: System-specific SQL for post-refresh operations.

## Safety & Precautions
- This utility performs **destructive operations** such as dropping databases and deleting archivelogs. Exercise extreme caution when modifying its behavior.
- **Dry-Run Mode**: Consider implementing a `--dry-run` flag or a `DRY_RUN=1` environment variable to skip destructive commands during testing.
- **Smoke Checks**: Implement unit-style smoke checks (e.g., `tnsping`, `status_DEST_DB_LSNR`) before executing destructive steps.
- **Password Security**: Never hard-code passwords or commit password files to version control. The `.orapwd_*` files in `$SYS_DIR` should be excluded by `.gitignore`.
- **File Permissions**: The `umask 0066` setting in `rman_dup_refresh.env` ensures that newly created files and directories are only accessible by the Oracle OS user. Maintain secure file modes for any new files.

## Contributing & Extending
- When making modifications, keep changes focused and include a manual testing checklist in your pull request description with example invocations.
- If runtime behavior is altered, add a non-destructive test mode and provide at least one example `-func=` invocation to verify the change.
- Update this `README.md` whenever CLI flags or key environment variables are changed.

## Useful Commands for Exploration
- **List available functions**:
  ```bash
  grep ^function rman_dup_refresh.func | awk '{print $2}' | cut -d\( -f1 | sort
  ```
- **View environment-derived values**:
  Examine `rman_dup_refresh.env` to understand ORATAB parsing and `ORAVER` detection logic.

---
