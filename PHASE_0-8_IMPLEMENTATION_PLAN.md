# Baby AI - Piano di Implementazione Fase 0-8

## App Agent POC - Piano di Esecuzione Dettagliato
**Versione:** 1.2  
**Data:** 16 Novembre 2025  
**Stato:** Pronto per l'Esecuzione

---

## Versioni

### Componenti Principali

- **Ollama Server** (binario macOS da includere con Tauri)
  - **Versione:** v0.12.11 ([Release Notes](https://github.com/ollama/ollama/releases))
  - **Fonte:** https://github.com/ollama/ollama/releases
  - **Scopo:** Server di inferenza LLM locale (incluso nella Fase 1.1)
- **ollama-python SDK** (libreria client Python)
  - **Versione:** v0.6.0
  - **Fonte:** https://github.com/ollama/ollama-python/releases
  - **Scopo:** Client API Python per comunicare con il server Ollama
  - **Installazione:** `pip install ollama==0.6.0` (in requirements.txt)

> Nota: Questi due componenti hanno versioni indipendenti.

### Tabella Versioni Componenti

| Componente      | Versione    | Scopo                                      |
|-----------------|-------------|---------------------------------------------|
| Ollama Server   | v0.12.11    | Server di inferenza LLM (incluso)           |
| LLM Model       | baby-ai-qwen3-tool | Modello per Tool Calling (da GGUF)    |
| ollama-python   | v0.6.0      | Client Python per Ollama                    |
| Python          | 3.14.0      | Runtime (incluso con PyInstaller)           |
| Pydantic        | 2.12.4      | Validazione Schema JSON (Tool Calls)        |
| FastAPI         | 0.115.0     | Framework API Backend                       |
| Appscript       | 1.4.0       | Automazione macOS (Apple Events)            |
| PyInstaller     | 6.11.1      | Inclusione backend Python                   |
| Tauri           | 2.9.x       | Framework app Desktop                       |
| Rust            | 1.91.1      | Linguaggio backend Tauri                    |
| Node.js         | 25.1.0      | Tooling di build Frontend                   |
| React           | 19.2.0      | Framework UI Frontend                       |
| TypeScript      | 5.9.3       | Type safety Frontend                        |
| Tailwind CSS    | 4.1.17      | Styling Frontend                            |

---

## Panoramica

Questo documento fornisce un piano di implementazione dettagliato, passo dopo passo, per la Fase 1.1 del rewrite Baby AI in Python. Il piano stabilisce le fondamenta per l'architettura di Tool Calling (Chiamata a Funzione) e Routing tramite un approccio nativo e leggero.

**Tempo Stimato:** 3 giorni
**Prerequisiti:**
- macOS con Python 3.14
- Ollama Server v0.12.11 installato (scarica da [GitHub Releases](https://github.com/ollama/ollama/releases))
- Modello baby-ai-qwen3-tool creato localmente da file GGUF
  - Scarica `Qwen3-4B-Function-Calling-Pro.gguf` (4.28 GB) da Hugging Face: [Pagina modello](https://huggingface.co/Manojb/Qwen3-4B-toolcalling-gguf-codex) | [Download diretto](https://huggingface.co/Manojb/Qwen3-4B-toolcalling-gguf-codex/blob/main/Qwen3-4B-Function-Calling-Pro.gguf)
- Node.js 25 (tramite nvm)
- Rust 1.91.1 per Tauri

---

## Struttura delle Directory

```text
baby-ai-python/
├── src/                           # Backend Python
│   ├── __init__.py
│   ├── main.py                    # Entry point app FastAPI
│   ├── config.py                  # Configurazione e variabili d'ambiente
│   ├── models/
│   │   ├── __init__.py
│   │   ├── schemas.py             # Modelli Pydantic (ToolCall, ExecutionResult, ecc.)
│   │   └── config.py              # OrchestratorConfig
│   ├── llm/
│   │   ├── __init__.py
│   │   ├── client.py              # Interfaccia astratta LLMClient
│   │   └── ollama_adapter.py      # Implementazione OllamaAdapter
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── base.py                # Classe base astratta BaseAgent
│   │   └── app_agent.py           # Implementazione AppAgent
│   ├── orchestrator/
│   │   ├── __init__.py
│   │   ├── prompts.py             # Prompt di Sistema
│   │   └── orchestrator.py        # Logica di Orchestrazione/Routing Principale
│   └── utils/
│       ├── __init__.py
│       └── logger.py              # Setup del logging strutturato
├── src-tauri/                     # App Desktop Tauri
├── tests/
├── .env.example
├── .gitignore
├── requirements.txt
├── README.md
├── Modelfile                      # File di configurazione Ollama per GGUF
├── models/llm/                    # Cartella per file GGUF
└── pyproject.toml
```

---

## Fase 0: Setup dell'Ambiente (30 minuti)

### Step 0.1: Crea la Directory di Progetto
```bash
# Naviga nella tua cartella di sviluppo, ad esempio:
cd ~/Developer
# Crea la directory di progetto e naviga al suo interno
mkdir -p "Nuova Baby AI"
cd "Nuova Baby AI"
```

### Step 0.2: Inizializza Git
```bash
# Inizializza la repository locale se necessario
# git init
# Collega il repository remoto se necessario
# git remote add origin <URL-REPO>
# Assicurati di essere sul branch main
# git checkout main
# git pull origin main
```

### Step 0.3: Crea l'Ambiente Virtuale Python
```bash
python3.14 -m venv venv
source venv/bin/activate
python --version  # Verifica 3.14.x
```

### Step 0.4: Installa le Dipendenze
```bash
cp /home/ubuntu/requirements.txt .
# Assicurati che 'ollama', 'pydantic', 'fastapi' e 'appscript' siano inclusi.
pip install --upgrade pip
pip install -r requirements.txt
```

**Validazione:**
- Python 3.14.x confermato
- Tutti i pacchetti installati senza errori

### Step 0.5: Verifica, Scarica e Configura Ollama (Aggiornato)
Obiettivo: Scaricare il GGUF e creare il modello locale baby-ai-qwen3-tool con il Modelfile specifico per il Tool Calling di Qwen.

1. **Scarica il modello GGUF corretto da Hugging Face:**
   - [Pagina modello](https://huggingface.co/Manojb/Qwen3-4B-toolcalling-gguf-codex)
   - [File GGUF](https://huggingface.co/Manojb/Qwen3-4B-toolcalling-gguf-codex/blob/main/Qwen3-4B-Function-Calling-Pro.gguf)
   - **Nome file:** `Qwen3-4B-Function-Calling-Pro.gguf` (4.28 GB)
   - Scarica il file e salvalo in `~/Downloads/` o nella cartella desiderata.
2. **Verifica Ollama:**
   ```bash
   ollama --version
   # Risultato Atteso: v0.12.11 (con tool calling stabile e fix JSON parsing)
   # Se hai una versione diversa, aggiorna da https://github.com/ollama/ollama/releases
   ```
3. **Crea la cartella per il modello:**
   ```bash
   mkdir -p models/llm
   ```
4. **Sposta il modello GGUF nella cartella:**
   ```bash
   mv ~/Downloads/Qwen3-4B-Function-Calling-Pro.gguf models/llm/
   ```
5. **Scarica il chat template ufficiale (opzionale ma raccomandato):**
   ```bash
   # Scarica chat_template.jinja dal repository HuggingFace
   curl -o models/llm/chat_template.jinja https://huggingface.co/Manojb/Qwen3-4B-toolcalling-gguf-codex/raw/main/chat_template.jinja
   ```
6. **Crea Modelfile (Modelfile):**
   - Crea il Modelfile nella root del progetto (`touch Modelfile`).
   - Contenuto di Modelfile (Ottimizzato per Tool Calling):
   ```
   FROM ./models/llm/Qwen3-4B-Function-Calling-Pro.gguf

   # Stop tokens per ChatML format
   PARAMETER stop "<|im_start|>"
   PARAMETER stop "<|im_end|>"

   # Template ChatML per Qwen3 (compatibile con tool calling)
   TEMPLATE """{{- if .System }}<|im_start|>system
   {{ .System }}<|im_end|>
   {{- end -}}
   {{- if .Prompt }}<|im_start|>user
   {{ .Prompt }}<|im_end|>
   {{- end -}}
   <|im_start|>assistant
   {{- if .Response }}{{ .Response }}{{- end -}}"""

   # Parametri ottimizzati per tool calling
   PARAMETER temperature 0.0
   PARAMETER top_p 0.9
   PARAMETER num_ctx 8192
   ```
7. **Crea il Modello Locale in Ollama:**
   ```bash
   ollama create baby-ai-qwen3-tool -f Modelfile
   ```
8. **Verifica il modello:**
   ```bash
   ollama list | grep baby-ai-qwen3-tool
   ```

**Validazione:**
- Ollama v0.12.11 confermato
- Modello baby-ai-qwen3-tool presente in ollama list

---

## Fase 1: Core Data Models (1 ora)

### Step 1.1: Crea la Struttura di Progetto
(Eseguire i comandi bash originali.)

### Step 1.2: Implementa Pydantic Schemas (`src/models/schemas.py`)
> Questi schemi Pydantic definiscono e validano la struttura JSON di Tool Call generata nativamente da Ollama (standard OpenAI).

**Validazione:**
```bash
python -c "from src.models.schemas import ChatRequest, ChatResponse; print('✅ Schemas OK')"
```

### Step 1.3: Implementa Orchestrator Config (`src/models/config.py`)
(Mantenere il contenuto originale.)

---

## Fase 2: Layer Client LLM (2 ore)

### Step 2.1: Crea il Client LLM Astratto (`src/llm/client.py`)
(Mantenere il contenuto originale.)

### Step 2.2: Implementa l'Adapter Ollama (`src/llm/ollama_adapter.py`)
> Il codice in questo file deve essere aggiornato per accettare l'array tools dal chiamante (l'Orchestrator) e passarlo al metodo `ollama.chat()` (o `ollama.generate()`) del client ollama-python.

---

## Fase 3: Implementazione App Agent (2 ore)

### Step 3.1: Crea l'Agent Base (`src/agents/base.py`)
(Mantenere il contenuto originale.)

### Step 3.2: Implementa l'App Agent (`src/agents/app_agent.py`)
> Mantenere il codice originale. Questo agent definisce le funzioni `open_app` e `close_app` e fornisce il loro schema JSON (che verrà raccolto dall'Orchestrator).

---

## Fase 4: Logica dell'Orchestrator (3 ore)  
**(IMPLEMENTAZIONE CHIAVE: Routing)**

### Step 4.1: Crea i Prompt di Sistema (`src/orchestrator/prompts.py`)
(Mantenere il contenuto originale.)

### Step 4.2: Implementa l'Orchestrator (`src/orchestrator/orchestrator.py`)  
**Logica di Routing Nativa**

**Azioni:**
1. Rimuovere l'importazione di qualsiasi libreria di orchestrazione esterna.
2. Preparazione:
   - Definire una mappa degli agenti attivi (es. `AGENT_MAP = {"app": AppAgent()}`).
   - Creare una funzione helper (`get_all_tools()`) per raccogliere gli schemi JSON delle funzioni da tutti gli agenti (es. `AppAgent.get_tools()`) e formattarli nell'array tools standard di Ollama/OpenAI.
3. Riscrivere il ciclo `orchestrate_with_retry`:
   - Inizializzazione: Ottenere l'elenco delle funzioni disponibili tramite `get_all_tools()`.
   - Ciclo LLM: Chiamare l'LLMClient (OllamaAdapter) passando il messaggio utente e l'elenco delle funzioni (tools).
   - Parsing e Routing:
     - Controllare la risposta LLM per il campo `tool_calls`.
     - Se `tool_calls` è presente, iterare su di essi e usare la `AGENT_MAP` per indirizzare la chiamata all'agente corretto (AppAgent).
     - Eseguire il tool e costruire un messaggio di risposta di tipo tool contenente il risultato.
     - Inviare il messaggio di risposta tool come feedback all'LLM (nel loop di conversazione) per generare la risposta finale amichevole.
   - Risposta Testuale: Se `tool_calls` non è presente, la risposta è la risposta finale all'utente.

---

## Fase 5: FastAPI Backend (2 ore)

### Step 5.1: Setup Logging (`src/utils/logger.py`)
(Mantenere il contenuto originale.)

### Step 5.2: Configuration (`src/config.py`)
(Mantenere il contenuto originale.)

### Step 5.3: Main FastAPI App (`src/main.py`)
(Mantenere il contenuto originale.)

### Step 5.4: Crea `.env.example` (Aggiornato)
```env
# Ollama Configuration
OLLAMA_BASE_URL=http://localhost:11434
LLM_MODEL=baby-ai-qwen3-tool
EMBED_MODEL=nomic-embed-text

# Server Configuration
HOST=0.0.0.0
PORT=8000
```

---

## Fase 6: Testing (3 ore)
> Mantenere tutti i passi e i test. I test convalidano ora il corretto funzionamento del nuovo Tool Calling e della logica di Routing manuale.

---

## Fase 7: Testing Manuale & Validazione (1 ora)
> Mantenere tutti i passi. Convalidare che il nuovo modello LLM e l'Orchestrator nativo generino correttamente le chiamate a funzione `open_app` e `close_app` e che il flusso di feedback funzioni.

---

## Fase 8: Documentazione (1 ora)

### Step 8.1: Crea README.md (Aggiornato)

```markdown
# Baby AI - Phase 1.1 POC

Python rewrite di Baby AI con architettura multi-agente. La Fase 1.1 implementa l'App Agent per il controllo delle applicazioni macOS.

## Prerequisiti
- macOS 13+ (Ventura o successivo)
- Python 3.14.0
- Ollama Server v0.12.11 installato localmente
- **Modello `baby-ai-qwen3-tool` creato localmente da GGUF**
  - File: `Qwen3-4B-Function-Calling-Pro.gguf` (4.28 GB)
  - Scarica da Hugging Face: [Pagina modello](https://huggingface.co/Manojb/Qwen3-4B-toolcalling-gguf-codex) | [Download diretto](https://huggingface.co/Manojb/Qwen3-4B-toolcalling-gguf-codex/blob/main/Qwen3-4B-Function-Calling-Pro.gguf)

## Setup
1. **Installa Ollama**
   ```bash
   # Scarica da https://ollama.ai
   ollama --version
   ```
2. **Prepara il Modello**
   ```bash
   # Scarica Qwen3-4B-Function-Calling-Pro.gguf (4.28 GB) da HuggingFace
   # Crea il Modelfile come descritto nella Fase 0.5
   ollama create baby-ai-qwen3-tool -f Modelfile
   ollama pull nomic-embed-text
   ```
3. **Crea Ambiente Virtuale**
   ```bash
   python3.14 -m venv venv
   source venv/bin/activate
   ```
4. **Installa le Dipendenze**
   ```bash
   pip install -r requirements.txt
   ```

## Architettura
- FastAPI Backend: Server API REST
- Orchestrator Logic (Custom): Gestisce il Tool Calling e il Routing manuale
- LLM Client: Interfaccia agnostica al modello (Adapter Ollama)
- App Agent: Controllo app macOS (open/close)

## Ambito della Fase 1.1
- ✅ App Agent solo (open_app, close_app)
- ✅ Supporto nativo Ollama Tool Calling (Qwen 3)
- ✅ Logica di Routing implementata nell'Orchestrator
- ✅ Retry di validazione (max 3 tentativi)
- ⏭️ Warm-up (Fase 1.2)
- ⏭️ Altri agenti (Fase 1.2+)
```

---
