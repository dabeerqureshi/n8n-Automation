# Ollama Setup (Local AI)

The booking assistant runs on a **local** Ollama model — no external AI API needed.

## 1. Install Ollama

- macOS/Windows: download from https://ollama.com/download
- Linux:
  ```bash
  curl -fsSL https://ollama.com/install.sh | sh
  ```

## 2. Pull a model

The workflow defaults to `llama3.1`. Any instruction-tuned model works; for
better structured-JSON reliability, `llama3.1:8b` or `qwen2.5:7b-instruct`
are good choices.

```bash
ollama pull llama3.1
```

Test it:
```bash
ollama run llama3.1 "Reply with only the word OK"
```

## 3. Make sure Ollama is reachable

By default Ollama listens on `http://localhost:11434`. Confirm:
```bash
curl http://localhost:11434/api/tags
```
You should get a JSON list of installed models back.

If n8n is running **inside Docker** and Ollama is on your host machine, use
`http://host.docker.internal:11434` (Mac/Windows) instead of `localhost`, or
run Ollama in the same Docker network on Linux and reference the container
name/IP.

## 4. Create the Ollama credential in n8n

1. In n8n, go to **Credentials → New → Ollama API**.
2. Base URL: `http://localhost:11434` (or `http://host.docker.internal:11434` if applicable).
3. Save it as **"Ollama (local)"** — this name matches the placeholder credential referenced in the workflow JSON.
4. Open the **Ollama Chat Model** node inside Workflow 1, select this credential, and confirm the model name (`llama3.1` or whichever you pulled) in the **Model** field.

## Notes on reliability

Local models are less reliable than hosted frontier models at strictly
following JSON schemas. If you see the agent occasionally returning
malformed output:
- Lower `temperature` further (already set to 0.3 in the workflow).
- Use a model that's better at instruction-following/JSON (e.g. `qwen2.5:7b-instruct`, `llama3.1:70b` if you have the hardware).
- n8n's Structured Output Parser will auto-retry a malformed response once by default — you can increase retries in the parser node's options if needed.
