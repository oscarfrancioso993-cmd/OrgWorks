# OrgWorks - GUIDA (per Oscar, non per Code)

Cosa fare a mano a ogni passaggio. Code non deve leggere questo file. Si aggiorna alla chiusura di ogni sotto-tappa.

## 1. Stato

- Ultima tappa chiusa: nessuna (SPEC v0.3, DESIGN, CLAUDE, TAPPE e GUIDA v0.2).
- Prossima: 0a (spike di Sheets e Calendar API).
- Barra settimanale all'ultima chiusura: da annotare.

Il repo è `/Users/oscar/GIT/OrgWorks` (git già inizializzato, nessun commit ancora: il primo commit contiene i cinque documenti).

## 2. Procedura per ogni sotto-tappa

Sostituisci `<n>` con il numero della sotto-tappa (per esempio `0a`).

| # | Passo | Cosa fare |
|---|---|---|
| 1 | Pulizia | `cd /Users/oscar/GIT/OrgWorks && git status -s`. Deve essere vuoto. Se non lo è, committa o metti da parte (`git stash`) prima di andare avanti. |
| 2 | Branch | `git switch main && git switch -c tappa-<n>` |
| 3 | Sessione nuova | Apri `claude` dalla radice del repo, oppure `/clear` se la sessione è già aperta. |
| 4 | Modello | `/model` e scegli quello indicato nella voce di TAPPE.md (O = Opus, S = Sonnet, H = Haiku). |
| 5 | Costo iniziale | Annota la barra settimanale prima di cominciare. |
| 6 | Tabella dei casi (solo voci con T) | Prima del prompt, scrivi (o fatti scrivere da Opus) la tabella degli input e dei risultati attesi, calcolati a mano. Non farla derivare dal codice. |
| 7 | Prompt | Incolla la voce di TAPPE.md. Aggiungi: "Leggi solo le sezioni di SPEC e DESIGN citate nella voce. Non toccare nulla fuori dalla voce. Prima di modificare, elenca i file che toccherai." Per la UI aggiungi il file dove si decide il comportamento. |
| 8 | Leggi l'elenco | Se tocca file che non ti aspetti, fermalo e correggi il prompt. |
| 9 | Esecuzione | Lascia lavorare Code. |
| 10 | Test | `swift test`. Con lo stesso errore due volte di seguito: `/model` al modello superiore, non riprovare un terzo giro con lo stesso. |
| 11 | Revisione del diff | `git diff main --stat`, poi `git diff main`. Controlla che i test non siano stati indeboliti, che non ci siano costanti al posto di valori dinamici, che non compaiano chiavi o ID del foglio. Il riepilogo di Code non basta. |
| 12 | Checklist M | Se la voce ha una M, esegui la checklist (5 minuti) e spunta ogni punto. |
| 13 | Aggiorna | In TAPPE.md compila il registro (modello, barra prima e dopo, esecuzioni fino al verde); in CLAUDE.md aggiorna "Tappe fatte"; in questa guida aggiorna lo Stato. |
| 14 | Commit | `git add -A && git commit -m "Tappa <n>: <titolo della voce>"` |
| 15 | Merge | `git switch main && git merge --ff-only tappa-<n>`. Se rifiuta, non forzare: significa che main è cambiato, riparti dal passo 1. |
| 16 | Chiusura | Annota la barra settimanale dopo. `git branch -d tappa-<n>` cancella solo il branch locale già fuso, i commit restano in main. |

Dopo tre sotto-tappe chiuse rivedi le assegnazioni dei modelli in TAPPE.md.

## 3. Prossima sotto-tappa: 0a

- Branch: `tappa-0a`. Modello: Opus. Verifica: M (punti 1, 2 e 4 di SPEC 13.2).
- Questa tappa tocca account Google reali: la chiave JSON del service account dà accesso al foglio e ai calendari condivisi. Non incollarla in chat, non metterla nel repo e non lasciarla in `Spike/`.

Da preparare a mano prima di aprire Code:

| # | Passo | Cosa fare |
|---|---|---|
| 1 | Progetto Google Cloud | Console Google Cloud: crea un progetto "orgworks" (o usane uno esistente). |
| 2 | API | Abilita "Google Sheets API" e "Google Calendar API" nel progetto. |
| 3 | Service account | IAM e amministrazione, Account di servizio: crea "orgworks" senza ruoli. Annota l'indirizzo (`...@....iam.gserviceaccount.com`). |
| 4 | Chiave | Nello stesso account di servizio crea una chiave JSON. Salvala in `~/Library/Application Support/OrgWorks/credenziali.json` e lancia `chmod 600 ~/Library/Application\ Support/OrgWorks/credenziali.json`. |
| 5 | Foglio | Crea un foglio Google vuoto "OrgWorks". Condividilo con l'indirizzo del service account come Editor. Annota l'ID (nell'URL, tra `/d/` e `/edit`). |
| 6 | Calendario di prova | Crea un calendario nuovo "OW prova" di cui sei proprietario (non il principale). Impostazioni del calendario, "Condividi con persone o gruppi specifici": aggiungi l'indirizzo del service account con "Apporta modifiche agli eventi". Annota l'ID (Impostazioni del calendario, "Integra il calendario"). |
| 7 | Calendario non tuo (per D3) | Scegli un calendario di cui non sei proprietario (per esempio uno condiviso da altri) e annotane l'ID. Il test dirà se il service account lo vede: se non lo vede, non rompere nulla, annota solo l'esito. |
| 8 | Dati a portata | Tieni pronti: ID del foglio, indirizzo del service account, ID del calendario di prova, ID del calendario non tuo. Passali a Code nel prompt, non nei file del repo. |

Poi parti dalla procedura della sezione 2, dal passo 1.

## 4. Dopo la tappa 0

Con gli esiti dello spike in mano si aggiorna SPEC.md (sezione 12: D2, D3, D7, D8) prima di aprire la tappa 1. Se D3 va male (calendari non propri non leggibili), decidi se per quei calendari condividere dal proprietario o rinunciarvi: cambia solo la sezione 2.6.
