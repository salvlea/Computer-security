# Spiegazione semplice: cos'è una sandbox LLM e cosa dice il file dei consigli

## 1. Cos'è una "sandbox LLM", in parole semplici

Immagina un LLM (tipo ChatGPT) che non si limita a *rispondere a parole*, ma può anche
**fare cose**: scrivere ed eseguire codice, leggere file, chiamare programmi esterni,
cercare su internet. Questo è quello che succede oggi con i "Code Interpreter" o con gli
"agenti AI" che usano tool.

Il problema: l'LLM genera il codice/le azioni sulla base di un testo (il prompt) scritto
da un utente. Se quel testo è malevolo (o contiene istruzioni nascoste), l'LLM potrebbe
generare ed eseguire codice pericoloso: cancellare file, rubare password, mandare dati a
un server esterno, bloccare il computer con un ciclo infinito, ecc.

**La sandbox è la "gabbia" in cui fai eseguire quel codice/quelle azioni**, isolata dal
resto del sistema, così che anche se il codice è cattivo, il danno resta *contenuto
dentro la gabbia* e non tocca il sistema vero (i tuoi file, la tua rete, le tue chiavi).

Un paragone semplice: è come far giocare un bambino con oggetti potenzialmente pericolosi
dentro una stanza imbottita, con la porta chiusa a chiave e senza finestre — qualsiasi
cosa rompa, resta lì dentro.

### Sandbox vs Guardrail (l'altro concetto menzionato dal prof)
Sono due cose diverse che si confondono facilmente:

- **Guardrail** = controlla **cosa l'LLM dice** (il testo). Es: se chiedi "come costruire
  una bomba", il guardrail intercetta la domanda o la risposta e blocca il contenuto.
  È un filtro sul *linguaggio*.
- **Sandbox** = controlla **cosa l'LLM fa** (le azioni). Non impedisce all'LLM di *scrivere*
  codice pericoloso, ma impedisce che quel codice, una volta *eseguito*, faccia danni veri.
  È un contenimento sulle *azioni/effetti*.

Il tuo progetto è **solo sulla sandbox** (le azioni), non sul guardrail (il testo) — questo
tema è stato dato a un altro studente, tu lavori da solo e in modo indipendente.

---

## 2. Riassunto brevissimo di cosa dice `consigli_progetto.md`

Il file è una guida per strutturare il progetto. In sintesi dice:

1. **Cosa vuole il prof**: studiare lo stato dell'arte (librerie esistenti), capire i
   principi di funzionamento (non solo usare uno strumento a scatola chiusa), progettare
   un'architettura sommaria e mostrarne una demo, ed essere onesti sui limiti.

2. **Scegli uno scope preciso**: "sandbox LLM" può voler dire tre cose diverse. Consiglia
   di concentrarti su: *(A)* isolare il codice che l'LLM genera ed esegue, e *(B)* isolare
   le azioni di un LLM-agente che usa dei tool. È lì che esistono librerie vere da studiare
   ed è il caso con la demo più convincente.

3. **Threat model**: prima di progettare, definisci chi attacca (utente malevolo, prompt
   injection), cosa vuoi proteggere (file, rete, risorse) e cosa la sandbox NON copre
   (es. non impedisce risposte testuali sbagliate — quello è il guardrail).

4. **Principi di funzionamento**: spiega una "scala" di modi per isolare codice, dal più
   debole al più forte:
   - livello linguaggio (debole, facile da aggirare)
   - livello processo/OS (seccomp, namespaces, cgroups)
   - container (Docker/Podman)
   - sandbox kernel user-space (gVisor)
   - microVM (Firecracker, Kata) — il più forte
   - WebAssembly (approccio alternativo, capability-based)

5. **Stato dell'arte**: elenca librerie open source reali da citare/studiare, es.
   `llm-sandbox`, E2B, Piston, gVisor, Firecracker, nsjail, bubblewrap.

6. **Architettura proposta**: un "broker" (motore di decisione) che intercetta ogni
   azione richiesta dall'LLM, decide se è sicura (con regole fisse, con un secondo LLM
   giudice, o un mix), e solo se approvata la fa eseguire dentro la sandbox isolata.

7. **Demo consigliata**: un "Code Interpreter sicuro" che esegue codice generato dall'LLM
   dentro un container, testato con prompt legittimi e con tentativi di attacco (lettura
   file sensibili, fork bomb, esfiltrazione di rete...), mostrando il confronto
   "con sandbox" vs "senza sandbox".

8. **Valutazione**: misurare quanti attacchi vengono bloccati, quanti falsi positivi,
   quanto overhead (rallentamento) introduce la sandbox.

9. **Struttura del report finale** e una **checklist** di cosa serve per il massimo dei voti.

10. **Prossimi passi pratici**: confermare lo scope, scegliere lo stack tecnico (Docker
    come base pragmatica), preparare i test di attacco prima di scrivere codice.

In una frase: *il file ti guida a scegliere bene cosa costruire (isolare l'esecuzione di
codice/azioni di un LLM), a motivarlo con un modello delle minacce, a conoscere le
tecnologie reali di isolamento, e a dimostrarlo con una demo che blocca attacchi concreti.*

---

## 3. Docker o Kubernetes? (risposta diretta)

**Usa Docker.** Kubernetes non serve, e aggiungerlo sarebbe controproducente per questo
progetto. Ecco perché:

| | Docker | Kubernetes |
|---|---|---|
| Cosa è | Motore per creare ed eseguire container isolati | Orchestratore che *gestisce tanti* container/nodi su più macchine |
| Problema che risolve | Isolamento di un singolo processo/esecuzione | Scalare, distribuire, riavviare automaticamente centinaia di container in un cluster |
| Serve alla tua demo? | Sì — è esattamente lo strumento di isolamento che ti serve | No — tu non hai bisogno di gestire un cluster, solo di isolare esecuzioni singole |
| Complessità da imparare | Bassa/media, si impara in poche ore | Alta: nodi, pod, networking, molto tempo per poco valore aggiunto qui |
| Cosa userebbe la libreria `llm-sandbox` citata nella guida | Docker (o Podman) come backend principale | K8s è supportato solo come opzione avanzata, non necessaria |

**Motivo tecnico di fondo**: il tuo problema è *"isolare una singola esecuzione di codice
non fidato"* (una richiesta → un container effimero → risultato → distruggi il container).
Questo è precisamente il caso d'uso di Docker. Kubernetes risolve un problema diverso:
*"come faccio girare in modo affidabile e scalabile centinaia/migliaia di container su
più server"* — un problema di infrastruttura di produzione su larga scala, non di sicurezza
per singola esecuzione.

Se nel report vuoi comunque dimostrare di conoscere Kubernetes, puoi **citarlo come nota**:
*"in un sistema di produzione con alto traffico, l'orchestrazione dei container sandbox
potrebbe essere gestita con Kubernetes (o Nomad), ma per la demo di questo progetto Docker
è sufficiente e più diretto da dimostrare"*. Questo ti fa guadagnare punti (mostri di
conoscere lo scaling) senza appesantire inutilmente l'implementazione.

**Percorso di crescita consigliato per il tuo isolamento** (puoi seguirlo nella demo, dal
più semplice al più robusto, ed è coerente con la "scala" già descritta in
`consigli_progetto.md`, sezione 3):

1. Docker semplice (baseline).
2. Docker **hardened**: `--network none`, `--read-only`, `--cap-drop=ALL`, `--user`
   non-root, limiti di CPU/RAM/tempo (`--memory`, `--cpus`, timeout).
3. (Opzionale, bonus) Provare **gVisor** come runtime alternativo di Docker
   (`--runtime=runsc`) per mostrare un livello di isolamento superiore, mantenendo la
   stessa interfaccia Docker — un confronto ottimo da mettere nel report.
