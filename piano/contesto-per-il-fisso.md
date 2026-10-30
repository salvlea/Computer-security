# Contesto per la nuova chat sul FISSO (passaggio di consegne)

> Questo file serve a una **nuova conversazione con Claude Code sul PC fisso Windows**, che
> non ha memoria della chat fatta sul Mac. Leggilo **per primo**, poi `piano-operativo.md`.
> Scritto sul Mac il 30/10/2026. Dove trovi **[DA COMPILARE]** il dato manca ancora.

---

## 0. Prompt da incollare nella nuova chat

```
Leggi, in quest'ordine: piano/contesto-per-il-fisso.md, piano/piano-operativo.md, poi
richiesta-professore/progetto.txt e i documenti in capire-progetto/ e linee-guida/.
Dove le vecchie guide contraddicono il piano operativo, vale il piano operativo.
Sono sul PC fisso Windows. Dobbiamo eseguire la FASE A del piano, un passo alla volta:
prima dimmi cosa vuoi controllare o installare e aspetta la mia conferma. Non lanciare
comandi che installano programmi, cambiano il firewall o le impostazioni di alimentazione
senza chiedermelo. Rispondi in italiano e spiegami i comandi in modo semplice.
```

---

## 1. Il progetto in breve

- **Corso:** Computer Security. Progetto **individuale**, tema **SANDBOX LLM**.
  Il tema "guardrail" è di un altro studente (Moscuza): qui è solo contesto, non è da fare.
- **Richiesta del professore** (testo completo in `richiesta-professore/progetto.txt`):
  stato dell'arte con librerie open source, principi di funzionamento, progettazione
  sommaria e **demo**. Idea chiave: un LLM che ne invoca un altro prima di agire
  (ricorsività, "AI che controlla AI"). Il progetto è volutamente poco specificato.
- **Esame:** demo **dal vivo** usando il **MacBook** (non il fisso) in università.
- **Scope scelto:** esecuzione sicura del codice generato da un LLM (+ azioni di un agente).
  Architettura: broker/policy engine (regole fisse + LLM giudice) davanti a una sandbox Docker.

## 2. Cosa è stato fatto finora (sul Mac)

- Analizzate tutte le cartelle del repo e l'hardware del Mac.
- Scritti **solo documenti**: `piano/piano-operativo.md` e questo file.
- **Non è stato installato né modificato nulla** sul Mac (nessun Ollama, nessun codice,
  nessun container). Le vecchie guide **non sono ancora state corrette** (vedi sezione 5).
- Nessun codice del progetto esiste ancora: si parte dalla Fase A del piano.

## 3. Hardware (misurato sul Mac, non stimato)

| | Mac (demo) | Fisso (sviluppo + LLM) |
|---|---|---|
| Modello | MacBookPro14,1, 13", 2017 | Desktop |
| CPU | Intel i5-7360U, 2 core / 4 thread, AVX2 | Ryzen 7 7700X |
| RAM | 8 GB, ~480 MB di swap già usati | 32 GB DDR5 |
| GPU | Intel Iris Plus 640 (inutile per l'LLM) | Radeon "7800" con 16 GB, **probabilmente RX 7800 XT [DA VERIFICARE]** |
| OS | macOS 13.7.8 (ultimo possibile per questo Mac) | Windows **[DA COMPILARE: versione]** |
| Disco | 15 GB liberi su 113; Docker ne occupa 8,6 | **[DA COMPILARE]** |
| Docker | Desktop **4.43.0** (Engine 28.3.0), VM 4 GB / 4 CPU / disco 61 GB, aggiornamenti automatici **disattivati** | da installare |
| Python | 3.13.1 | da installare (usare 3.13) |
| Altro | Homebrew 6.0.12; Ollama non installato | |

App che occupano più RAM sul Mac: Antigravity (~1 GB in più processi), Chrome, CleanMyMac.

## 4. Decisioni già prese (non rimetterle in discussione senza motivo)

1. Si sviluppa e si testa sul **fisso**; il Mac serve per la validazione e la demo.
2. All'esame l'**LLM gira sul fisso** (acceso a casa) e il Mac lo raggiunge con **Tailscale**.
   La **sandbox Docker gira sul Mac**, dal vivo.
3. **Ollama sul fisso: nativo su Windows** (la GPU AMD non arriva ai container su Windows).
   **Ollama sul Mac: in Docker**, con il 3B come riserva (l'app ufficiale richiede macOS 14).
4. Modelli sempre **con il tag** (`:3b`, `:7b`, `:14b`). Mai `qwen2.5-coder` senza tag.
5. Docker del Mac **non va aggiornato** (le versioni nuove richiedono macOS 14).
6. Il fisso imita i limiti del Mac per la sandbox con `.wslconfig` (4 GB, 4 CPU, 1 GB swap).
7. Scala del piano B: remoto → locale 3B → replay delle risposte salvate → video.
8. Il broker sta **fuori** dai container; i container della sandbox usano `--network none`.
9. Benchmark su più profili (3B/3B, 14B/7B, solo regole) come risultato per il report.

## 5. Cose da correggere nelle vecchie guide (ancora NON fatto)

Se la nuova chat legge `linee-guida/ollama.md` e `capire-progetto/guida-operativa.md`, deve
sapere che queste parti sono **superate** dal piano operativo:

| Dove | Dice | Va letto così |
|---|---|---|
| `ollama.md` §2 e checklist | `brew install ollama` | Sul **fisso**: installer/winget, nativo. Sul **Mac**: Ollama in Docker |
| `ollama.md` §3 e §4, `guida-operativa.md` 0.2 | `qwen2.5-coder` senza tag | Usare `:3b`, `:7b`, `:14b` |
| `ollama.md` §3 | secondo modello `llama3.1` come giudice | Il giudice è `qwen2.5-coder:7b` (o 3B sul Mac); niente `llama3.1` sul Mac per la RAM |
| `guida-operativa.md` 0.1 | `brew install --cask docker` | Sul Mac Docker è già installato (4.43.0): **non aggiornare**. Sul fisso: Docker Desktop con WSL2 |
| `guida-operativa.md` A.4 | gVisor "solo teoria" | Sul fisso (Linux/WSL2) si può provare per una misura extra, **opzionale** |
| `ollama.md` esempi Python | nessun timeout, host fisso | Host e timeout vengono dal profilo (`profili.yaml`); timeout LLM almeno 120 s |

Proposta: la nuova chat (o questa) può applicare queste correzioni ai file. Da decidere.

## 6. Verificato e non verificato

**Verificato (con fonte, vedi piano §15):**
- Ollama ufficiale richiede macOS 14 → non installabile in nativo sul Mac.
- Docker Desktop recente non supporta macOS 13 → restare alla 4.43.0.
- Homebrew non ha bottle di Ollama per Ventura (compilerebbe da sorgente).
- Tailscale indica macOS 12 come minimo.
- La documentazione Ollama elenca la RX 7800 XT per Windows.

**NON verificato — da controllare con una prova pratica:**
- Che la GPU del fisso venga davvero usata (`ollama ps` → `100% GPU`).
- Che il 14B e il 7B stiano insieme nei 16 GB di VRAM.
- Che Tailscale si installi e funzioni su macOS 13.
- Che Ollama sul fisso riparta dopo un riavvio senza login.
- Che l'immagine `ollama/ollama` entri nello spazio libero del Mac.
- Che la rete dell'università lasci passare Tailscale.
- **Tutti i tempi di risposta** citati nei documenti (sono stime, non misure).

## 7. Dati mancanti da compilare

| Dato | Valore |
|---|---|
| Data dell'esame | **[DA COMPILARE]** |
| Versione di Windows (`winver`) | **[DA COMPILARE]** |
| Modello esatto della scheda grafica | **[DA COMPILARE]** |
| Spazio libero su C: nel fisso | **[DA COMPILARE]** |
| Virtualizzazione (SVM) attiva nel BIOS? | **[DA COMPILARE]** |
| Il fisso è collegato via cavo o Wi-Fi? | **[DA COMPILARE]** |
| Il repo GitHub è pubblico o privato? | **[DA COMPILARE]** |

## 8. Cosa annotare sul fisso e riportare sul Mac

Durante la Fase A, appunta qui i valori (serviranno nella Fase B):

| Cosa | Dove si trova | Valore |
|---|---|---|
| Versione di Ollama | `ollama --version` | **[DA COMPILARE]** |
| IP Tailscale del fisso | `tailscale ip -4` | **[DA COMPILARE]** |
| Nome dell'account Tailscale usato | app Tailscale | **[DA COMPILARE]** |
| Modelli scaricati | `ollama list` | **[DA COMPILARE]** |
| Esito `ollama ps` con il 14B | colonna PROCESSOR | **[DA COMPILARE]** |
| Esito del test di riavvio senza login | piano F6 | **[DA COMPILARE]** |
| Versione di Python | `python --version` | **[DA COMPILARE]** |

L'IP di Tailscale va nel file `.env` del Mac (`OLLAMA_HOST_REMOTO=...`), **mai nel repo**.

## 9. Stato del repository

- Remote: `https://github.com/salvlea/Computer-security.git`
- Branch di lavoro: `master` (esiste anche `skill`, con commit su una skill poi rimossa: non serve).
- Non esistono ancora `.gitignore` né `.gitattributes`: vanno creati **prima di scrivere
  codice** (`* text=auto eol=lf`; ignorare `.venv/`, `__pycache__/`, `.env`).
- `.DS_Store` è tracciato da git sul Mac: non committarlo.
- Convenzione: un commit per fase del piano, messaggi chiari in italiano o inglese.

## 10. Preferenze di lavoro dell'utente

- Parla e scrive in **italiano**: rispondere in italiano.
- Ha poca esperienza con Ollama e Docker: spiegare i comandi, un passo alla volta.
- Vuole **capire come funziona**, non solo usare librerie (valutazione del professore):
  il cuore del progetto va scritto da lui/lei con guida, non copiato da `llm-sandbox`.
- Ha chiesto di fare le cose **in ordine e con conferma** prima di installare o cambiare
  impostazioni del sistema.
- I file di piano e di guida sono in Markdown dentro il repo.

## 11. Ordine di lavoro sul fisso

1. Compilare la sezione 7 (dati mancanti) insieme a Claude.
2. **Fase A** del piano (F1 → F7), un passo alla volta, annotando la sezione 8.
3. Creare `.gitignore` e `.gitattributes`, fare il primo commit sul fisso.
4. **M1**: far partire `curl` dal Mac verso il fisso (serve la Fase B del Mac, Tailscale).
5. **Fase C**: sviluppo, partendo dalla batteria di test (`tests/prompts/*.yaml`) **prima** del codice.
6. Tornare sul Mac per la Fase D (misure e taratura dei timeout).

## 12. Cose da non dimenticare

- Il **nucleo minimo** del progetto è: sandbox hardened + broker con regole + benchmark +
  validazione sul Mac. Giudice LLM, gVisor e il resto sono extra.
- I tempi misurati sul fisso **non** vanno nel report come tempi della sandbox: quelli si
  misurano sul Mac.
- Il giudice LLM **non è affidabile al 100%**: è un risultato da misurare e dichiarare.
- Test d'attacco solo con **file finti** e solo in container con limiti (mai fork bomb senza `--pids-limit`).
