# Presentation Speech Notes: Introduction & Layer 1 — [SIMPLIFIED / CONVERSATIONAL VERSION]

**Speaker:** Matteo Zylfo  
**Target Duration:** ~4–4.5 minutes  
**Slides Covered:** Slide 1, Slide 2, Slide 3, Slide 4  
**Project:** Chat Bot for Telemetry Analysis on AROL Capping Machines  
**Course:** System and Device Programming (Prof. Stefano Quer)  

---

## 🎙️ Full Spoken Script (English)

### [0:00 – 0:45] Slide 1 & 2: Introduction & Agenda
*(Display Slide 1, then advance to Slide 2)*

> "Good morning Professor Quer, and good morning everyone. Today we present our project for System and Device Programming: an end-to-end industrial telemetry pipeline and intelligent chatbot developed for **AROL Capping Machines**.
>
> Our architecture has four decoupled layers:
> * **Layer 1 — Ingestion & Preprocessing:** turns massive raw telemetry into a clean, event-based dataset.
> * **Layer 2 — Analytic Functions:** provides deterministic statistical, diagnostic, and charting tools.
> * **Layer 3 — AI Agent:** handles tool execution and semantic routing deterministically.
> * **Layer 4 — Bot Interface:** offers both a guided operational menu and a conversational CLI.
>
> I will present the industrial problem, our data observations, and Layer 1. Then I will hand over to Irene for Layers 2 and 3, and Cristian for Layer 4, the execution results, and our live demo."

---

### [0:45 – 1:50] Slide 3: Industrial Context & Real-World Data Observations
*(Display Slide 3)*

> "Let us look at the machine and the telemetry. AROL is a world leader in high-speed rotary capping equipment. The dataset comes from an industrial machine with **36 independent capping heads**, running continuously at high speed.
>
> The raw data has **89 daily CSV files** covering three months, totaling **4.6 GB** and over **77 million rows**.
> Every second, for all 36 heads, the machine logs three main signals from the PLC (Programmable Logic Controller):
> * `Count`: a cumulative counter of completed capping cycles.
> * `AppTorque`: the peak applied torque in Newton-meters.
> * `Status`: a diagnostic code indicating success, no-load, or specific error states.
>
> When we inspected the raw data against the project specification, we found three important surprises:
> 1. **Column grouping:** columns were not arranged in triplets per head, but **grouped by signal type** (all 36 Counts first, then all Torques, then all Statuses).
> 2. **Timestamp semantics:** timestamps had no UTC designator (`Z`). In fact, file boundaries shifted by exactly one hour on the second Sunday of March, proving the logs follow **U.S. Daylight Saving Time**, not UTC.
> 3. **Status code distribution:** out of 13 documented status codes, **only five ever appear in practice**: 0 for OK, 2 for No Load, and codes 4, 9, and 65 for specific mechanical errors."

---

### [1:50 – 3:00] Slide 4: Layer 1 — Streaming Ingestion & Event Detection
*(Display Slide 4)*

> "Now, let's look at Layer 1. Our main goal here is simple: turn raw, periodic telemetry into real capping events.
>
> First, loading 4.6 GB into memory all at once would crash the system. So, we built a **streaming pipeline** that processes data file by file. To avoid losing events at midnight when changing files, we carry over the last counter and timestamp to the next day. We tested this against a full merge, and the result was 100% identical, but using minimal RAM.
>
> So, how do we detect an actual closure?
> * The PLC only logs a cumulative counter. Whenever `Count` goes up, that's a closure.
> * We distinguish between **observed events**—where we actually measured torque and status—and **inferred closures**, which count every single piece even if the counter jumped by 2 or more.
> * If the counter drops, it's just a shift reset, not an error.
> * And if electrical noise causes a quick drop to zero, our filter ignores the flicker so we don't invent fake production."

---

### [3:00 – 3:55] Slide 4 (continued): Deduplication & Idle Detection
*(Still on Slide 4)*

> "Next, we solved two tricky problems: duplicates and machine downtime.
>
> For deduplication, if you simply delete repeated counter values, you create a huge issue: when the machine resets and starts back from zero, all new bottles would be deleted! To prevent this, we tag each run with a **segment ID**. We only remove duplicates within the exact same segment.
>
> Finally, for downtime: the machine is officially idle when **all 36 heads show Status 2 (No Load) for at least 30 seconds**.
> Here, we added a key rule for **sampling gaps**: if communication drops and no data arrives, we stop the idle counter. Missing packets are not machine downtime! This fix saved nearly three hours of misclassified downtime, giving us a true machine utilization of **33.63%**."

---

### [3:55 – 4:25] Wrap-up & Handover to Irene
*(Advance to Slide 5 as you hand over)*

> "Finally, Layer 1 saves everything into **Apache Parquet** format—specifically `closure_events.parquet` with over 55 million rows, and `idle_periods.parquet`—along with a data quality summary.
>
> Using a columnar binary format cuts disk usage by over 80% and lets our analytics queries run in milliseconds.
>
> With our data clean and validated, I will now hand over to Irene to present **Layer 2** with our analytics tools, followed by **Layer 3** and our AI agent architecture."

---

## 📋 Quick Cue Card (Speaking Notes)

1. **Intro & Agenda (Slide 1–2, ~40s)**
   - Welcome Prof. Quer and audience.
   - 4 decoupled layers.
   - Handover plan: Matteo (L1), Irene (L2 & L3), Cristian (L4, Results & Demo).
2. **Context & Surprises (Slide 3, ~1m)**
   - 36 heads, continuous rotary machine, 1 Hz from PLC.
   - 89 CSVs, 4.6 GB, 77M rows.
   - 3 surprises: grouped columns, US Daylight Saving Time shift, only 5 real status codes.
3. **Streaming & Closures (Slide 4, ~1m 10s)**
   - Streaming pipeline (O(1) RAM, no OOM crashes).
   - Carry over state across daily boundaries (100% match with full merge).
   - Closure rule: `Count` goes up ($\Delta 	ext{Count} > 0$).
   - Observed events vs inferred closures.
   - Resets are not errors; sensor flicker filtered out.
4. **Duplicates & Downtime (Slide 4, ~55s)**
   - Deduplication by `segment ID`: keeps new bottles after counter resets.
   - Idle = all 36 heads on Status 2 for $\ge 30$ seconds.
   - Sampling gaps: missing data is not downtime (saved ~3 hours, utilization = 33.63%).
5. **Parquet & Handover (~30s)**
   - Saved to Apache Parquet (fast columnar storage, Snappy compression).
   - Handover to Irene for Layers 2 and 3.

---

## 🛡️ Q&A Defense Sheet for Matteo

* **Q: Why separate observed events from inferred closures?**
  * *Answer:* "A row where the counter jumps by 5 gives only one reading of torque and status. We compute the success rate on observed events (99.9965%) and total production on inferred closures (55.95M)."

* **Q: How do you handle counter resets without deleting valid closures?**
  * *Answer:* "We assign a `segment ID` that increments whenever the counter drops. We only deduplicate within the same segment, so when a new shift starts from zero, its bottles are safely kept."

* **Q: Why the sampling gap rule in idle detection?**
  * *Answer:* "Missing communication is not machine downtime. If data stops arriving for more than a few seconds, we stop the idle counter. This prevented misclassifying nearly 3 hours of lost packets as downtime."

* **Q: Why Parquet?**
  * *Answer:* "It is a columnar binary format with Snappy compression. It reads only the needed columns from disk, making queries run in milliseconds while saving 80% disk space."
