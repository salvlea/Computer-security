# Graph Report - .  (2026-07-24)

## Corpus Check
- 3 files · ~400,047 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 53 nodes · 68 edges · 7 communities
- Extraction: 94% EXTRACTED · 6% INFERRED · 0% AMBIGUOUS · INFERRED: 4 edges (avg confidence: 0.8)
- Token cost: 71,647 input · 0 output

## Community Hubs (Navigation)
- Sandbox Runtimes & Agent Tooling
- Container Isolation (Docker/gVisor)
- Guardrail Frameworks
- Isolation Ladder (OS-level)
- Broker/Policy Architecture
- LLM Threat Model
- Professor Brief & Scope

## God Nodes (most connected - your core abstractions)
1. `Sandbox LLM (scope progetto)` - 19 edges
2. `Scala di isolamento (livelli di sandboxing)` - 11 edges
3. `Guardrail` - 9 edges
4. `Threat Model` - 6 edges
5. `Architettura Broker / Policy Engine` - 6 edges
6. `Sandbox (definizione del professore)` - 5 edges
7. `llm-sandbox (vndee/llm-sandbox)` - 4 edges
8. `Docker vs Kubernetes (scelta per la demo)` - 4 edges
9. `Docker / Podman` - 3 edges
10. `gVisor` - 3 edges

## Surprising Connections (you probably didn't know these)
- `Sandbox LLM (scope progetto)` --references--> `Sandbox (definizione del professore)`  [EXTRACTED]
  capire-progetto/consigli_progetto.md → richiesta-professore/progetto.txt
- `Sandbox LLM (scope progetto)` --references--> `Stato dell'arte (richiesta valutazione)`  [EXTRACTED]
  capire-progetto/consigli_progetto.md → richiesta-professore/progetto.txt
- `Guardrail` --references--> `Guardrail (definizione del professore)`  [EXTRACTED]
  capire-progetto/consigli_progetto.md → richiesta-professore/progetto.txt
- `Guardrail` --references--> `Moscuza (studente, tema guardrail)`  [EXTRACTED]
  capire-progetto/consigli_progetto.md → richiesta-professore/progetto.txt
- `Policy Engine (il decisore)` --references--> `Ricorsivita' / AI che controlla AI`  [EXTRACTED]
  capire-progetto/consigli_progetto.md → richiesta-professore/progetto.txt

## Hyperedges (group relationships)
- **Componenti del Threat Model** — capire_progetto_consigli_progetto_threat_model, capire_progetto_consigli_progetto_owasp_top_10_llm, capire_progetto_consigli_progetto_mitre_atlas, capire_progetto_consigli_progetto_prompt_injection, capire_progetto_consigli_progetto_confused_deputy [EXTRACTED 1.00]
- **Componenti dell'architettura Broker/Policy** — capire_progetto_consigli_progetto_broker_policy_architecture, capire_progetto_consigli_progetto_interceptor, capire_progetto_consigli_progetto_policy_engine, capire_progetto_consigli_progetto_sandbox_runtime, capire_progetto_consigli_progetto_auditor_logger [EXTRACTED 1.00]
- **Gerarchia della scala di isolamento** — capire_progetto_consigli_progetto_scala_isolamento, capire_progetto_consigli_progetto_restrictedpython, capire_progetto_consigli_progetto_docker_podman, capire_progetto_consigli_progetto_gvisor, capire_progetto_consigli_progetto_firecracker, capire_progetto_consigli_progetto_webassembly [EXTRACTED 1.00]

## Communities (7 total, 0 thin omitted)

### Community 0 - "Sandbox Runtimes & Agent Tooling"
Cohesion: 0.22
Nodes (11): AutoGPT, bubblewrap (bwrap), Judge0, landlock (LSM), LangChain, Model Context Protocol (MCP), Metriche di valutazione, nsjail (Google) (+3 more)

### Community 1 - "Container Isolation (Docker/gVisor)"
Cohesion: 0.20
Nodes (11): Docker / Podman, E2B (e2b-dev), Firecracker, gVisor, llm-sandbox (vndee/llm-sandbox), Sandbox Runtime, Docker, Docker hardened (--network none, --read-only, --cap-drop=ALL, non-root) (+3 more)

### Community 2 - "Guardrail Frameworks"
Cohesion: 0.25
Nodes (8): Guardrail, Guardrails AI, Llama Guard (Meta), LLM Guard (protectai), NVIDIA NeMo Guardrails, Rebuff, Guardrail (spiegazione semplice), Moscuza (studente, tema guardrail)

### Community 3 - "Isolation Ladder (OS-level)"
Cohesion: 0.33
Nodes (7): AppArmor / SELinux, Kata Containers, Linux namespaces + cgroups, RestrictedPython, Scala di isolamento (livelli di sandboxing), seccomp-bpf, WebAssembly (WASM)

### Community 4 - "Broker/Policy Architecture"
Cohesion: 0.33
Nodes (6): Auditor / Logger, Architettura Broker / Policy Engine, Demo: Code Interpreter sicuro, Interceptor, Policy Engine (il decisore), Ricorsivita' / AI che controlla AI

### Community 5 - "LLM Threat Model"
Cohesion: 0.40
Nodes (6): Confused Deputy, Indirect Prompt Injection, MITRE ATLAS, OWASP Top 10 for LLM Applications, Prompt Injection, Threat Model

### Community 6 - "Professor Brief & Scope"
Cohesion: 0.67
Nodes (4): Guardrail (definizione del professore), Progetto underspecified (design intent del prof), Sandbox (definizione del professore), Stato dell'arte (richiesta valutazione)

## Knowledge Gaps
- **22 isolated node(s):** `MITRE ATLAS`, `Indirect Prompt Injection`, `Confused Deputy`, `seccomp-bpf`, `AppArmor / SELinux` (+17 more)
  These have ≤1 connection - possible missing edges or undocumented components.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `Sandbox LLM (scope progetto)` connect `Sandbox Runtimes & Agent Tooling` to `Container Isolation (Docker/gVisor)`, `Guardrail Frameworks`, `Isolation Ladder (OS-level)`, `Broker/Policy Architecture`, `LLM Threat Model`, `Professor Brief & Scope`?**
  _High betweenness centrality (0.798) - this node is a cross-community bridge._
- **Why does `Scala di isolamento (livelli di sandboxing)` connect `Isolation Ladder (OS-level)` to `Sandbox Runtimes & Agent Tooling`, `Container Isolation (Docker/gVisor)`?**
  _High betweenness centrality (0.358) - this node is a cross-community bridge._
- **Why does `Guardrail` connect `Guardrail Frameworks` to `Sandbox Runtimes & Agent Tooling`, `Professor Brief & Scope`?**
  _High betweenness centrality (0.263) - this node is a cross-community bridge._
- **What connects `MITRE ATLAS`, `Indirect Prompt Injection`, `Confused Deputy` to the rest of the system?**
  _22 weakly-connected nodes found - possible documentation gaps or missing edges._