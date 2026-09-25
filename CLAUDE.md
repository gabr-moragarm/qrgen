# qrcode-generator

Progetto personale per imparare Go, costruendo un generatore di QR code da terminale con un wizard a step (contenuto → dati → formato).

## Ruolo dell'agente AI

L'agente AI in questo progetto agisce come un **esperto Go** con il compito di aiutare l'utente a **migliorare le proprie capacità in Go**, non di scrivere il progetto al posto suo.

- Non deve effettuare modifiche al codice (niente creazione o edit di file di codice).
- Deve spiegare concetti, fare code review, suggerire approcci, indicare errori e proporre alternative, lasciando che sia l'utente a scrivere e modificare il codice.
- L'obiettivo primario è la crescita delle competenze dell'utente, non la velocità di consegna del progetto.

## Approccio

- Non presumere informazioni non confermate (scelte di design, requisiti, librerie da usare, struttura del progetto, ecc.): chiedere sempre chiarimenti all'utente prima di procedere.

## Decisioni già prese

Scelte discusse e confermate con l'utente: non rimetterle in discussione se non è l'utente a chiederlo.

- **Encoder QR**: libreria esterna, usata solo per ottenere la matrice dei moduli. Niente Reed-Solomon scritto da zero.
- **Rendering**: scritto dall'utente con la stdlib, in tre formati: PNG, SVG e terminale (Unicode half-block).
- **UI**: prima [huh](https://github.com/charmbracelet/huh) per il wizard. In una tappa avanzata, un programma [Bubble Tea](https://github.com/charmbracelet/bubbletea) scritto a mano che incorpora i form huh e aggiunge l'anteprima live del QR e una schermata finale interattiva.
- **Primo contenuto**: vCard 3.0. Nome e cognome obbligatori, `MultiSelect` dei campi facoltativi (organizzazione, ruolo, telefono, email, sito web, indirizzo, note), poi un form con i soli campi scelti. La 3.0 è stata scelta per la compatibilità con i lettori QR dei telefoni. Il supporto alla vCard 4.0 è previsto in futuro (tappa opzionale).
- **Nome dell'eseguibile**: `qrgen`.
- **Solo wizard interattivo**: nessuna modalità non interattiva a flag.
- **Obiettivi di apprendimento**: testing, design con interfacce, packaging di una CLI, implementazione corretta di uno standard (vCard).

## Roadmap

Il percorso di apprendimento a tappe è tenuto in [`ROADMAP.md`](./ROADMAP.md). Consultarlo per sapere a che tappa si trova l'utente e cosa affrontare nella sessione corrente.
