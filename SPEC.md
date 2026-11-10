# OrgWorks - SPEC

Versione 0.3 del 2026-11-10. Stato: bozza completa per i moduli Calendario e To do, da validare con lo spike della tappa 0.

Deriva da agenda v0.1 (repo `/Users/oscar/GIT/agenda`, commit 86fe511, che resta come riferimento). Tenuti: Sheets come fonte di verità con id stabili e compare-and-swap, cache derivata, server MCP, test con fake. Caduti: scrivania a griglia, design system Production OF, moduli Clienti, In corso, Timeline, Riviste, Concorsi, Ricorrenze (come modulo), Proposte e regole R1-R4, backup JSON, telefono.

Dove un punto è una proposta e non una decisione di Oscar è marcato [D#] e ricompare nella sezione 12. Le decisioni nuove vanno solo in questo file, mai in CLAUDE.md.

## 1. Scopo e confini

### 1.1 Cos'è

"OrgWorks" (OW) è un'app macOS stand alone, per uso personale di Oscar. Metafora della scrivania: ogni modulo è una finestra indipendente della stessa app. Interfaccia solo con componenti di sistema (SwiftUI nativo). Per ora due moduli:

- **Calendario**: mostra Google Calendar e i suoi calendari. Viste Anno, Mese, Settimana, Giorno (il Giorno è un'agenda a elenco). Gli eventi si spostano e si creano; la sincronizzazione è a due vie.
- **To do**: ogni evento è un task, con priorità e sotto task. Vista a lista e a matrice di Eisenhower. I task si possono creare anche da qui; con una data diventano eventi del calendario.

Dati dei task in Google Sheets, eventi in Google Calendar (API). Un server MCP permette a Claude (Desktop e Code, sul Mac) di leggere e modificare gli stessi dati. Nessun modello di linguaggio nell'app e nessuna lettura di Gmail: l'intelligenza sta in Claude, che scrive nell'app tramite MCP.

### 1.2 Fuori ambito v1

- Gli altri moduli di agenda (Clienti, Progetti, Timeline, Riviste, Concorsi, Proposte e regole) e la finanza.
- Inviti: creare eventi con invitati o cambiarne i partecipanti (il service account non può invitare da un account gmail.com). Gli eventi con invitati si leggono e restano in sola lettura (5.2).
- Creare o modificare regole di ricorrenza dall'app: gli eventi ricorrenti creati in Google Calendar si leggono e si gestiscono (3.3), ma la regola non si tocca.
- Notifiche di sistema, ricerca full-text, più utenti, telefono e iPad, backup e ripristino propri (resta la cronologia versioni di Google Sheets), stile proprio.
- Stato "in corso" dei task, scadenza separata dalla data dell'evento, progetti, persone, assegnazioni.

### 1.3 Principi

1. I dati sono di Oscar: foglio leggibile da altri programmi, nessun archivio proprietario.
2. Un campo, una fonte: titolo e data vivono nell'evento, priorità e sotto task nel foglio (3.1).
3. Logica separata dalla UI: tutto ciò che si può testare sta in OrgWorksCore.
4. Solo componenti di sistema: nessun controllo, font o colore proprio, salvo il colore dei calendari, che viene da Google.
5. Costo token zero nell'app: i modelli girano solo quando Oscar li invoca in Claude.

## 2. Architettura

### 2.1 Target

Un solo repo, `/Users/oscar/GIT/OrgWorks` (nuovo, D9).

| Target | Tipo | Contiene | Dipende da |
|---|---|---|---|
| OrgWorksCore | Swift Package, libreria | modelli, SheetsClient, CalendarClient, Store, logica dei task e della matrice, sincronizzazione, parser | Foundation, Security |
| OrgWorks | app macOS SwiftUI | scene, moduli, pannelli, impostazioni | OrgWorksCore |
| orgworks-mcp | eseguibile da riga di comando | server MCP su stdio | OrgWorksCore |

Regole: OrgWorksCore non importa SwiftUI né AppKit. La UI non contiene regole di business. I test di Core girano con `swift test` con `FakeSheets` e `FakeCalendar` in memoria.

Piattaforma: macOS su Apple Silicon. La versione minima si fissa in tappa 1a (D14). Dalla sessione cloud Swift non compila: test, build, commit e merge li lancia Oscar dal suo terminale (GUIDA.md).

### 2.2 Finestre

- Scene dichiarate dall'app: `Window("Calendario", id: "calendario")` e `Window("To do", id: "todo")`. Una sola istanza per modulo [D4]. Menu Finestra di sistema; posizione e dimensione le ricorda macOS.
- Un solo `Store` (`@MainActor`, osservabile) creato all'avvio e iniettato in ogni scena. I moduli non si parlano: leggono e scrivono lo Store.
- Collegamenti tra moduli con `openWindow`: dal To do "Mostra nel calendario" apre il Calendario sulla data del task; dal Calendario "Apri nel To do" porta il task selezionato nel To do.
- Chiusa l'ultima finestra, l'app termina [D4].
- Menu: Aggiorna (⌘R), Impostazioni (⌘,), Nuovo evento o Nuovo task (⌘N, secondo la finestra attiva).

### 2.3 Flusso dei dati

```
Google Sheets   <-> SheetsClient   \
                                     Store (Core: cache + unione) <-> OrgWorks (UI)
Google Calendar <-> CalendarClient /
        ^
        +-- orgworks-mcp (stdio): stessi client, stesse regole <-> Claude Desktop / Code
```

- Sheets e Calendar sono le due fonti di verità. La cache locale è derivata e si può cancellare. Il server MCP legge sempre dalle fonti, mai dalla cache.
- Lo Store unisce eventi e righe in un elenco di Task (3.2, 3.7).

### 2.4 Lettura, cache e aggiornamento

- Sheets: una sola `batchGet` su tutte le schede a ogni ciclo.
- Calendar: `events.list` incrementale per calendario con `syncToken`. Se l'API risponde 410, quel calendario si risincronizza da zero. Eventi ricorrenti espansi (`singleEvents=true`).
- Finestra di eventi in cache: da 3 mesi prima a 12 mesi dopo oggi; fuori da questo intervallo si legge a richiesta (D10). La finestra vale per la sincronizzazione iniziale. Le letture con `syncToken` non accettano `timeMin` e `timeMax`, quindi gli eventi che arrivano fuori dalla finestra si filtrano nel Core (da verificare nello spike).
- Avvio dalla cache con "Agg. gg/mm hh:mm" nella toolbar, poi dalla rete. Cicli: all'avvio, quando l'app torna attiva se sono passati più di 60 secondi, ogni 2 minuti con l'app attiva, a mano con ⌘R.
- Limiti: scritture Sheets accorpate in `batchUpdate`, retry con backoff su 429 e 5xx. Le quote si misurano nello spike. Offline o senza credenziali: moduli in sola lettura dalla cache o "non autorizzato" (sezione 8). Nessuna coda di scrittura in v1.

### 2.5 Scrittura e concorrenza

Scrivono tre soggetti: l'app, orgworks-mcp e Oscar a mano (sul foglio, su Google Calendar web o da telefono). Né Sheets né Calendar hanno transazioni tra le due fonti.

- Righe Sheets sempre per `id`, mai per numero di riga. Compare-and-swap: prima di scrivere si rilegge la riga per `id` e si confronta con la copia letta.
- Eventi: `If-Match` con l'`etag` letto. Una risposta 412 ha lo stesso trattamento: scrittura rifiutata e modulo in sola lettura con "L'elemento è cambiato altrove. Ricarica per modificare." e `Ricarica`. Il server MCP risponde `elemento_cambiato`.
- Le scritture che toccano evento e riga seguono l'ordine di 3.4: un'interruzione lascia al peggio un evento senza riga, che è già uno stato valido (3.2).
- Inserimenti Sheets sempre `append`. Cancellazioni solo dall'interfaccia dell'app, con conferma. Colonne sconosciute lette per nome e mai riscritte.
- Lista bianca: l'app e l'MCP toccano solo il foglio indicato in config.json, solo le schede della sezione 4 e solo i calendari elencati in config.json. Un test lo garantisce.

### 2.6 Autenticazione

Un service account Google per Sheets e Calendar (D2). Scope `spreadsheets` e `calendar`. Il token si ottiene con un JWT firmato RS256 (Security.framework, da verificare nello spike). Nessuna schermata di consenso e nessuna scadenza a sette giorni.

- Foglio: Oscar lo condivide con l'indirizzo del service account come editor.
- Calendari: ogni calendario di cui Oscar è proprietario si condivide con lo stesso indirizzo ("Apporta modifiche agli eventi"); l'app lo aggiunge con `calendarList.insert` dalle Impostazioni (id del calendario). I calendari di cui Oscar non è proprietario funzionano solo se il proprietario li condivide col service account: da verificare nello spike (D3).
- Nome e colore dei calendari si leggono da Google (`summary`, `backgroundColor`).
- Chiave JSON in `~/Library/Application Support/OrgWorks/credenziali.json`, permessi 600, mai nel repo (D7: Keychain se funziona senza richieste continue).

### 2.7 File locali

Tutto in `~/Library/Application Support/OrgWorks/`.

| File | Contenuto | Sacrificabile |
|---|---|---|
| config.json | ID del foglio, calendari elencati e visibili, calendario predefinito per i nuovi eventi | no |
| credenziali.json | chiave del service account (600) | no |
| cache.json | ultima lettura di Sheets e Calendar, con i `syncToken` | sì |

Non c'è un file di disposizione: posizione e dimensione delle finestre le ricorda macOS.

## 3. Task = evento

### 3.1 Principio

Ogni evento di calendario è un task. Ogni task con data è un evento. Un task può non avere data (e allora non è nel calendario). Ogni campo ha una sola fonte.

| Campo | Fonte |
|---|---|
| titolo, data, ora, durata, tutto il giorno, luogo, descrizione, calendario, ricorrenza | l'evento, se il task ne ha uno; altrimenti la riga Task (solo titolo, perché senza evento non ci sono gli altri) |
| stato, priorità, nota, sotto task, occorrenze fatte | il foglio |

La riga Task contiene anche `titolo`: è la fonte per i task senza evento; per gli altri è una copia leggibile, aggiornata dall'app a ogni scrittura e mai letta come verità.

### 3.2 Task materializzati e virtuali

- **Materializzato**: ha una riga nella scheda Task. La riga nasce quando si crea un task dal To do, oppure quando su un evento si assegna o si svuota la priorità, si aggiunge un sotto task, si scrive una nota o si segna fatto.
- **Virtuale**: un evento senza riga. Ha priorità `normale` (derivata, mai scritta) e stato `da_fare`. Il To do lo mostra se non è finito (fine uguale o successiva all'inizio di oggi) e se il suo calendario è visibile. Nessuna riga viene scritta finché non serve. Un evento ricorrente virtuale compare come una riga per serie, con la prima occorrenza da oggi in poi.
- Un evento passato senza riga non compare nel To do. Un evento passato con riga compare finché non è fatto, poi solo con "Mostra fatti".
- **Collegamento**: la riga contiene `calendario_id` ed `evento_id`; l'evento contiene `extendedProperties.private.ow_task_id`. All'aggiornamento, se un evento con `ow_task_id` ha cambiato calendario, si aggiorna `calendario_id`. Se l'`evento_id` non esiste più (evento eliminato fuori dall'app), la riga resta come task senza data, con `evento_id` e `calendario_id` svuotati.

### 3.3 Ricorrenze

Il task di un evento ricorrente è uno per serie: la riga ha l'`evento_id` dell'evento master e `ricorrente=TRUE`. Priorità, nota e sotto task sono della serie.

- Nel To do compare una riga per serie, con la prima occorrenza non fatta da oggi in poi. Le occorrenze passate non fatte si considerano perse e non si accumulano (D12).
- "Fatto" vale per occorrenza: una riga nella scheda Occorrenze (`<task_id>@<data>`). Il task di una serie non passa a `fatto` a ogni occorrenza; con `stato=fatto` sulla serie esce dal To do.
- Nel Calendario una serie è un normale evento ricorrente. Spostare, modificare ed eliminare chiede "Solo questo evento / Tutti gli eventi" (Google: istanza o master). Modificare una singola occorrenza non crea un altro task.

### 3.4 Creazione, cambio di data, eliminazione

- **Creare un task con data** (da To do o da Claude): 1) si genera l'id e si crea l'evento con `ow_task_id`; 2) si scrive la riga con `evento_id`. Se il passo 2 fallisce, l'evento resta come task virtuale.
- **Creare un evento dal Calendario**: solo evento, task virtuale.
- **Dare una data a un task senza data**: crea l'evento, nel calendario predefinito di config.json. Senza ora è un evento di tutto il giorno; con l'ora dura un'ora (D13).
- **Togliere la data a un task con evento**: dialogo "Eliminare anche l'evento dal calendario?" con Annulla / Elimina evento. Senza conferma la data non si toglie.
- **Eliminare un task con evento, o un evento che ha una riga**, dall'app: un solo dialogo con le conseguenze ("Elimina l'evento, N sotto task e la nota"). Ordine: prima sotto task, occorrenze e riga, poi l'evento.
- **Eliminare un evento virtuale**: solo l'evento, con un dialogo semplice.
- **Fatto**: `stato=fatto`, `fatto_il=ora`. L'evento resta nel calendario, mostrato con titolo barrato. Togliere la spunta riporta a `da_fare`.

### 3.5 Priorità e urgenza (matrice di Eisenhower)

- `priorita`: `alta`, `normale` oppure vuota (da classificare). Due valori reali (D5): con tre livelli la matrice non guadagnerebbe nulla. Un task virtuale conta come `normale`: sta in Sbriga o Poi secondo la data. Da classificare ci finiscono solo i task con riga e priorità vuota.
- **Importante** = `alta`.
- **Urgente** (derivato, mai memorizzato) = task non fatto con data di inizio entro `giorni_urgente` giorni da oggi, compresi quelli già passati non fatti. Parametro `giorni_urgente`, default 2. Il conto è in giorni di calendario, non in ore. Un task senza data non è mai urgente.
- **Quadranti** (etichette D11): importante e urgente = Subito; importante non urgente = Pianifica; urgente non importante = Sbriga; né l'uno né l'altro = Poi. I task con priorità vuota stanno fuori dalla matrice, in una colonna "Da classificare".

### 3.6 Sotto task

Un solo livello. Campi: titolo, fatto, ordine. Senza data e senza priorità propria. Il padre si chiude a mano, anche con sotto task aperti, e mostra l'avanzamento "2/5". Aggiungere un sotto task a un task virtuale lo materializza. Eliminare il padre elimina i sotto task.

### 3.7 Calcolo dell'elenco

Funzione pura del Core: `elencoTask(eventi, righe, sottotask, occorrenze, oggi, parametri)` restituisce gli `OWTask` con i campi derivati (virtuale, urgente, quadrante, avanzamento, prossima occorrenza). È la base del To do, degli indicatori nel Calendario e degli strumenti MCP.

## 4. Schede

Foglio dedicato "OrgWorks" (D1), condiviso solo con Oscar e il service account.

Convenzioni:
- Riga 1 = intestazioni (minuscolo, senza accenti, trattino basso), bloccata. Id `<prefisso>_<8 caratteri base36>`: `tsk` task, `sub` sotto task.
- Date `YYYY-MM-DD`; marche temporali ISO 8601 con offset (`2026-11-10T09:12:00+01:00`). Si scrive con `valueInputOption=RAW` in colonne a testo semplice. Il lettore è tollerante (accetta anche `gg/mm/aaaa` e i seriali di Sheets) e riscrive sempre in ISO.
- Booleani `TRUE` o `FALSE` (si legge anche `si`, `1`, `true`; vuoto = falso). Enumerazioni minuscole, senza accenti.
- Ogni scheda ha `creato` e `modificato`, scritte da chi scrive.

**Task**: id, titolo, stato (`da_fare`, `fatto`), priorita (`alta`, `normale`, vuoto), calendario_id, evento_id, ricorrente, nota, fatto_il, creato, modificato.

**Sottotask**: id, task_id, titolo, fatto, ordine, creato, modificato.

**Occorrenze**: chiave (`<task_id>@<data>`), task_id, data, fatto_il. Solo occorrenze fatte; l'assenza significa "da fare".

**Parametri** (chiave e valore): versione_schema (1), giorni_urgente (2), mcp_ultimo_accesso.

Inizializzazione: un comando idempotente del Core crea le schede mancanti con le loro intestazioni e non sovrascrive mai nulla. Lo esegue l'app al primo avvio e `orgworks-mcp --init`.

## 5. Modulo Calendario

Finestra "Calendario". `NavigationSplitView`: barra laterale con i calendari (casella di visibilità e colore di Google), area centrale con la vista. Toolbar: selettore di vista, frecce, "Oggi", ↻ (⌘R), "+".

### 5.1 Viste

- **Anno**: 12 mesi in griglia 4x3, con i giorni che hanno eventi segnati. Clic su un giorno apre il Giorno, clic sul nome del mese apre il Mese. Sola lettura.
- **Mese**: griglia da lunedì a domenica, 5 o 6 righe. Ogni cella elenca gli eventi (colore del calendario, ora, titolo) fino a un limite, poi "+N". Clic su un giorno apre il Giorno.
- **Settimana**: 7 colonne, griglia oraria di 24 ore con scorrimento (si apre già posizionata sull'ora corrente), riga "Tutto il giorno" fissa in alto, linea dell'ora corrente nel giorno di oggi, sfondo diverso nel weekend.
- **Giorno** (agenda): elenco cronologico degli eventi del giorno, non una griglia oraria. Prima "Tutto il giorno", poi per ora. Riga: orario, titolo, pallino del calendario, luogo, avanzamento dei sotto task se ci sono.
- Su ogni evento che è un task: simbolo per la priorità alta, titolo barrato se fatto.
- Settimana da lunedì, lingua italiana, fuso orario del Mac (D10).

### 5.2 Spostare e creare

- **Settimana**: trascinare sposta (scatto da 15 minuti, tra giorni e ore); la maniglia in basso cambia la durata; gli eventi di tutto il giorno si trascinano tra le colonne. **Mese**: trascinare su un altro giorno sposta mantenendo ora e durata. **Giorno** e **Anno**: nessun trascinamento (si modifica dal pannello). Durante il gesto compare un segnaposto con l'orario risultante.
- **Creare**: doppio clic o trascinamento su uno spazio vuoto (Settimana), doppio clic su un giorno (Mese), "+" o ⌘N. Il nuovo evento si apre nel pannello con il calendario predefinito e l'ora e la durata dello spazio scelto (un'ora se viene da "+").
- **Pannello Evento** (`.inspector`, a destra): Titolo, Tutto il giorno, Inizio, Fine, Calendario, Luogo, Descrizione; sezione Task (6.3): Stato, Priorità, Sotto task, Nota; in fondo Elimina e "Apri in Google Calendar". Salvataggio automatico all'uscita da un campo o dopo un secondo di inattività, con "Salvato hh:mm". `Esc` chiude. Lo stesso pannello vale per il To do.
- **Sola lettura**: eventi con invitati o su calendari non scrivibili non si trascinano. Il pannello mostra il tag "Sola lettura" e il testo "Spostarlo da qui manderebbe una notifica agli invitati."
- **Ricorrenti**: dialogo "Solo questo evento / Tutti gli eventi" per spostare, modificare ed eliminare. "Questo e i successivi" è fuori v1.
- **Eliminare**: dialogo di conferma (3.4).

### 5.3 Sincronizzazione a due vie

Modifiche dell'app: scrittura subito con `If-Match`; l'evento si muove subito e torna indietro con un avviso se la scrittura fallisce. Modifiche fatte fuori (Google Calendar web o telefono, Claude via MCP): arrivano al ciclo successivo (2.4). Un evento aperto nel pannello e modificato altrove passa in sola lettura con "Ricarica" (2.5).

## 6. Modulo To do

Finestra "To do". Toolbar: selettore Lista / Matrice, filtro (Tutti, Alta priorità, Da classificare), campo di ricerca sul titolo, interruttore "Mostra fatti", "+".

### 6.1 Lista

Sezioni, in quest'ordine: Da classificare (priorità vuota), In ritardo (data passata, non fatti), Oggi, Prossimi (data futura), Senza data. Un task compare in una sola sezione e "Da classificare" ha la precedenza. Ordine nella sezione: per data e ora; i senza data per `creato`.

Riga: casella (fatto), titolo, data relativa (oggi, domani, gg/mm), pallino del calendario, simbolo della priorità alta, avanzamento dei sotto task, simbolo di ricorrenza. Clic apre il pannello Task. "Mostra fatti" aggiunge la sezione Fatti (ultimi 30 giorni).

### 6.2 Matrice

Quattro quadranti (3.5) in griglia 2x2 e, a sinistra, la colonna "Da classificare". Ogni scheda: titolo, data, avanzamento. Trascinare:

- tra le due righe (importante e non importante) cambia `priorita` (`alta` e `normale`);
- dalla colonna "Da classificare" a un quadrante assegna la priorità della riga; la colonna urgente o non urgente non si sceglie, perché l'urgenza è derivata e il task resta dove la data lo mette;
- verso la colonna "Da classificare" svuota la priorità.

Un trascinamento che richiederebbe di cambiare urgenza (da una colonna all'altra) viene ignorato con il segnale di sistema.

### 6.3 Creazione e pannello Task

- "+" apre una riga di inserimento rapido: Titolo, invio per creare. Il task nasce senza data, con priorità vuota (da classificare).
- Il pannello Task è lo stesso del Calendario (5.2). Aggiungendo Data, Ora, Durata e Calendario il task diventa un evento (3.4) e compare subito nel Calendario.
- Sotto task: elenco modificabile (aggiungi, riordina, spunta, elimina).
- Elimina: dialogo di conferma (3.4).

## 7. Server MCP

### 7.1 Trasporto e installazione

- Eseguibile `orgworks-mcp`, trasporto stdio. Su stdout passa solo il protocollo; i log vanno su stderr.
- Registrazione in Claude Desktop (`claude_desktop_config.json`, voce `mcpServers.orgworks`) e in Claude Code (`claude mcp add orgworks -- <percorso>`). Percorso previsto: `/Users/oscar/GIT/OrgWorks/bin/orgworks-mcp` (D8). Implementazione: SDK Swift ufficiale di MCP oppure JSON-RPC minimale, scelta nello spike (D8).
- Usa config.json e credenziali.json come l'app, funziona con l'app chiusa e scrive sia su Sheets sia su Calendar con le stesse regole del Core. Non è raggiungibile dalle sessioni cloud di Claude, solo da quelle sul Mac.
- Strumenti di lettura con `readOnlyHint`. Risposte in italiano; ogni scrittura restituisce l'elemento così come è stato scritto. Aggiorna `mcp_ultimo_accesso` al massimo una volta al minuto.

### 7.2 Strumenti

Gli `elenca_*` hanno filtri e un limite di 50 righe.

| Strumento | Tipo | Effetto |
|---|---|---|
| oggi | lettura | eventi di oggi, task non fatti con data oggi o in ritardo, conteggio dei task da classificare |
| elenca_calendari | lettura | calendari con id, nome, colore, scrivibile, visibile |
| elenca_eventi | lettura | per intervallo e calendario; ogni evento con i dati del task se materializzato |
| crea_evento | scrittura | titolo, inizio, fine o tutto_il_giorno, calendario, luogo, descrizione; mai invitati; task virtuale |
| aggiorna_evento | scrittura | sposta o modifica; `ambito` istanza o serie per i ricorrenti; rifiuta eventi in sola lettura (`non_consentito`) |
| elenca_task | lettura | filtri: stato, priorità, da_classificare, con_data, intervallo |
| leggi_task | lettura | task con sotto task, evento collegato, urgenza, quadrante, descrizione dell'evento |
| crea_task | scrittura | titolo, priorità (vuota se non indicata), nota, sotto task, data, ora e calendario opzionali (con data crea l'evento, 3.4) |
| aggiorna_task | scrittura | titolo, priorità, stato, nota; data e ora (crea o sposta; togliere la data non è consentito, 7.4) |
| completa_task | scrittura | per i ricorrenti accetta la data dell'occorrenza |
| aggiungi_sottotask, aggiorna_sottotask | scrittura | titolo, fatto, ordine |
| leggi_parametri, imposta_parametro | lettura, scrittura | solo `giorni_urgente` |

Gli strumenti dei task funzionano anche su un evento virtuale: lo materializzano quando serve.

### 7.3 Cosa fa solo l'app

Cancellazioni (task, sotto task, eventi), togliere la data a un task con evento, gestire i calendari (aggiungerli o toglierli dalla lista), importare le credenziali.

### 7.4 Autonomia e cancellazione

Le esecuzioni non presidiate (task pianificate di Claude Desktop) usano solo gli strumenti di lettura. Tutto il resto solo su richiesta esplicita di Oscar in una conversazione. Nessuno strumento cancella nulla: né eventi, né task, né sotto task.

### 7.5 Errori

In italiano, con codice: `elemento_cambiato` (riga o evento modificati da altri: rileggere), `non_trovato`, `non_consentito` (sola lettura, invitati, cancellazione), `non_autorizzato` (credenziali, o calendario non condiviso col service account), `quota` (limite persistente di Sheets o Calendar). Nei log mai il contenuto delle descrizioni.

## 8. Stati dei moduli

Cinque stati, per modulo e indipendenti tra moduli, con componenti di sistema.

1. **Caricamento**: `ProgressView` con "Leggo i dati ...".
2. **Vuoto**: `ContentUnavailableView` con l'azione che lo risolve.
3. **Errore**: messaggio con `Riprova`; gli altri moduli continuano.
4. **Non autorizzato**: messaggio e `Apri Impostazioni`.
5. **Sola lettura**: banner con `Ricarica`. Vale anche dopo una scrittura rifiutata (2.5).

## 9. Impostazioni e avvio

- Finestra Impostazioni (⌘,, scena `Settings`) con quattro pannelli: **Fonti** (foglio: id e stato; credenziali: importa il file; verifica dell'accesso), **Calendari** (elenco, visibilità, calendario predefinito, aggiungi per id), **Matrice** (`giorni_urgente`), **Claude** (percorso del server MCP, ultimo accesso da `mcp_ultimo_accesso`).
- Primo avvio: scelta del foglio (ID), import del file delle credenziali, verifica dell'accesso, inizializzazione delle schede (sezione 4), aggiunta dei calendari, con il promemoria di condividerli prima con l'indirizzo del service account.

## 10. Sicurezza e privacy

- L'app e orgworks-mcp parlano solo con le API di Google (Sheets e Calendar). Nessuna telemetria, nessun altro servizio.
- Foglio e calendari contengono dati personali e di lavoro: il foglio lo condividono solo Oscar e il service account.
- La chiave del service account non entra mai nel repo (`.gitignore`), nei log o negli output. Il server MCP è locale su stdio e non apre porte.
- Le risposte degli strumenti MCP passano nella sessione di Claude: ogni strumento restituisce solo i campi necessari, con filtri e limiti. La descrizione degli eventi si restituisce solo da `leggi_task`.

## 11. Decisioni prese il 10/11/2026

- Metafora della scrivania come finestre indipendenti della stessa app; solo componenti di sistema; due moduli (Calendario, To do); server MCP; dati in Google Sheets; connessione a Google Calendar.
- Calendar API con lo stesso service account di Sheets, così Claude può spostare e creare eventi via MCP.
- Giorno e agenda sono la stessa vista. Eventi con invitati e su calendari non scrivibili in sola lettura. Ricorrenti: dialogo "solo questo / tutti".
- Ogni evento è un task; ogni task ha sotto task (un livello) e priorità; matrice di Eisenhower con importante = priorità e urgente derivato dalla data.
- Un evento creato fuori dall'app è un task virtuale: la riga nasce solo quando serve (3.2).
- Priorità e sotto task sulla serie per gli eventi ricorrenti; "fatto" per occorrenza (3.3).
- Task con data ma senza ora: evento di tutto il giorno.
- I task virtuali contano come priorità `normale`; il task creato al volo dal To do nasce da classificare (priorità vuota).
- Nel codice il task si chiama `OWTask`, per non scontrarsi con `Task` della concorrenza di Swift.

## 12. Decisioni aperte

"Assunta" = Oscar ha accettato la proposta del 10/11 e la spec la dà per buona.

| # | Tema | Stato | Come si chiude |
|---|---|---|---|
| D1 | Foglio dedicato "OrgWorks" invece di schede nuove nel foglio del Finance Tracker | proposta: dedicato (nessun rischio per la finanza, lista bianca banale) | Oscar conferma |
| D2 | Un solo service account per Sheets e Calendar | assunta | spike: lettura e scrittura di una cella, firma JWT RS256 con Security.framework |
| D3 | Calendari di cui Oscar non è proprietario, leggibili dal service account | aperta | spike; se non funziona, quei calendari non compaiono o serve la condivisione del proprietario |
| D4 | Una istanza per finestra; l'app termina all'ultima finestra chiusa | proposta | Oscar |
| D5 | Priorità a due valori; `giorni_urgente` = 2 | assunta | si regola dalle Impostazioni |
| D6 | L'app non crea né modifica regole di ricorrenza (1.2) | proposta | Oscar, se serve creare ricorrenze dall'app |
| D7 | Credenziali: file con permessi 600 o Keychain | proposta: file | spike |
| D8 | Server MCP: SDK ufficiale o implementazione minima; percorso del binario | aperta | spike |
| D9 | Nome del repo e della cartella | chiusa il 10/11: `OrgWorks` in `/Users/oscar/GIT`, separato dal repo `agenda` | |
| D10 | Settimana da lunedì, 24 ore con scorrimento, scatto da 15 minuti, finestra di cache da -3 a +12 mesi | proposta | prima della tappa 5 |
| D11 | Etichette dei quadranti: Subito, Pianifica, Sbriga, Poi | proposta | Oscar |
| D12 | Le occorrenze passate non fatte di una serie si considerano perse | proposta | Oscar |
| D13 | Un task con data e ora dura un'ora; senza ora è di tutto il giorno | proposta | Oscar |
| D14 | Versione minima di macOS | aperta | tappa 1a |

## 13. Verifica

### 13.1 Test automatici di OrgWorksCore

Con `FakeSheets` e `FakeCalendar`; i valori attesi si calcolano a mano dall'input.

- Lettura tollerante di date (ISO, `gg/mm/aaaa`, seriali), booleani e liste; scrittura sempre ISO; generazione degli id.
- Ricerca delle righe per id dopo riordino del foglio; colonne sconosciute preservate; compare-and-swap con riga cambiata; lista bianca di schede e calendari.
- CalendarClient: mapping degli eventi (`date` contro `dateTime`, fusi, tutto il giorno), `syncToken` e risposta 410, `If-Match` e 412, `extendedProperties`, istanza contro serie.
- `elencoTask`: task virtuali (solo futuri o di oggi, priorità `normale`), materializzati, ricorrenti (una riga per serie, prima occorrenza non fatta), evento eliminato fuori dall'app, evento passato in un altro calendario.
- Urgenza e quadranti ai confini: oggi, oggi più 2, oggi più 3, passato, senza data.
- Sotto task: avanzamento, materializzazione del padre.
- Operazioni composte (3.4): creazione con data, dare e togliere la data, eliminazione, fatto; ordine delle scritture e fallimento a metà (FakeCalendar che fallisce al secondo passo).
- Effetti del trascinamento in matrice sulla priorità; trascinare un virtuale nella Matrice lo materializza; trascinamento tra colonne ignorato.
- Strumenti MCP con i fake; nessuno strumento cancella.

### 13.2 Spike della tappa 0 (verifica manuale)

1. Lettura e scrittura di una cella del foglio da Swift con il service account (firma JWT RS256).
2. Calendar API: `calendarList.insert` di un calendario condiviso; `events.list` con `syncToken` (con una finestra `timeMin`/`timeMax` iniziale) e risposta 410; `patch` con `If-Match` e risposta 412; `extendedProperties.private` scritta e cercata; spostamento di un'istanza di una serie; un calendario di cui Oscar non è proprietario (D3); cosa succede su un evento con invitati.
3. `orgworks-mcp` registrato in Claude Desktop con uno strumento visibile e richiamabile, che scrive un evento.
4. Quote di Sheets e Calendar in uso reale; Keychain condiviso tra app e orgworks-mcp.

### 13.3 Checklist M per modulo (5 minuti ciascuna, elencata nella voce di TAPPE.md)

- **Calendario**: apertura e stati (caricamento, vuoto, errore, sola lettura); le quattro viste con dati reali; creare ed eliminare un evento; spostarne uno e controllare su Google Calendar web; ricorrente con dialogo; evento con invitati in sola lettura; modifica dello stesso evento da Google mentre l'app è aperta.
- **To do**: lista con le sezioni; creare un task con e senza data (con data compare nel Calendario); sotto task; matrice con trascinamento; "Da classificare"; fatto e titolo barrato nel Calendario; eliminazione con conferma.

## 14. Nomi nel codice

| Termine | Nome nel codice |
|---|---|
| Task (evento materializzato o virtuale) | `OWTask` |
| Sotto task | `Sottotask` |
| Occorrenza fatta | `Occorrenza` |
| Evento di Google Calendar | `EventoCalendario` |
| Stato condiviso dell'app | `Store` |
| Calcolo dell'elenco | `elencoTask` |
| Quadrante della matrice | `Quadrante` |

Non `Task`, che in Swift è il tipo della concorrenza.
