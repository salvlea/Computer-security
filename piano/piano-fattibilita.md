# Piano di fattibilità — Sandbox LLM

> Scritto il **3/11/2026** sul fisso. Esame: **giovedì 3/12/2026** → restano **30 giorni**.
> Risponde a una domanda sola: *si riesce a fare il progetto in tempo, e a quali condizioni?*
> Il "come" tecnico resta in `piano-operativo.md`; qui ci sono tempi, priorità, punti di
> controllo e cosa tagliare se si va in ritardo.

---

## 1. Verdetto

**Sì, è fattibile**, a tre condizioni:

1. **Tempo disponibile:** circa **25 ore a settimana** fino al 26/11 (es. 3 h nei feriali,
   5 h nel weekend). Con meno tempo resta fattibile il **nucleo**, non gli extra (sezione 6).
2. **Scope ridotto sulla parte "agente" (B):** la parte A (esecuzione sicura del codice
   generato) si fa completa; la parte B (azioni di un agente) si fa con **2-3 azioni** che
   passano dallo stesso broker, più la parte di progettazione nel report. Un agente completo
   non sta in 30 giorni insieme a tutto il resto.
3. **Congelamento del codice il 26/11:** dopo quella data solo correzioni, report e prove.

Stima totale: **~100 ore** (forbice 80-125). Il nucleo da solo: **~75 ore**.

---

## 2. Cosa è già verificato oggi (riduce il rischio)

| Fatto | Esito | Effetto sulla fattibilità |
|---|---|---|
| GPU del fisso | RX 7800 XT confermata | Il 14B/7B in locale è realistico (resta da provare `ollama ps`) |
| Spazio su disco del fisso | 707 GB liberi | Nessun vincolo |
| Virtualizzazione (SVM) | Attiva, WSL2 presente | Docker funziona senza toccare il BIOS |
| Docker Desktop sul fisso | Già installato | Fase A più corta |
| Git | Già installato, repo già clonato | Nessun lavoro |
| Rete del fisso | Cavo Ethernet 1 Gbps | Collegamento remoto più stabile |
| Repo GitHub | **Pubblico** | Attenzione a `.env` e IP (già previsto nel piano) |

**Ancora da verificare (sono i veri punti incerti):**

| Incognita | Quando si scopre | Se va male |
|---|---|---|
| Ollama usa davvero la GPU (non l'integrata del Ryzen) | Fase A, 4/11 | 7B al posto del 14B |
| Tailscale funziona su macOS 13 | Fase B, entro 8/11 | Niente profilo remoto: demo con 3B locale + replay |
| Il Mac (8 GB, 15 GB di disco) regge Ollama 3B in Docker | Fase B, entro 8/11 | Niente 3B locale: demo con remoto + replay |
| La rete dell'università lascia passare Tailscale | Prova in uni, 30/11 | Hotspot del telefono |
| Tempi della sandbox sul Mac | Fase D, 23-25/11 | Si alzano i timeout |

Nessuna di queste incognite può far fallire il progetto: ognuna ha un ripiego già previsto,
e la sandbox **gira comunque dal vivo sul Mac** in tutti i casi.

---

## 3. Priorità (cosa è obbligatorio e cosa no)

**MUST — senza questi il progetto non regge**

| Componente | Ore stimate | Perché è obbligatorio |
|---|---|---|
| Ambiente fisso (Fase A) | 4 | Base di tutto |
| Ambiente Mac (Fase B, senza 3B locale) | 3 | La demo è sul Mac |
| Threat model (1 pagina) | 3 | Lo chiede la struttura del report |
| Batteria di test (15-20 casi) | 5 | È il bersaglio del codice e del benchmark |
| `runner_docker.py` hardened + Dockerfile | 10 | È la sandbox |
| Broker con regole fisse | 6 | È il "decisore" |
| Giudice LLM semplice (safe/unsafe) | 3 | È l'idea esplicita del professore (LLM che ne invoca un altro) |
| Auditor (una riga di log per esecuzione) | 2 | Serve alla demo e al report |
| Benchmark con/senza sandbox | 6 | La tabella è il risultato principale |
| `demo.py` con modalità `--codice` e `--replay` | 4 | Garantisce che la demo vada anche senza LLM |
| Validazione sul Mac (Fase D) | 5 | I numeri del report si misurano lì |
| Stato dell'arte (studio + prova di `llm-sandbox`) | 9 | È la prima richiesta del professore |
| Report | 15 | Deliverable |
| Prova in uni + prova generale + video | 4 | Assicura la demo |
| **Totale MUST** | **~79** | |

> Nota: nel piano operativo il giudice LLM è tra gli "extra". Qui lo sposto nel MUST nella
> versione **semplice** (3 ore), perché è il cuore della richiesta del professore. La
> misura approfondita della sua affidabilità resta SHOULD.

**SHOULD — alzano molto il voto, ma si possono sacrificare**

| Componente | Ore |
|---|---|
| Profilo remoto via Tailscale (LLM sul fisso durante la demo) | 3 |
| Policy ibrida (regole + giudice) e misura dei falsi negativi del giudice | 4 |
| Benchmark su più profili (3B/3B, 14B/7B, solo regole) | 4 |
| Parte B ridotta: 2-3 azioni d'agente passate dal broker | 5 |
| Riserva Ollama 3B in Docker sul Mac | 2 |
| `preflight.py` | 2 |
| Slide della presentazione | 5 |

**COULD — solo se avanza tempo**

- gVisor su WSL2 per una misura in più.
- RestrictedPython e un suo bypass (per mostrare perché non basta).

**WON'T — dichiarati nel report come sviluppi futuri**

- MicroVM (Firecracker), E2B, Kubernetes, interfaccia web.

---

## 4. Calendario

Date reali. Le settimane vanno da martedì a domenica per allinearsi a oggi.

| Periodo | Lavoro | Uscita attesa |
|---|---|---|
| **mar 3 – mer 4/11** | Fase A sul fisso: Ollama, modelli, `.wslconfig`, Tailscale, Python | `ollama ps` → GPU; `docker run` ok |
| **gio 5 – ven 6/11** | `.gitignore`/`.gitattributes`, threat model, batteria di test | `tests/prompts/*.yaml` scritti |
| **sab 7 – dom 8/11** | Fase B sul Mac + collegamento M1. Inizio lettura stato dell'arte | **Controllo G1** |
| **lun 9 – mer 11/11** | `runner_docker.py`: baseline, poi un flag di hardening alla volta | Attacchi scritti a mano contenuti |
| **gio 12 – dom 15/11** | Broker con regole, auditor, `demo.py --codice` | **Controllo G2** |
| **lun 16 – mer 18/11** | Giudice LLM, policy ibrida, profili e `--replay` | Giro completo prompt → LLM → broker → sandbox |
| **gio 19 – dom 22/11** | Benchmark; prima prova completa sul Mac. Bozza sezioni stato dell'arte e principi | **Controllo G3** |
| **lun 23 – gio 26/11** | Fase D sul Mac (timeout, misure), parte B ridotta se in tempo, `preflight.py` | **Controllo G4: codice congelato** |
| **ven 27 – dom 29/11** | Report completo, slide, video della demo | Report in bozza finale |
| **lun 30/11** | **Prova in università** (rete vera) | Remoto ok, oppure ripiego provato |
| **mar 1/12** | Correzioni dalla prova, rilettura report | |
| **mer 2/12** | Prova generale cronometrata; checklist del giorno prima | Nessuna modifica di configurazione dopo |
| **gio 3/12** | **Esame** | |

Il report non si scrive tutto alla fine: le sezioni "stato dell'arte" e "principi" si
scrivono man mano dalla settimana 1, perché non dipendono dal codice.

---

## 5. Punti di controllo (go / no-go)

A ogni controllo si decide **subito** se proseguire come da piano o passare al ripiego.
Rimandare la decisione è la causa più comune di ritardo.

| Controllo | Data | Domanda | Se NO |
|---|---|---|---|
| **G1** | dom 8/11 | Fisso pronto con GPU? Mac con Docker ok? `curl` dal Mac al fisso risponde? | GPU no → 7B. Tailscale no → si toglie il profilo remoto: l'LLM gira sul fisso **prima** e si salvano le risposte (replay); al Mac resta il 3B locale se ci sta |
| **G2** | dom 15/11 | La sandbox hardened contiene tutti gli attacchi scritti a mano? Il broker con regole funziona? | Si taglia la parte B e il benchmark multi-profilo; si recupera tempo per sandbox e broker |
| **G3** | dom 22/11 | Giro completo funzionante sul Mac? Il benchmark produce la tabella? | Benchmark su un solo profilo; giudice solo nella versione semplice |
| **G4** | gio 26/11 | Tutti i legittimi passano e tutti gli attacchi sono contenuti sul Mac? | Si rimuove dalla demo ciò che non è stabile; si mostra con `--codice` / `--replay` |
| **G5** | lun 30/11 | In università il collegamento al fisso funziona? | Hotspot; se neanche così → demo con 3B locale o replay, dichiarandolo |

---

## 6. Cosa tagliare se il tempo non basta (in quest'ordine)

1. gVisor e RestrictedPython (COULD).
2. Parte B oltre la progettazione: resta solo nel report.
3. Riserva 3B locale sul Mac: si fa affidamento su remoto + replay + video.
4. Benchmark multi-profilo → un solo profilo (il 14B/7B o il 3B).
5. `preflight.py` → checklist scritta a mano.
6. Profilo remoto: si usa solo replay sul Mac, con le risposte generate sul fisso.

**Non si taglia mai:** sandbox hardened, batteria di test, broker con regole, giudice LLM
semplice, validazione sul Mac, stato dell'arte, report.

**Scenari di tempo:**

| Ore a settimana | Cosa si riesce a fare |
|---|---|
| ~25 o più | MUST + quasi tutti gli SHOULD |
| ~18 | MUST + remoto + benchmark multi-profilo; niente parte B pratica, niente 3B locale |
| ~12 o meno | Solo MUST, con demo in modalità `--codice` / `--replay`. Ancora un progetto completo, ma senza effetto "LLM dal vivo" |

---

## 7. Rischi per la fattibilità (non tecnici)

| Rischio | Contromisura |
|---|---|
| Altri esami o impegni a novembre riducono le ore | Decidere ora quanto tempo c'è davvero e scegliere lo scenario (sezione 6) |
| Curva di apprendimento di Docker SDK / Ollama | Fase 3 con un flag alla volta; usare `llm-sandbox` come esempio da leggere, non da copiare |
| Perfezionismo sul giudice LLM (prompt infiniti) | Massimo 3 ore sulla versione semplice; i suoi errori sono un **risultato**, non un difetto da eliminare |
| Report lasciato all'ultima settimana | Scrivere stato dell'arte e principi già dalle settimane 1-2 |
| Lavoro sul Mac concentrato alla fine | Prima prova completa sul Mac già il 21-22/11, non il 25 |

---

## 8. Ipotesi da confermare

Il piano regge se queste ipotesi sono vere. Se una è sbagliata, il calendario va rivisto.

1. Si possono dedicare **circa 25 ore a settimana** al progetto.
2. Il Mac è disponibile a casa nel weekend 7-8/11 per la Fase B.
3. All'esame si consegna un **report** e si fa una **demo** (eventualmente con slide). Non
   c'è una scadenza di consegna del report **prima** del 3/12.
4. La prova in università è possibile lunedì 30/11 (in alternativa martedì 1/12).
