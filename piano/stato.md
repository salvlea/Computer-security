# Stato del progetto — dove siamo arrivati

> **Questo è il file da leggere per primo**, su qualsiasi macchina, subito dopo `git pull`.
> Dice su quale macchina si lavora, a che punto siamo e qual è il prossimo passo.
> Si aggiorna **alla fine di ogni sessione** e **sempre prima di cambiare macchina**.
>
> Gli altri documenti: `piano-operativo.md` = *come* si fa ogni cosa (comandi) ·
> `piano-fattibilita.md` = priorità e cosa tagliare · `contesto-per-il-fisso.md` = il
> passaggio iniziale dal Mac (i suoi dati delle sezioni 7-8 ora si aggiornano **qui**).
>
> **Repo pubblico:** qui non vanno mai IP di Tailscale, nomi di account, chiavi o `.env`.

---

## 1. Adesso

| | |
|---|---|
| **Macchina attiva** | **FISSO** |
| **Fase** | 1 — Ambiente del fisso (Fase A del piano operativo) |
| **Fatto in questa fase** | F1 (verifiche preliminari, solo lettura) |
| **Prossimo passo** | F2: installare Ollama, Tailscale ed eventualmente Python 3.13, con conferma prima di ogni installazione |
| **Prossimo cambio di macchina** | Alla fine della fase 2 → si passa al MAC |
| **Ultimo aggiornamento** | 3/11/2026, sul fisso |

---

## 2. Mappa delle fasi tra fisso e Mac

Le fasi sono in ordine. La colonna **Cambio** dice se alla fine della fase si fa push e si
passa all'altra macchina. Le date non sono fissate: conta l'ordine.

| # | Fase | Macchina | Cosa si fa | Si passa alla successiva quando | Cambio |
|---|---|---|---|---|---|
| 1 | Ambiente fisso (A) | FISSO | Ollama con GPU, modelli, `.wslconfig`, Docker, Tailscale, firewall, Python, `.venv` | `ollama ps` → GPU; `docker run` ok; limiti WSL attivi | — |
| 2 | Base del repo | FISSO | `.gitignore`, `.gitattributes`, threat model, batteria di test (`tests/prompts/*.yaml`) | Test scritti, primo push riuscito | **push → MAC** |
| 3 | Ambiente Mac (B) + collegamento (M1) | MAC | Spazio disco, repo + `.venv`, Docker 4.43 verificato, Tailscale, `.env`, `curl` verso il fisso, riserva 3B se c'è spazio | **Controllo G1** superato (o ripiego deciso) | **push → FISSO** |
| 4 | Sandbox hardened | FISSO | `docker/Dockerfile.sandbox`, `runner_docker.py`, un flag di hardening alla volta | Gli attacchi scritti a mano sono contenuti | — |
| 5 | Broker + regole + auditor | FISSO | `broker.py`, `policy_engine.py` (regole), `auditor.py`, `demo.py --codice` | **Controllo G2** superato | — (facoltativo: push → MAC per una prova veloce di `demo.py --codice`, poi push → FISSO) |
| 6 | Giudice LLM + profili | FISSO | Giudice safe/unsafe, policy ibrida, `profili.yaml`, `llm/client.py`, `--replay` | Giro completo prompt → LLM → broker → sandbox | — |
| 7 | Benchmark | FISSO | `run_benchmark.py`, confronto con/senza sandbox, profili | La tabella dei risultati esce | **push → MAC** |
| 8 | Prima prova completa sul Mac | MAC | Build dell'immagine, batteria completa, `--profilo remoto/locale/replay` | **Controllo G3** superato | Se ci sono bug grossi: **push → FISSO**, si corregge e si torna; altrimenti si resta sul Mac |
| 9 | Validazione (D) e congelamento | MAC | Misure, taratura timeout, `preflight.py`, parte B ridotta se c'è tempo | **Controllo G4**: codice congelato | — |
| 10 | Report e slide | Mac o fisso, indifferente | Report, slide, video della demo | Report completo | push a ogni sessione |
| 11 | Prova in università (M6) | MAC in università | Rete vera, collegamento al fisso, ripieghi | **Controllo G5** | — |
| 12 | Prova generale ed esame (E) | MAC | Demo cronometrata, checklist del giorno prima | — | Fisso acceso e intoccato fino all'esame |

Note:
- Il fisso resta **sempre** il server dell'LLM, anche quando si lavora sul Mac: non va spento.
- I test d'attacco "pesanti" (fork bomb, prove di fuga dal container) si provano **prima sul
  fisso**; sul Mac si ripetono solo quelli già verificati (rischio R18).
- I tempi della sandbox per il report si misurano **solo sul Mac** (fase 8-9).
- Il report si può scrivere in qualsiasi momento; le sezioni "stato dell'arte" e "principi"
  conviene scriverle già durante le fasi 2-7.

---

## 3. Procedura di cambio macchina

**Prima di lasciare una macchina**
- [ ] Aggiornare la sezione 1 (adesso), la 4 (cosa c'è su ogni macchina) e il registro (sezione 8).
- [ ] `git status`: nessun `.env`, `.venv`, IP o file personali tra le modifiche.
- [ ] Commit e push.

**Arrivati sull'altra macchina**
- [ ] `git pull` (non basta `git fetch`: scarica ma non aggiorna i file).
- [ ] Aprire Claude Code e incollare il prompt qui sotto.
- [ ] Controllare la sezione 4: cosa manca su questa macchina (modelli, immagini, `.env`).

**Prompt per la nuova sessione di Claude**
```
Leggi piano/stato.md e riparti da lì. Per i comandi usa piano/piano-operativo.md, per le
priorità piano/piano-fattibilita.md. Prima di installare o cambiare impostazioni chiedimi
conferma. Quando è il momento di cambiare macchina dimmelo, aggiorna stato.md e prepara
commit e push. Rispondi in italiano.
```

---

## 4. Cosa c'è su ogni macchina (non viaggia con git)

| Elemento | Fisso | Mac |
|---|---|---|
| Git | ✅ 2.52 | ✅ |
| `git push` funziona | da verificare | da verificare |
| Python | ⚠️ 3.11.9 (Microsoft Store); il piano dice 3.13 | ✅ 3.13.1 |
| `.venv` del progetto | da creare | da creare |
| Docker | ✅ installato (CLI 29.8.0), WSL2 attivo | ✅ Desktop 4.43.0 — **non aggiornare** |
| Limiti VM Docker | `.wslconfig` da creare (4 GB, 4 CPU, 1 GB swap) | già 4 GB / 4 CPU |
| Immagine `sandbox-py` | da costruire (fase 4) | da costruire (fase 8) |
| Ollama | da installare (nativo) | da fare in Docker (riserva 3B), se c'è spazio |
| Modelli | nessuno | nessuno |
| Tailscale | da installare | da installare (login fatto a casa) |
| Firewall Ollama solo Tailscale | da fare (F5) | — |
| `.env` | da creare (LLM su localhost) | da creare (IP Tailscale del fisso) |

---

## 5. Dati annotati

| Dato | Valore |
|---|---|
| Data dell'esame | **giovedì 3/12/2026** |
| Windows del fisso | 11 Pro 26H2, build 26300 (non Insider) |
| Scheda grafica del fisso | AMD Radeon RX 7800 XT, driver 32.0.32015.7029 (+ grafica integrata del Ryzen) |
| Spazio libero su C: | 707 GB |
| Virtualizzazione (SVM) | Attiva |
| Rete del fisso | Cavo Ethernet 1 Gbps |
| Repo GitHub | Pubblico |
| Versione di Ollama (fisso) | — |
| Modelli scaricati (fisso) | — |
| `ollama ps` con il 14B | — |
| 14B + 7B insieme in VRAM? | — |
| Test di riavvio senza login | — |
| Versione di Python usata | — |
| Spazio libero sul Mac | — |
| Tailscale su macOS 13 funziona? | — |
| IP Tailscale del fisso | **non scriverlo qui**: solo nel `.env` del Mac |

---

## 6. Controlli go / no-go

Il dettaglio di cosa fare se un controllo fallisce è in `piano-fattibilita.md`, sezione 5.

| Controllo | Fine della fase | Domanda | Esito |
|---|---|---|---|
| G1 | 3 | Fisso con GPU, Mac con Docker, `curl` Mac → fisso ok? | — |
| G2 | 5 | Sandbox hardened contiene gli attacchi a mano? Broker con regole ok? | — |
| G3 | 8 | Giro completo sul Mac e tabella del benchmark? | — |
| G4 | 9 | Sul Mac passano tutti i legittimi e sono contenuti tutti gli attacchi? | — |
| G5 | 11 | In università il collegamento al fisso funziona? | — |

---

## 7. Problemi aperti e decisioni in sospeso

- **Python sul fisso:** c'è il 3.11 dello Store, il Mac ha il 3.13. Decidere in F2 se
  installare il 3.13 (consigliato, per avere la stessa versione).
- **Grafica integrata del Ryzen:** Ollama potrebbe vederla oltre alla RX 7800 XT. Da
  controllare in F3 con `ollama ps`.
- **VirtualBox installato sul fisso:** di solito convive con Docker; tenerlo presente se
  Docker dà problemi.
- **Login automatico di Windows:** si decide solo se il test di riavvio (F6) fallisce.
- **Ore disponibili a settimana:** non note. Se il tempo stringe si taglia seguendo
  `piano-fattibilita.md`, sezione 6.
- **Vecchie guide** (`linee-guida/ollama.md`, `capire-progetto/guida-operativa.md`): non
  corrette per scelta; vale il piano operativo.
- **Non ancora committati:** `piano/piano-fattibilita.md`, `piano/stato.md`.

---

## 8. Registro delle sessioni

Una riga per sessione, la più recente in fondo.

| Data | Macchina | Cosa è stato fatto |
|---|---|---|
| 30/10/2026 | Mac | Analisi del repo e dell'hardware del Mac; scritti `piano-operativo.md` e `contesto-per-il-fisso.md` |
| 3/11/2026 | Fisso | Letti tutti i documenti; verifiche F1 in sola lettura; scritti `piano-fattibilita.md` e `stato.md` |
