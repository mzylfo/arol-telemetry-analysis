# Note Discorso Presentazione: Introduzione e Layer 1 (Data Preparation) — [VERSIONE ESTESA / DETTAGLIATA IN ITALIANO]

**Relatore:** Matteo Zylfo  
**Durata target:** 5–6 minuti  
**Slide trattate:** Slide 1, Slide 2, Slide 3, Slide 4  
**Progetto:** Chat Bot for Telemetry Analysis on AROL Capping Machines  
**Corso:** System and Device Programming (Prof. Stefano Quer)  

---

## 🎙️ Discorso Completo (Italiano)

### [0:00 – 1:00] Slide 1 & 2: Introduzione & Agenda
*(Mostra Slide 1, poi passa a Slide 2)*

> "Buongiorno Professor Quer e buongiorno a tutti. Oggi vi presentiamo il nostro progetto per System and Device Programming: una pipeline end-to-end di telemetria industriale e un chatbot intelligente sviluppato per le **macchine tappatrici AROL**.
>
> La nostra architettura è strutturata in quattro layer disaccoppiati:
> * **Layer 1 — Ingestion e Preprocessing:** trasforma i flussi grezzi di telemetria ad alta frequenza in un dataset pulito basato su eventi.
> * **Layer 2 — Funzioni Analitiche:** fornisce strumenti deterministici per la statistica, la diagnostica e la generazione di grafici.
> * **Layer 3 — Agente AI:** orchestra l'esecuzione dei tool e il routing semantico in modo deterministico.
> * **Layer 4 — Interfaccia Bot:** offre sia un menu operativo guidato per i tecnici d'impianto, sia una CLI conversazionale.
>
> Io mi occuperò del contesto industriale, delle osservazioni sui dati grezzi e dell'architettura del Layer 1, per poi passare la parola a Irene per i Layer 2 e 3, e a Cristian per il Layer 4, i risultati e la demo dal vivo."

---

### [1:00 – 2:15] Slide 3: Contesto Industriale e Dati Reali
*(Mostra Slide 3)*

> "Partiamo dalla macchina e dalla telemetria. AROL è leader mondiale nelle macchine rotative di tappatura. Il dataset che abbiamo analizzato proviene da una macchina industriale equipaggiata con **36 teste di tappatura indipendenti**, operante a ciclo continuo ad altissima cadenza produttiva.
>
> Il dataset grezzo comprende **89 file CSV giornalieri** per un periodo di quasi tre mesi, per un totale di **4.6 GB** e oltre **77 milioni di righe**.
> Ogni secondo, per ciascuna delle 36 teste contemporaneamente, la macchina registra tre grandezze principali trasmesse dal PLC (Programmable Logic Controller):
> * `Count`: contatore intero cumulativo dei cicli di chiusura effettuati.
> * `AppTorque`: coppia massima di serraggio registrata sul tappo, in Newton-metri.
> * `Status`: codice diagnostico che indica la riuscita dell'operazione, lo stato di *No Load*, o specifici codici di errore.
>
> Ispezionando i dati reali a confronto con le specifiche teoriche fornite, abbiamo riscontrato tre discrepanze fondamentali:
> 1. **Disposizione delle colonne:** i campi non erano raggruppati a triplette per testa come da specifica, ma **raggruppati per tipologia di segnale** (prima tutti i 36 Count, poi tutti i 36 Torque, infine tutti i 36 Status).
> 2. **Semantica temporale:** i timestamp non avevano il suffisso UTC (`Z`). Tra l'8 e il 9 marzo l'inizio del file giornaliero slitta esattamente di un'ora, dimostrando che i dati seguono l'**ora legale americana (Daylight Saving Time)** e non il tempo UTC.
> 3. **Distribuzione dei codici di stato:** dei 13 codici teorici documentati, nella realtà operativa ne compaiono **solo cinque**: 0 per Chiusura OK, 2 per No Load, e i codici 4, 9 e 65 per specifici guasti meccanici."

---

### [2:15 – 3:45] Slide 4: Layer 1 — Ingestion in Streaming e Riconoscimento Chiusure
*(Mostra Slide 4)*

> "Passando al **Layer 1**, l'obiettivo primario era convertire la telemetria periodica campionata a 1 Hz in un **dataset basato su eventi**, in cui ogni riga corrisponda a un'effettiva operazione di chiusura e non a un secondo di clock.
>
> Poiché caricare in memoria 4.6 GB tutti insieme causerebbe crash per Out-Of-Memory, abbiamo progettato una **pipeline streaming a blocchi**.
> Per garantire che l'elaborazione file per file non perda eventi al confine tra un giorno e l'altro, la pipeline trasporta lo stato da un file al successivo attraverso un oggetto `CarryState` (ultimo contatore, timestamp e segmento di reset). Abbiamo verificato sperimentalmente che questa esecuzione a memoria costante produce esattamente lo stesso conteggio di eventi rispetto a un caricamento monolitico dell'intero archivio.
>
> Come riconosciamo una chiusura reale?
> * Il PLC (Programmable Logic Controller) non emette un log di eventi discreti, ma solo un contatore. Quindi una chiusura avviene quando **il `Count` cresce rispetto alla lettura precedente valida** ($\Delta \text{Count} > 0$).
> * Facciamo una distinzione fondamentale: una singola riga di telemetria è un **evento osservato**—che ci dà una lettura di coppia e di stato—mentre il numero reale di bottiglie tappate è una **chiusura inferita**, calcolata dall'incremento del contatore. Quindi, se in un secondo il contatore salta di 3 pezzi, registriamo un singolo evento di misura, ma 3 chiusure complessive.
> * Classifichiamo ciascun evento con un tag di qualità del dato `data_quality`:
>   * `single` per i normali avanzamenti unitari a 1 Hz,
>   * `aggregated` quando una testa chiude più bottiglie in un solo secondo per l'alta velocità della macchina,
>   * e `gap` quando l'incremento si trova a cavallo di un buco di trasmissione.
> * Se il contatore scende, viene identificato come reset di turno o cambio lotto, non come guasto.
> * Per gestire i disturbi elettrici sui sensori (*flicker* a zero), confrontiamo il valore successivo al blocco di zeri (`post`) con quello precedente (`pre`): se `post >= pre`, si tratta di un errore di trasmissione e mascheriamo il `Count` come NaN. Si noti che mascheriamo *esclusivamente* il `Count`: preserviamo intatte le letture di coppia `AppTorque` e di stato `Status`, che continuano a essere valide."

---

### [3:45 – 4:45] Slide 4 (continua): Deduplica per Segmento e Rilevazione dell'Idle
*(Sempre su Slide 4)*

> "Altri due nodi ingegneristici del Layer 1 sono stati la **deduplica** e il **rilevamento del fermo macchina (Idle)**.
>
> Per la deduplica, una semplice eliminazione dei duplicati basata globalmente sulla coppia `(Head, Count)` cancellerebbe produzione lecita non appena il contatore riparte da zero dopo un reset. Per evitarlo tracciamo un **ID di segmento di reset**: un record è considerato duplicato solo se si ripete **all'interno dello stesso identico segmento**.
>
> Nelle metriche derivate, calcoliamo la velocità oraria istantanea per testa (`capping_speed_pph`) rapportando le chiusure inferite al tempo trascorso, riflettendo la reale cadenza della macchina anche in presenza di salti.
>
> Per i tempi di fermo, un periodo di Idle richiede che **tutte le 36 teste registrino contemporaneamente `Status = 2` (No Load) per almeno 30 secondi continuativi**.
> Abbiamo tuttavia introdotto un raffinamento cruciale: la gestione dei **Sampling Gap**. Se tra due letture consecutive c'è un buco temporale superiore a una soglia adattiva (pari a `GAP_FACTOR` volte l'intervallo mediano, cioè oltre 2 secondi), il periodo di idle viene interrotto. Non possiamo presumere che la macchina fosse ferma durante una perdita di comunicazione. Questa correzione ha evitato di conteggiare come fermo macchina circa **due ore e cinquantadue minuti di pacchetti persi**, portando il tasso di utilizzo effettivo al **33,63%**."

---

### [4:45 – 5:30] Conclusione del Layer 1 e Passaggio della Parola
*(Passa a Slide 5 mentre passi la parola)*

> "Infine, il Layer 1 serializza le tabelle normalizzate in formato **Apache Parquet** con compressione Snappy: nello specifico `closure_events.parquet` (oltre 55 milioni di righe su 16 colonne) e `idle_periods.parquet` (3.493 intervalli di fermo), accompagnate da un report di qualità in JSON e Markdown.
>
> Questo formato binario colonnare, grazie al column pruning, riduce l'ingombro su disco di oltre l'80% e consente alle query analitiche del Layer 2 di girare in millisecondi.
>
> Avendo così validato e ripulito le fondamenta dei dati, passo la parola a Irene per illustrarvi il **Layer 2** con gli strumenti analitici e il **Layer 3** con l'architettura dell'agente AI."

---

## 📋 Cue Card Rapida (Punti Chiave per Parlare a Braccio)

1. **Intro & Agenda (Slide 1–2, ~45s)**
   - Saluto al Prof. Quer e introduzione del gruppo.
   - I 4 layer disaccoppiati: Ingestion, Analytics, AI Agent, Bot Interface.
2. **Contesto & Dati Grezzi (Slide 3, ~1m 15s)**
   - Macchina rotativa a 36 teste, campionamento 1 Hz dal PLC (Programmable Logic Controller).
   - 89 file CSV, 4.6 GB, oltre 77M di righe.
   - Le 3 discrepanze reali: layout colonne raggruppate per segnale, shift ora legale USA a marzo, solo 5 codici di stato attivi su 13.
3. **Streaming & Riconoscimento Chiusure (Slide 4, ~1m 30s)**
   - Pipeline streaming a chunk con `CarryState` tra file (RAM $O(1)$, zero perdita di dati).
   - Regola chiusura: $\Delta \text{Count} > 0$.
   - **Eventi osservati** vs **Chiusure inferite** (somma totale dei salti).
   - **Tag di qualità**: `single`, `aggregated` (alta cadenza meccanica), `gap` (attraversa un buco di segnale).
   - **Filtro flicker**: verifica matematica `post` vs `pre` su 2.088 blocchi a zero. Mascheramento solo di `Count` per preservare coppia e stato.
4. **Deduplica, Velocità & Correzione Idle (Slide 4, ~1m 15s)**
   - Deduplica per `(Head, segment, Count)` per salvare le chiusure valide dopo i reset.
   - Velocità istantanea (`capping_speed_pph`) calcolata con `inferred_closure_count`.
   - Definizione Idle: tutte le 36 teste a `Status = 2` per $\ge 30$ secondi.
   - **Sampling Gap adattivo**: assenza di segnale $\neq$ fermo macchina (salvate 2h 52m, utilization al 33,63%).
5. **Persistenza & Chiusura (~30s)**
   - Salvataggio in Apache Parquet colonnare (`closure_events.parquet` con 16 colonne, `idle_periods.parquet`).
   - Column pruning e compressione Snappy.
   - Passaggio della parola a Irene per i Layer 2 e 3.

---

## 🛡️ Scheda Domande e Risposte per Matteo (Possibili Domande di Quer)

* **D1: Perché avete separato gli eventi osservati dalle chiusure inferite?**
  * *Risposta:* "Una riga in cui il contatore sale di 5 pezzi fornisce una sola misura istantanea di coppia e stato. Contarla come una sola chiusura sottostimerebbe la produzione; applicare quello stato a tutte e 5 le bottiglie distorcerebbe la qualità. Calcoliamo la success rate sugli eventi osservati (99.9965%) e la produzione totale sulle chiusure inferite (55.95M)."

* **D2: Come gestite i reset del contatore senza cancellare chiusure lecite con la deduplica?**
  * *Risposta:* "Assegniamo un `segment_id` monotonico che si incrementa ogni volta che il contatore decresce. La deduplica verifica la terna `(head_id, segment_id, counter)`. Quando il contatore riparte da zero in un nuovo segmento, quei valori sono riconosciuti come nuove chiusure legittime."

* **D3: Come distinguete un vero reset da un disturbo temporaneo del sensore (flicker)?**
  * *Risposta:* "Analizziamo il blocco di zeri confrontando la lettura successiva (`post`) con quella precedente (`pre`). Se `post >= pre`, il contatore fisico ha continuato a contare ed era solo un difetto di telemetria; se `post << pre` (rapporto < 0.05), è un vero reset. Validato su 2.088 blocchi a zero con zero casi dubbi. Inoltre mascheriamo solo `Count`, preservando le misure fisiche di `AppTorque` e `Status`."

* **D4: Cosa rappresenta l'attributo `data_quality`?**
  * *Risposta:* "Etichetta la natura di ciascun incremento: `single` per avanzamenti unitari a 1 Hz, `aggregated` quando una testa chiude più bottiglie in un secondo a causa dell'alta velocità della macchina, e `gap` quando il salto scavalca un buco di comunicazione."

* **D5: Perché avete modificato la rilevazione dell'idle con una soglia adattiva?**
  * *Risposta:* "Inizialmente due letture idle separate da ore prive di dati venivano fuse in un unico gigantesco fermo macchina. Ma l'assenza di dati non è downtime. Fissando una soglia adattiva a `GAP_FACTOR` volte l'intervallo mediano (> 2s), spezziamo l'idle in presenza di buchi di trasmissione, escludendo 10.337 secondi (~2h 52m) di mancata comunicazione."

* **D6: Perché Parquet invece di SQLite o CSV?**
  * *Risposta:* "Parquet è un formato binario colonnare compresso con Snappy. Riduce 4.6 GB di CSV a poche centinaia di megabyte, supporta il column pruning (leggendo da disco solo le colonne richieste dalla query) e si integra nativamente con PyArrow e Pandas per aggregazioni istantanee."
