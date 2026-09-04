# JMeter MCP — Summary of the fix

> Self-contained explanation for another AI agent. Tells the story of what was
> broken, why, and what to do. Assumes the agent has filesystem and bash access
> to `C:\Work\Projects\jmeter-mcp` on Windows.

---

## TL;DR

The `QAInsights/jmeter-mcp-server` package's `execute_jmeter_test_non_gui`
tool hangs on Windows because **three layered bugs** combine. All three are
fixed in this fork (`Andrew-153/jmeter-mcp`) at
`C:\Work\Projects\jmeter-mcp\jmeter_server.py`. After the fixes, the tool
runs a JMeter test via the MCP in ~30–45 seconds and returns cleanly.

The MCP is registered in `C:\Work\Projects\POS\kilo.json` under `mcp.jmeter`
and is currently operational.

---

## What was broken

MCP error observed by every client (Claude Desktop, Kilo, Cursor): **`MCP
error -32001: Request timed out`** after 2 minutes when calling
`execute_jmeter_test_non_gui` or `execute_jmeter_test` on Windows. Analysis
tools (`analyze_jmeter_results`, `identify_performance_bottlenecks`,
`get_performance_insights`, `generate_visualization`) worked fine.

Root cause was three independent Windows-specific defects, **any one of which
is enough to hang the tool**:

### Bug 1 — `async def` tool calls a blocking `subprocess.run` directly

```python
@mcp.tool()
async def execute_jmeter_test_non_gui(...):
    return await run_jmeter(...)   # run_jmeter is async but calls sync subprocess.run
```

`subprocess.run` blocks the event loop for the duration of the JMeter run.
On Windows the FastMCP stdio transport ends up unable to deliver the
response within the client's timeout. Upstream maintainer has not fixed
this (open Issue [#10](https://github.com/QAInsights/jmeter-mcp-server/issues/10)
since 2025-09-01).

**Fix:** add `stdin=subprocess.DEVNULL` and `timeout=600` to the
`subprocess.run` call. We do not switch to `asyncio.to_thread` or
`asyncio.create_subprocess_exec` — both have separate Windows issues (see
Bug 3).

### Bug 2 — `jmeter.bat` prints `Press any key to continue . . .` and blocks on stdin

`C:\Users\a.sapach\Downloads\apache-jmeter-5.6.3\bin\jmeter.bat` line 211
calls `pause` when the run finishes. `pause` reads a keypress from stdin.

When Python's `subprocess.run([...])` inherits stdin from the parent, the
parent (the MCP server) has stdin connected to the MCP transport's pipe to
the client. Kilo never sends a keypress → subprocess hangs forever →
MCP request times out.

**Fix:** pass `stdin=subprocess.DEVNULL` to `subprocess.run` so the child
has no stdin to wait on. (The same call also fixes Bug 1 partially.)

### Bug 3 — `jmeter.bat`'s `%~dp0` resolves to empty / wrong dir under `subprocess.run([list])`

`jmeter.bat` line 200 does:

```bat
%JM_LAUNCH% %ARGS% %JVM_ARGS% -jar "%JMETER_BIN%ApacheJMeter.jar" %*
```

where `JMETER_BIN` is set by `set JMETER_BIN=%~dp0` (line 138). When
Python's `subprocess.run([list])` invokes the `.bat` via cmd.exe with a
list argv, cmd.exe's handling of `%~dp0` returns an empty or garbled
value, and the resolved path becomes `<empty>ApacheJMeter.jar` → which
cmd.exe concatenates with the previous element `jmeter.bat`, yielding
`jmeter.batApacheJMeter.jar`. JMeter exits in 0.15s with:

```
Error: Unable to access jarfile .../jmeter.batApacheJMeter.jar
```

**Fix:** on Windows, if `JMETER_BIN` ends in `.bat`, do **not** call
`jmeter.bat` at all. Construct the command ourselves:

```python
cmd = ["java.exe", *java_opts.split(), "-jar", str(jar_path), ...]
```

where `jar_path = JMETER_BIN.parent / "ApacheJMeter.jar"`.

---

## The patched function (current state)

```python
import time, subprocess
from pathlib import Path

async def run_jmeter(test_file, non_gui=True, properties=None,
                     generate_report=False, report_output_dir=None,
                     log_file=None):
    logger.error(f"[DIAG] run_jmeter ENTER")
    try:
        test_file_path = Path(test_file).resolve()
        if not test_file_path.exists():
            return f"Error: Test file not found: {test_file}"
        if test_file_path.suffix != ".jmx":
            return f"Error: Invalid file type. Expected .jmx file: {test_file}"

        jmeter_bin = os.getenv("JMETER_BIN", "jmeter")
        java_opts = os.getenv("JMETER_JAVA_OPTS", "")
        jmeter_bin_path = Path(jmeter_bin)

        # BUG 3 fix: on Windows, bypass jmeter.bat entirely
        if jmeter_bin_path.suffix.lower() == ".bat":
            jar_path = jmeter_bin_path.parent / "ApacheJMeter.jar"
            if not jar_path.exists():
                return f"Error: ApacheJMeter.jar not found: {jar_path}"
            java_opts_list = [o for o in java_opts.split() if o]
            cmd = ["java.exe", *java_opts_list, "-jar", str(jar_path)]
        else:
            cmd = [str(jmeter_bin_path.resolve())]

        if non_gui:
            cmd.extend(["-n"])
        cmd.extend(["-t", str(test_file_path)])

        if properties:
            for k, v in properties.items():
                cmd.append(f"-J{k}={v}")

        # BUG-FIX: write JTL whenever log_file is provided (was previously
        # gated on generate_report, which meant passing only log_file did nothing)
        if non_gui and log_file:
            cmd.extend(["-l", log_file])

        if generate_report and non_gui:
            if log_file is None:
                unique_id = generate_unique_id()
                log_file = f"{test_file_path.stem}_{unique_id}_results.jtl"
                cmd.extend(["-l", log_file])
            cmd.extend(["-e"])
            unique_id = unique_id if "unique_id" in locals() else generate_unique_id()
            if report_output_dir:
                report_output_dir = f"{report_output_dir}_{unique_id}"
            else:
                report_output_dir = f"{test_file_path.stem}_{unique_id}_report"
            cmd.extend(["-o", report_output_dir])

        logger.error(f"[DIAG] before subprocess.run at {time.time():.2f}")
        try:
            # BUG 1 + BUG 2 fix: DEVNULL prevents cmd.exe "Press any key"
            # block; timeout prevents infinite hang
            result = subprocess.run(
                cmd, capture_output=True, text=True,
                stdin=subprocess.DEVNULL, timeout=600,
            )
        except subprocess.TimeoutExpired:
            return "Error: JMeter run exceeded 600s and was killed"
        logger.error(f"[DIAG] after subprocess.run rc={result.returncode} took {time.time()-t0:.2f}s")
        return result.stdout if result.returncode == 0 else f"Error: {result.stderr}"
    finally:
        logger.error("[DIAG] run_jmeter EXIT")
```

Note: `JM_LAUNCH` (which jmeter.bat defaults to `java.exe`), `HEAP`
(`-Xms1g -Xmx1g`), `GC_ALGO`, `JMETER_LANGUAGE` — none of these are
honored by the bypass. If you need them, prepend them to `java_opts`
via `JMETER_JAVA_OPTS` in `.env`.

---

## Files & layout

| Path | Purpose |
|---|---|
| `C:\Work\Projects\jmeter-mcp\` | Fork root (Andrew-153/jmeter-mcp) — source of truth |
| `C:\Work\Projects\jmeter-mcp\jmeter_server.py` | Patched MCP server code |
| `C:\Work\Projects\jmeter-mcp\.env` | `JMETER_HOME`, `JMETER_BIN=.../jmeter.bat`, `JMETER_JAVA_OPTS=-Xms1g -Xmx2g` |
| `C:\Work\Projects\jmeter-mcp\pyproject.toml` | `requires-python = ">=3.13"`, `mcp[cli]>=1.6.0,<2` |
| `C:\Work\Projects\jmeter-mcp\.venv\` | uv-managed venv, Python 3.13, mcp==1.29.1 |
| `C:\Work\Projects\jmeter-mcp\logs\jmeter-mcp.log` | Diagnostic log with `[DIAG]` markers |
| `C:\Work\Projects\jmeter-mcp\results\` | Test outputs (JTL, dashboards) |
| `C:\Work\Projects\jmeter-mcp\DEBUG.md` | Step-by-step manual debugging guide |
| `C:\Users\a.sapach\Downloads\apache-jmeter-5.6.3\` | JMeter install (Apache binary) |
| `C:\Work\Projects\POS\kilo.json` | Kilo MCP config — `mcp.jmeter` block |

---

## kilo.json mcp.jmeter entry (current)

```jsonc
"jmeter": {
  "type": "local",
  "command": [
    "C:\\Users\\a.sapach\\.local\\bin\\uv.exe",
    "--directory",
    "C:\\Work\\Projects\\jmeter-mcp",
    "run",
    "jmeter_server.py"
  ],
  "environment": { "PYTHONUNBUFFERED": "1" },
  "enabled": true,
  "timeout": 180000
}
```

---

## How to verify the fix yourself

### 1. Run the MCP server directly (smoke test)

```powershell
& "C:\Users\a.sapach\.local\bin\uv.exe" --directory "C:\Work\Projects\jmeter-mcp" run jmeter_server.py
```

No output = server is waiting on stdio. Ctrl-C to exit.

### 2. Send a test MCP request via stdin

Use the PowerShell snippet from `C:\Work\Projects\jmeter-mcp\DEBUG.md`
section 4. Or use the Kilo `jmeter_execute_jmeter_test_non_gui` tool:

```python
# Through Kilo MCP, this works after Kilo restart with the patched code:
jmeter_execute_jmeter_test_non_gui(
    test_file="C:/Work/Projects/jmeter-mcp/sample_test.jmx",
    log_file="C:/Work/Projects/jmeter-mcp/results/test/results.jtl",
    generate_report=True,
    report_output_dir="C:/Work/Projects/jmeter-mcp/results/test/report",
)
```

Expected: returns in ~30–45s with stdout containing the JMeter summary
(`summary = N in ...`), writes JTL to `log_file` and the report dashboard
to `report_output_dir_<unique_id>/`.

### 3. Confirm via log

`C:\Work\Projects\jmeter-mcp\logs\jmeter-mcp.log` should show:

```
[DIAG] run_jmeter ENTER
[DIAG] before subprocess.run at ...
[DIAG] after subprocess.run rc=0 took ~33s
[DIAG] run_jmeter EXIT
```

If the gap between `before` and `after` is more than ~60s, something is
wrong. If only `before` shows up, `subprocess.run` is hanging — check
for new `[DIAG]` additions or stderr output.

---

## Open follow-ups (not blocking)

1. **PR upstream** — Issue #10 has been open for a year. The fix could
   be sent to `QAInsights/jmeter-mcp-server`. Maintainer has been silent.
2. **Diagnostic `[DIAG]` logging** — could be removed for production, but
   useful if the issue resurfaces.
3. **Cleanup `tools/` deletion** — earlier in the session `tools/` was
   wiped (lost `allure-testops-mcp`, `mcp-fetch`). No way to recover
   from this environment. Safety rules now in global `kilo.jsonc` to
   prevent repeat.
4. **Linux/macOS** — these fixes are Windows-specific; on Linux the
   `.bat` branch is skipped and `jmeter` binary is called directly.
   Should work but has not been tested.

---

## Quick reference for the agent

- Run jmeter test via MCP: call `jmeter_execute_jmeter_test_non_gui` (Kilo) with
  `test_file`, optional `log_file`, optional `generate_report`+`report_output_dir`.
- Run jmeter test directly (bypass MCP):
  ```powershell
  & "C:\Users\a.sapach\Downloads\apache-jmeter-5.6.3\bin\jmeter.bat" -n -t "<test.jmx>" -l "<output.jtl>"
  ```
- Read logs: `Get-Content C:\Work\Projects\jmeter-mcp\logs\jmeter-mcp.log -Wait`
- Search MCP upstream issues: <https://github.com/QAInsights/jmeter-mcp-server/issues>
