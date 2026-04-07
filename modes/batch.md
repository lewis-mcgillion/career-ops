# Modo: batch — Procesamiento Masivo de Ofertas

Dos modos de uso: **conductor with browser** (navega portales en tiempo real) o **standalone** (script para URLs ya recolectadas).

## Arquitectura

### Copilot CLI — Subagent Model

```
Copilot CLI Conductor (main session with browser)
  │
  │  Chrome DevTools: navega portales (sesiones logueadas)
  │  Lee DOM directo — el usuario ve todo en tiempo real
  │
  ├─ Oferta 1: lee JD del DOM + URL
  │    └─► task(agent_type="general-purpose", mode="background") → report .md + PDF + tracker-line
  │
  ├─ Oferta 2: click siguiente, lee JD + URL
  │    └─► task(agent_type="general-purpose", mode="background") → report .md + PDF + tracker-line
  │
  └─ Fin: merge tracker-additions → applications.md + resumen
```

Each subagent runs in a separate context. The conductor only orchestrates.

### Claude Code — Pipe Worker Model

```
Claude Conductor (claude --chrome --dangerously-skip-permissions)
  │
  │  Chrome: navega portales (sesiones logueadas)
  │
  ├─ Oferta 1: lee JD del DOM + URL
  │    └─► claude -p worker → report .md + PDF + tracker-line
  │
  ├─ Oferta 2: click siguiente, lee JD + URL
  │    └─► claude -p worker → report .md + PDF + tracker-line
  │
  └─ Fin: merge tracker-additions → applications.md + resumen
```

## Archivos

```
batch/
  batch-input.tsv               # URLs (por conductor o manual)
  batch-state.tsv               # Progreso (auto-generado, gitignored)
  batch-runner.sh               # Script orquestador standalone (Claude Code only)
  batch-prompt.md               # Prompt template para workers
  logs/                         # Un log por oferta (gitignored)
  tracker-additions/            # Líneas de tracker (gitignored)
```

## Modo A: Conductor with browser

1. **Leer estado**: `batch/batch-state.tsv` → saber qué ya se procesó
2. **Navegar portal**: Browser → URL de búsqueda
3. **Extraer URLs**: Leer DOM de resultados → extraer lista de URLs → append a `batch-input.tsv`
4. **Para cada URL pendiente**:
   a. Browser: click en la oferta → leer JD text del DOM
   b. Guardar JD a `/tmp/batch-jd-{id}.txt`
   c. Calcular siguiente REPORT_NUM secuencial
   d. **Launch subagent:**

   **Copilot CLI:**
   ```
   task(
     agent_type="general-purpose",
     mode="background",
     name="batch-worker-{id}",
     description="Evaluate offer {id}",
     prompt="[content of batch/batch-prompt.md]\n\nProcesa esta oferta. URL: {url}. JD: [JD text]. Report: {num}. ID: {id}"
   )
   ```

   **Claude Code:**
   ```bash
   claude -p --dangerously-skip-permissions \
     --append-system-prompt-file batch/batch-prompt.md \
     "Procesa esta oferta. URL: {url}. JD: /tmp/batch-jd-{id}.txt. Report: {num}. ID: {id}"
   ```

   e. Actualizar `batch-state.tsv` (completed/failed + score + report_num)
   f. Log a `logs/{report_num}-{id}.log`
   g. Browser: volver atrás → siguiente oferta
5. **Paginación**: Si no hay más ofertas → click "Next" → repetir
6. **Fin**: Merge `tracker-additions/` → `applications.md` + resumen

## Modo B: Script standalone (Claude Code only)

```bash
batch/batch-runner.sh [OPTIONS]
```

Opciones:
- `--dry-run` — lista pendientes sin ejecutar
- `--retry-failed` — solo reintenta fallidas
- `--start-from N` — empieza desde ID N
- `--parallel N` — N workers en paralelo
- `--max-retries N` — intentos por oferta (default: 2)

**Note:** `batch-runner.sh` uses `claude -p` which is Claude Code-specific. In Copilot CLI, use Modo A (conductor with browser) or launch `task()` subagents manually for each pending URL.

## Formato batch-state.tsv

```
id	url	status	started_at	completed_at	report_num	score	error	retries
1	https://...	completed	2026-...	2026-...	002	4.2	-	0
2	https://...	failed	2026-...	2026-...	-	-	Error msg	1
3	https://...	pending	-	-	-	-	-	0
```

## Resumabilidad

- Si muere → re-ejecutar → lee `batch-state.tsv` → skip completadas
- Lock file (`batch-runner.pid`) previene ejecución doble
- Cada worker es independiente: fallo en oferta #47 no afecta a las demás

## Workers (subagents)

Each worker receives `batch-prompt.md` as system prompt. It is self-contained.

**Copilot CLI:** Each worker is a `task(agent_type="general-purpose")` subagent.
**Claude Code:** Each worker is a `claude -p` child process with clean 200K context.

El worker produce:
1. Report `.md` en `reports/`
2. PDF en `output/`
3. Línea de tracker en `batch/tracker-additions/{id}.tsv`
4. JSON de resultado por stdout

## Gestión de errores

| Error | Recovery |
|-------|----------|
| URL inaccesible | Worker falla → conductor marca `failed`, siguiente |
| JD detrás de login | Conductor intenta leer DOM. Si falla → `failed` |
| Portal cambia layout | Conductor razona sobre HTML, se adapta |
| Worker crashea | Conductor marca `failed`, siguiente. Retry con `--retry-failed` |
| Conductor muere | Re-ejecutar → lee state → skip completadas |
| PDF falla | Report .md se guarda. PDF queda pendiente |
