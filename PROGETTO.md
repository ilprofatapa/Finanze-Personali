# Finanze Personali — scheda di progetto

App personale di contabilità in **partita doppia** (ispirata a Cashfulness) con inserimento da foto/screenshot via AI e assistente integrato.
Questo file serve per riprendere il lavoro in futuro: incollalo (o allegalo) in una nuova chat con Claude e chiedi la modifica che ti serve.

---

## 1. Architettura

| Pezzo | Dove sta | Cosa fa |
|---|---|---|
| **Google Sheet "Finanze Personali"** | Google Drive | È il database: piano dei conti, prima nota, regimi fiscali, guida, ammortamenti, config, bilancio, riepilogo fiscale |
| **Apps Script** (`Codice.gs` + `Form.html`) | Foglio → Estensioni → Apps Script | Logica contabile, menu 💰 Finanze, API per l'app, chiamate a Gemini |
| **App web (PWA)** | GitHub `ilprofatapa/Finanze-Personali` → GitHub Pages | Interfaccia per telefono e computer |
| **Gemini** (Google AI Studio, piano gratuito) | Chiave salvata nelle *Proprietà script* (`GEMINI_API_KEY`) | Lettura foto/screenshot e assistente |

Indirizzo app: `https://ilprofatapa.github.io/Finanze-Personali/`

Flusso: App → `POST` all'URL `/exec` dell'Apps Script (body JSON `{token, azione, dati}`, content-type text/plain) → lo script legge/scrive il foglio e, se serve, chiama Gemini → risponde in JSON.

Sicurezza: il token API (Proprietà script `API_TOKEN`, visibile da *💰 Finanze → Mostra token API*) protegge l'API. URL e token stanno solo nel `localStorage` di ogni dispositivo. La chiave Gemini non esce mai dallo script.

---

## 2. File

**Apps Script**
- `Codice.gs` (versione attuale **1.6**): tutto il backend.
- `Form.html`: pannello "Nuova transazione" dentro Google Sheets (menu 💰 Finanze).

**Repository GitHub (PWA)**
- `index.html`: tutta l'app (HTML + CSS + JS in un file, nessuna libreria esterna).
- `manifest.webmanifest`, `sw.js` (service worker, cache `finanze-v5`), `icon-192.png`, `icon-512.png`, `icon-maskable-512.png`, `LEGGIMI.md`.

---

## 3. Fogli del Google Sheet

| Foglio | Colonne |
|---|---|
| Piano dei conti | Codice · Nome · Sezione · Padre · Tipo (Cartella/Conto) · Liquido · Regime fiscale · Valore di mercato · Attivo · Suggerimento |
| Prima nota | ID scrittura · Data · Descrizione · Riga · Codice conto · Nome conto · Dare · Avere · Etichetta · Origine · Allegato · Registrato il · Stato (ATTIVA/CESTINO) |
| Regimi fiscali | Codice · Nome · Aliquota · Tassato alla fonte · Gruppo compensazione · Anni riporto perdite · Note · Da verificare |
| Guida | Chiave (`tipo.X`, `campo.X`, `caso.X`) · Titolo · Suggerimento · Esempio corretto · Errore comune |
| Ammortamenti | Codice conto bene · Descrizione · Data inizio · Costo · Valore residuo finale · Vita utile (mesi) · Ultima quota registrata · Conto ammortamenti |
| Config | Parametro · Valore · Note (conti di sistema; opzionale riga `Modello Gemini`) |
| Ricorrenti | ID · Nome · Tipo (FISSO / SALDO_CARTA) · Conto origine (Avere) · Conto destinazione (Dare) · Importo · Giorno · Ogni quanti mesi · Prossima scadenza · Rate residue · Giorno chiusura carta · Attivo · Note (creato in automatico al primo uso) |
| Bilancio / Riepilogo fiscale | Generati dallo script |

**Sezioni**: `ATT+` attività produttive · `ATT-` liquidità e beni d'uso · `PAS` passività · `PN` patrimonio netto · `RIC` ricavi · `COS` costi.
Attività e costi aumentano in Dare; passività, patrimonio netto e ricavi in Avere.

**Conti di sistema** (non si spostano/nascondono/eliminano): 4.01 Saldi di apertura · 4.02 Rettifiche da ricognizione · 5.3.06 Rivalutazioni non realizzate · 6.7.05 Svalutazioni non realizzate · 6.8.01 Ammortamenti · 3.1.02 Debiti tributari.

**Codici**: colonna in formato testo (`6.10` ≠ `6.1`). Il codice non cambia quando si sposta un conto: la gerarchia vera è la colonna *Padre*.

---

## 4. Regole di registrazione (maschera semplice)

Primo conto = **Avere**, secondo = **Dare**.

| Tipo | Avere | Dare |
|---|---|---|
| Entrata | conto di ricavo (RIC) | dove arrivano i soldi (ATT/PAS) |
| Uscita | da dove escono i soldi (ATT/PAS) | categoria di costo (COS) |
| Trasferimento | conto di origine (ATT/PAS) | conto di destinazione (ATT/PAS) |
| Altro | righe libere, Totale Dare = Totale Avere | |

Gioco/trading sportivo: ricarica e prelievo = Trasferimento; a fine sessione solo il **netto** (positivo = Entrata su 5.2.x, negativo = Uscita su 6.6.x).
Controlli anti-errore: `controlliCoerenza_()` in Codice.gs (rimborsi, ricariche, estratto carta, acquisto titoli, rate, conti gioco, data futura).

---

## 5. Movimenti ricorrenti (v1.3)

- La regola non registra mai da sola: alla scadenza (data ≤ oggi) il movimento compare in Home in "Da confermare"; Conferma registra la scrittura con la data di scadenza e sposta la regola alla prossima, Salta la sposta senza registrare.
- **Rata finanziamento**: tipo FISSO con rate residue; alla creazione può generare un conto Passività "Finanziamento …" con debito = rata × rate residue. Ogni rata = Trasferimento conto corrente → debito. All'ultima rata la regola si disattiva.
- **Saldo carta di credito** (SALDO_CARTA): importo = saldo della carta al giorno di chiusura del mese precedente l'addebito (es. chiusura 15/10 → addebito 5/11). Trasferimento conto corrente → carta.
- **Movimento fisso**: qualunque coppia di conti, ogni N mesi, opzionalmente per N volte.

## Registro, correzioni e grafici (v1.4)

- **Registro**: cerca su tutta la Prima nota (non solo gli ultimi 40), filtro per conto, scheda **Cestino** con Ripristina. Da ogni conto: "📜 Movimenti di questa voce".
- **Correggi**: apre il movimento nel modulo "Altro"; al salvataggio la nuova scrittura viene registrata e la vecchia va nel cestino (`registraScrittura` con `sostituisce`).
- **Eliminare una voce**: possibile se non ha movimenti attivi; le scritture nel cestino che la usano vengono cancellate definitivamente.
- **Grafici** (Home → 📊 Grafici, libreria Chart.js 4.4.1 da cdnjs, caricata solo in quella schermata): entrate/uscite/risparmio per mese, costi per categoria e per voce, entrate per fonte, patrimonio netto e liquidità a fine mese, gioco e trading (netto mensile e cumulato), composizione del patrimonio. Periodi: 12 mesi, anno corrente, anno scorso, mese corrente.

## Registro con periodi, totali ed etichette (v1.5)

- Filtri: conto (anche cartelle), periodo (sempre, mese, mese scorso, anno, anno scorso, personalizzato), etichetta, testo.
- Riquadro totali calcolato su tutti i movimenti trovati: per un conto patrimoniale saldo iniziale, entrate, uscite, saldo finale; per ricavi/costi il totale; per un'etichetta il conto economico (ricavi, costi, risultato).
- Etichette multiple separate da virgola (es. "Poker, Bratislava 2026"), suggerite nel modulo.
- Poker/trasferte: costi per natura sotto Costi › Gioco e trading sportivo, la trasferta si raggruppa con l'etichetta, non con un conto dedicato. Bankroll live su un conto separato tra i Conti gioco. Sponsor (staking): i buy-in pagati dallo sponsor non sono costi propri; le vincite di sua spettanza sono un debito verso di lui.

## Attività di gioco (v1.6)

- Quattro attività: **Poker live**, **Poker online**, **Trading sportivo** (exchange), **Scommesse** (quota fissa/schedine). Il conto dei soldi (Betfair, bookmaker, piattaforma poker, Bankroll poker live) è separato dal conto di risultato.
- Convenzione: il nome del conto inizia con l'attività seguita da " – " (es. "Poker live – vincite", "Trading sportivo – perdite", "Poker live – viaggi e alloggio"). Grafici e ricerca riconoscono l'attività dal nome.
- Funzione `impostaContiGioco()` (menu del foglio o Conti → 🎲 Conti di gioco): rinomina i conti di default e crea quelli mancanti, idempotente.
- Grafici: risultato mensile per attività (barre impilate) + cumulato, totali per attività, totali per etichetta (luogo/club/trasferta).
- Registrazione: solo il netto di sessione/giornata. Betfair: netto exchange e netto sportsbook registrati separatamente.

## 6. API (`doPost`, campo `azione`)

`datiApp` · `registraSemplice` · `registraScrittura` · `saldi` · `aggiungiConto` · `modificaConto` · `eliminaConto` · `cestina` · `saldoIniziale` · `allinea` (RIVALUTAZIONE / RETTIFICA) · `fiscale` · `ammortamenti` · `analizzaImmagine` · `assistente` · `salvaRicorrente` · `eliminaRicorrente` · `confermaScadenza` · `movimenti` (ricerca su tutta la Prima nota, filtri conto/periodo/etichetta/testo/cestino, con riepilogo) · `ripristina` · `statistiche` (dati dei grafici) · `impostaContiGioco`

Gemini: funzione `gemini_()` → modello da Config (default `gemini-flash-latest`), con riprova e riserva `gemini-flash-lite-latest` se sovraccarico.

---

## 7. Come aggiornare

**Script**: incolla le modifiche in Apps Script → salva → **Esegui il deployment → Gestisci deployment → ✏️ → Versione: Nuova versione → Esegui il deployment** (l'URL `/exec` resta lo stesso). Senza nuova versione l'app continua a usare il codice vecchio.

**App**: su GitHub apri il file (es. `index.html`) → ✏️ modifica, oppure *Add file → Upload files* per sostituirlo → Commit. GitHub Pages si aggiorna in 1-2 minuti; sul telefono chiudi e riapri l'app (se serve, ricarica la pagina). Se modifichi `sw.js`, cambia il nome della cache (`finanze-v2`, …).

**Preferenza per le modifiche**: indicare sempre la stringa esatta da cercare e quella da sostituire (trova e sostituisci).

---

## 8. Idee per le prossime versioni

- Mutuo/finanziamento con piano di ammortamento alla francese
- Budget per categoria con semaforo
- Accantonamento automatico delle imposte stimate su Debiti tributari
- Controllo AI della scrittura prima del salvataggio
- Inserimento vocale
- Allegare la foto dello scontrino al movimento (Drive)
- Riconciliazione con l'estratto conto

## 9. Note e limiti noti

- Le regole fiscali nel foglio "Regimi fiscali" sono stime da far verificare al commercialista (crypto 33% dal 2026, gioco ADM tassato alla fonte, ETF redditi di capitale non compensabili, minusvalenze riportabili 4 anni).
- Il piano gratuito di Gemini può dare "high demand" o limiti orari: riprovare dopo qualche minuto.
- Il Registro dell'app mostra gli ultimi 40 movimenti; lo storico completo è nel foglio Prima nota.
- Per correggere un movimento: cestino + nuova registrazione (non modificare a mano la Prima nota).
