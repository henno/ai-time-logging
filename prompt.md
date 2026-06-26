You are an expert systems integrator and AI engineering assistant. Your task is to set up an automated, project-scoped task timer and time tracking system for an AI coding agent in a new Unix/macOS environment.

Please follow the instructions below to create the necessary scripts, set the correct permissions, and document how the agent should use this setup.

---

### Architecture Overview

The system consists of three parts:
1. **State Directory:** A location where CLI-specific and project-scoped state files are saved (default: `~/.local/state/agent-task/<cli>/<project-path>/task.env`).
2. **Global Orchestrator Script (`agent-task`):** Manages start, refresh, restart, and done states. It calculates the elapsed time.
3. **Log Writing Script (`task-log.sh`):** Optional but recommended custom logging script that appends formatted time tracking entries to a central Markdown file (`~/tasks.md`).

---

### Step 1: Create the Time Tracker Script (`~/.local/bin/agent-task`)

Create a script at `~/.local/bin/agent-task` and make it executable (`chmod +x`). 

This script must support four main commands:
* `start` – Starts a new timer (or restores an existing active one for the current project + CLI process).
* `refresh` – Refreshes timestamps without resetting the timer.
* `restart` – Force-starts a clean new timer.
* `done` – Computes active minutes, logs it, and marks the task complete.

Here is the implementation of `agent-task`:

```bash
#!/usr/bin/env bash
# Save this file to ~/.local/bin/agent-task and make it executable (chmod +x)
set -euo pipefail

state_dir="${AGENT_TASK_STATE_DIR:-${XDG_STATE_HOME:-$HOME/.local/state}/agent-task}"

usage() {
  cat >&2 <<'EOF'
Usage:
  eval "$(agent-task start [--new])"
  eval "$(agent-task refresh)"
  eval "$(agent-task restart)"
  agent-task done [--force] [--minutes N] [--issue ISSUE_ID] "task summary"

Commands:
  start    Start a timer, refresh title/timestamp, and print context.
  refresh  Refresh title/timestamp/context without resetting current timer.
  restart  Start a fresh timer for the next work slice.
  done     Log completed work using stored start time or manual --minutes.
EOF
}

lower_value() { printf '%s' "$1" | tr '[:upper:]' '[:lower:]'; }
sanitize_name() { lower_value "$1" | sed 's/[^a-z0-9._-]/-/g; s/--*/-/g; s/^-//; s/-$//'; }
process_parent() { ps -o ppid= -p "$1" 2>/dev/null | tr -d '[:space:]' || true; }
process_comm() {
  local comm
  comm="$(ps -o comm= -p "$1" 2>/dev/null | awk 'NR == 1 {print $1}' || true)"
  comm="${comm##*/}"
  printf '%s\n' "$comm"
}
process_args() { ps -o args= -p "$1" 2>/dev/null | sed 's/^[[:space:]]*//; s/[[:space:]]*$//' || true; }
process_start_stamp() { ps -o lstart= -p "$1" 2>/dev/null | sed 's/^[[:space:]]*//; s/[[:space:]]*$//' || true; }
is_pid_alive() {
  local pid="${1:-}"
  [ -n "$pid" ] || return 1
  case "$pid" in ''|*[!0-9]*) return 1 ;; esac
  kill -0 "$pid" 2>/dev/null
}
pid_matches_start() {
  local pid="${1:-}" expected="${2:-}"
  [ -n "$expected" ] || return 0
  [ "$expected" = "unknown" ] && return 0
  [ "$(process_start_stamp "$pid")" = "$expected" ]
}

known_cli_from_text() {
  local text
  text="$(lower_value "$1")"
  case "$text" in
    *pi-coding-agent*|*/pi\ *|pi|*' pi '*) printf 'pi\n'; return 0 ;;
    *claude-code*|*claude*) printf 'claude-code\n'; return 0 ;;
    *codex*) printf 'codex\n'; return 0 ;;
    *antigravity*|*agy*) printf 'antigravity\n'; return 0 ;;
    *opencode*) printf 'opencode\n'; return 0 ;;
    *gemini*) printf 'gemini\n'; return 0 ;;
    *cursor-agent*) printf 'cursor-agent\n'; return 0 ;;
    *) return 1 ;;
  esac
}

detect_agent_context() {
  local pid parent comm args cli depth
  if [ -n "${AGENT_TASK_CLI:-}" ]; then
    agent_cli="$(sanitize_name "$AGENT_TASK_CLI")"
    session_pid="${AGENT_TASK_OWNER_PID:-$PPID}"
    session_start="$(process_start_stamp "$session_pid")"
    [ -n "$session_start" ] || session_start="unknown"
    return 0
  fi
  pid="$$"
  depth=0
  while [ -n "$pid" ] && [ "$pid" != "0" ] && [ "$depth" -lt 30 ]; do
    comm="$(process_comm "$pid")"
    args="$(process_args "$pid")"
    if cli="$(known_cli_from_text "$comm $args")"; then
      agent_cli="$(sanitize_name "$cli")"
      session_pid="$pid"
      session_start="$(process_start_stamp "$pid")"
      [ -n "$session_start" ] || session_start="unknown"
      return 0
    fi
    parent="$(process_parent "$pid")"
    [ -n "$parent" ] || break
    pid="$parent"
    depth=$((depth + 1))
  done
  agent_cli="unknown"
  session_pid="$PPID"
  session_start="$(process_start_stamp "$session_pid")"
  [ -n "$session_start" ] || session_start="unknown"
}

canonical_project_dir() {
  local dir
  if [ -n "${AGENT_TASK_PROJECT_DIR:-}" ]; then
    dir="$AGENT_TASK_PROJECT_DIR"
  elif git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
    dir="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
  else
    dir="$PWD"
  fi
  if [ -d "$dir" ]; then
    (cd "$dir" && pwd -P)
  else
    printf '%s\n' "$dir"
  fi
}

write_env_var() { printf '%s=%q\n' "$1" "$2"; }
state_value() {
  local file="$1" key="$2"
  [ -f "$file" ] || return 1
  bash -c '
    set -euo pipefail
    file=$1; key=$2
    . "$file"
    eval "printf '\''%s\\n'\'' \"\${${key}:-}\""
  ' _ "$file" "$key"
}
task_field() { state_value "$task_file" "$1" 2>/dev/null || true; }
is_truthy() { case "$(lower_value "$1")" in 1|true|yes|y|on) return 0 ;; *) return 1 ;; esac; }
is_falsey() { case "$(lower_value "$1")" in 0|false|no|n|off) return 0 ;; *) return 1 ;; esac; }
is_subagent_role() {
  case "$(lower_value "$1")" in
    subagent|sub-agent|sub_agent|reviewer|review|worker|planner|scout|security|maintainability|performance) return 0 ;;
    *) return 1 ;;
  esac
}

should_skip_done_log() {
  if [ -n "${AGENT_TASK_LOGGING:-}" ] && is_truthy "$AGENT_TASK_LOGGING"; then return 1; fi
  if [ -n "${AGENT_TASK_LOGGING:-}" ] && is_falsey "$AGENT_TASK_LOGGING"; then return 0; fi
  if [ -n "${AGENT_TASK_SUBAGENT:-}" ] && is_truthy "$AGENT_TASK_SUBAGENT"; then return 0; fi
  if [ -n "${AGENT_IS_SUBAGENT:-}" ] && is_truthy "$AGENT_IS_SUBAGENT"; then return 0; fi
  if [ -n "${AGENT_ROLE:-}" ] && is_subagent_role "$AGENT_ROLE"; then return 0; fi
  return 1
}

generate_task_id() {
  local uuid rand
  if command -v uuidgen >/dev/null 2>&1; then
    uuid="$(uuidgen | tr '[:upper:]' '[:lower:]')"
    printf 'task-%s\n' "$uuid"
    return 0
  fi
  rand="$( (od -An -N4 -tx1 /dev/urandom 2>/dev/null || printf '%s' "$RANDOM$RANDOM") | tr -d '[:space:]')"
  printf 'task-%s-%s-%s\n' "$(date +%s)" "$$" "$rand"
}

write_task_state() {
  local task_id="$1" status="$2" start_epoch="$3" last_epoch="$4" done_epoch="${5:-}" done_minutes="${6:-}" done_summary="${7:-}"
  local tmp dir
  dir="$(dirname "$task_file")"
  mkdir -p "$dir"
  chmod 700 "$state_dir" 2>/dev/null || true
  tmp="$task_file.$$"
  {
    write_env_var AGENT_TASK_ID "$task_id"
    write_env_var AGENT_TASK_STATUS "$status"
    write_env_var AGENT_TASK_START_EPOCH "$start_epoch"
    write_env_var AGENT_TASK_LAST_TOUCH_EPOCH "$last_epoch"
    write_env_var AGENT_TASK_CLI "$agent_cli"
    write_env_var AGENT_TASK_PROJECT_DIR "$project_dir"
    write_env_var AGENT_TASK_CWD "$PWD"
    write_env_var AGENT_TASK_OWNER_PID "$session_pid"
    write_env_var AGENT_TASK_OWNER_START "$session_start"
    write_env_var AGENT_TASK_STATE_FILE "$task_file"
    if [ -n "$done_epoch" ]; then
      write_env_var AGENT_TASK_DONE_EPOCH "$done_epoch"
      write_env_var AGENT_TASK_DONE_MINUTES "$done_minutes"
      write_env_var AGENT_TASK_DONE_SUMMARY "$done_summary"
    fi
  } > "$tmp"
  mv "$tmp" "$task_file"
}

refresh_now() {
  local now_exports epoch timestamp dir branch
  epoch="$(date +%s)"
  timestamp="$(date '+%Y-%m-%d %H:%M:%S %Z')"
  dir="$(basename "$PWD")"
  branch="$(git branch --show-current 2>/dev/null || true)"
  [ -n "$branch" ] || branch="no-git"
  printf '\033]0;%s@%s\007' "$dir" "$branch" >&2
  AGENT_NOW_EPOCH="$epoch"
  AGENT_NOW_TIMESTAMP="$timestamp"
  export AGENT_NOW_EPOCH AGENT_NOW_TIMESTAMP
}

read_start_epoch() {
  local file value
  for file in "${AGENT_TASK_STATE_FILE:-}" "$task_file"; do
    [ -n "$file" ] || continue
    value="$(state_value "$file" AGENT_TASK_START_EPOCH 2>/dev/null || true)"
    if [ -n "$value" ]; then
      printf '%s\n' "$value"; return 0
    fi
  done
  if [ -n "${AGENT_TASK_START_EPOCH:-}" ]; then
    printf '%s\n' "$AGENT_TASK_START_EPOCH"; return 0
  fi
  return 1
}

emit_exports() {
  local task_id="$1" start_epoch="$2"
  printf 'export AGENT_NOW_EPOCH=%q\n' "${AGENT_NOW_EPOCH:-}"
  printf 'export AGENT_NOW_TIMESTAMP=%q\n' "${AGENT_NOW_TIMESTAMP:-}"
  printf 'export AGENT_TASK_ID=%q\n' "$task_id"
  printf 'export AGENT_TASK_START_EPOCH=%q\n' "$start_epoch"
  printf 'export AGENT_TASK_STATE_FILE=%q\n' "$task_file"
  printf 'export AGENT_TASK_CLI=%q\n' "$agent_cli"
  printf 'export AGENT_TASK_PROJECT_DIR=%q\n' "$project_dir"
}

print_context() {
  local task_id="$1" start_epoch="$2" status="$3" existing="$4" owner_alive="$5" conflict="$6"
  local repo_root branch default_branch git_status owner_pid last_touch
  printf 'cwd=%s\n' "$PWD" >&2
  if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
    repo_root="$(git rev-parse --show-toplevel 2>/dev/null || true)"
    branch="$(git branch --show-current 2>/dev/null || true)"
    default_branch="$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null | sed 's#^origin/##' || true)"
    printf 'git=true\nrepo_root=%s\nbranch=%s\ndefault_branch=%s\n' "$repo_root" "${branch:-detached}" "${default_branch:-unknown}" >&2
  else
    printf 'git=false\n' >&2
  fi
  owner_pid="$(task_field AGENT_TASK_OWNER_PID)"
  printf 'AGENT_TASK_ID=%s\nAGENT_TASK_STATUS=%s\nAGENT_TASK_CONFLICT=%s\n' "$task_id" "$status" "$conflict" >&2
}

start_cmd() {
  local reset=0 status task_id start_epoch existing owner_alive conflict owner_pid owner_start
  while [ "$#" -gt 0 ]; do
    case "$1" in
      --new|--reset) reset=1; shift ;;
      *) usage; exit 2 ;;
    esac
  done
  refresh_now
  status="$(task_field AGENT_TASK_STATUS)"
  if [ "$reset" -eq 1 ] || [ "$status" != "active" ]; then
    task_id="$(generate_task_id)"
    start_epoch="$AGENT_NOW_EPOCH"
    existing=false
    owner_alive=false
    conflict=false
    write_task_state "$task_id" active "$start_epoch" "$AGENT_NOW_EPOCH"
  else
    task_id="$(task_field AGENT_TASK_ID)"
    start_epoch="$(task_field AGENT_TASK_START_EPOCH)"
    existing=true
    owner_pid="$(task_field AGENT_TASK_OWNER_PID)"
    owner_start="$(task_field AGENT_TASK_OWNER_START)"
    if is_pid_alive "$owner_pid" && pid_matches_start "$owner_pid" "$owner_start"; then owner_alive=true; else owner_alive=false; fi
    if [ "$owner_alive" = true ] && [ "${owner_pid:-}" != "$session_pid" ]; then
      conflict=true
    else
      conflict=false
      write_task_state "$task_id" active "$start_epoch" "$AGENT_NOW_EPOCH"
    fi
  fi
  print_context "$task_id" "$start_epoch" active "$existing" "$owner_alive" "$conflict"
  emit_exports "$task_id" "$start_epoch"
}

refresh_cmd() {
  local status task_id start_epoch existing owner_alive conflict owner_pid owner_start
  refresh_now
  status="$(task_field AGENT_TASK_STATUS)"
  if [ "$status" = "active" ]; then
    task_id="$(task_field AGENT_TASK_ID)"
    start_epoch="$(task_field AGENT_TASK_START_EPOCH)"
    existing=true
    owner_pid="$(task_field AGENT_TASK_OWNER_PID)"
    owner_start="$(task_field AGENT_TASK_OWNER_START)"
    if is_pid_alive "$owner_pid" && pid_matches_start "$owner_pid" "$owner_start"; then owner_alive=true; else owner_alive=false; fi
    if [ "$owner_alive" = true ] && [ "${owner_pid:-}" != "$session_pid" ]; then conflict=true; else conflict=false; write_task_state "$task_id" active "$start_epoch" "$AGENT_NOW_EPOCH"; fi
  else
    task_id="$(generate_task_id)"
    start_epoch="$AGENT_NOW_EPOCH"
    existing=false
    owner_alive=false
    conflict=false
    write_task_state "$task_id" active "$start_epoch" "$AGENT_NOW_EPOCH"
  fi
  print_context "$task_id" "$start_epoch" active "$existing" "$owner_alive" "$conflict"
  emit_exports "$task_id" "$start_epoch"
}

validate_minutes() {
  case "$1" in ''|*[!0-9]*) return 1 ;; *) [ "$1" -ge 1 ] ;; esac
}

done_cmd() {
  local issue="${ISSUE_ID:-}"
  local force_log=0 manual_minutes=""
  local summary status task_id start_epoch end_epoch minutes owner_pid owner_start
  while [ "$#" -gt 0 ]; do
    case "$1" in
      --issue) issue="$2"; shift 2 ;;
      --issue=*) issue="${1#--issue=}"; shift ;;
      --force) force_log=1; shift ;;
      --minutes|--estimated-minutes|--estimate) manual_minutes="$2"; shift 2 ;;
      --minutes=*|--estimated-minutes=*|--estimate=*) manual_minutes="${1#*=}"; shift ;;
      *) break ;;
    esac
  done
  [ "$#" -gt 0 ] || { usage; exit 2; }
  summary="$*"
  if [ "$force_log" -ne 1 ] && should_skip_done_log; then
    printf 'Skipped task log for subagent role (%s): %s\n' "${AGENT_ROLE:-subagent}" "$summary" >&2
    return 0
  fi
  status="$(task_field AGENT_TASK_STATUS)"
  owner_pid="$(task_field AGENT_TASK_OWNER_PID)"
  owner_start="$(task_field AGENT_TASK_OWNER_START)"
  if [ "$force_log" -ne 1 ] && [ "$status" = active ] && is_pid_alive "$owner_pid" && pid_matches_start "$owner_pid" "$owner_start" && [ "${owner_pid:-}" != "$session_pid" ]; then
    printf 'agent-task: refusing to log active task owned by live process %s. Use --force.\n' "$owner_pid" >&2; exit 2
  fi
  end_epoch="$(date +%s)"
  task_id="$(task_field AGENT_TASK_ID)"
  [ -n "$task_id" ] || task_id="$(generate_task_id)"

  if [ -n "$manual_minutes" ]; then
    validate_minutes "$manual_minutes"
    minutes="$manual_minutes"
    start_epoch="$(task_field AGENT_TASK_START_EPOCH)"
    [ -n "$start_epoch" ] || start_epoch=$(( end_epoch - minutes * 60 ))
  else
    start_epoch="$(read_start_epoch)" || start_epoch="$end_epoch"
    minutes=$(( (end_epoch - start_epoch + 59) / 60 ))
    [ "$minutes" -lt 1 ] && minutes=1
  fi

  # Call custom task-log.sh if available, otherwise append directly to ~/tasks.md
  if [ -x "$HOME/.claude/task-log.sh" ]; then
    if [ -n "$issue" ]; then
      "$HOME/.claude/task-log.sh" --issue "$issue" --minutes "$minutes" Done "$summary"
    else
      "$HOME/.claude/task-log.sh" --minutes "$minutes" Done "$summary"
    fi
  else
    printf '[%s] [%smin] [%s]\n' "$(date '+%Y-%m-%d %H:%M')" "$minutes" "$summary" >> "$HOME/tasks.md"
  fi
  write_task_state "$task_id" done "$start_epoch" "$end_epoch" "$end_epoch" "$minutes" "$summary"
}

agent_cli=""
session_pid=""
session_start=""
detect_agent_context
project_dir="$(canonical_project_dir)"
project_relative="${project_dir#/}"
[ -n "$project_relative" ] || project_relative="root"
task_file="$state_dir/$agent_cli/$project_relative/task.env"

cmd="${1:-}"
case "$cmd" in
  start) shift; start_cmd "$@" ;;
  refresh) shift; refresh_cmd "$@" ;;
  restart) shift; restart_cmd ;;
  done|log) shift; done_cmd "$@" ;;
  *) usage; exit 2 ;;
esac
```

---

### Step 2: Create the Custom Log Formatter (`~/.claude/task-log.sh`)

Create a script at `~/.claude/task-log.sh` and make it executable (`chmod +x`). 

This script is called by `agent-task done` and is responsible for adding the path context (relative to `$HOME` using `~`) and appending a uniform line to `~/tasks.md`.

```bash
#!/bin/bash
# Save this file to ~/.claude/task-log.sh and make it executable (chmod +x)
issue=""
minutes=""

while [[ "$1" == --* ]]; do
  case "$1" in
    --issue) issue="$2"; shift 2 ;;
    --minutes) minutes="$2"; shift 2 ;;
    *) shift ;;
  esac
done

action="$1"
shift
description="$*"

suffix=""
if [ "$action" = "Done" ] && [ -n "$minutes" ]; then
  suffix=" (${minutes}min)"
fi

if [ -n "$issue" ]; then
  echo "[$(date '+%Y-%m-%d %H:%M') $(pwd | sed "s|$HOME|~|")] $action $issue: $description$suffix" >> ~/tasks.md
else
  echo "[$(date '+%Y-%m-%d %H:%M') $(pwd | sed "s|$HOME|~|")] $action $description$suffix" >> ~/tasks.md
fi
```

---

### Step 3: Configure Agent System Rules (`~/AGENTS.md`)

Add the following instructions to the system prompt or agent configuration file (e.g., `~/AGENTS.md` or `.agents/AGENTS.md` at the project root) to instruct the AI agent to use the time tracking system:

```markdown
## Task Time Logging Process

1. **Refresh Context and Start Timer**: When meaningful work begins, call the global helper once. This records the task start time, refreshes timestamp variables, updates the terminal title, and outputs project context:
   ```bash
   eval "$(agent-task start)"
```
2. **Refresh Context on Directory Change**: If the working directory changes or after initial analysis, call the refresh helper:
   ```bash
   eval "$(agent-task refresh)"
   ```
3. **Log Completed Work**: Before returning control to the user, log completed meaningful work using:
   ```bash
   agent-task done "your concise summary of what was completed"
   ```
   If a specific Issue/Ticket ID (e.g., `GH-123`, `BB-45`) is known, always attach it using:
   ```bash
   agent-task done --issue "ISSUE_ID" "your concise summary of what was completed"
   ```
   *Note: Subagents and reviewer agents must not call `agent-task done` unless explicitly instructed.*
```

---

### Step 4: Verification

Test the setup by running these commands in your shell:
1. Initialize the task:
   ```bash
   eval "$(agent-task start)"
```
2. Wait a minute and complete the task:
   ```bash
   agent-task done "Setup and tested time tracker"
   ```
3. Verify that the task is logged in `~/tasks.md`. It should format as:
   ```text
   [YYYY-MM-DD HH:MM ~/path/to/project] Done Setup and tested time tracker (1min)
   ```
