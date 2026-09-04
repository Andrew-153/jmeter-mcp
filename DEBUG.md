# JMeter MCP — ручная диагностика

> Гайд для самостоятельной проверки и для передачи другой ИИ. Покрывает:
> где что лежит, как поднять MCP руками, как отправить тестовый запрос через stdio,
> как читать логи и куда смотреть при зависании.

---

## 1. Где что лежит

| Что | Путь |
|---|---|
| Код MCP-сервера (форк Andrew-153/jmeter-mcp) | `C:\Work\Projects\jmeter-mcp\jmeter_server.py` |
| Конфиг JMeter (env vars) | `C:\Work\Projects\jmeter-mcp\.env` |
| Зависимости Python | `C:\Work\Projects\jmeter-mcp\pyproject.toml` |
| venv (Python 3.13) | `C:\Work\Projects\jmeter-mcp\.venv\` |
| **Логи MCP-сервера** (для диагностики) | `C:\Work\Projects\jmeter-mcp\logs\jmeter-mcp.log` |
| Результаты тестов (JTL, дашборды) | `C:\Work\Projects\jmeter-mcp\results\` |
| Пример теста | `C:\Work\Projects\jmeter-mcp\sample_test.jmx` |
| Конфиг Kilo | `C:\Work\Projects\POS\kilo.json` (секция `mcp.jmeter`) |
| Глобальный конфиг Kilo | `C:\Users\a.sapach\.config\kilo\kilo.jsonc` |
| JMeter (Windows) | `C:\Users\a.sapach\Downloads\apache-jmeter-5.6.3\bin\jmeter.bat` |

---

## 2. Конфиг Kilo — что должно быть в `kilo.json`

```jsonc
{
  "mcp": {
    "jmeter": {
      "type": "local",
      "command": [
        "C:\\Users\\a.sapach\\.local\\bin\\uv.exe",
        "--directory",
        "C:\\Work\\Projects\\jmeter-mcp",
        "run",
        "jmeter_server.py"
      ],
      "environment": {
        "PYTHONUNBUFFERED": "1"
      },
      "enabled": true,
      "timeout": 180000
    }
  }
}
```

---

## 3. Запуск MCP-сервера руками (для отладки)

В PowerShell:

```powershell
cd C:\Work\Projects\jmeter-mcp
& "C:\Users\a.sapach\.local\bin\uv.exe" --directory "C:\Work\Projects\jmeter-mcp" run jmeter_server.py
```

**Что должно произойти:** никакого вывода (сервер слушает stdio). Если `ImportError` или `ModuleNotFoundError` — будут в stderr.

---

## 4. Отправка тестового MCP-запроса вручную (через stdin)

### Вариант А: PowerShell (без отдельного файла)

```powershell
# Стартуем сервер, сразу шлём initialize + tools/call
$psi = New-Object System.Diagnostics.ProcessStartInfo
$psi.FileName = "C:\Users\a.sapach\.local\bin\uv.exe"
$psi.Arguments = "--directory `"C:\Work\Projects\jmeter-mcp`" run jmeter_server.py"
$psi.UseShellExecute = $false
$psi.RedirectStandardInput = $true
$psi.RedirectStandardOutput = $true
$psi.RedirectStandardError = $true
$proc = [System.Diagnostics.Process]::Start($psi)

Start-Sleep -Milliseconds 1500  # даём серверу стартовать

# 1. initialize
$proc.StandardInput.WriteLine('{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"diag","version":"0.1"}}}')
$proc.StandardInput.Flush()
Start-Sleep -Milliseconds 500
"INIT: $($proc.StandardOutput.ReadLine())"

# 2. initialized notification
$proc.StandardInput.WriteLine('{"jsonrpc":"2.0","method":"notifications/initialized"}')
$proc.StandardInput.Flush()

# 3. tools/call — execute_jmeter_test_non_gui
$call = @'
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"execute_jmeter_test_non_gui","arguments":{"test_file":"C:/Work/Projects/jmeter-mcp/sample_test.jmx","log_file":"C:/Work/Projects/jmeter-mcp/results/diag/results.jtl"}}}
'@
$proc.StandardInput.WriteLine($call)
$proc.StandardInput.Flush()
"calling... ждём 60 сек"

# Читаем всё, что приходит в stdout 60 секунд
$sw = [System.Diagnostics.Stopwatch]::StartNew()
while ($sw.Elapsed.TotalSeconds -lt 60 -and -not $proc.HasExited) {
    if ($proc.StandardOutput.Peek() -ne -1) {
        $line = $proc.StandardOutput.ReadLine()
        "OUT (${sw.Elapsed.TotalSeconds}s): $line"
    } else {
        Start-Sleep -Milliseconds 500
    }
}
"elapsed: $([math]::Round($sw.Elapsed.TotalSeconds,1))s; exited=$($proc.HasExited)"
if (-not $proc.HasExited) { $proc.Kill() }
```

### Вариант Б: Python one-shot

```powershell
& "C:\Users\a.sapach\.local\bin\uv.exe" --directory "C:\Work\Projects\jmeter-mcp" run python -c @"
import subprocess, sys, json, time
proc = subprocess.Popen(
    [r'C:\Users\a.sapach\.local\bin\uv.exe', '--directory', r'C:\Work\Projects\jmeter-mcp', 'run', 'jmeter_server.py'],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True, bufsize=1
)
def send(msg):
    proc.stdin.write(json.dumps(msg) + '\n'); proc.stdin.flush()
send({'jsonrpc':'2.0','id':1,'method':'initialize','params':{'protocolVersion':'2024-11-05','capabilities':{},'clientInfo':{'name':'diag','version':'0.1'}}})
time.sleep(0.5)
print('INIT:', proc.stdout.readline().strip())
send({'jsonrpc':'2.0','method':'notifications/initialized'})
send({'jsonrpc':'2.0','id':2,'method':'tools/call','params':{'name':'execute_jmeter_test_non_gui','arguments':{'test_file':'C:/Work/Projects/jmeter-mcp/sample_test.jmx','log_file':'C:/Work/Projects/jmeter-mcp/results/diag/results.jtl'}}})
print('CALL sent, waiting 60s for response...')
t0 = time.time()
import select
while time.time() - t0 < 60:
    rl, _, _ = select.select([proc.stdout], [], [], 0.5)
    if rl:
        line = proc.stdout.readline()
        print(f'OUT ({time.time()-t0:.1f}s):', line.rstrip())
proc.kill()
print(f'elapsed {time.time()-t0:.1f}s; stderr tail:', proc.stderr.read()[-500:])
"@
```

---

## 5. Чтение логов

```powershell
# Текущий лог
Get-Content C:\Work\Projects\jmeter-mcp\logs\jmeter-mcp.log

# Следить в реальном времени (как tail -f)
Get-Content C:\Work\Projects\jmeter-mcp\logs\jmeter-mcp.log -Wait

# Только ошибки/диагностика
Select-String -Path C:\Work\Projects\jmeter-mcp\logs\jmeter-mcp.log -Pattern "\[DIAG\]|\[ERROR\]"
```

В логе маркеры `[DIAG]` — это диагностические точки, добавленные при зависании. Их ожидаемая последовательность при успехе:
```
[DIAG] execute_jmeter_test_non_gui ENTER
[DIAG] run_jmeter ENTER: test_file=... non_gui=True
[DIAG] before subprocess.run at 1693823123.45
[DIAG] after subprocess.run at 1693823156.78 (took 33.33s) rc=0
[DIAG] run_jmeter EXIT
[DIAG] execute_jmeter_test_non_gui RETURN (len=1234)
```

Если процесс виснет, в логе будет видно, **где остановился** последний `[DIAG]`.

---

## 6. Прямой запуск JMeter (обход MCP для сравнения)

```powershell
$env:JMETER_HOME = "C:\Users\a.sapach\Downloads\apache-jmeter-5.6.3"
New-Item -ItemType Directory -Path C:\Work\Projects\jmeter-mcp\results\diag -Force | Out-Null
& "$env:JMETER_HOME\bin\jmeter.bat" -n -t "C:\Work\Projects\jmeter-mcp\sample_test.jmx" -l "C:\Work\Projects\jmeter-mcp\results\diag\results.jtl"
```

Это занимает ~33 секунды для `sample_test.jmx`. Если MCP зависает, а прямой запуск работает — проблема 100% в `jmeter_server.py`.

---

## 7. Известная проблема: timeout на Windows

Upstream-баг [#10](https://github.com/QAInsights/jmeter-mcp-server/issues/10) — `execute_jmeter_test_non_gui` отдаёт `MCP error -32001: Request timed out` на Windows. В этом форке (`Andrew-153/jmeter-mcp`) пытаемся починить.

**Что работает:** `analyze_jmeter_results` и другие анализ-тулы.
**Что зависает:** `execute_jmeter_test` и `execute_jmeter_test_non_gui`.

### Куда копать в коде

`jmeter_server.py:25-124` — `async def run_jmeter`. Проблемные места:
- строка 101: `subprocess.run(cmd, capture_output=True, text=True)` — блокирующий вызов внутри `async def`
- `await run_jmeter(...)` в tool-функциях (строки 130, 143)

### Возможные направления фикса
1. **`asyncio.to_thread(subprocess.run, ...)`** — пробовали, на Windows не помогло
2. **`asyncio.create_subprocess_exec(*cmd, stdout=PIPE, stderr=PIPE)`** + `await proc.communicate()` — ломает argv на `.bat` файлах
3. **Сделать tool-функции sync (`def`, не `async def`)** — FastMCP сам уведёт в thread pool
4. **`shell=True`** с правильным экранированием через `subprocess.list2cmdline`
5. **Прямой вызов Java** (`java -jar ApacheJMeter.jar ...`) вместо `.bat` wrapper

---

## 8. Чеклист для передачи другой ИИ

1. Прочитай `kilo.json` — какой путь в `mcp.jmeter.command`.
2. Прочитай `C:\Work\Projects\jmeter-mcp\jmeter_server.py` — какая сейчас реализация `run_jmeter`.
3. Запусти MCP-сервер руками (раздел 3) — стартует ли?
4. Отправь тестовый MCP-запрос (раздел 4) — возвращается ли ответ за <60 сек?
5. Прочитай `logs\jmeter-mcp.log` (раздел 5) — где остановились `[DIAG]` маркеры?
6. Если висит — попробуй один из вариантов фикса (раздел 7).
