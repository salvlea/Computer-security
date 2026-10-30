# Guida a Ollama — come si usa e dove entra nel progetto

> Documento pratico. Se non hai mai usato Ollama, leggilo tutto prima di lanciare comandi:
> spiega cos'è, come si installa, come si usa da terminale e da Python, quale modello
> scegliere e in quali punti esatti del progetto (broker/sandbox) lo userai.

---

## 1. Cos'è Ollama, in parole semplici

Ollama è un programma che fa girare **modelli LLM open source direttamente sul tuo Mac**,
senza bisogno di internet (dopo il primo download) e senza pagare nulla.

Funziona un po' come Docker, ma per i modelli invece che per i container:
- **Docker**: scarichi un'immagine → la fai girare in un container isolato.
- **Ollama**: scarichi un modello (es. `qwen2.5-coder`) → lo fai "girare" e ci parli.

Ollama espone il modello in due modi:
1. **Da terminale**, con comandi tipo `ollama run nome-modello "il tuo prompt"`.
2. **Come server HTTP locale** (`http://localhost:11434`), a cui il tuo codice Python si
   collega per mandare prompt e ricevere risposte — è **questo** il modo che userai nel
   progetto, perché il broker deve parlare con l'LLM in modo automatico, non a mano.

---

## 2. Installazione (macOS)

```bash
brew install ollama
```

Poi devi avviare il **server** (il processo che sta in ascolto e risponde alle richieste).
Ha bisogno di restare acceso in un terminale mentre lavori:

```bash
ollama serve
```

Lascia questa finestra di terminale aperta con `ollama serve` in esecuzione. Apri **un
secondo terminale** per tutto il resto (scaricare modelli, lanciare il tuo codice Python,
ecc.).

> Nota: su macOS, dopo l'installazione, Ollama a volte parte già in background da solo
> (icona nella barra in alto). Se `ollama serve` dice che la porta è già in uso, va bene
> comunque: significa che il server è già acceso.

Verifica che il server risponda:
```bash
curl http://localhost:11434
```
Se stampa `Ollama is running`, sei a posto. ✅

---

## 3. Quale modello scegliere

I modelli si scaricano con `ollama pull nome-modello`. Ogni modello ha una dimensione
(GB) e ti serve spazio su disco + un po' di RAM libera per farlo girare.

Per il tuo progetto ti servono **due usi diversi** dell'LLM (vedi sezione 5), quindi ti
consiglio **un solo modello per iniziare**, buono per generare codice:

```bash
ollama pull qwen2.5-coder
```

| Modello | A cosa serve | Dimensione circa |
|---|---|---|
| **`qwen2.5-coder`** (consigliato) | Generare codice Python da un prompt in linguaggio naturale — è specializzato per coding | ~4–5 GB |
| `qwen2.5-coder:1.5b` | Versione piccola, più veloce ma meno precisa — utile se il Mac è lento o ha poca RAM | ~1 GB |
| `llama3.1` | Modello generalista, va bene anche come "secondo LLM giudice" (valuta se un'azione è sicura) | ~4.7 GB |

**Per iniziare**: scarica solo `qwen2.5-coder` e usalo per entrambi i ruoli (generare
codice + fare da giudice). Se poi vuoi differenziare, aggiungi `llama3.1` per il ruolo di
giudice — nel report puoi motivare che usare due modelli diversi riduce il rischio che lo
stesso "bias" del modello generi il codice pericoloso *e* lo approvi.

Scaricare un modello (una tantum, ci mette qualche minuto):
```bash
ollama pull qwen2.5-coder
```

Elenco dei modelli già scaricati:
```bash
ollama list
```

Prova rapida da terminale:
```bash
ollama run qwen2.5-coder "scrivi una funzione python che somma due numeri"
```
(Con `ollama run` entri anche in una modalità interattiva a chat; per uscire scrivi `/bye`.)

---

## 4. Come parlargli da Python (quello che userà il tuo codice)

Nel progetto non userai `ollama run` a mano: il tuo codice Python parlerà con il server
Ollama via libreria. Nella Fase 0 della guida operativa hai già `ollama` in
`requirements.txt`.

Installazione libreria (dentro l'ambiente virtuale del progetto):
```bash
pip install ollama
```

Esempio minimo — **generare codice da un prompt** (questo è, in sostanza,
`llm/client.py` descritto nella guida operativa):

```python
import ollama

def genera_codice(prompt_utente: str) -> str:
    risposta = ollama.chat(
        model="qwen2.5-coder",
        messages=[
            {
                "role": "system",
                "content": (
                    "Sei un assistente che scrive SOLO codice Python. "
                    "Rispondi con il codice, senza spiegazioni."
                ),
            },
            {"role": "user", "content": prompt_utente},
        ],
    )
    return risposta["message"]["content"]

if __name__ == "__main__":
    codice = genera_codice("calcola i primi 20 numeri primi")
    print(codice)
```

Esempio minimo — **il secondo LLM come "giudice"** (questo è il cuore del
`policy_engine.py` in modalità *LLM-based*, descritto in `consigli_progetto.md`):

```python
import ollama

def valuta_sicurezza(codice: str) -> str:
    """Ritorna 'safe' o 'unsafe' analizzando il codice prima di eseguirlo."""
    risposta = ollama.chat(
        model="qwen2.5-coder",
        messages=[
            {
                "role": "system",
                "content": (
                    "Sei un revisore di sicurezza. Analizza il codice Python che segue. "
                    "Se prova ad accedere alla rete, leggere file sensibili, cancellare "
                    "dati, o consumare risorse in modo eccessivo (loop infiniti, fork "
                    "bomb), rispondi SOLO con la parola 'unsafe'. Altrimenti rispondi "
                    "SOLO con la parola 'safe'."
                ),
            },
            {"role": "user", "content": codice},
        ],
    )
    esito = risposta["message"]["content"].strip().lower()
    return "unsafe" if "unsafe" in esito else "safe"
```

Nota importante da scrivere nel report: **questo giudice LLM non è affidabile al 100%**
(può sbagliare, può essere aggirato con prompt injection anche lui). Per questo la guida
consiglia un approccio **ibrido**: regole fisse (deny-list di pattern come `os.system`,
`socket`, `subprocess`, `rm -rf`, `/etc/passwd`...) *insieme* al giudice LLM — le regole
fisse coprono i casi certi, l'LLM copre i casi ambigui che le regole non prevedono.

---

## 5. Dove entra Ollama nell'architettura del progetto

Guardando lo schema in `capire-progetto/consigli_progetto.md` (sezione 5), Ollama serve
in **due punti distinti** della pipeline:

```
Utente
  │ prompt ("analizza questo CSV")
  ▼
┌───────────────┐   codice Python    ┌──────────────────────┐
│  LLM #1        │ ─────────────────▶│  BROKER / POLICY      │
│  (Ollama:      │   generato dal     │  ENGINE                │
│  qwen2.5-coder)│   prompt           │                        │
└───────────────┘                    │  ┌──────────────────┐  │
   ▲ genera il codice                │  │ LLM #2 (Ollama)   │  │  ← secondo uso di Ollama
   │                                 │  │ "è sicuro? sì/no" │  │     (il "giudice")
   │                                 │  └──────────────────┘  │
   │                                 └──────────┬─────────────┘
   │                                            │ se approvato
   │                                            ▼
   │                                 ┌─────────────────────┐
   │  risultato                     │   SANDBOX DOCKER      │
   └─────────────────────────────── │  (esegue il codice)   │
                                     └─────────────────────┘
```

1. **LLM #1 — generatore**: riceve il prompt dell'utente e scrive il codice Python da
   eseguire. È la parte "normale" di un assistente tipo Code Interpreter.
2. **LLM #2 — giudice** (dentro il policy engine del broker): riceve il codice generato
   *prima* che venga eseguito e decide se è sicuro. È la parte a cui il professore si
   riferiva parlando di *ricorsività* / *"un bot che invoca un altro prima di rispondere"*
   — solo che qui il secondo LLM non giudica il *testo* (guardrail), ma se le *azioni* del
   codice sono da bloccare (sandbox).

Puoi usare lo **stesso modello** (`qwen2.5-coder`) per entrambi i ruoli, cambiando solo il
prompt di sistema (uno dice "scrivi codice", l'altro dice "valuta se è pericoloso").
Questo è sufficiente per la demo. Nel report puoi comunque menzionare che, in un sistema
di produzione, si userebbero modelli diversi o un modello specializzato per la sicurezza.

---

## 6. Perché Ollama e non le API a pagamento (OpenAI/Anthropic)

Come già indicato in `capire-progetto/guida-operativa.md` (sezione A.3):

- **Gratis**: puoi fare mille prove per costruire e testare la batteria di attacchi senza
  spendere nulla.
- **Nessuna chiave API da proteggere**: ironicamente, una chiave API è proprio uno degli
  "asset" che la tua sandbox dovrebbe difendere (evitare che il codice generato la legga
  e la esfiltri) — con Ollama locale questo problema semplicemente non c'è durante lo
  sviluppo.
- **Funziona offline** (dopo aver scaricato il modello una volta).
- Nel report scrivi che l'architettura è **agnostica rispetto al modello**: sostituendo
  Ollama con una chiamata API a GPT/Claude cambia solo `llm/client.py`, il resto
  (broker, policy engine, sandbox) resta identico. Così dimostri di aver capito il
  disaccoppiamento senza dover spendere soldi per la demo.

---

## 7. Comandi utili da tenere a portata di mano

```bash
ollama serve                     # avvia il server (lascialo acceso in un terminale)
ollama pull qwen2.5-coder        # scarica un modello (una tantum)
ollama list                      # elenca i modelli scaricati
ollama run qwen2.5-coder "..."   # prova rapida da terminale (chat interattiva)
ollama ps                        # modelli attualmente caricati in memoria
ollama rm nome-modello           # rimuove un modello per liberare spazio
```

---

## 8. Checklist rapida per iniziare

- [ ] `brew install ollama`
- [ ] `ollama serve` (in un terminale dedicato, lasciato acceso)
- [ ] `ollama pull qwen2.5-coder` (in un secondo terminale)
- [ ] `ollama run qwen2.5-coder "scrivi una funzione python che somma due numeri"` → verifica che risponda
- [ ] `pip install ollama` dentro l'ambiente virtuale del progetto (`.venv`)
- [ ] Provare lo script Python di esempio della sezione 4 (`genera_codice`)
- [ ] Provare lo script Python di esempio della sezione 4 (`valuta_sicurezza`)

Quando questi passaggi funzionano, sei pronto per collegare Ollama al broker e alla
sandbox Docker (Fase 3-4 della guida operativa).
