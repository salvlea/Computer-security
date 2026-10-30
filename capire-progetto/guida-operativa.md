# Guida operativa: strumenti e passo-passo per costruire il progetto

> Documento **pratico**. Mentre `consigli_progetto.md` dice *cosa* costruire e *perché*,
> questo file dice **con quali programmi** e **in che ordine** farlo.
> Pensato per **macOS** (il tuo sistema) e per il fatto che lavori **da solo** sulla
> sandbox. Segui le fasi in ordine: ognuna produce qualcosa di concreto.

---

## Parte A — Gli strumenti che ti servono (lo "stack")

Divido gli strumenti in **essenziali** (senza questi non parti) e **opzionali/bonus**
(alzano il voto ma non sono obbligatori).

### A.1 Essenziali

| Strumento | A cosa serve nel progetto | Come si installa (macOS) |
|-----------|---------------------------|--------------------------|
| **Python 3.11+** | Linguaggio principale: broker, policy engine, script della demo | `brew install python@3.12` (o già presente) |
| **Docker Desktop** | Il motore di isolamento: ogni esecuzione di codice va in un container effimero | Scarica da docker.com oppure `brew install --cask docker` |
| **Git + GitHub** | Versionare il progetto e mostrarlo (già configurato in questo repo) | già presente |
| **VS Code** | Editor (lo stai già usando) | già presente |
| **Un LLM** | Genera il codice dal prompt + fa da "secondo LLM giudice" | vedi sezione A.3 |

### A.2 Librerie Python (dentro un ambiente virtuale)

Le installerai nella Fase 0. Le principali:

- **`docker`** — SDK Python per controllare Docker dal codice (creare/avviare/distruggere
  container, leggere i limiti). È il cuore tecnico della sandbox.
- **`ollama`** *oppure* **`openai`** / **`anthropic`** — per parlare con l'LLM (vedi A.3).
- **`pytest`** — per far girare la batteria di test (attacchi + casi legittimi) in modo
  automatico e riproducibile.
- **`rich`** — output colorato in terminale (comodo per la demo: tabelle con/senza sandbox).
- **`pyyaml`** — per leggere il file di configurazione della policy (i livelli di isolamento).

> Nota: **`llm-sandbox`** (la libreria citata nei consigli) puoi installarla e **studiarla**
> come riferimento dello stato dell'arte, ma per la lode conviene che il cuore lo scriva tu,
> così dimostri di aver capito *come funziona* e non solo di saper usare una libreria.

### A.3 Quale LLM usare? (decisione importante)

Hai due strade. Ti consiglio di partire con **Ollama** (gratis, locale) e citare le API
come alternativa "di produzione".

| | **Ollama (locale)** — consigliato per iniziare | **API (OpenAI/Anthropic)** |
|---|---|---|
| Costo | Gratis | A consumo (serve carta / crediti) |
| Chiave API da proteggere | No | Sì (ironia: è proprio uno degli asset che la sandbox difende!) |
| Qualità del codice generato | Buona con modelli come `llama3.1` / `qwen2.5-coder` | Superiore |
| Serve internet | No (dopo il download del modello) | Sì |
| Installazione | `brew install ollama` poi `ollama pull qwen2.5-coder` | `pip install openai` + chiave |

**Consiglio**: usa Ollama per lo sviluppo e i test (fai mille prove senza spendere). Nel
report scrivi che l'architettura è agnostica rispetto al modello e che con una API
commerciale la qualità salirebbe. Così mostri consapevolezza senza costi.

### A.4 Opzionali / bonus (per spingere verso il top)

- **gVisor** (`runsc`): runtime alternativo di Docker con isolamento più forte.
  **⚠️ Attenzione macOS**: gVisor gira **solo su Linux**. Su Mac non lo installi
  nativamente. Se vuoi il confronto Docker-vs-gVisor nel report hai due opzioni:
  1. Farlo su una **VM Linux** (es. una piccola VM cloud, oppure locale con UTM/Lima/
     multipass) — è il modo "vero".
  2. Trattarlo **teoricamente** nel report (spieghi cosa cambia) senza demo live.
  Non è obbligatorio: la demo forte la fai già con Docker hardened.
- **`RestrictedPython`**: la sandbox "a livello di linguaggio" (debole). Vale la pena
  installarla e mostrarne **un bypass** per dimostrare *perché* non basta — è un ottimo
  argomento da lode.
- **Podman**: alternativa rootless a Docker, citabile come variante più sicura.

> **Promemoria macOS**: Docker Desktop su Mac esegue già i container **dentro una VM Linux**
> leggera. Questo è un *vantaggio* di isolamento (c'è già un confine di virtualizzazione tra
> container e il tuo macOS) ed è un punto interessante da scrivere nel report.

---

## Parte B — Passo-passo: cosa fare, in ordine

Ogni fase ha un **obiettivo** e un **risultato concreto** (una "spunta"). Non saltare la
Fase 2: preparare i test *prima* del codice è ciò che rende il progetto scientifico.

### Fase 0 — Preparare l'ambiente di lavoro
**Obiettivo**: avere tutti gli strumenti pronti e una struttura di cartelle pulita.

> Stato attuale del tuo Mac (già verificato): **Python 3.13**, **Homebrew** e **Git** sono
> già presenti. Ti mancano solo **Docker** e **Ollama**. I comandi qui sotto sono
> copia-incolla, uno alla volta.

**0.1 — Installa Docker Desktop**
```bash
brew install --cask docker
```
Poi apri l'app **Docker Desktop** (icona balena), accetta i termini e aspetta che l'icona
nella barra in alto diventi fissa (= Docker avviato). Verifica che funzioni:
```bash
docker run hello-world
```
Se stampa "Hello from Docker!" sei a posto. ✅

**0.2 — Installa Ollama e scarica il modello per il codice**
```bash
brew install ollama
ollama serve            # avvia il server: lascialo girare in QUESTA finestra
```
Apri **un secondo terminale** e scarica il modello (~4–5 GB, ci mette un po'):
```bash
ollama pull qwen2.5-coder
```
Prova che risponda:
```bash
ollama run qwen2.5-coder "scrivi una funzione python che somma due numeri"
```

**0.3 — Crea la struttura del progetto e l'ambiente virtuale Python**
Dalla cartella del progetto (`Computer-security/`):
```bash
# cartelle
mkdir -p sandbox/config llm tests/prompts benchmark

# ambiente virtuale Python (isola le librerie del progetto)
python3 -m venv .venv
source .venv/bin/activate      # da rifare a ogni nuova sessione di terminale
```

**0.4 — Crea il file `requirements.txt` e installa le librerie**
Metti questo contenuto in `Computer-security/requirements.txt`:
```
docker
ollama
pytest
rich
pyyaml
```
Poi installale (con l'ambiente virtuale attivo):
```bash
pip install -r requirements.txt
```

**0.5 — Verifica finale**
```bash
docker run hello-world      # Docker ok
ollama list                 # mostra qwen2.5-coder
pip list                    # mostra docker, ollama, pytest, rich, pyyaml
```

**Struttura di cartelle proposta** (la creeremo insieme):
```
Computer-security/
├── capire-progetto/          # documenti teorici (già esistente)
├── sandbox/                  # il codice della sandbox
│   ├── broker.py             # intercetta l'azione e decide
│   ├── policy_engine.py      # regole + secondo LLM giudice
│   ├── runner_docker.py      # esegue il codice nel container isolato
│   ├── auditor.py            # logging delle azioni e decisioni
│   └── config/
│       └── policy.yaml       # livelli di isolamento configurabili
├── llm/
│   └── client.py             # parla con l'LLM (genera il codice)
├── tests/
│   └── prompts/              # la batteria di prompt buoni + malevoli
│       ├── legittimi.yaml
│       └── malevoli.yaml
├── benchmark/
│   └── run_benchmark.py      # esegue tutti i test e produce la tabella metriche
├── requirements.txt
└── README.md
```
**Risultato**: `docker run hello-world` funziona, `ollama run` risponde, `pip list` mostra
le librerie. ✅

---

### Fase 1 — Congelare scope e threat model
**Obiettivo**: mettere per iscritto (in una pagina) cosa la sandbox difende e cosa no.

- Scope: **A + B** (esecuzione di codice generato + azioni di agente), come da consigli.
- Scrivi il threat model: attori, asset, capacità, e **cosa è fuori scope** (il guardrail).
- Cita OWASP LLM Top 10 e MITRE ATLAS.

Questa parte è **già quasi pronta** in `consigli_progetto.md` (sezioni 1 e 2): qui la
sintetizzi in forma definitiva per il report.

**Risultato**: una paginetta "scope + threat model" pronta da incollare nel report. ✅

---

### Fase 2 — Costruire la batteria di test PRIMA del codice
**Obiettivo**: definire *cosa devi dimostrare*, così il codice ha un bersaglio chiaro.

Prepara due liste di prompt (file YAML in `tests/prompts/`):

- **Legittimi** (la sandbox deve lasciarli passare): "calcola i primi 100 numeri primi",
  "analizza questo CSV e dammi la media", ecc.
- **Malevoli** (la sandbox deve contenerli):
  - lettura file sensibile (finto `/etc/passwd`, finta `~/.ssh/id_rsa`);
  - esfiltrazione di rete (`curl`/`requests` verso un host esterno);
  - comando distruttivo (`rm -rf`);
  - fork bomb / loop infinito (deve scattare il timeout);
  - tentativo di leggere una variabile d'ambiente con una finta chiave API.

Bastano **15–20 casi** ben scelti. Ognuno ha un esito atteso (`consentito` / `bloccato`).

**Risultato**: `tests/prompts/legittimi.yaml` e `malevoli.yaml` compilati. ✅

---

### Fase 3 — Prototipo minimo: eseguire codice in Docker
**Obiettivo**: la prima "gabbia" funzionante, dalla più debole alla più forte.

1. **Baseline**: prendi del codice Python e fallo eseguire dentro un container Docker,
   catturando output e errori (`runner_docker.py`).
2. **Hardening** (il passaggio che conta), aggiungendo un flag alla volta:
   - `--network none` (niente rete → blocca l'esfiltrazione);
   - `--read-only` + directory temporanea scrivibile usa-e-getta;
   - `--cap-drop=ALL` e `--user` non-root;
   - `--memory`, `--cpus`, `--pids-limit` (anti fork-bomb) e **timeout** di esecuzione;
   - container **effimero**: distrutto dopo ogni run (`--rm`).

**Risultato**: dai un pezzo di codice → gira isolato → torna il risultato, e gli attacchi
di rete/risorse sono già contenuti. ✅

---

### Fase 4 — Il broker + policy engine (il "secondo LLM")
**Obiettivo**: mettere un decisore *davanti* alla sandbox (la ricorsività chiesta dal prof).

1. **Interceptor**: il broker riceve il codice generato dall'LLM prima di eseguirlo.
2. **Policy engine**, tre modalità (implementane almeno due):
   - *rule-based*: allow-list / deny-list di pattern pericolosi;
   - *LLM-based*: un **secondo LLM** classifica l'azione come `safe`/`unsafe`;
   - *ibrida* (consigliata): regole rigide per i casi certi + LLM per gli ambigui.
3. La policy è configurabile via `policy.yaml` (così mostri trade-off diversi).

**Risultato**: il broker approva/rifiuta un'azione e, se approvata, la manda alla sandbox. ✅

---

### Fase 5 — Auditor / logging
**Obiettivo**: registrare ogni azione e decisione (serve alla demo e alla valutazione).

- Ogni run produce una riga di log: prompt → codice → decisione → esito → tempo.
- Utile anche per l'osservabilità (uno degli "assi di controllo" dei consigli).

**Risultato**: un file di log leggibile che racconta cosa è successo a ogni test. ✅

---

### Fase 6 — Esperimenti: con vs senza sandbox + metriche
**Obiettivo**: il confronto che trasforma il compitino in un progetto da lode.

1. Fai girare tutta la batteria della Fase 2 in due modalità: **senza** sandbox (danno) e
   **con** sandbox (contenuto).
2. Produci la **tabella degli esiti** e calcola le metriche:
   - % di attacchi contenuti;
   - falsi positivi (legittimi bloccati per errore);
   - overhead (latenza introdotta dalla sandbox).

**Risultato**: `benchmark/run_benchmark.py` stampa la tabella finale con le metriche. ✅

---

### Fase 7 — Report e presentazione
**Obiettivo**: raccontare tutto seguendo la struttura della sezione 8 dei consigli.

- Introduzione → threat model → principi (scala di isolamento) → stato dell'arte →
  architettura → demo con attacchi → metriche → limiti onesti → cenno ai guardrail.
- Inserisci le tabelle prodotte nelle Fasi 2 e 6 e lo schema dell'architettura.

**Risultato**: report + demo pronti per l'esame. ✅

---

## Parte C — Ordine rapido delle decisioni da prendere subito

1. **LLM**: Ollama locale (consigliato) o API? → decidiamolo nella Fase 0.
2. **Livello di isolamento target della demo**: Docker hardened (consigliato); gVisor solo
   come bonus/teoria vista la limitazione macOS.
3. **Policy engine**: parti rule-based, poi aggiungi il secondo LLM (arrivi all'ibrido).

> Quando hai letto questo file, torna in chat: partiamo dalla **Fase 0** e installiamo/
> verifichiamo gli strumenti uno alla volta.
