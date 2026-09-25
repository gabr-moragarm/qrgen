# Roadmap di apprendimento — qrcode-generator

Percorso a tappe per imparare Go costruendo un generatore di QR code da terminale, guidato da un wizard in tre step: "1. scegli il contenuto", "2. inserisci i dati", "3. scegli il formato".

Vincoli:

- Una libreria QR esterna fornisce solo la **matrice dei moduli**. Il rendering (PNG, SVG, terminale) si scrive a mano con la stdlib.
- Il wizard si costruisce prima con **huh**. **Bubble Tea** scritto a mano arriva solo quando serve davvero (Tappa 6).
- Il dominio (contenuti e renderer) non deve dipendere dalla UI.

Livello di partenza (da go-pong, da confermare): buona conoscenza di struct e interface, poca esperienza con goroutine e channel.

Obiettivi trasversali: testing, design con interfacce, packaging di una CLI, implementazione corretta di uno standard.

## Tappa 0 — Setup e struttura del modulo

Obiettivo: avere un modulo Go ben organizzato che compila e si esegue.

- `go mod init` e scelta del module path (es. `github.com/gabr-moragarm/qrcode-generator`, da decidere).
- Layout: `cmd/qrgen/` per l'eseguibile e `internal/` per i package del dominio. Capire cosa impedisce `internal` e perché conviene.
- Nomi dei package: brevi, al singolare, senza `utils`/`common`.
- `git init`, `.gitignore`, `gofmt`, `go vet`, `go build`/`go run`.

## Tappa 1 — Il dominio vCard (senza UI)

Obiettivo: data una struct con i dati del biglietto da visita, produrre una stringa vCard 3.0 valida.

- Leggere le parti rilevanti di RFC 2426 (vCard 3.0): `BEGIN`/`END`, `VERSION`, `N`, `FN` (entrambi obbligatori nella 3.0), `ORG`, `TITLE`, `TEL;TYPE=...`, `EMAIL`, `URL`, `ADR`, `NOTE`.
- Dettagli dello standard: fine riga CRLF, escaping di `\`, `,`, `;` e dei newline, line folding a 75 ottetti (attenzione ai caratteri UTF-8 multibyte: non spezzarli).
- Modellare i campi facoltativi: zero value (`""`), puntatore o flag di presenza? Pro e contro di ciascuno.
- Tenere presente che in futuro arriverà anche la vCard 4.0 (Tappa 8): quali parti del codice dipendono dalla versione e quali no?
- `strings.Builder` e perché è preferibile alla concatenazione.
- Validazione ed errori: `errors.New`, errori sentinella, `fmt.Errorf` con `%w`, `errors.Is`/`errors.As`, `errors.Join` per riportare più errori insieme.
- Test: **table-driven test** con `t.Run`, `t.Helper`, **golden file** in `testdata/` aggiornabili con un flag `-update`.

## Tappa 2 — Astrazioni e libreria QR

Obiettivo: separare "cosa codifico" da "come lo disegno", e ottenere la matrice del QR.

- Definire un'interfaccia `Content` (qualcosa che produce il payload testuale da codificare), pensando già a URL, testo e WiFi.
- Confrontare le librerie QR (es. `skip2/go-qrcode`, che espone `Bitmap() [][]bool`, e `yeqown/go-qrcode`). Scegliere quella che dà accesso alla matrice dei moduli.
- Principi: interfacce piccole, "accept interfaces, return structs", l'interfaccia la definisce chi la usa, direzione delle dipendenze.
- Livelli di correzione d'errore (L/M/Q/H): come influenzano la densità del QR e perché una vCard con tanti campi diventa difficile da leggere.
- Gestione delle dipendenze: `go get`, `go.sum`, `go mod tidy`.

## Tappa 3 — I renderer (terminale, SVG, PNG)

Obiettivo: disegnare la stessa matrice in tre formati diversi dietro un'unica interfaccia.

- Interfaccia `Renderer` che scrive su un `io.Writer`: perché `io.Writer` e non un path o un `[]byte`.
- **Terminale**: half-block Unicode (`▀`, `▄`, `█`, spazio) per mettere due righe di moduli in una riga di testo, quiet zone (margine bianco), terminali con sfondo scuro.
- **SVG**: generazione testuale (`fmt` o `text/template`), `viewBox`, un `<rect>` per modulo oppure un unico `<path>`, e come cambia la dimensione del file.
- **PNG**: `image`, `image/color`, `image/png`, scala in pixel per modulo.
- Registry dei formati (es. `map[string]Renderer`): aggiungere un formato senza toccare il resto del codice.
- Test: golden file per terminale e SVG. Per il PNG, decodificare l'immagine generata e verificarne dimensioni e colori di alcuni pixel.

## Tappa 4 — Il wizard con huh

Obiettivo: il flusso completo contenuto → dati → formato, che scrive il file.

- Step 1: `Select` del tipo di contenuto. Per ora solo vCard, ma già estendibile.
- Step 2: nome e cognome obbligatori, `MultiSelect` dei campi facoltativi, poi un secondo form costruito dinamicamente con i soli campi scelti.
- Step 3: scelta del formato e del percorso di output (per il terminale: stampa a schermo).
- Concetti: generics (`huh.NewSelect[T]`), API fluent/builder, puntatori come destinazione dei valori (`Value(&x)`), closure per la validazione (`Validate(func(string) error)`), gestione di `huh.ErrUserAborted` (Ctrl+C).
- Separazione UI/dominio: il wizard produce solo un `Content`, un formato e una destinazione. Generazione e scrittura avvengono fuori dalla UI.
- Test: la costruzione dei campi e il mapping delle risposte verso il dominio devono essere testabili senza terminale.

## Tappa 5 — Packaging della CLI

Obiettivo: un eseguibile installabile e con un comportamento prevedibile.

- `main` sottile: `func run() error` con tutta la logica, `os.Exit` solo in `main`, exit code significativi.
- stdout vs stderr: cosa va dove, soprattutto quando il QR viene stampato nel terminale.
- Versione iniettata a build time con `-ldflags "-X main.version=..."`.
- `go install`, `golangci-lint`, eventuale `Makefile`, eventuale goreleaser per le release multipiattaforma.

## Tappa 6 — Bubble Tea: anteprima live e schermata finale

Obiettivo: controllare tutto lo schermo, non solo il form, per aggiungere ciò che huh da solo non sa fare.

- Architettura Elm: `Model`, `Update`, `View`. Type switch su `tea.Msg`, receiver per valore e perché `Update` restituisce un nuovo model.
- Un model proprio che **incorpora** l'`huh.Form` (che implementa `tea.Model`) e gli inoltra i messaggi.
- Layout con Lip Gloss: form, pannello di anteprima e barra degli step ("① Contenuto → ② Dati → ③ Formato").
- Anteprima live del QR che riusa il renderer terminale della Tappa 3.
- `tea.Cmd` e `tea.Tick`: rigenerare l'anteprima con un debounce invece che a ogni tasto. Bubble Tea esegue i comandi in goroutine: capire cosa succede sotto senza gestire direttamente channel e mutex.
- Schermata finale interattiva: salva anche in un altro formato, ricomincia, esci.
- Test con `teatest`.
- Verifica del design: il passaggio da huh a Bubble Tea non deve richiedere modifiche al dominio.

## Tappa 7 (opzionale) — Nuovi tipi di contenuto

Obiettivo: verificare che le astrazioni reggano.

- URL e testo semplice.
- WiFi (`WIFI:T:WPA;S:<ssid>;P:<password>;;`) con le sue regole di escaping.
- Se aggiungere un contenuto richiede di modificare renderer o wizard in punti inattesi, ridiscutere il design di `Content`.

## Tappa 8 (opzionale) — Supporto vCard 4.0

Obiettivo: serializzare gli stessi dati del biglietto da visita anche in vCard 4.0 (RFC 6350), senza duplicare il dominio.

- Leggere le differenze principali con la 3.0: solo `FN` obbligatorio, UTF-8 obbligatorio, `TEL` come URI (`VALUE=uri:tel:...`), parametro `PREF=1` al posto di `TYPE=PREF`, nuove proprietà (`KIND`, ecc.).
- Design: dati di dominio unici e due serializzatori. Dove vive la scelta della versione: nel wizard o come impostazione fissa?
- Riusare la logica comune (escaping, line folding, CRLF) invece di copiarla.
- Test: golden file per entrambe le versioni a partire dagli stessi dati.
- Verifica pratica: scansionare con più telefoni lo stesso biglietto generato in 3.0 e in 4.0 e confrontare quali campi vengono importati correttamente.

## Note

- Vedi `CLAUDE.md` per il ruolo dell'agente AI in questo progetto: guida/spiega, non scrive codice al posto dell'utente.
- Procedere una tappa alla volta; tornare a discutere dubbi concettuali o revisione del codice già scritto prima di passare alla tappa successiva.
