# Note Discorso Presentazione: Introduzione e Layer 1 — [VERSIONE SEMPLIFICATA E SCORREVOLE]

**Relatore:** Matteo Zylfo  
**Durata target:** ~4–4.5 minuti  
**Slide trattate:** Slide 1, Slide 2, Slide 3, Slide 4  
**Progetto:** Chat Bot for Telemetry Analysis on AROL Capping Machines  
**Corso:** System and Device Programming (Prof. Stefano Quer)  

---

## 🎙️ Discorso Completo (Italiano)

### [0:00 – 0:45] Slide 1 & 2: Introduzione & Agenda
*(Mostra Slide 1, poi passa a Slide 2)*

> "Buongiorno Professor Quer e buongiorno a tutti. Oggi vi presentiamo il nostro progetto per System and Device Programming: una pipeline end-to-end di telemetria industriale e un chatbot intelligente sviluppato per le **macchine tappatrici AROL**.
>
> La nostra architettura è divisa in quattro layer disaccoppiati:
> * **Layer 1 — Ingestion e Preprocessing:** trasforma la telemetria grezza in un dataset pulito basato su eventi.
> * **Layer 2 — Funzioni Analitiche:** fornisce strumenti deterministici per la statistica, la diagnostica e i grafici.
> * **Layer 3 — Agente AI:** gestisce l'esecuzione dei tool e il routing semantico in modo deterministico.
> * **Layer 4 — Interfaccia Bot:** offre sia un menu guidato per i tecnici d'impianto, sia una CLI conversazionale.
>
> Io presenterò il contesto industriale, le analisi sui dati grezzi e il Layer 1. Poi passerò la parola a Irene per i Layer 2 e 3, e a Cristian per il Layer 4, i risultati reali e la demo dal vivo."

---

### [0:45 – 1:50] Slide 3: Contesto Industriale e Dati Reali
*(Mostra Slide 3)*

> "Partiamo dalla macchina e dalla telemetria. AROL è leader mondiale nelle macchine rotative di tappatura. I dati provengono da un impianto industriale con **36 teste di tappatura indipendenti**, che lavora a ciclo continuo ad altissima velocità.
>
> Il dataset grezzo comprende **89 file CSV giornalieri** per quasi tre mesi, per un totale di **4.6 GB** e oltre **77 milioni di righe**.
> Ogni secondo, per tutte le 36 teste contemporaneamente, la macchina registra tre grandezze trasmesse dal PLC (Programmable Logic Controller):
> * `Count`: contatore cumulativo dei cicli di chiusura effettuati.
> * `AppTorque`: la coppia massima di serraggio in Newton-metri.
> * `Status`: codice diagnostico che indica successo, no-load, o specifici errori.
>
> Ispezionando i dati reali a confronto con le specifiche teoriche, abbiamo trovato tre sorprese importanti:
> 1. **Disposizione delle colonne:** le colonne non erano affiancate a triplette per testa, ma **raggruppate per tipologia di segnale** (prima tutti i 36 Count, poi tutti i Torque, infine tutti gli Status).
> 2. **Semantica dei timestamp:** non c'era il suffisso UTC (`Z`). Tra l'8 e il 9 marzo l'inizio del file slitta esattamente di un'ora, dimostrando che i log seguono l'**ora legale americana (Daylight Saving Time)** e non il tempo UTC.
> 3. **Codici di stato effettivi:** dei 13 codici documentati, nella realtà ne compaiono **solo cinque**: 0 per Chiusura OK, 2 per No Load, e i codici 4, 9 e 65 per specifici guasti meccanici."

---

### [1:50 – 3:00] Slide 4: Layer 1 — Ingestion in Streaming e Riconoscimento Chiusure
*(Mostra Slide 4)*

> "Passiamo al Layer 1. L'obiettivo qui è semplice: trasformare la telemetria periodica a 1 Hz in eventi reali di chiusura.
>
> Prima di tutto, caricare 4.6 GB tutti insieme farebbe crashare la memoria per Out-Of-Memory. Abbiamo quindi creato una **pipeline in streaming** che legge un file alla volta. Per non perdere dati al cambio file a mezzanotte, passiamo l'ultimo valore del contatore e del timestamp al giorno successivo. Abbiamo verificato questo approccio confrontandolo con un caricamento completo: il conteggio delle chiusure è identico al 100%, ma con un consumo di RAM minimo e costante.
>
> Come troviamo una chiusura reale?
> * Il PLC registra solo un contatore cumulativo. Quindi, ogni volta che il `Count` sale, abbiamo una chiusura.
> * Distinguiamo tra **eventi osservati**—dove abbiamo misurato fisicamente coppia e stato—e **chiusure inferite**, che contano tutti i pezzi anche se il contatore è saltato di 2 o più.
> * Se il contatore scende, è solo un reset di turno, non un errore.
> * E se un disturbo elettrico fa comparire uno zero fasullo, il nostro filtro riconosce il disturbo ed evita di inventare produzione inesistente."

---

### [3:00 – 3:55] Slide 4 (continua): Deduplica e Rilevazione dell'Idle
*(Sempre su Slide 4)*

> "Infine, abbiamo risolto due punti delicati: duplicati e fermo macchina.
>
> Per la deduplica: se cancellassimo semplicemente i contatori già visti, al primo reset da zero cancelleremmo tutta la produzione del nuovo turno! Per evitare questo usiamo un **ID di segmento**, eliminando i duplicati solo all'interno dello stesso segmento.
>
> Per il fermo macchina: l'impianto è ufficialmente in idle quando **tutte le 36 teste registrano Status 2 (No Load) per almeno 30 secondi**.
> Abbiamo però introdotto una regola chiave sui **buchi di campionamento**: se la comunicazione salta e non arrivano dati, fermiamo il conteggio dell'idle. La perdita di pacchetti non è fermo macchina! Questo ha corretto quasi 3 ore di finto downtime, portando l'utilizzo reale al **33,63%**."

---

### [3:55 – 4:25] Conclusione del Layer 1 e Passaggio della Parola
*(Passa a Slide 5 mentre passi la parola)*

> "Infine, il Layer 1 salva tutti i risultati puliti nel formato binario **Apache Parquet**—nello specifico `closure_events.parquet` con oltre 55 milioni di righe e `idle_periods.parquet`—insieme a un report completo di qualità.
>
> L'uso di un formato colonnare compresso con Snappy riduce l'ingombro su disco di oltre l'80% e consente alle query del Layer 2 di girare in millisecondi.
>
> Avendo così ripulito e validato le fondamenta dei dati, passo la parola a Irene per illustrarvi il **Layer 2** con gli strumenti analitici, e il **Layer 3** con l'architettura dell'agente AI."

---

## 📋 Cue Card Rapida (Punti Chiave)

1. **Intro & Agenda (Slide 1–2, ~40s)**
   - Saluto al Prof. Quer e panoramica sui 4 layer.
   - Passaggio parola: Matteo (L1), Irene (L2 & L3), Cristian (L4, Risultati & Demo).
2. **Contesto & Sorprese (Slide 3, ~1m)**
   - Macchina rotativa a 36 teste, 1 Hz dal PLC.
   - 89 CSV, 4.6 GB, 77M righe.
   - 3 sorprese: colonne per tipo, cambio ora legale USA a marzo, solo 5 codici di stato reali.
3. **Streaming & Chiusure (Slide 4, ~1m 10s)**
   - Pipeline streaming (RAM O(1), nessun crash).
   - Continuità tra i giorni con CarryState (100% uguale al merge totale).
   - Chiusura: `Count` che sale ($\Delta 	ext{Count} > 0$).
   - Eventi osservati vs chiusure inferite.
   - Reset non sono errori; zeri fasulli (flicker) mascherati.
4. **Deduplica & Idle (Slide 4, ~55s)**
   - Deduplica con `segment ID`: non cancella la produzione dopo i reset.
   - Idle: tutte le 36 teste a Status 2 per $\ge 30$ secondi.
   - Sampling Gap: assenza di pacchetti non è downtime (salvate quasi 3 ore, utilization al 33,63%).
5. **Parquet & Handover (~30s)**
   - Formato Apache Parquet colonnare veloce con compressione Snappy.
   - Passaggio della parola a Irene per i Layer 2 e 3.

---

## 🛡️ Scheda Domande e Risposte per Matteo

* **D: Perché separare eventi osservati da chiusure inferite?**
  * *Risposta:* "Un salto di 5 pezzi dà una sola lettura di coppia e stato. Calcoliamo la success rate sugli eventi osservati (99.9965%) e la produzione totale sulle chiusure inferite (55.95M)."

* **D: Come gestite i reset senza cancellare dati lecite con la deduplica?**
  * *Risposta:* "Assegniamo un `segment ID` che cresce quando il contatore cala. Deduplichiamo solo nello stesso segmento: quando un turno riparte da zero, le nuove bottiglie vengono tenute."

* **D: Perché la regola sui sampling gap nell'idle?**
  * *Risposta:* "La mancata comunicazione non è fermo macchina. Se mancano i dati, fermiamo il conteggio dell'idle. Abbiamo evitato di conteggiare quasi 3 ore di pacchetti persi come downtime."

* **D: Perché Parquet?**
  * *Risposta:* "È un formato binario colonnare compresso. Legge da disco solo le colonne necessarie, velocizzando le query a millisecondi e risparmiando oltre l'80% di spazio su disco."
