# OrgWorks - TAPPE

Modelli: O = Opus, S = Sonnet, H = Haiku, C = Codex (solo logica con test già scritti).
Verifiche: T = test automatici; G = confronto con un file di riferimento; M = checklist manuale di 5 minuti elencata nella voce (obbligatoria per ogni voce con UI).

## Regole di lavoro

- Una sotto-tappa per branch (`tappa-<n>`) e per sessione, con `/clear` tra l'una e l'altra.
- Merge `--ff-only` su main solo con test verdi e checklist M spuntata.
- I test di logica delicata (parser, client Sheets e Calendar, Store, operazioni composte) li scrive Opus, mai Haiku.
- I valori attesi si calcolano a mano dall'input, non dall'output del codice.
- Haiku solo per lavori meccanici con esito specificato e verifica automatica.
- Due esecuzioni con lo stesso errore: si passa al modello superiore.
- Un modello inferiore non modifica test scritti da uno superiore senza dirlo.
- `grep` dei nomi esistenti prima di scrivere; elenco dei file da toccare prima di modificare.
- La logica sta in OrgWorksCore, la UI nell'app.
- Nei prompt per la UI indicare il file dove si decide il comportamento.
- A fine sotto-tappa annotare qui: modello, costo (barra settimanale prima e dopo), esecuzioni fino al verde.
- Dopo tre sotto-tappe si rivedono le assegnazioni dei modelli.

## Tappa 0 - Spike (SPEC 13.2)

Chiude D2, D3, D7, D8. Codice di prova in `Spike/`, da buttare o riusare solo dopo revisione.

- **0a** Sheets e Calendar API da Swift: JWT RS256 con Security.framework, lettura e scrittura di una cella, `calendarList.insert`, `events.list` con `syncToken` (con una finestra `timeMin`/`timeMax` iniziale) e 410, `patch` con `If-Match` e 412, `extendedProperties.private`, istanza di una serie, calendario non proprio, evento con invitati. Modello: O. Verifica: M (punti 1, 2 e 4 di SPEC 13.2). Annotare i risultati in SPEC 12.
- **0b** `orgworks-mcp` minimale (uno strumento, `oggi`, anche finto) registrato in Claude Desktop e richiamabile; scelta tra SDK ufficiale e JSON-RPC minimale. Modello: S. Verifica: M (punto 3 di SPEC 13.2).

## Tappa 1 - Core: modelli e Sheets (SPEC 2.1, 2.5, 4, 13.1)

- **1a** `Package.swift`, modelli (`OWTask`, `Sottotask`, `Occorrenza`, `EventoCalendario`, parametri), generazione degli id, parser e serializzatore tolleranti (date ISO, `gg/mm/aaaa`, seriali; booleani; liste). Modello: O. Verifica: T con tabella di casi. Fissa la versione minima di macOS (SPEC D14) e la struttura in CLAUDE.md.
- **1b** `SheetsClient`, `FakeSheets`, ricerca per id, compare-and-swap, colonne sconosciute preservate, lista bianca di schede, inizializzazione idempotente. Modello: O. Verifica: T.

## Tappa 2 - Core: Calendar (SPEC 2.4, 2.5, 3.2, 3.3, 5.3)

- **2a** `CalendarClient` (protocollo), `FakeCalendar`, mapping degli eventi (`date` contro `dateTime`, fusi, tutto il giorno), sincronizzazione incrementale con `syncToken` e 410. Modello: O. Verifica: T.
- **2b** Scritture: insert, patch, delete con `If-Match`, `extendedProperties`, istanza contro serie, errore 412. Modello: O. Verifica: T.

## Tappa 3 - Core: modello dei task (SPEC 3, 13.1)

- **3a** `elencoTask`: virtuali, materializzati, ricorrenti (una riga per serie), evento eliminato fuori dall'app, evento cambiato di calendario. Test: O. Implementazione: O. Verifica: T.
- **3b** Urgenza, quadranti, avanzamento dei sotto task, effetti del trascinamento in matrice. Test: O. Implementazione: S. Verifica: T con confini (oggi, oggi più 2, oggi più 3, passato, senza data).
- **3c** Operazioni composte nello `Store`: creazione con data, dare e togliere la data, eliminazione, fatto, con ordine delle scritture e fallimento a metà. Test: O. Implementazione: O. Verifica: T.

## Tappa 4 - Guscio dell'app (SPEC 2.2, 8, 9; DESIGN 2)

- **4a** Progetto Xcode, due scene `Window`, `Store` iniettato, menu, `openWindow` tra moduli, `run.sh`. Modello: S. Verifica: M (le due finestre si aprono, si chiudono, la posizione viene ricordata, l'ultima chiusa termina l'app).
- **4b** Impostazioni (quattro pannelli), primo avvio, componenti degli stati dei moduli. Modello: S. Verifica: M (importa credenziali, verifica accesso, aggiungi un calendario, stati vuoto, errore e non autorizzato).

## Tappa 5 - Modulo Calendario (SPEC 5; DESIGN 5)

- **5a** Barra laterale dei calendari, viste Mese e Settimana in sola lettura con dati reali. Modello: S. Verifica: M (eventi coerenti con Google Calendar web, colori, tutto il giorno, linea dell'ora, scorrimento).
- **5b** Viste Giorno (agenda) e Anno. Modello: S. Verifica: M (clic dall'Anno al Giorno e al Mese, frecce, "Oggi").
- **5c** Pannello Evento: creare, modificare, eliminare, sezione Task, sola lettura per gli invitati. Modello: S. Verifica: M (creare ed eliminare con conferma; evento con invitati bloccato; salvataggio e `Esc`).
- **5d** Spostare e ridimensionare in Settimana e Mese, dialogo "Solo questo evento / Tutti gli eventi", rifiuto 412 con "Ricarica". Modello: O (gesti e concorrenza). Verifica: M (spostare e controllare su Google Calendar web; modificare lo stesso evento da Google mentre l'app è aperta).

## Tappa 6 - Modulo To do (SPEC 6; DESIGN 6)

- **6a** Lista, filtri, ricerca, "Mostra fatti", pannello Task con sotto task. Modello: S. Verifica: M (sezioni nell'ordine giusto, sotto task, fatto, avanzamento).
- **6b** Matrice, colonna "Da classificare", trascinamento della priorità, trascinamento tra colonne ignorato. Modello: S. Verifica: M (i quattro quadranti con dati reali, trascinamenti, urgenza che segue la data).
- **6c** Creazione dal To do con e senza data, collegamento tra finestre ("Mostra nel calendario", "Apri nel To do"). Modello: S. Verifica: M (un task con data compare subito nel Calendario; togliere la data chiede conferma).

## Tappa 7 - Server MCP (SPEC 7)

- **7a** Strumenti di lettura (`oggi`, `elenca_calendari`, `elenca_eventi`, `elenca_task`, `leggi_task`, `leggi_parametri`). Test: O. Implementazione: S. Verifica: T.
- **7b** Strumenti di scrittura (`crea_evento`, `aggiorna_evento`, `crea_task`, `aggiorna_task`, `completa_task`, `aggiungi_sottotask`, `aggiorna_sottotask`, `imposta_parametro`), con `elemento_cambiato` e verifica che nessuno strumento cancelli. Modello: O. Verifica: T.
- **7c** Registrazione in Claude Desktop e Code, prova reale. Modello: S. Verifica: M (chiedere a Claude di spostare e creare un evento e di aggiungere un sotto task; ricompare nell'app al ciclo successivo).

## Tappa 8 - Chiusura

- **8a** Pulizia con la skill `cleanup-swift` (codice morto, duplicati, costanti, file di contesto) senza cambiare funzionalità. Modello: S. Verifica: T e M finale su entrambi i moduli.

## Registro (si compila a fine sotto-tappa)

| Sotto-tappa | Modello | Barra prima | Barra dopo | Esecuzioni fino al verde |
|---|---|---|---|---|
