# OrgWorks - DESIGN

Versione 0.2 del 2026-11-10. Il comportamento e i dati stanno in SPEC.md; qui solo l'aspetto. Quando i due documenti divergono, SPEC.md vince sul comportamento, DESIGN.md sull'aspetto.

## 1. Principio

Solo componenti di sistema (SwiftUI nativo). Nessun token, font, bordo, ombra o animazione personalizzati. L'app segue l'aspetto di sistema: chiaro e scuro, colore d'accento, dimensione del testo, "Riduci movimento". Il design system Production OF e tutto DESIGN.md v0.1 di agenda decadono (restano nel repo `agenda`, commit 86fe511).

L'unico colore "proprio" è quello dei calendari, che arriva da Google (`backgroundColor`). Il significato non dipende solo dal colore: priorità, fatto, sola lettura hanno sempre anche un simbolo o un testo.

## 2. Finestre

| Finestra | Dimensione predefinita | Minima |
|---|---|---|
| Calendario | 1100 x 720 | 800 x 520 |
| To do | 900 x 640 | 640 x 480 |
| Impostazioni | scena `Settings`, pannelli a schede | |

Barra del titolo standard con toolbar unificata. Posizione e dimensione le ricorda macOS.

## 3. Componenti di sistema usati

- `NavigationSplitView` per il Calendario (barra laterale dei calendari e area centrale).
- `List` con sezioni per la Lista del To do, Giorno e barra laterale.
- `Picker` segmentato per le viste (Anno, Mese, Settimana, Giorno; Lista, Matrice).
- `.inspector` (larghezza 320-400 pt) per il pannello Evento e Task.
- `.searchable` per la ricerca sul titolo, `Toggle`, `Menu`, `DatePicker`, `TextField`.
- `.confirmationDialog` e `.alert` per i dialoghi (eliminazioni, "Solo questo evento / Tutti gli eventi", togliere la data).
- `ContentUnavailableView` per gli stati vuoti, `ProgressView` per il caricamento.
- Griglie di Anno, Mese, Settimana e matrice con `Grid`, `LazyVGrid` e `Canvas` solo dove serve il disegno (blocchi degli eventi, linea dell'ora).

## 4. Simboli

SF Symbols, tratto di sistema:
- priorità alta: `exclamationmark.circle.fill`
- fatto: `checkmark.circle.fill` (casella: `circle` e `checkmark.circle.fill`)
- ricorrente: `repeat`
- sola lettura: `lock.fill`
- link a Google Calendar: `arrow.up.right.square`
- aggiorna: `arrow.clockwise`
- aggiungi: `plus`

## 5. Calendario

- Blocco evento (Settimana, Mese): riempimento del colore del calendario con opacità ridotta, striscia piena a sinistra da 3 pt, titolo con testo a contrasto (si sceglie testo chiaro o scuro dalla luminanza del colore). Fatto: titolo barrato e opacità 0.6. Sola lettura: simbolo `lock.fill` accanto al titolo, nessuna maniglia.
- Settimana: altezza di un'ora fissa (48 pt), etichette delle ore a sinistra, linea dell'ora corrente nel colore d'accento di sistema.
- Giorno (agenda): `List` con riga a due linee, orario in `.secondary`, titolo, luogo.
- Segnaposto di trascinamento: blocco semi trasparente con l'orario risultante in `.caption`.

## 6. To do

- Lista: riga con casella, titolo, meta in `.secondary` (data relativa, avanzamento, simboli). Altezza minima della riga 32 pt.
- Matrice: griglia 2x2 con le etichette dei quadranti (Subito, Pianifica, Sbriga, Poi) come intestazione di ciascuno, colonna "Da classificare" a sinistra. Schede con titolo, data e avanzamento; nessun colore proprio per i quadranti.

## 7. Tastiera e accessibilità

- Tasti: Esc chiude il pannello, ⌘R aggiorna, ⌘N nuovo evento o task, ⌘, apre le Impostazioni, frecce e ⌘ frecce per navigare nelle viste del Calendario (da definire in tappa 5).
- Ogni controllo ha un'etichetta di accessibilità in italiano e un ordine di focus logico.
- Bersagli minimi come da standard di sistema; "Riduci movimento" rispettato (nessuna animazione oltre quelle di sistema).
