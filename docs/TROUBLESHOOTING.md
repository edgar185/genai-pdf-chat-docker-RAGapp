# Troubleshooting Log

A record of the issues hit while deploying this stack end to end, and how each
was diagnosed and resolved. None of these were in the original lab script — they
came up in a real cloud environment and had to be worked through from the logs.

---

## 1. Neo4j: `ProcedureNotFound` on vector index creation

**Symptom**

```
neo4j.exceptions.ClientError: {code: Neo.ClientError.Procedure.ProcedureNotFound}
There is no procedure with the name `db.index.vector.createNodeIndex`
registered for this database instance.
```

The PDF uploaded fine, but creating the vector index failed.

**Diagnosis**

The app's LangChain version creates its vector index by calling the procedure
`db.index.vector.createNodeIndex`. That was a beta procedure in Neo4j 5.11 and
has since been removed in favor of a native `CREATE VECTOR INDEX` command. The
app was pointed at a managed cloud Neo4j instance running a current version,
which no longer exposes that procedure.

**Fix**

Repoint the app from the managed instance to the **local Neo4j 5.11 container**
that Docker Compose already runs — the exact version that supports the procedure.

```ini
# .env  — change only the URI
NEO4J_URI=neo4j://database:7687
```

`database` is the Compose service hostname; `neo4j://` (not `neo4j+s://`) because
the local container is unencrypted. Username and password already matched the
local container, which initializes its password from the same env variable.

---

## 2. Ollama: segmentation fault during model warm-up

**Symptom**

```
ValueError: Ollama call failed with status code 500.
Details: {"error":"llama-server process has terminated:
signal: segmentation fault (core dumped)"}
```

Reproducible outside the app with a direct `ollama run tinyllama "hello"`.

**Diagnosis**

Ruled out the easy causes first:

- **Memory** — `free -h` showed 38 GiB available. Not the issue.
- **Corrupt model** — `ollama rm` + `ollama pull` re-downloaded it; same crash.

Container logs showed the crash happening at `warming up the model with an
empty run`, inside an **AMX** (Advanced Matrix Extensions) CPU code path. The
image was `ollama/ollama:latest` — a bleeding-edge build whose newer llama.cpp
backend segfaults on this VM's CPU. Updating could not help, since `latest` was
already the broken build.

**Fix**

Pin the Ollama image to a known-stable older version instead of `latest`.

```yaml
# docker-compose.yaml  — under the ollama service
image: ollama/ollama:0.9.0
```

```bash
docker compose pull ollama
docker compose up -d ollama
docker exec -it $(docker ps -qf "name=ollama") ollama run tinyllama "hello"
```

The model returned text instead of a segfault. The pin carries forward to the
llama2 and neural-chat runs as well.

---

## 3. Operational gotchas

- **Stray character in source.** A `SyntaxError: invalid syntax` on `app.py`
  line 1 turned out to be a stray `f` (`fimport os`) introduced during an editor
  mix-up. The app bakes source into the image at build time, so the fix needed a
  rebuild, not just a restart: `docker compose up -d --build server`.
- **Wrong editor.** `nano .env` opened in vim once; `Ctrl+X` typed literal text
  instead of exiting. Escape hatch: `Esc`, then `:q!`, then Enter.
- **SSH timeouts mid-download.** Long pulls dropped with "broken pipe." Solved by
  running them inside `tmux` so the process survives a disconnect — reconnect and
  `tmux attach` to resume.
- **Running commands from the wrong directory.** `docker compose down` from `~`
  returns "no configuration file provided"; it must run from the project folder.

---

## Quick diagnostic reference

```bash
docker compose ps                                   # what's running / healthy
docker logs $(docker ps -qf "name=ollama") --tail 40   # ollama crash detail
free -h                                             # available memory
grep LLM .env                                       # confirm active model
docker exec -it $(docker ps -qf "name=ollama") ollama list   # pulled models
```
