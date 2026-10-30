# Piano operativo — Sandbox LLM (fisso Windows + Mac in università)

> Documento **operativo**. Dice cosa installare, su quale macchina, in che ordine, con quali
> comandi, cosa può andare storto e cosa fare in quel caso.
> Si appoggia a `capire-progetto/guida-operativa.md` (le fasi del progetto) e a
> `linee-guida/ollama.md`, **ma dove i due documenti sono in conflitto con questo piano
> vale questo piano** (vedi sezione 1 per le differenze).
>
> Legenda: **[FISSO]** = si fa sul PC Windows di casa · **[MAC]** = si fa sul MacBook ·
> **[ENTRAMBI]** = stessa cosa su entrambi.
> Le affermazioni che non ho potuto verificare sono marcate **[DA VERIFICARE]**.
> I numeri di velocità sono **stime**, non misure: le misurazioni vere sono un compito del piano.

---

## 0. L'idea in una pagina

```
 UNIVERSITÀ                                         CASA
┌──────────────────────────────┐   Tailscale    ┌─────────────────────────────┐
│ MAC (MacBook Pro 2017)        │  rete privata  │ FISSO (Ryzen 7 7700X)       │
│  - broker + policy engine     │  cifrata,      │  - Ollama (nativo Windows)  │
│  - sandbox Docker             │  passa SOLO    │  - modelli 7B / 14B         │
│    (qui gira il codice        │  testo         │    su RX 7800 XT 16 GB      │
│     generato dall'LLM)        │ ◀────────────▶ │  - sempre acceso            │
│  - Ollama locale 3B (riserva) │                │                             │
└──────────────────────────────┘                └─────────────────────────────┘
```

- Il **fisso** fa il lavoro pesante: sviluppo, test, benchmark e, il giorno dell'esame,
  l'inferenza dell'LLM.
- Il **Mac** fa la parte che deve essere dimostrata dal vivo: il broker decide e la
  **sandbox Docker esegue il codice** sul Mac, davanti al professore.
- Tra i due viaggia solo testo (prompt e codice generato). Basta una connessione lenta.
- La **demo non dipende da una sola cosa**: se il fisso non risponde, si passa al modello
  piccolo sul Mac; se anche quello è troppo lento, si usano le risposte già salvate
  (`replay`). Vedi sezione 4.

**Principio guida:** il codice è lo stesso su entrambe le macchine. Cambia solo un file di
configurazione (profilo) che dice dove si trova l'LLM.

---

## 1. Decisioni prese e differenze rispetto alle altre guide

| Tema | Decisione | Perché |
|---|---|---|
| Dove sviluppo | Sul **fisso** | Molto più veloce; il Mac ha 8 GB di RAM e un i5 dual-core |
| Dove gira l'LLM all'esame | Sul **fisso**, via Tailscale | Il Mac non regge modelli buoni; il 3B locale è solo riserva |
| Dove gira la sandbox all'esame | Sul **Mac** | È ciò che si dimostra dal vivo |
| Ollama sul fisso | **Nativo su Windows**, non in Docker | Nei container su Windows la GPU AMD non è utilizzabile; in nativo sì |
| Ollama sul Mac (riserva) | **In Docker**, con il 3B | L'app Ollama ufficiale richiede macOS 14, il Mac ha il 13.7.8 |
| Docker sul Mac | Resta la **4.43.0**, mai aggiornare | Le versioni nuove richiedono macOS 14 |
| Modello di riferimento | `qwen2.5-coder:3b` (Mac-equivalente) e `14b`/`7b` (demo) | Il benchmark si fa con entrambi: diventa un risultato per il report |
| `brew install ollama` | **Non usare** sul Mac | Non c'è un pacchetto precompilato per Ventura: compilerebbe da sorgente |
| `qwen2.5-coder` senza taglia | **Evitare**, indicare sempre il tag (`:3b`, `:7b`, `:14b`) | Senza tag scarica il 7B, troppo pesante per il Mac |
| Rete | **Tailscale**, mai porte aperte su internet | Ollama non ha password |
| Broker | **Fuori** dai container | Dentro un container servirebbe `docker.sock`, che equivale a permessi di amministratore |

---

## 2. Hardware: cosa implica per il progetto

| | Mac (demo) | Fisso (sviluppo + LLM) |
|---|---|---|
| CPU | i5-7360U, 2 core / 4 thread, AVX2 | Ryzen 7 7700X, 8 core / 16 thread |
| RAM | 8 GB, già ~480 MB di swap in uso | 32 GB DDR5 |
| GPU | Intel Iris Plus 640 (inutile per l'LLM) | RX 7800 XT, 16 GB di VRAM |
| OS | macOS 13.7.8 (ultimo possibile) | Windows (versione da controllare) |
| Disco libero | ~15 GB (Docker ne occupa già 8,6) | da controllare |
| Docker | Desktop 4.43.0, VM con 4 GB / 4 CPU | da installare |

**Cosa dipende dall'hardware e cosa no:**

| Cosa | Dipende dall'hardware? | Dove si misura |
|---|---|---|
| La sandbox blocca l'attacco? (rete, file, fork bomb) | No: Docker si comporta allo stesso modo | Fisso, poi si conferma sul Mac |
| Che codice scrive l'LLM, cosa decide il giudice | No, dipende da modello e prompt | Fisso |
| Tempo di avvio del container, timeout, overhead | **Sì** | **Solo sul Mac** |
| Velocità dell'LLM | **Sì** | Si misura dove gira (fisso o Mac) |

Un modello da 14B sulla GPU del fisso risponde in pochi secondi; lo stesso 3B sul Mac può
impiegare 20-30 secondi (stima). Per questo i tempi del fisso **non** vanno scritti nel
report come tempi della sandbox.

---

## 3. Architettura del software

```
Computer-security/
├── piano/
│   └── piano-operativo.md          ← questo file
├── sandbox/
│   ├── broker.py                   # riceve il codice, chiede alla policy, esegue
│   ├── policy_engine.py            # regole fisse + giudice LLM (ibrido)
│   ├── runner_docker.py            # lancia il container hardened, cattura output
│   ├── auditor.py                  # log di ogni decisione e esito
│   ├── preflight.py                # controllo "è tutto pronto?" prima della demo
│   └── config/
│       ├── policy.yaml             # livelli di isolamento (baseline/hardened/...)
│       └── profili.yaml            # remoto / locale / replay
├── llm/
│   └── client.py                   # parla con Ollama; unico file che sa dove sta l'LLM
├── tests/
│   └── prompts/
│       ├── legittimi.yaml
│       └── malevoli.yaml
├── benchmark/
│   └── run_benchmark.py
├── results/
│   ├── replay/                     # risposte dell'LLM salvate (si committano)
│   └── benchmark-*.json
├── docker/
│   ├── Dockerfile.sandbox          # immagine della sandbox
│   └── compose.fallback.yaml       # Ollama locale 3B, solo sul Mac
├── demo.py                         # punto di ingresso: --profilo remoto|locale|replay
├── requirements.txt                # versioni FISSATE (pip freeze)
├── .gitattributes                  # * text=auto eol=lf
├── .gitignore                      # .venv, __pycache__, .env
└── README.md
```

**Regole di progetto che rendono il passaggio fisso → Mac indolore:**
1. `llm/client.py` è l'unico punto che conosce l'indirizzo dell'LLM. L'indirizzo viene dal
   profilo scelto.
2. Tutti i valori che dipendono dalla macchina (timeout, memoria, CPU) stanno in
   `policy.yaml` / `profili.yaml`, mai nel codice.
3. Immagini Docker con **versione fissa** (`python:3.12-slim`, `ollama/ollama:<versione>`),
   mai `latest`.
4. Il broker non gira in un container e non ha accesso a `docker.sock` da dentro uno.
   I container della sandbox usano sempre `--network none`: il codice generato non può
   raggiungere né l'LLM né la rete.

**Esempio di `profili.yaml`:**
```yaml
profili:
  remoto:                              # demo normale: LLM sul fisso
    host_env: OLLAMA_HOST_REMOTO       # es. http://100.x.y.z:11434 (da .env, non committato)
    generatore: qwen2.5-coder:14b
    giudice: qwen2.5-coder:7b
    timeout_llm_s: 120
  locale:                              # riserva: 3B nel container sul Mac
    host: http://127.0.0.1:11434
    generatore: qwen2.5-coder:3b
    giudice: qwen2.5-coder:3b
    timeout_llm_s: 180
  replay:                              # ultima riserva: nessun LLM, risposte salvate
    cartella: results/replay/
ordine_fallback: [remoto, locale, replay]
```

**Esempio di `policy.yaml` (valori di partenza, da ritarare sul Mac):**
```yaml
livelli:
  baseline:                            # solo per il confronto "senza sandbox"
    rete: bridge
    fs_readonly: false
    capabilities: tutte
  hardened:
    rete: none                         # --network none
    fs_readonly: true                  # --read-only + tmpfs per /tmp
    cap_drop: ALL
    user: "65534:65534"                # nobody, non root
    no_new_privileges: true
    memoria: 256m
    cpus: 1
    pids_limit: 64                     # anti fork bomb
    timeout_esecuzione_s: 10           # ATTENZIONE: da misurare sul Mac (sez. 4, R6)
```

---

## 4. Rischi e contromisure

### 4.1 Tabella dei rischi

Probabilità e impatto sono una mia valutazione (B = bassa, M = media, A = alta).

| ID | Rischio | Prob. | Impatto | Contromisura |
|---|---|---|---|---|
| R1 | Il **fisso è spento, in sospensione o riavviato** da Windows Update durante l'esame | M | A | Sezione 6, passo F5: sospensione disattivata, aggiornamenti in pausa, BIOS "Restore on AC Power Loss". Il giorno prima controllare da remoto |
| R2 | La **rete dell'università blocca** Tailscale o è instabile | M | A | Provare in università prima dell'esame (M6). Piano B: hotspot del telefono, Tailscale funziona anche via rete mobile |
| R3 | **Internet di casa cade** o ha un guasto | B | A | Passare al profilo `locale`, poi a `replay` |
| R4 | **Ollama non riparte** dopo un riavvio del fisso (parte al login, non al boot) **[DA VERIFICARE]** | M | A | Test di riavvio senza login (F6). Eventuale login automatico, oppure non riavviare nei giorni dell'esame |
| R5 | **La GPU non viene usata** e il 14B gira sul processore, lentissimo | M | M | Controllare `ollama ps`: deve dire `100% GPU`. Aggiornare i driver AMD Adrenalin. Altrimenti scendere al 7B |
| R6 | **Timeout calibrati sul fisso** bloccano codice legittimo sul Mac (falsi positivi in demo) | A | M | Tarare `timeout_esecuzione_s` **sul Mac** (fase D). Lasciare margine di 2-3 volte |
| R7 | **Il Mac va in swap** e la demo rallenta o si blocca | M | M | Chiudere Chrome, Antigravity, CleanMyMac prima della demo; riavviare il Mac; caricatore collegato |
| R8 | **Disco del Mac pieno** (15 GB liberi) | M | M | Liberare spazio prima; controllare `docker system df`. L'immagine `ollama/ollama` pesa alcuni GB **[DA VERIFICARE dopo il pull]** |
| R9 | **Docker Desktop si aggiorna** e smette di funzionare su macOS 13 | B | A | Aggiornamenti automatici già disattivati (`autoDownloadUpdates: false`). Rifiutare ogni proposta di aggiornamento |
| R10 | **Il giudice LLM sbaglia** (falso negativo: lascia passare codice pericoloso) | A | M | È atteso e va misurato, non nascosto. Metodo ibrido: le regole fisse decidono i casi certi, e comunque la sandbox contiene. Diventa un risultato del report |
| R11 | **Il modello piccolo scrive codice d'attacco che non funziona**, e sembra che la sandbox non abbia fatto nulla | A | M | Per i test di attacco usare anche codice **malevolo scritto a mano** (modalità "codice diretto"): il threat model dice che l'attaccante controlla tutto il codice |
| R12 | **Fork bomb senza limiti** blocca la VM di Docker | M | A | Mai lanciarla senza `--pids-limit`. Per il confronto "senza sandbox" usare un limite alto ma presente, e dichiararlo nel report |
| R13 | **File con fine riga Windows (CRLF)** rompono script e Dockerfile in Linux | A | M | `.gitattributes` con `* text=auto eol=lf` dal primo commit |
| R14 | **Versioni diverse** tra fisso e Mac (Ollama, immagini, librerie) danno risultati diversi | M | M | Versioni fissate; `ollama --version` uguale sul fisso e nel container del Mac; `requirements.txt` da `pip freeze` |
| R15 | **Prima autenticazione Tailscale** sul Mac richiede un browser e non va in università | M | M | Fare login a casa, almeno una volta |
| R16 | Qualcuno raggiunge Ollama dal Wi-Fi di casa o da internet | B | A | Regola firewall solo per Tailscale (F5), nessun port forwarding sul router |
| R17 | La chiave/account Tailscale viene compromessa | B | A | Autenticazione a due fattori sull'account usato per il login (Google/GitHub); ACL che ammettono solo il Mac verso il fisso |
| R18 | Il codice malevolo di test **esce dal container** e tocca il Mac | B | A | Fisso come ambiente principale dei test d'attacco. File finti (`id_rsa`, chiave API) creati apposta, mai quelli veri. Mac non usato per sperimentare exploit di escape |
| R19 | **Il repository è pubblico** e contiene qualcosa di sensibile | B | M | Niente `.env`, indirizzi Tailscale o chiavi nel repo. Controllare con `git status` prima di ogni commit |
| R20 | **Il tempo non basta** | M | A | Fasi con criteri di uscita (sezione 5); il nucleo minimo è M2+M3+M5, il resto è bonus |

### 4.2 La scala del piano B (cosa fare durante l'esame)

```
1. REMOTO   Mac → Tailscale → fisso (14B/7B, GPU)       risposta in pochi secondi
      │ non risponde entro 120 s, o errore di rete
      ▼
2. LOCALE   Mac → Ollama in Docker (3B, CPU)             risposta in ~20-40 s (stima)
      │ troppo lento o memoria esaurita
      ▼
3. REPLAY   nessun LLM: si rilegge il codice già generato e salvato in results/replay/
            e si mostra comunque broker + sandbox dal vivo
      │
      ▼
4. VIDEO    registrazione dello schermo di una demo completa riuscita
```

In ogni modalità la **sandbox gira dal vivo sul Mac**. È la parte che conta davvero.
Il programma deve scrivere a schermo con chiarezza quale profilo sta usando, così non
sembra di nascondere un ripiego.

---

## 5. Roadmap per fasi (con criteri di uscita)

**Data dell'esame:** `____ / ____ / ______` (da compilare)
Pianifica all'indietro: congela il codice **7 giorni** prima, prova in università **2-3
giorni** prima, prova generale **il giorno prima**.

| Fase | Dove | Obiettivo | Criterio di uscita |
|---|---|---|---|
| **A** | FISSO | Ambiente pronto (Ollama GPU, Docker, Python, Git, Tailscale) | `ollama ps` dice `100% GPU`; `docker run hello-world` va; limiti WSL attivi |
| **B** | MAC | Ambiente pronto (Docker 4.43, Tailscale, repo, riserva 3B) | `docker run python:3.12-slim` va; Tailscale connesso; 3B risponde |
| **C** | FISSO | Sviluppo: guida-operativa Fasi 1-7 (test, sandbox, broker, auditor, benchmark) | 20 casi di test scritti; benchmark produce la tabella |
| **M1** | MAC+FISSO | **Collegamento Mac → fisso funziona**, provato con il Mac sull'hotspot del telefono (simula un'altra rete) | `curl` dal Mac risponde; `demo.py --profilo remoto` completa un giro |
| **D** | MAC | Validazione: stessi test sul Mac, taratura timeout, misure di latenza | Tutti i legittimi passano, tutti gli attacchi contenuti, timeout tarati |
| **M6** | MAC in uni | Prova in università, sulla rete vera | Collegamento al fisso OK, oppure fallback provato |
| **E** | MAC | Prova generale: demo completa con cronometro, piano B incluso | Demo di 10 minuti senza interventi manuali imprevisti |

Il nucleo minimo per il progetto è: sandbox hardened (C) + broker con regole (C) +
benchmark (C) + validazione sul Mac (D). Giudice LLM, gVisor e il resto sono extra.

---

## 6. FASE A — Preparare il FISSO (Windows)

Tutti i comandi sono in **PowerShell** (aprila come amministratore dove indicato).

### F1. Verifiche preliminari
```powershell
winver                                   # versione di Windows
Get-CimInstance Win32_VideoController | Select-Object Name, DriverVersion
Get-PSDrive C                            # spazio libero
wsl --status                             # WSL2 presente?
```
- Servono Windows 10 22H2 / 11 aggiornato, WSL2 e la virtualizzazione attiva nel BIOS
  (SVM Mode su AMD).
- Aggiorna il driver grafico dal sito AMD (**Adrenalin**, ultima versione stabile).
- Spazio libero consigliato: almeno 40 GB (modelli + immagini Docker).

### F2. Installare i programmi
```powershell
winget install Git.Git
winget install Python.Python.3.13        # stessa minor del Mac (3.13)
winget install Docker.DockerDesktop
winget install Ollama.Ollama
winget install Tailscale.Tailscale
```
Se `winget` non c'è, scarica gli installer dai siti ufficiali. Riavvia se richiesto.

### F3. Ollama nativo con GPU
1. Avvia Ollama (icona nella barra).
2. Scarica i modelli, **sempre con il tag**:
   ```powershell
   ollama pull qwen2.5-coder:3b      # ~1,9 GB   - Mac-equivalente
   ollama pull qwen2.5-coder:7b      # ~4,7 GB   - giudice / alternativa
   ollama pull qwen2.5-coder:14b     # ~9 GB     - generatore per la demo
   ollama --version                  # ANNOTARE: serve per l'immagine del Mac (B5)
   ```
3. Verifica che usi la GPU:
   ```powershell
   ollama run qwen2.5-coder:14b "scrivi una funzione python che somma due numeri"
   ollama ps                         # colonna PROCESSOR: deve dire "100% GPU"
   ```
4. Se dice `100% CPU`: aggiorna il driver Adrenalin, riavvia Ollama, riprova.
   Se la GPU resta non utilizzata, usa il 7B (sul Ryzen è comunque accettabile).

**Compatibilità della scheda — [DA VERIFICARE con `ollama ps`]:** la documentazione di
Ollama elenca la RX 7800 XT tra le schede AMD supportate su Windows, ma le pagine che ho
consultato riportano versioni diverse della libreria ROCm richiesta. Il test pratico sopra
è l'unica prova affidabile.

**Memoria video:** 14B (~9 GB) + 7B (~4,7 GB) = ~14 GB, quasi tutti i 16 GB. Con la cache
del contesto e Windows che usa un po' di VRAM, potrebbe non starci tutto. Controllare con
`ollama ps` mentre entrambi sono caricati. Se non entrano, usare il 7B per tutti e due i
ruoli, oppure 14B come generatore e 3B come giudice.

### F4. Docker Desktop con gli stessi limiti del Mac
Crea il file `C:\Users\<tuo-utente>\.wslconfig` con:
```ini
[wsl2]
memory=4GB
processors=4
swap=1GB
```
Poi:
```powershell
wsl --shutdown
# riavvia Docker Desktop
docker info | Select-String "Total Memory","CPUs"     # devono dire ~4 GiB e 4
docker run --rm hello-world
docker run --rm python:3.12-slim python -c "print('ok')"
```
Il limite riguarda solo la VM di Docker (quindi la sandbox). Ollama gira in nativo e non
ne è toccato. Così se un test non sta in 4 GB lo scopri subito sul fisso.

Imposta anche, in Docker Desktop → Settings → General: **disattiva** "Start Docker Desktop
when you sign in" solo se non ti serve; per la sandbox basta avviarlo a mano.

### F5. Esporre Ollama SOLO via Tailscale e tenere il fisso acceso

**1. Tailscale**
- Apri Tailscale, accedi con un account protetto da 2FA (Google/GitHub).
- Annota l'IP del fisso (`100.x.y.z`): `tailscale ip -4`.
- Tailscale su Windows gira come servizio e parte all'avvio anche prima del login.

**2. Ollama in ascolto sulla rete**
```powershell
setx OLLAMA_HOST "0.0.0.0:11434"
setx OLLAMA_KEEP_ALIVE "-1"           # modello sempre in memoria
setx OLLAMA_CONTEXT_LENGTH "4096"
```
Poi chiudi Ollama dall'icona nella barra e riavvialo. (Alcune versioni hanno
l'opzione "Expose Ollama to the network" nelle impostazioni: fa la stessa cosa.)

**3. Firewall: ammettere solo la rete Tailscale** (PowerShell come amministratore)
```powershell
New-NetFirewallRule -DisplayName "Ollama solo Tailscale" -Direction Inbound `
  -Protocol TCP -LocalPort 11434 -RemoteAddress 100.64.0.0/10 -Action Allow -Profile Any

# controllare che NON esista una regola più larga:
Get-NetFirewallRule -Direction Inbound -Action Allow |
  Where-Object { $_.DisplayName -like "*ollama*" } |
  Format-Table DisplayName, Enabled, Profile
```
Se esiste la regola automatica "ollama.exe" (creata dalla finestra di Windows al primo
avvio) che ammette reti Private/Pubbliche, **disabilitala**: altrimenti anche il Wi-Fi di
casa potrebbe raggiungere Ollama.
`100.64.0.0/10` è l'intervallo che Tailscale usa per i suoi indirizzi.

**4. Sul router: nessun port forwarding.** Non serve e non deve esserci.

**5. Tenere il fisso acceso**
```powershell
powercfg /change standby-timeout-ac 0         # mai sospensione
powercfg /change hibernate-timeout-ac 0
powercfg /hibernate off
powercfg /change monitor-timeout-ac 15        # il monitor può spegnersi
```
- **Windows Update:** Impostazioni → Windows Update → *Sospendi aggiornamenti* (fino a 5
  settimane), e imposta le *Ore di attività*. Un riavvio forzato durante l'esame è il
  rischio R1.
- **BIOS/UEFI:** abilita "Restore on AC Power Loss" / "AC Back = Power On", così se salta
  la corrente il PC riparte da solo.
- **Raffreddamento:** l'LLM sulla GPU è un carico lungo. Controlla le temperature e lascia il
  PC in un posto arieggiato.
- **Opzionale ma utile:** una presa smart (Wi-Fi) per spegnere e riaccendere il fisso da
  remoto, oppure una persona di fiducia a casa che possa riavviarlo.

### F6. Provare tutto, compreso il riavvio
```powershell
curl http://localhost:11434/api/tags          # lista modelli in locale
```
Poi, **dal Mac** (fase M1), `curl http://100.x.y.z:11434/api/tags`.

**Test di riavvio (importante, rischio R4):** riavvia il fisso, **non fare il login**, e
dal Mac (o dal telefono con Tailscale) prova a raggiungere `:11434`.
- Risponde → ottimo.
- Non risponde → Ollama parte solo al login. Soluzioni: login automatico di Windows
  (`netplwiz`, con il compromesso che il PC di casa non chiede password all'avvio), oppure
  non riavviare mai il fisso nei giorni dell'esame.

### F7. Repo e ambiente Python
```powershell
git config --global core.autocrlf input
git clone https://github.com/salvlea/Computer-security.git
cd Computer-security
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```
Il primo commit del progetto deve contenere `.gitattributes` (`* text=auto eol=lf`) e
`.gitignore` (`.venv/`, `__pycache__/`, `.env`).

---

## 7. FASE B — Preparare il MAC

### B1. Liberare spazio e memoria
```bash
df -h /
docker system df                     # quanto occupano immagini e cache
```
Libera disco (cache, download, vecchi progetti) fino ad avere almeno **10 GB** di margine
oltre a ciò che serve. Non cancellare `Docker.raw` a mano.

### B2. Repo e Python
```bash
cd ~/Desktop/progetti/Computer-security
git pull
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### B3. Docker (NON aggiornare)
```bash
open -a Docker                       # aspettare che l'icona sia ferma
docker --version                     # deve restare 28.3.0 / Desktop 4.43.0
docker pull python:3.12-slim
docker build -f docker/Dockerfile.sandbox -t sandbox-py:1 .
docker run --rm --network none sandbox-py:1 python -c "print('ok')"
```
Se compare una finestra "Update available": **chiudila / Skip**.

### B4. Tailscale sul Mac
- Scarica il pacchetto "Standalone" dal sito Tailscale (o la versione App Store).
- La documentazione di Tailscale indica **macOS 12 Monterey come minimo**, quindi il 13
  dovrebbe andare, ma non l'ho trovato scritto esplicitamente per Ventura
  **[DA VERIFICARE installandolo]**. Se non funziona: WireGuard, oppure un tunnel SSH verso
  il fisso.
- Fai **il login a casa**, con l'account protetto da 2FA (rischio R15).
- Nelle ACL di Tailscale (console web) consenti al Mac di raggiungere il fisso sulla porta
  11434 e niente altro.

```bash
tailscale status
curl http://100.x.y.z:11434/api/tags       # deve elencare i modelli del fisso
```
Metti l'indirizzo in un file `.env` (non committato):
```
OLLAMA_HOST_REMOTO=http://100.x.y.z:11434
```
Preferisci l'indirizzo numerico al nome: se il DNS dell'università fa problemi, l'IP
funziona comunque.

### B5. Riserva locale: Ollama 3B in Docker
Usa **la stessa versione di Ollama del fisso** (annotata in F3):
```yaml
# docker/compose.fallback.yaml
services:
  ollama:
    image: ollama/ollama:<versione-del-fisso>
    ports: ["127.0.0.1:11434:11434"]
    environment:
      OLLAMA_KEEP_ALIVE: "-1"
      OLLAMA_CONTEXT_LENGTH: "2048"
    volumes: ["ollama:/root/.ollama"]
    mem_limit: 3g
volumes:
  ollama:
```
```bash
docker compose -f docker/compose.fallback.yaml up -d
docker exec ollama ollama pull qwen2.5-coder:3b
curl http://127.0.0.1:11434/api/tags
docker system df                     # R8: quanto spazio hanno preso immagine e modello?
```
Il container lascia 1 GB di margine sui 4 GB della VM per la sandbox. Se l'immagine di
Ollama è troppo grande per il tuo disco, **salta questa riserva** e fai affidamento su
`replay` + hotspot (decisione da prendere dopo aver letto `docker system df`).

> Nota: questa riserva è per la demo. Se la sandbox e Ollama sono entrambi in esecuzione, la
> memoria è giusta-giusta. Non lanciarli insieme a Chrome aperto.

### B6. Controllo "è tutto pronto?"
`sandbox/preflight.py` (da scrivere nella fase C) stampa una tabella verde/rossa con:
Docker attivo · immagine `sandbox-py` presente · Tailscale connesso · fisso raggiungibile ·
modelli presenti sul fisso · Ollama locale attivo · file `replay` presenti · spazio su disco
· memoria libera. Va lanciato prima di ogni prova e **5 minuti prima dell'esame**.

---

## 8. FASE C — Sviluppo (sul FISSO)

Segui le fasi di `capire-progetto/guida-operativa.md`, con queste precisazioni.

| Fase guida | Cosa fare sul fisso | Nota |
|---|---|---|
| 1. Scope e threat model | Scrivere la pagina definitiva | Cita OWASP LLM Top 10 e MITRE ATLAS |
| 2. Batteria di test | `legittimi.yaml` e `malevoli.yaml`, 15-20 casi | **Prima** del codice |
| 3. Docker hardened | `runner_docker.py`, un flag alla volta | Misura l'effetto di ogni flag |
| 4. Broker + policy | Prima regole fisse, poi giudice LLM, poi ibrido | Vedi sotto |
| 5. Auditor | Una riga di log per esecuzione | Serve a demo e report |
| 6. Benchmark | Con e senza sandbox, su **più profili** | Vedi sotto |
| 7. Report | Struttura della sezione 8 dei consigli | Aggiungi la sezione sul setup distribuito |

**Tre modalità di ingresso nella demo (`demo.py`):**
1. `--prompt "..."`: percorso completo, con LLM generatore e giudice.
2. `--codice file.py`: codice malevolo scritto a mano, senza LLM, istantaneo e prevedibile.
3. `--replay`: usa le risposte salvate in `results/replay/`.

Ogni generazione dell'LLM viene salvata automaticamente in `results/replay/`.

**Benchmark su più profili (ottimo risultato per il report):**

| Profilo | Generatore | Giudice | Scopo |
|---|---|---|---|
| Mac-equivalente | 3B | 3B | Cosa succederebbe con l'hardware del Mac |
| Demo | 14B | 7B | Cosa mostrerai all'esame |
| Solo regole | nessuno | nessuno | Quanto fanno le regole fisse da sole |

Misura per ciascuno: attacchi contenuti, falsi positivi, falsi negativi del giudice. La
differenza tra i profili dimostra quanto pesa la qualità del modello giudice, e che la
sandbox protegge **anche quando il giudice sbaglia**.

**Il confronto "senza sandbox" non va fatto sul sistema reale.** Usa un container senza le
protezioni, con file finti (`id_rsa` e chiave API inventate) e un ricevitore in ascolto su
localhost per mostrare l'esfiltrazione. Per la fork bomb usa comunque un limite di processi
alto (rischio R12) e scrivilo nel report.

---

## 9. FASE D — Validazione sul MAC

1. `git pull` sul Mac, poi riavvia il Mac e chiudi le altre app.
2. Esegui l'intera batteria con il profilo `locale` e con il profilo `remoto`.
3. **Misura sul Mac:**
   - tempo di avvio del container (da comando a prima riga di output);
   - tempo di esecuzione dei casi legittimi (tipicamente quelli con `pandas`, se li usi);
   - memoria libera durante l'esecuzione (Monitoraggio Attività);
   - tempo di risposta dell'LLM per ciascun profilo.
4. **Tara i timeout:** `timeout_esecuzione_s` = 2-3 volte il caso legittimo più lento
   misurato sul Mac.
5. Confronta i risultati con quelli del fisso: devono coincidere (stesso insieme di attacchi
   contenuti). Se non coincidono, è un bug di configurazione da risolvere prima di tutto.
6. Registra le misure di overhead del Mac: sono i **numeri da mettere nel report**.

---

## 10. Test in università (M6)

Da fare 2-3 giorni prima, **nello stesso posto e sulla stessa rete** dell'esame, se
possibile.
- [ ] Il Mac si collega al Wi-Fi dell'università.
- [ ] `tailscale status` mostra il fisso.
- [ ] `curl http://100.x.y.z:11434/api/tags` risponde.
- [ ] `demo.py --profilo remoto` completa un giro intero.
- [ ] Se la rete blocca Tailscale: prova con l'**hotspot del telefono**.
- [ ] Prova il passaggio automatico a `locale` e a `replay` (stacca la rete a metà).
- [ ] Misura la latenza reale del collegamento.
- [ ] Verifica che il fisso sia rimasto acceso e che Ollama abbia ancora il modello caricato.

---

## 11. Checklist del giorno prima e di 5 minuti prima

**Il giorno prima**
- [ ] Fisso: acceso, Windows Update in pausa, Ollama risponde, `ollama ps` ok.
- [ ] Fisso: temperature normali, nessun programma pesante in background.
- [ ] Mac: carico al 100%, spazio su disco sufficiente, Docker 4.43.0.
- [ ] Repo allineato su entrambe le macchine (`git status` pulito).
- [ ] `preflight.py` tutto verde.
- [ ] Registrazione video della demo completa fatta e salvata anche sul telefono.
- [ ] Immagini Docker e modello 3B già scaricati (la demo non deve scaricare nulla).

**5 minuti prima**
- [ ] Mac riavviato, solo Docker, Terminale e (se serve) il browser aperti.
- [ ] Caricatore collegato.
- [ ] Docker avviato; `preflight.py` tutto verde.
- [ ] Una richiesta di prova all'LLM già fatta (modello caricato in memoria).
- [ ] Notifiche e aggiornamenti del Mac disattivati.
- [ ] Telefono pronto come hotspot.

---

## 12. Cosa NON fare

- Non aggiornare Docker Desktop o macOS sul Mac nei giorni prima dell'esame.
- Non lasciare `qwen2.5-coder` senza tag (scarica il 7B).
- Non aprire la porta 11434 sul router e non usare `0.0.0.0` senza la regola firewall.
- Non dare accesso a `docker.sock` a un container.
- Non lanciare la fork bomb senza `--pids-limit`.
- Non usare file veri (chiavi SSH, `.env`) come esca nei test d'attacco: solo file finti.
- Non scrivere nel report i tempi del fisso come se fossero quelli del Mac.
- Non presentare il giudice LLM come affidabile al 100%: è il limite che devi mostrare.
- Non fare modifiche di configurazione nelle ultime 24 ore prima dell'esame.

---

## 13. Risoluzione dei problemi

| Sintomo | Causa probabile | Cosa fare |
|---|---|---|
| `Cannot connect to the Docker daemon` (Mac) | Docker Desktop non avviato | `open -a Docker` e attendere |
| `curl` al fisso va in timeout | Fisso spento/in sospensione, Tailscale giù, firewall | `tailscale status` su entrambi; controllare la regola firewall |
| `curl` risponde ma la generazione non parte | Modello non caricato / VRAM piena | Sul fisso `ollama ps`; riavviare Ollama |
| Il 14B è lentissimo | Gira sul processore | `ollama ps` → se non è GPU, aggiornare driver, usare il 7B |
| Il codice legittimo viene ucciso dal timeout | Timeout calibrato sul fisso | Rialzare `timeout_esecuzione_s` (fase D) |
| `exec format error` / script che non parte nel container | Fine riga CRLF | `.gitattributes` e rifare il checkout |
| Il Mac diventa lentissimo durante la demo | Swap | Chiudere app, riavviare, usare il profilo `replay` |
| Errore OOM nel container | `--memory` troppo basso o VM Docker piena | `docker stats`; alzare il limite del singolo container, non della VM |
| Tailscale non si autentica in università | Portale captive / rete filtrata | Login fatto a casa; usare l'hotspot |
| Ollama sul fisso non riparte dopo il riavvio | Parte al login | Vedi F6 / rischio R4 |

---

## 14. Come questo setup entra nel report

Il setup distribuito non è solo logistica: è materiale per il report.
- **Architettura**: LLM remoto e sandbox locale mostrano il disaccoppiamento
  (`llm/client.py` è l'unico punto che cambia).
- **Sicurezza dell'infrastruttura**: Ollama non ha autenticazione, quindi viene esposto solo
  via rete privata cifrata con firewall e ACL. È un esempio concreto di riduzione della
  superficie d'attacco.
- **Il broker non ha accesso a `docker.sock` dall'interno di un container**, e i container
  non hanno rete: il codice generato non può raggiungere l'LLM.
- **Docker su Mac gira già in una VM Linux**, quindi c'è un confine di virtualizzazione tra
  container e macOS.
- **Benchmark su più modelli**: mostra quanto cambia l'affidabilità del giudice con la
  dimensione del modello, e che la sandbox contiene comunque.
- **Limiti onesti**: i container condividono il kernel della VM; gVisor/microVM sono
  trattati come evoluzione (gVisor non gira su macOS, ma si può provare sul fisso con
  Linux/WSL2 per una misura in più, **opzionale**).

---

## 15. Fonti consultate

- [Ollama — requisiti macOS](https://docs.ollama.com/macos) (richiede macOS 14 Sonoma)
- [Ollama — GPU supportate](https://docs.ollama.com/gpu.md) (RX 7800 XT elencata per Windows)
- [Tailscale — installazione su macOS](https://tailscale.com/docs/install/mac) (minimo macOS 12)
- [Docker Desktop — installazione su Mac](https://docs.docker.com/desktop/setup/install/mac-install.md)
- [Shipyard — Docker su macOS vecchi](https://shipyard.build/blog/installing-docker-on-older-mac/)
