# OrgWorks

App macOS personale di Oscar (SwiftUI). Due moduli per ora, Calendario e To do, come finestre indipendenti della stessa app. Dati in Google Sheets, eventi in Google Calendar (API), server MCP per Claude. Repo: `/Users/oscar/GIT/OrgWorks` (nato dal repo `agenda`, che resta come riferimento della v0.1).

## Stack

Swift e SwiftUI, macOS su Apple Silicon. Versione minima di macOS da fissare in tappa 1a (SPEC D14). Nessuna dipendenza esterna, salvo l'SDK Swift di MCP se lo spike (tappa 0) lo sceglie.

## Struttura (prevista, si fissa in tappa 1)

```
Package.swift           OrgWorksCore e orgworks-mcp
Sources/OrgWorksCore/   logica, client, Store (nessun SwiftUI, nessun AppKit)
Sources/orgworks-mcp/   server MCP su stdio
Tests/OrgWorksCoreTests/
OrgWorks/               progetto Xcode dell'app (solo UI)
SPEC.md  DESIGN.md  TAPPE.md  GUIDA.md
```

## Comandi

- `swift test` dalla radice: test di OrgWorksCore.
- `swift build -c release --product orgworks-mcp`: server MCP.
- `./run.sh`: build e avvio dell'app (si crea in tappa 4).

Test, build, commit e merge li lancia Oscar dal suo terminale: dalla sessione cloud Swift non compila e git lascia file `.lock`.

## Regole

- Leggi solo le sezioni di SPEC.md e DESIGN.md citate dalla tappa corrente di TAPPE.md. Non leggere GUIDA.md (è per Oscar).
- Le decisioni nuove vanno solo in SPEC.md, mai qui.
- OrgWorksCore non importa SwiftUI né AppKit. La UI non contiene regole di business.
- Un campo, una fonte (SPEC 3.1): titolo e data nell'evento, priorità e sotto task nel foglio.
- Prima di creare un nome, `grep` dei nomi esistenti. Prima di modificare, elenca i file che toccherai.
- Una costante al posto di un valore dinamico è un cambio di comportamento.
- Nei prompt per la UI indica il file dove si decide il comportamento, non solo il sintomo.
- La chiave del service account non entra mai nel repo, nei log, negli output (`.gitignore`).
- Nessuno strumento MCP cancella nulla (SPEC 7.4).
- Test: valori attesi calcolati a mano dall'input, mai dall'output del codice.
- Testi dell'interfaccia in italiano, senza emoji; nel codice e nei documenti niente em dash.

## Tappe fatte

Nessuna. Prossima: 0a (vedi TAPPE.md).
