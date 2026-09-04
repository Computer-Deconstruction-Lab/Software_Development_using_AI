# LLM Written Python code

> Write a python program that uses notify-send to send alerts to the console. Read an ical file for daily events and schedule them via cron, using at, to display those alerts.

[Start thinking]

The user wants a Python program that:
1. Reads an iCalendar (.ics) file.
2. Parses daily events.
3. Schedules alerts for those events.
4. Uses `at` to schedule the alerts.
5. Uses `notify-send` to display the alerts.
6. Runs in the background or is set up to run continuously/cronned.

Let's break down the requirements and constraints:
- **Language**: Python
- **Input**: `.ics` file
- **Parsing**: `python-dateutil` and `icalendar` libraries are standard for this.
- **Scheduling**: `at` command. The prompt says "schedule them via cron, using at". This is slightly contradictory or means "use a cron job to run the Python script, and have the Python script use `at` to schedule individual events". I'll design it so a cron job runs the script periodically (e.g., every minute or hour), and the script checks for upcoming events, uses `at` to schedule a `notify-send` command at the exact event time, and marks them as scheduled.
- **Output**: `notify-send` for desktop notifications.
- **Robustness**: Need to handle timezones, avoid rescheduling already notified events, and ensure the script runs safely as a background process.

Key components:
1. Parse `.ics` file to get events with start times.
2. Filter events for "today" or "upcoming".
3. Check a state file (e.g., `scheduled_events.json` or `notify.log`) to avoid duplicates.
4. For each unscheduled upcoming event, calculate the delay until it starts.
5. Use `at <time>` or `at now + <X> minutes` to schedule a command.
6. The command executed by `at` should run `notify-send` with the event details.
7. Since `at` runs in a minimal environment, it might need `DBUS_SESSION_BUS_ADDRESS` or `DISPLAY` to work with `notify-send`. I'll need to capture these from the parent process or instruct the user on how to set it up, or use a helper script that preserves the environment. Actually, `notify-send` often fails in `at` jobs unless the D-Bus session bus is exported. I'll make the script capture `DBUS_SESSION_BUS_ADDRESS` and `DISPLAY` from `/proc/$$/environ` or pass them, or better, use a small wrapper that reads the environment. Actually, `at` inherits environment variables by default. I'll add a note about `DBUS_SESSION_BUS_ADDRESS`.

Wait, `at` jobs run in a separate shell. `notify-send` requires a D-Bus session bus. I'll make the Python script write a small shell script to `/tmp` that exports the necessary environment variables and then runs `notify-send`, then pass that to `at`.

Let's design the Python script:
- Dependencies: `icalendar`, `python-dateutil`
- State management: JSON file to track notified events (by UID or date+summary).
- Main loop or cron entry: `0 * * * * /path/to/script.py` or `*/5 * * * *`. The prompt says "using cron, using at". I'll assume a cron job triggers the script periodically.
- The script will:
  1. Read ICS.
  2. Filter events in the next hour (or day).
  3. Check state file.
  4. For new events, calculate time difference.
  5. Use `subprocess.run(['at', '-t', f'{timestamp}'])` or `echo "command" | at now + {minutes} minutes`. Actually, `at` expects a specific time format or relative. I'll use `at now + {delta} minutes`.
  6. The command sent to `at` will be a shell script that calls `notify-send`.

Important caveat: `notify-send` in background/`at` often fails without proper D-Bus. I'll explicitly handle exporting `DBUS_SESSION_BUS_ADDRESS` and `DISPLAY` by reading them from the parent process's `/proc/self/environ`.

Let's draft the code structure:

```python
#!/usr/bin/env python3
import os
import sys
import json
import subprocess
import sqlite3
from datetime import datetime, timedelta
from pathlib import Path
import icalendar
from dateutil import parser as dateutil_parser

# Config
ICS_FILE = Path("/path/to/calendar.ics")
STATE_DB = Path("/tmp/ical_notifier_state.json")
CHECK_INTERVAL_MINUTES = 5 # How often cron runs this
NOTIFY_DELAY_MINUTES = 5 # Notify X minutes before event? Or at start time. Prompt says "schedule them... to display those alerts". I'll notify at start time.

def get_env_for_at():
    """Capture D-Bus and display env vars from parent process."""
    with open(f"/proc/{os.getppid()}/environ", "rb") as f:
        env = f.read().decode("utf-8", errors="ignore")
    vars = {}
    for line in env.split("\x00"):
        if "=" in line:
            k, _, v = line.partition("=")
            vars[k] = v
    return vars

def parse_ics(filepath):
    cal = icalendar.Calendar.from_ical(Path(filepath).read_bytes())
    events = []
    for component in cal.walk():
        if component.name == "VEVENT":
            start = component.get("DTSTART").dt
            summary = str(component.get("SUMMARY", "No Title"))
            desc = str(component.get("DESCRIPTION", ""))
            # Handle naive vs aware datetime
            if hasattr(start, 'tzinfo') and start.tzinfo is None:
                start = start.replace(tzinfo=None)
            elif start.tzinfo is not None:
                start = start.astimezone()
            events.append({"start": start, "summary": summary, "description": desc})
    return events

def load_state():
    if STATE_DB.exists():
        return json.loads(STATE_DB.read_text())
    return {"notified": []}

def save_state(state):
    STATE_DB.write_text(json.dumps(state))

def schedule_alert(summary, description):
    env = get_env_for_at()
    display = env.get("DISPLAY", ":0")
    dbus_addr = env.get("DBUS_SESSION_BUS_ADDRESS")

    # Escape quotes for shell
    safe_summary = summary.replace('"', '\\"')
    safe_desc = description.replace('"', '\\"')

    cmd = f'export DISPLAY="{display}"; {f"export DBUS_SESSION_BUS_ADDRESS=\\"{dbus_addr}\\";" if dbus_addr else ""}notify-send "{safe_summary}" "{safe_desc}"'

    try:
        subprocess.run(["at", "-m", "now + 0 minutes"], input=cmd.encode(), check=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    except Exception as e:
        print(f"Failed to schedule alert: {e}", file=sys.stderr)

def main():
    if not ICS_FILE.exists():
        print(f"Ics file not found: {ICS_FILE}", file=sys.stderr)
        sys.exit(1)

    now = datetime.now()
    window_end = now + timedelta(minutes=CHECK_INTERVAL_MINUTES)

    events = parse_ics(ICS_FILE)
    state = load_state()

    for event in events:
        start = event["start"]
        uid = event.get("uid", f"{event['summary']}_{start.isoformat()}")

        # Skip past events or events outside the window
        if start < now or start > window_end:
            continue

        if uid in state["notified"]:
            continue

        # Schedule via at
        # We use "at now + X minutes" where X is minutes until event
        delta = (start - now).total_seconds() / 60
        minutes = max(0.1, delta) # Ensure it's > 0

        cmd = f'echo "export DISPLAY=\\"{os.environ.get("DISPLAY", ":0")}\\"; export DBUS_SESSION_BUS_ADDRESS=\\"{os.environ.get("DBUS_SESSION_BUS_ADDRESS", "")}\\"; notify-send \\"{event["summary"]}\\" \\"{event["description"]}\\"" | at now + {minutes} minutes'
        # Actually, at expects a script or command. Let's construct a temp script for reliability.

        script_path = Path(f"/tmp/ical_notify_{uid.replace('/', '_').replace(' ', '_')}.sh")
        dbus = os.environ.get("DBUS_SESSION_BUS_ADDRESS", "")
        display = os.environ.get("DISPLAY", ":0")
        script_path.write_text(f"""#!/bin/sh
export DISPLAY="{display}"
{f'export DBUS_SESSION_BUS_ADDRESS="{dbus}"' if dbus else ""}
exec notify-send "{event['summary'].replace('"', '\\"')}" "{event['description'].replace('"', '\\"')}"
""")
        script_path.chmod(0o755)

        subprocess.run(["at", "now + 0 minutes"], input=f"/tmp/ical_notify_{uid.replace('/', '_').replace(' ', '_')}.sh\n".encode())

        state["notified"].append(uid)

    save_state(state)
    print(f"Checked {len(events)} events. Notified {sum(1 for e in events if e['start'] >= now and e['start'] <= window_end and any(e['start'] == ...))}") # simplified print

if __name__ == "__main__":
    main()
```

Wait, `at` might not be installed by default, but the prompt says "using at". I should add a check and install instructions or just assume it's there. I'll add a note about dependencies.
Also, `at`'s behavior with "now + X minutes" can be flaky if the system time changes or if `atd` isn't running. I'll make sure to check for `atd`.
The prompt says "schedule them via cron, using at". This implies a cron job runs the script periodically. I'll structure it so the script is meant to be called by cron (e.g., `*/1 * * * * /path/to/ical_notifier.py`).

Let's refine the script for robustness:
1. Handle timezone properly. `dateutil` and `icalendar` can be tricky with naive datetimes. I'll convert everything to local time.
2. State file management: Keep it simple.
3. `notify-send` environment: Crucial to export `DBUS_SESSION_BUS_ADDRESS`. I'll read it from `/proc/self/environ` or `os.environ`.
4. `at` usage: I'll use `at -f <script> now + X minutes` if available, or pipe a script. `at -f` is cleaner.
5. I'll add a `schedule()` function that calculates the exact time and uses `at -t` (time format YYYYMMDDhhmm.ss) for precision, or `at now + X minutes`. `at now + X minutes` is easier.

Let's rewrite carefully.

```python
#!/usr/bin/env python3
"""
ical_notifier.py
Reads an .ics file, checks for upcoming events, and schedules desktop notifications using `at` and `notify-send`.
Intended to be run periodically via cron (e.g., every minute or 5 minutes).
"""

import os
import sys
import json
import subprocess
import tempfile
from datetime import datetime, timedelta
from pathlib import Path

# --- Configuration ---
ICS_FILE = Path("/path/to/your/calendar.ics")  # Update this path
STATE_FILE = Path("/tmp/ical_notifier_state.json")
WINDOW_MINUTES = 10  # Look ahead this many minutes for new events
NOTIFY_BEHIND_MINUTES = 0  # Notify X minutes before event (0 = exactly on time)
# ---------------------

def ensure_at_running():
    if not subprocess.run(["which", "at"], capture_output=True).returncode == 0:
        print("Error: `at` is not installed. Install it via your package manager.", file=sys.stderr)
        sys.exit(1)
    # Check if atd daemon is running
    if not subprocess.run(["pgrep", "atd"], capture_output=True).returncode == 0:
        print("Warning: `atd` daemon is not running. Alerts may not trigger on time.", file=sys.stderr)

def parse_ics(filepath):
    try:
        from icalendar import Calendar
        from dateutil import parser as dateutil_parser
    except ImportError:
        print("Error: Missing dependencies. Install them via `pip install icalendar python-dateutil`", file=sys.stderr)
        sys.exit(1)

    events = []
    if not filepath.exists():
        print(f"Error: ICS file not found at {filepath}", file=sys.stderr)
        sys.exit(1)

    with open(filepath, "rb") as f:
        cal = Calendar.from_ical(f.read())

    now = datetime.now().astimezone()

    for component in cal.walk():
        if component.name == "VEVENT":
            start_dt = component.get("DTSTART").dt
            # Handle timezone naive vs aware
            if hasattr(start_dt, 'tzinfo') and start_dt.tzinfo is None:
                start_dt = start_dt.replace(tzinfo=now.tzinfo)
            else:
                start_dt = start_dt.astimezone()

            # Only keep future events in our lookup window
            if start_dt <= now:
                continue

            summary = str(component.get("SUMMARY", "Untitled Event"))
            description = str(component.get("DESCRIPTION", ""))
            uid = str(component.get("UID", f"{summary}_{start_dt.isoformat()}"))

            events.append({
                "uid": uid,
                "summary": summary,
                "description": description,
                "start": start_dt
            })

    return events

def load_state():
    if STATE_FILE.exists():
        return json.loads(STATE_FILE.read_text())
    return {"notified": []}

def save_state(state):
    STATE_FILE.write_text(json.dumps(state))

def schedule_at_command(script_path, minutes_delay):
    """Schedules a script execution using `at`."""
    try:
        # at now + X minutes
        proc = subprocess.run(
            ["at", f"now + {minutes_delay} minutes"],
            input=script_path.read_text().encode(),
            stdout=subprocess.DEVNULL,
            stderr=subprocess.PIPE,
            text=True
        )
        if proc.returncode != 0:
            print(f"at command failed: {proc.stderr.strip()}", file=sys.stderr)
            return False
        return True
    except Exception as e:
        print(f"Failed to schedule via at: {e}", file=sys.stderr)
        return False

def main():
    ensure_at_running()

    now = datetime.now().astimezone()
    window_end = now + timedelta(minutes=WINDOW_MINUTES)

    events = parse_ics(ICS_FILE)
    state = load_state()
    notified_count = 0

    # Environment for notify-send in background jobs
    display = os.environ.get("DISPLAY", ":0")
    dbus_addr = os.environ.get("DBUS_SESSION_BUS_ADDRESS", "")

    for event in events:
        # Check if within the look-ahead window
        if event["start"] > window_end:
            continue

        # Skip already notified
        if event["uid"] in state["notified"]:
            continue

        # Calculate delay
        delta = (event["start"] - now).total_seconds() / 60
        delay_minutes = max(0.1, delta + NOTIFY_BEHIND_MINUTES) # +0 for exact, adjust if needed

        # Create a temporary shell script for at to execute
        # `at` jobs run in a minimal environment, so we explicitly export D-Bus vars
        with tempfile.NamedTemporaryFile(mode="w", suffix=".sh", delete=False, dir="/tmp") as tmp:
            tmp.write(f"""#!/bin/sh
export DISPLAY="{display}"
{f'export DBUS_SESSION_BUS_ADDRESS="{dbus_addr}"' if dbus_addr else ""}
exec /usr/bin/notify-send "{event['summary'].replace('"', '\\"')}" "{event['description'].replace('"', '\\"')}"
""")
            tmp_path = Path(tmp.name)

        if schedule_at_command(tmp_path, delay_minutes):
            state["notified"].append(event["uid"])
            notified_count += 1
            print(f"Scheduled notification for: {event['summary']} at {event['start'].strftime('%H:%M')}")
        else:
            print(f"Failed to schedule: {event['summary']}", file=sys.stderr)

    save_state(state)
    if notified_count == 0:
        print("No new events to schedule in the current window.")

if __name__ == "__main__":
    main()
```

Wait, `at` might not like reading from a temporary file created in Python if it's deleted immediately, but I'm not deleting it until the script exits or it's handled. Actually, `at` copies the script contents into its own queue, so the temp file can be deleted after `schedule_at_command` returns. I'll add `os.unlink(tmp_path)` after scheduling to keep `/tmp` clean.

Also, `at`'s `now + X minutes` can be imprecise if the system is under load. For a robust cron+at setup, it's fine. I'll add a note about `atd` and D-Bus.

One important detail: `notify-send` might fail if `DBUS_SESSION_BUS_ADDRESS` isn't correctly passed. Reading from `os.environ` works if the script is run interactively or via cron in a session that has it. Cron usually strips `DBUS_SESSION_BUS_ADDRESS`. If running via cron, the user might need to export it in their crontab or use `systemd` user timers. I'll add a robust fallback: try to get it from `/proc/$PPID/environ` or just document it. Actually, if cron runs it, `os.environ.get("DBUS_SESSION_BUS_ADDRESS")` will be empty. I'll add a comment explaining this and suggest a workaround (setting it in crontab or using `sh -c 'export ...'`).

Let's adjust the environment handling to be more robust for cron:
If `DBUS_SESSION_BUS_ADDRESS` is missing, `notify-send` often falls back to a default session bus or fails. I'll leave it as is but add a clear configuration note.

Also, the prompt says "schedule them via cron, using at". I'll make the script self-contained and provide the exact `crontab` line.

I'll refine the code to be production-ready:
- Add shebang and comments.
- Handle missing dependencies gracefully.
- Ensure temp files are cleaned up.
- Provide installation and cron setup instructions.

One thing: `at` queue names might be an issue on some systems, but default is fine.
I'll also add a check to avoid scheduling the same event twice if the script runs while `at` is still pending. The `state["notified"]` list handles this, but I should clear old notifications after the event time to prevent the list from growing infinitely. I'll add a cleanup step.

Revised cleanup logic:
```python
    # Clean up old notified events (older than window_end)
    state["notified"] = [uid for uid in state["notified"] if any(e["uid"] == uid and e["start"] > window_end for e in events)]
```
This is a bit complex with nested loops. Simpler: keep state file, but prune entries older than 1 day or where the event time has passed. I'll just keep it simple: state file grows, but that's fine for personal use. I'll add a comment.

Actually, I'll just track notified events and prune any where `event['start'] < now - timedelta(days=1)`.

Let's put it all together carefully.

Structure of response:
1. Python script
2. Dependencies installation
3. Setup instructions (cron, paths, D-Bus note)
4. Direct, no fluff.

Check constraints: "Answer directly, after thinking. Lead with the answer... Never open with preamble... Keep essential steps, caveats... Keep the final answer lean."

I will output the code block first, then the concise setup/caveats.

Code structure:
```python
#!/usr/bin/env python3
import os, sys, json, subprocess, tempfile
from datetime import datetime, timedelta
from pathlib import Path

ICS_FILE = Path("/path/to/calendar.ics")
STATE_FILE = Path("/tmp/ical_notifier_state.json")
WINDOW_MINUTES = 10

def main():
    if not shutil.which("at"):
        sys.exit("Error: `at` is not installed.")
    # ... rest of logic ...
```
Wait, `shutil` is standard. I'll use it.

I'll make sure the code is complete and runnable.

One detail: `at` on some systems (like macOS) uses `launchd`, but the prompt specifies `at`, implying Linux. I'll note Linux requirement.

Final check of the code logic:
- Parses ICS
- Filters upcoming events within WINDOW_MINUTES
- Checks state file
- Creates temp shell script exporting DISPLAY/DBUS
- Schedules via `at now + X minutes`
- Updates state file
- Cleans up old state
- Prints status

All solid. I'll generate it now.
I will ensure the output strictly follows the prompt. No greetings. Direct code. Then brief setup notes.
Proceeds.
Output matches response.
(Self-Correction/Refinement during thought)
I should make sure `at` handles the script correctly. `at` expects a command or script. Piping to `at` works, but creating a temp file and running `at -f` is safer. I'll use `at -f <script_path> now + X minutes`.
Also, `notify-send` might require `libnotify-bin`. I'll note that.
The `DBUS_SESSION_BUS_ADDRESS` is the biggest hurdle for cron+notify. I'll explicitly handle it by reading it from the parent shell if available, or instruct to set it in crontab. Actually, a robust way is to use `user-dbus` or just note that `DISPLAY` and `DBUS` must be exported in the cron job. I'll add a `crontab` example that exports them.

Let's write it out carefully.
Code:
```python
#!/usr/bin/env python3
import os
import sys
import json
import subprocess
import tempfile
from datetime import datetime, timedelta
from pathlib import Path

# --- Configuration ---
ICS_FILE = Path("/path/to/your/calendar.ics")
STATE_FILE = Path("/tmp/ical_notifier_state.json")
LOOKAHEAD_MINUTES = 15  # Check for events within this window
# ---------------------

def ensure_dependencies():
    if not subprocess.run(["which", "at"], capture_output=True).returncode == 0:
        print("Error: `at` is not installed. Install via: sudo apt install at / sudo dnf install at", file=sys.stderr)
        sys.exit(1)
    if not subprocess.run(["pgrep", "atd"], capture_output=True).returncode == 0:
        print("Warning: `atd` daemon is not running. Start it via: sudo systemctl enable --now atd", file=sys.stderr)

def parse_ics(filepath):
    try:
        import icalendar
    except ImportError:
        print("Error: Missing dependency. Install: pip install icalendar python-dateutil", file=sys.stderr)
        sys.exit(1)

    if not filepath.exists():
        print(f"Error: ICS file not found at {filepath}", file=sys.stderr)
        sys.exit(1)

    events = []
    now = datetime.now().astimezone()

    with open(filepath, "rb") as f:
        cal = icalendar.Calendar.from_ical(f.read())

    for component in cal.walk():
        if component.name != "VEVENT":
            continue

        start_dt = component.get("DTSTART").dt
        if hasattr(start_dt, 'tzinfo') and start_dt.tzinfo is None:
            start_dt = start_dt.replace(tzinfo=now.tzinfo)
        else:
            start_dt = start_dt.astimezone()

        if start_dt <= now:
            continue

        events.append({
            "uid": str(component.get("UID", f"{component.get('SUMMARY')}_{start_dt.isoformat()}")),
            "summary": str(component.get("SUMMARY", "Untitled")),
            "description": str(component.get("DESCRIPTION", "")),
            "start": start_dt
        })
    return events

def load_state():
    if STATE_FILE.exists():
        return json.loads(STATE_FILE.read_text())
    return {"notified": []}

def save_state(state):
    STATE_FILE.write_text(json.dumps(state))

def schedule_alert(event, minutes_delay):
    display = os.environ.get("DISPLAY", ":0")
    dbus_addr = os.environ.get("DBUS_SESSION_BUS_ADDRESS", "")

    # Escape quotes for shell safety
    safe_summary = event["summary"].replace('"', '\\"')
    safe_desc = event["description"].replace('"', '\\"')

    # Create a temporary script for `at` to execute
    with tempfile.NamedTemporaryFile(mode="w", suffix=".sh", delete=False, dir="/tmp") as tmp:
        tmp.write(f"""#!/bin/sh
export DISPLAY="{display}"
{f'export DBUS_SESSION_BUS_ADDRESS="{dbus_addr}"' if dbus_addr else ""}
exec /usr/bin/notify-send "{safe_summary}" "{safe_desc}"
""")
        script_path = Path(tmp.name)

    try:
        subprocess.run(
            ["at", "-f", script_path, "now + 0 minutes"], # at -f reads script
            # Actually, at -f is better combined with time. Let's use: at -f <script> now + X min
            # Correction: `at` syntax for script is: echo "script" | at now + X min
            # Or: at -f <script> now + X min (if supported)
            # Standard POSIX `at` doesn't always support -f. I'll stick to piping to be safe.
        )
        # Actually, let's just pipe it directly to `at` for maximum compatibility
        # ... wait, piping a complex script with quotes is messy.
        # I'll write it to /tmp/notify.sh and run: at now + X min < /tmp/notify.sh
    except Exception as e:
        print(f"at scheduling failed: {e}", file=sys.stderr)
        return False
    finally:
        os.unlink(script_path)
    return True
```
Wait, `at` syntax: `at -f script_file now + X minutes` works on GNU `at`. To be strictly safe and portable, I'll use `echo "script_content" | at now + X minutes`. But escaping is hard. I'll use a fixed temp path `/tmp/ical_notify.sh` and run `at now + X min < /tmp/ical_notify.sh`. Actually, `at` allows `-f` in GNU at. I'll assume GNU at (Linux). I'll use `subprocess.run(["at", "now + 0 minutes"], input=script_content.encode())`. It's fine.

Let's simplify the `at` call:
```python
        try:
            subprocess.run(
                ["at", f"now + {minutes_delay} minutes"],
                input=script_content.encode(),
                stdout=subprocess.DEVNULL,
                stderr=subprocess.PIPE
            )
```
This is standard and works.

I will finalize the code with this approach, ensuring robustness and clear comments. I'll also add the necessary setup instructions concisely.
All constraints checked. Output matches.
Proceeds.
[Output Generation] -> *Output matches the final refined version.*
*(Self-Correction/Note during output gen prep)*: I'll make sure the script handles the `DBUS_SESSION_BUS_ADDRESS` gracefully, as cron often drops it. I'll add a comment to export it in crontab, or use `busctl`/`user-bus` fallback, but keeping it simple is better. I'll stick to environment export in crontab as best practice.

Final code structure looks solid. I'll output it now.
[Output Generation] -> *Proceeds*
[End thinking]

```python
#!/usr/bin/env python3
"""
ical_notifier.py
Reads an .ics file, checks for upcoming events, and schedules desktop notifications via `at` and `notify-send`.
Designed to run periodically via cron (e.g., every 1-5 minutes).
"""

import os
import sys
import json
import subprocess
import tempfile
from datetime import datetime, timedelta
from pathlib import Path

# --- Configuration ---
ICS_FILE = Path("/path/to/your/calendar.ics")
STATE_FILE = Path("/tmp/ical_notifier_state.json")
LOOKAHEAD_MINUTES = 15  # Only schedule events within this window
# ---------------------

def ensure_environment():
    if not subprocess.run(["which", "at"], capture_output=True).returncode == 0:
        print("Error: `at` is not installed.", file=sys.stderr)
        sys.exit(1)
    if not subprocess.run(["pgrep", "atd"], capture_output=True).returncode == 0:
        print("Warning: `atd` daemon is not running. Start with: sudo systemctl enable --now atd", file=sys.stderr)

def parse_ics(filepath):
    try:
        import icalendar
    except ImportError:
        print("Error: Missing `icalendar`/`python-dateutil`. Install: pip install icalendar python-dateutil", file=sys.stderr)
        sys.exit(1)

    if not filepath.exists():
        print(f"Error: ICS file not found at {filepath}", file=sys.stderr)
        sys.exit(1)

    events = []
    now = datetime.now().astimezone()

    with open(filepath, "rb") as f:
        cal = icalendar.Calendar.from_ical(f.read())

    for component in cal.walk():
        if component.name != "VEVENT":
            continue

        start_dt = component.get("DTSTART").dt
        # Normalize timezones to local
        if hasattr(start_dt, 'tzinfo') and start_dt.tzinfo is None:
            start_dt = start_dt.replace(tzinfo=now.tzinfo)
        else:
            start_dt = start_dt.astimezone()

        if start_dt <= now:
            continue

        events.append({
            "uid": str(component.get("UID", f"{component.get('SUMMARY')}_{start_dt.isoformat()}")),
            "summary": str(component.get("SUMMARY", "Untitled Event")),
            "description": str(component.get("DESCRIPTION", "")),
            "start": start_dt
        })
    return events

def load_state():
    if STATE_FILE.exists():
        return json.loads(STATE_FILE.read_text())
    return {"notified": []}

def save_state(state):
    STATE_FILE.write_text(json.dumps(state))

def schedule_at_notification(summary, description, minutes_delay):
    display = os.environ.get("DISPLAY", ":0")
    dbus_addr = os.environ.get("DBUS_SESSION_BUS_ADDRESS", "")

    # Shell-escape quotes
    safe_summary = summary.replace('"', '\\"')
    safe_desc = description.replace('"', '\\"')

    # `at` runs in a minimal environment; explicitly export D-Bus/Display vars
    script = f"""#!/bin/sh
export DISPLAY="{display}"
{f'export DBUS_SESSION_BUS_ADDRESS="{dbus_addr}"' if dbus_addr else ""}
exec /usr/bin/notify-send "{safe_summary}" "{safe_desc}"
"""
    try:
        # Pipe script content directly to `at`
        proc = subprocess.run(
            ["at", f"now + {minutes_delay} minutes"],
            input=script.encode(),
            stdout=subprocess.DEVNULL,
            stderr=subprocess.PIPE
        )
        if proc.returncode != 0:
            print(f"at error: {proc.stderr.decode().strip()}", file=sys.stderr)
            return False
        return True
    except Exception as e:
        print(f"Failed to schedule: {e}", file=sys.stderr)
        return False

def main():
    ensure_environment()

    now = datetime.now().astimezone()
    window_end = now + timedelta(minutes=LOOKAHEAD_MINUTES)

    events = parse_ics(ICS_FILE)
    state = load_state()
    scheduled = 0

    for event in events:
        if event["start"] > window_end:
            continue
        if event["uid"] in state["notified"]:
            continue

        delta = (event["start"] - now).total_seconds() / 60
        delay = max(0.1, delta)  # Ensure >0 minutes for `at`

        if schedule_at_notification(event["summary"], event["description"], delay):
            state["notified"].append(event["uid"])
            scheduled += 1
            print(f"Scheduled: {event['summary']} @ {event['start'].strftime('%H:%M')}")
        else:
            print(f"Failed: {event['summary']}", file=sys.stderr)

    # Prune old entries to prevent state file bloat
    state["notified"] = [uid for uid in state["notified"] if any(e["uid"] == uid and e["start"] > window_end for e in events)]
    save_state(state)

    if scheduled == 0:
        print("No new events to schedule in the current window.")

if __name__ == "__main__":
    main()
```

### Setup & Caveats

1. **Dependencies**: `sudo apt install at libnotify-bin icalendar python-dateutil` (Debian/Ubuntu) or equivalent. Ensure `pip install icalendar python-dateutil` is run if Python packages are missing.
2. **D-Bus Session**: `notify-send` requires a live user session bus. Cron jobs strip environment variables by default. Add these lines to your crontab (`crontab -e`) before the script line:
   ```bash
   export DISPLAY=:0
   export DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$(id -u)/bus"
   * * * * * /usr/bin/python3 /path/to/ical_notifier.py >> /tmp/ical_notifier.log 2>&1
   ```
3. **Daemon**: Run `sudo systemctl enable --now atd` to ensure the `at` queue processes jobs.
4. **State Management**: The script uses `/tmp/ical_notifier_state.json` to prevent duplicate notifications. It auto-prunes entries older than the `LOOKAHEAD_MINUTES` window.
5. **Timezones**: The parser normalizes all events to your system's local timezone. Ensure your system clock/timezone is accurate.

[ Prompt: 4.7 t/s | Generation: 4.1 t/s ]

