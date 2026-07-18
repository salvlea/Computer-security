# Progetto: Sandbox per LLM — Guida alla strutturazione (livello 30 e lode)

> Documento **teorico** di pianificazione. Nessun codice qui: serve a decidere *cosa*
> costruire, *perché* e *come* raccontarlo.
>
> **Nota sullo scope**: il progetto lo svolgo **da solo**, concentrandomi
> **esclusivamente sulla sandbox LLM**. Il tema "guardrail" è stato assegnato a un altro
> studente (Moscuza) solo perché eravamo insieme a lezione: qui viene citato unicamente
> come *contesto concettuale* per chiarire la differenza rispetto alla sandbox — **non è
> un lavoro condiviso** e non fa parte del mio deliverable.

---

## 0. Cosa vuole davvero il professore (lettura tra le righe)

Il progetto è **individuale** e riguarda **solo la sandbox**. Il guardrail (assegnato a un
altro studente) compare in questo documento esclusivamente come termine di paragone, per
delimitare bene cosa la sandbox *è* e cosa *non è*.

Dalle sue parole si ricavano gli obiettivi di valutazione:

1. **Stato dell'arte**: dimostrare di aver studiato la letteratura e le librerie open
   source esistenti ("ho trovato queste librerie, le posso riusare / oppure non esistono
   e spiego perché").
2. **Principi di funzionamento**: capire *come* funziona una sandbox per LLM, non solo
   usarne una a scatola chiusa.
3. **Progettazione + prototipo**: progettare "sommariamente" un sistema e mostrarne una
   **demo** funzionante (anche minimale, ma reale).
4. **Consapevolezza dei limiti**: il progetto è *underspecified* di proposito. Chi prende
   il massimo è chi **definisce bene lo scope**, motiva le scelte e riconosce onestamente
   cosa la sandbox *non* può fare.

Frase chiave del prof: *"la sandbox è per l'esplosione interna, dalle viscere"*.
Traduzione: mentre il **guardrail filtra il contenuto** (input/output, un proxy testuale),
la **sandbox contiene le AZIONI** — ciò che l'LLM (o l'agente che lo usa) riesce
concretamente a *fare* sul sistema: eseguire codice, chiamare tool, leggere/scrivere file,
aprire connessioni di rete.

> **Guardrail = "cosa può dire"** (semantica del testo).
> **Sandbox = "cosa può fare"** (capacità ed effetti sul mondo).
> I due si incastrano: sono le due metà dello stesso problema di contenimento.

---

## 1. Definizione dello scope (la scelta che vale mezzo voto)

Il termine "sandbox LLM" è ambiguo. Esistono **tre interpretazioni** possibili; conviene
sceglierne una come *focus principale* e citare le altre come contesto.

| # | Interpretazione | Cosa si contiene | Rischio tipico |
|---|-----------------|------------------|----------------|
| A | **Sandbox di esecuzione codice** | Il codice che l'LLM *genera* e che viene eseguito (code interpreter, data analysis) | RCE, lettura file sensibili, esfiltrazione dati |
| B | **Sandbox di agente / tool-use** | Le *azioni* di un LLM-agente: chiamate a tool, API, filesystem, shell, browser | Confused deputy, azioni distruttive, catena di tool |
| C | **Sandbox del modello stesso** | L'inferenza del modello isolata dal resto del sistema (contenimento dell'inferenza) | Meno rilevante per un progetto studente |

**Raccomandazione**: focus su **A + B** (esecuzione di codice generato + azioni di
agente), perché sono i casi dove la sandbox è *tecnicamente distinta* dal guardrail ed è
dove esistono librerie open source vere da studiare. È anche il caso che genera la demo
più convincente.

### La demo suggerita dal prof, riletta
"Un bot ChatGPT che prima di darmi la risposta ne invoca un altro" → questo è il pattern
**LLM-as-a-judge / broker**: un componente di controllo intercetta la richiesta o l'azione
e la valida *prima* di lasciarla eseguire. Nel tuo caso (sandbox) il secondo LLM non serve
tanto a riscrivere il testo (quello è guardrail), quanto a **decidere se un'azione è
sicura da eseguire dentro l'ambiente isolato**.

---

## 2. Modello delle minacce (threat model) — la spina dorsale del progetto

Un progetto da lode parte **sempre** da un threat model esplicito. Definisci:

**Attore / origine della minaccia**
- Utente malevolo che fa *prompt injection* per indurre l'LLM a eseguire codice dannoso.
- *Indirect prompt injection*: dati esterni (pagine web, file, email) che contengono
  istruzioni nascoste raccolte dall'agente.
- L'LLM stesso che "sbaglia" e genera codice distruttivo senza intenzione malevola.

**Beni da proteggere (assets)**
- Il sistema host (filesystem, processi, credenziali, chiavi API).
- La rete interna (movimenti laterali, accesso a servizi privati/metadata cloud).
- Dati sensibili di altri utenti (multi-tenancy).
- Risorse (CPU/RAM/disco → denial of service, fork bomb, loop infiniti).

**Capacità dell'attaccante**
- Può controllare *interamente* il prompt e quindi il codice generato.
- Non può (idealmente) rompere l'isolamento del kernel/hypervisor.

**Cosa la sandbox NON difende** (dichiararlo è un punto a favore, non contro)
- Non impedisce che l'LLM *dica* cose sbagliate → quello è il guardrail.
- Non protegge da vulnerabilità del modello stesso (jailbreak semantico).
- Le sandbox a livello di linguaggio (es. RestrictedPython) sono notoriamente bypassabili.

Riferimenti da citare nel report:
- **OWASP Top 10 for LLM Applications** (in particolare *LLM01 Prompt Injection*,
  *Insecure Output Handling*, *Excessive Agency*, *Insecure Plugin/Tool Design*).
- **MITRE ATLAS** (tattiche e tecniche di attacco a sistemi ML/LLM).
- Letteratura su *prompt injection* e *confused deputy* negli agenti LLM.

---

## 3. Principi di funzionamento di una sandbox (il "come funziona")

Qui dimostri di conoscere i **meccanismi di isolamento**, dal più debole al più forte.
Ordina la trattazione come una scala di garanzie:

1. **Isolamento a livello di linguaggio** (debole)
   - Es. `RestrictedPython`, AST allow-list, rimozione di `__builtins__`, `import` bloccati.
   - *Perché è debole*: superfici di escape enormi (introspezione, `getattr`, sandbox
     escape noti). Utile spiegarlo proprio per mostrare *perché non basta*.

2. **Isolamento a livello di processo / OS** (medio)
   - `seccomp-bpf` (filtra le syscall), `AppArmor`/`SELinux`, Linux `namespaces` +
     `cgroups` (limiti CPU/RAM/PID/rete), utente non-root, filesystem read-only,
     `no-new-privileges`.
   - È il cuore di come funzionano i container.

3. **Container** (medio-alto)
   - **Docker / Podman**: isolamento pratico e riproducibile. Con hardening
     (drop capabilities, network `none`, rootless, quota) è un buon compromesso.
   - *Limite*: condividono il kernel dell'host → una vuln del kernel = escape.

4. **Sandbox del kernel / user-space kernel** (alto)
   - **gVisor** (Google): intercetta le syscall in user space, riduce la superficie del
     kernel host. Usato da OpenAI/altri per code interpreter.

5. **MicroVM / virtualizzazione** (molto alto)
   - **Firecracker** (AWS), **Kata Containers**, VM tradizionali: isolamento hardware,
     ogni sessione in una micro-macchina virtuale. È lo standard "serio" per eseguire
     codice non fidato di terzi.

6. **WebAssembly (WASM)** (alto, diverso approccio)
   - `Pyodide`, `wasmtime`, `wasmer`: sandbox by design, capability-based, niente accesso
     al sistema salvo esplicita concessione. Ottimo per codice deterministico e leggero.

**Assi di controllo trasversali** (validi per qualsiasi livello, da elencare come
"checklist di sandbox"):
- **Filesystem**: read-only, directory temporanea usa-e-getta, no accesso a `/etc`, home, chiavi.
- **Rete**: default *deny*, allow-list di host, blocco metadata cloud (169.254.169.254).
- **Risorse**: timeout di esecuzione, limiti CPU/RAM, max processi (anti fork-bomb).
- **Privilegi**: utente non privilegiato, capabilities minime.
- **Effimero**: ambiente distrutto e ricreato ad ogni esecuzione (no stato persistente).
- **Osservabilità**: logging di syscall/azioni per audit.

> Il messaggio da far passare: **"sandbox forte = isolamento + limiti risorse + rete
> negata + effimero + osservabile"**, e più si scende nella scala più le garanzie sono
> forti ma il costo (latenza, complessità) cresce. Il progetto sceglie un punto su questa
> scala e lo motiva.

---

## 4. Stato dell'arte: librerie e prodotti open source da studiare

Questa sezione risponde direttamente al prof ("ho trovato queste librerie... o non ne ho
trovate"). Da presentare come tabella comparativa nel report.

**Sandbox di esecuzione codice per LLM (il cuore del tuo tema)**
- **`llm-sandbox`** (vndee/llm-sandbox): esegue codice generato da LLM in Docker/Podman/
  Kubernetes. Diretto per il tuo caso A. Ottimo come base per la demo.
- **E2B** (`e2b-dev`): sandbox cloud per agenti AI basate su Firecracker microVM. Open source,
  SDK Python/JS. Rappresenta lo stato dell'arte "serio".
- **Piston** (engineer-man/piston): motore di esecuzione codice multi-linguaggio in container.
- **Judge0**: sistema open source di esecuzione codice (nato per giudici di programmazione,
  riusabile).
- **Pyodide / Jupyter kernel** in WASM o container per data analysis isolata.

**Infrastruttura di isolamento (i mattoni)**
- **gVisor**, **Firecracker**, **Kata Containers**, **Docker/Podman rootless**.
- **`nsjail`** (Google), **`bubblewrap`** (bwrap): sandbox leggere basate su namespaces.
- **`seccomp`**, **`landlock`** (LSM recente per restringere l'accesso al FS).

**Lato guardrail / agente (da citare come contesto, è il tema del compagno)**
- **NVIDIA NeMo Guardrails**, **Guardrails AI**, **LLM Guard** (protectai), **Rebuff**,
  **Llama Guard** (Meta). Servono a mostrare la *complementarità*: guardrail filtra, sandbox contiene.

**Framework di agenti dove il problema si manifesta**
- **LangChain**, **AutoGPT**, **OpenAI/Anthropic tool-use**, **Model Context Protocol (MCP)**:
  qui l'agente esegue tool → è esattamente il punto in cui serve la sandbox.

> Conclusione da scrivere: *le librerie esistono ma sono frammentate* — alcune fanno solo
> esecuzione codice, altre solo isolamento infra, poche integrano il tutto con una logica
> di decisione LLM-driven. **Questo "buco" è lo spazio del tuo contributo.**

---

## 5. Architettura proposta (progettazione "sommaria" ma seria)

Proposta di architettura a **broker/proxy di azioni** — coerente con la metafora del prof
(un LLM che ne invoca un altro) ma applicata alle *azioni* invece che al testo.

```
Utente
  │  prompt
  ▼
┌─────────────┐   codice/azione     ┌──────────────────┐
│   LLM        │ ──────────────────▶ │  BROKER /         │
│  principale  │                     │  POLICY ENGINE    │
└─────────────┘                     │  (decisore)       │
       ▲                            └────────┬─────────┘
       │ risultato                           │ se permesso
       │                                     ▼
       │                          ┌─────────────────────┐
       │                          │   SANDBOX            │
       │                          │  (container/microVM) │
       │  ◀───────────────────────│  - FS effimero       │
        risultato + log audit     │  - rete deny         │
                                  │  - limiti risorse     │
                                  └─────────────────────┘
```

**Componenti**
1. **Interceptor**: cattura codice/azioni prodotte dall'LLM prima dell'esecuzione.
2. **Policy engine (il decisore)**: decide se un'azione è ammessa. Due modalità:
   - *rule-based*: allow-list di comandi/tool, deny di pattern pericolosi (`rm -rf`,
     accessi a chiavi, socket di rete).
   - *LLM-based ("l'altro LLM")*: un secondo modello classifica l'azione come safe/unsafe →
     è il pattern di *ricorsività/AI-che-controlla-AI* che il prof ha citato esplicitamente.
   - *ibrido* (consigliato): regole rigide per i casi certi + LLM per i casi ambigui.
3. **Sandbox runtime**: esegue l'azione approvata in ambiente isolato (Docker come default
   pragmatico; menzionare gVisor/microVM come upgrade).
4. **Auditor / logger**: registra ogni azione e decisione per analisi e per la valutazione.

**Punto di forza per la lode**: rendere la **policy configurabile** (file di
configurazione con i livelli di isolamento) così puoi *dimostrare* trade-off diversi.

---

## 6. La demo / prototipo (cosa mostrare concretamente)

Idea minima ma d'impatto: **"Code Interpreter sicuro"**.

- Input: l'utente chiede all'LLM di svolgere un compito che richiede eseguire codice
  (es. "analizza questo CSV", "calcola...").
- L'LLM genera il codice.
- Il broker valuta e la sandbox lo esegue **isolato**.
- Mostri il risultato **corretto** per i casi legittimi.

**Poi la parte che fa la differenza: gli attacchi.**
Prepara una batteria di **prompt malevoli / injection** e mostra che la sandbox li contiene:
- lettura di un file sensibile (es. finto `~/.ssh/id_rsa` o `/etc/passwd`);
- esfiltrazione via rete (una `curl`/`requests` verso un host esterno → bloccata);
- comando distruttivo (`rm -rf`, cancellazione file);
- fork bomb / loop infinito → fermato da timeout e limiti risorse;
- tentativo di accesso alle chiavi API dell'app.

Per ciascuno: **"senza sandbox" (danno) vs "con sandbox" (contenuto)** → tabella di esiti.
Questo confronto è ciò che trasforma un compitino in un progetto da lode.

**Bonus avanzati (se hai tempo, spingono verso il top)**
- Confronto tra livelli di isolamento (RestrictedPython bypassabile vs Docker vs gVisor).
- Il "secondo LLM" come classificatore di azioni + misura di quanto è affidabile.
- Test di *sandbox escape* noti per mostrare i limiti onestamente.

---

## 7. Valutazione e metriche (spesso dimenticata = qui guadagni)

Un progetto scientifico si *misura*. Definisci metriche:

- **Efficacia di sicurezza**: % di attacchi contenuti / bypassati (su una suite di test).
- **Falsi positivi**: azioni legittime erroneamente bloccate dalla policy.
- **Overhead**: latenza e risorse introdotte dalla sandbox (Docker vs microVM vs WASM).
- **Copertura**: quali classi di minaccia (dal threat model) sono coperte e quali no.

Costruisci un piccolo **benchmark di prompt** (buoni + malevoli) e riportalo in tabella.
Anche 15–20 casi ben scelti sono sufficienti per essere convincente.

---

## 8. Struttura del report / presentazione finale

Ordine consigliato (allineato ai criteri del prof):

1. **Introduzione**: problema, differenza guardrail vs sandbox, scope scelto.
2. **Threat model**: attori, asset, capacità, cosa è fuori scope.
3. **Principi di funzionamento**: la scala di isolamento (sez. 3).
4. **Stato dell'arte**: tabella comparativa librerie open source (sez. 4).
5. **Architettura proposta**: schema broker + policy + sandbox (sez. 5).
6. **Prototipo/Demo**: implementazione + batteria di attacchi (sez. 6).
7. **Valutazione**: metriche e risultati (sez. 7).
8. **Limiti e lavori futuri**: onestà sui bypass, escape, costi.
9. **Posizionamento rispetto ai guardrail** (opzionale, mezza pagina): accennare come una
   sandbox si comporrebbe con un filtro di input/output in una pipeline completa
   (input filter → decisione → esecuzione contenuta → output filter). Serve a mostrare la
   *visione di sistema*, ma resta contesto: il deliverable è la sandbox.

---

## 9. Cosa rende questo progetto "30 e lode" (checklist)

- [ ] Scope definito con precisione e motivato (non "una sandbox generica").
- [ ] Threat model esplicito con riferimenti a OWASP LLM Top 10 / MITRE ATLAS.
- [ ] Padronanza della *scala di isolamento* (sai perché RestrictedPython non basta e
      perché una microVM è più forte di un container).
- [ ] Rassegna reale delle librerie open source, con giudizio critico (non solo elenco).
- [ ] Architettura chiara con il "secondo LLM" come decisore (la ricorsività chiesta dal prof).
- [ ] Demo che mostra il confronto **con/senza sandbox** su attacchi concreti.
- [ ] Metriche di valutazione quantitative.
- [ ] Discussione onesta dei limiti (i bypass esistono, li conosci).
- [ ] (Opzionale) Cenno a come la sandbox si comporrebbe con un guardrail → visione di sistema.

---

## 10. Prossimi passi (prima di scrivere codice)

1. **Confermare lo scope** (A+B: esecuzione codice + azioni agente) e concordarlo col prof.
2. **Scegliere lo stack della demo**: probabilmente Python + Docker (pragmatico) come base,
   con gVisor/E2B citati come evoluzione.
3. **Preparare la suite di test** (prompt buoni + malevoli) *prima* di implementare: così
   sai cosa devi dimostrare.
4. Solo dopo → prototipo.

> Da decidere insieme prima di partire: livello di isolamento target per la demo (Docker vs
> microVM), e se il "secondo LLM" decisore sarà rule-based, LLM-based o ibrido.
