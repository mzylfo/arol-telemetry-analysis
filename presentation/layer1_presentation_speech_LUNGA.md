# Presentation Speech Notes: Introduction & Layer 1 (Data Preparation) — [EXTENDED / DETAILED VERSION]

**Speaker:** Matteo Zylfo  
**Target Duration:** 5–6 minutes  
**Slides Covered:** Slide 1, Slide 2, Slide 3, Slide 4  
**Project:** Chat Bot for Telemetry Analysis on AROL Capping Machines  
**Course:** System and Device Programming (Prof. Stefano Quer)  

---

## 🎙️ Full Spoken Script (English)

### [0:00 – 1:00] Slide 1 & 2: Introduction & Agenda
*(Display Slide 1, then advance to Slide 2)*

> "Good morning Professor Quer, and good morning everyone. Today we are presenting our project for System and Device Programming: an end-to-end industrial telemetry analysis pipeline and intelligent chatbot developed for **AROL Capping Machines**.
>
> Our architecture is organized into four decoupled layers:
> * **Layer 1 — Ingestion and Preprocessing:** transforms massive, high-frequency raw telemetry into a clean, event-based dataset.
> * **Layer 2 — Analytic Functions:** provides deterministic statistical, diagnostic, and visualization tools.
> * **Layer 3 — AI Agent:** orchestrates tool execution and deterministic semantic routing.
> * **Layer 4 — Bot Interface:** offers both a guided operational menu and a conversational CLI.
>
> I will cover the industrial problem, our data observations, and the Layer 1 engineering pipeline, before handing over to Irene for Layers 2 and 3, and Cristian for Layer 4, the execution results, and the live demo."

---

### [1:00 – 2:15] Slide 3: Industrial Context & Real-World Data Observations
*(Display Slide 3)*

> "Let us look at the machine and the telemetry. AROL is a world leader in high-speed rotary capping equipment. The dataset we analyzed comes from an industrial machine equipped with **36 independent capping heads**, operating continuously at high production rates.
>
> The raw data consists of **89 daily CSV files** spanning three months, totaling **4.6 GB** and over **77 million rows**.
> Every second, for all 36 heads simultaneously, the machine logs three primary signals from the PLC (Programmable Logic Controller):
> * `Count`: a cumulative integer counter of completed capping cycles.
> * `AppTorque`: the peak applied torque in Newton-meters.
> * `Status`: a diagnostic code indicating successful closure, no-load, or specific error states.
>
> When we inspected the raw data against the initial project specification, we identified three critical real-world discrepancies:
> 1. **Column grouping:** columns were not arranged in head triplets as specified, but **grouped by signal type** (all 36 Counts first, then all 36 Torques, then all 36 Statuses).
> 2. **Timestamp semantics:** timestamps lacked a UTC designator (`Z`). In fact, the file boundary shifted by exactly one hour on the second Sunday of March, demonstrating that the timestamps follow **U.S. Daylight Saving Time**, not UTC.
> 3. **Status code distribution:** out of 13 theoretical status codes, in practice **only five appear in real operations**: 0 for OK, 2 for No Load, and codes 4, 9, and 65 for specific mechanical errors."

---

### [2:15 – 3:45] Slide 4: Layer 1 — Streaming Ingestion & Event Detection
*(Display Slide 4)*

> "Moving to **Layer 1**, our core task was converting raw 1 Hz periodic telemetry into an **event-based dataset**, where each record corresponds to an actual capping operation rather than an arbitrary second of time.
>
> Because loading 4.6 GB into memory all at once is impractical and causes Out-Of-Memory crashes, we designed a **streaming chunk-based pipeline**.
> To ensure that processing file-by-file does not lose events at day boundaries, our pipeline carries over state across files—specifically the last counter, timestamp, and reset segment via a `CarryState` object. We experimentally verified that this chunked execution yields the exact same closure count as loading the entire archive at once, while maintaining a minimal, constant memory footprint.
>
> How do we detect a closure?
> * The PLC (Programmable Logic Controller) does not log discrete closure events; it only increments a counter. Therefore, an operation occurs when **`Count` increases compared to the previous valid reading** ($\Delta \text{Count} > 0$).
> * Crucially, we make a distinction here: a single row of telemetry is an **observed event**—giving us one torque and status reading—while the actual number of capped bottles is an **inferred closure**, calculated from the counter increment. So, if the counter jumps by 3 in one second, we register one measurement event, but 3 total closures.
> * We classify every closure with a `data_quality` tag:
>   * `single` for standard 1-step increments,
>   * `aggregated` when a head closes multiple bottles within one second due to high machine speed,
>   * and `gap` when an increment spans an unobserved communication interval.
> * If the counter drops, it is treated as a lot reset, not an error.
> * To handle sensor flicker—where electrical noise drops `Count` temporarily to zero—we compare the value after the zero run against the value before: if `post >= pre`, it is corrupted reporting, so we mask `Count` as NaN. Notice that we mask *only* `Count`: we deliberately preserve the physical `AppTorque` and `Status` readings, which remain valid throughout."

---

### [3:45 – 4:45] Slide 4 (continued): Segment-Aware Deduplication & Idle Detection
*(Still on Slide 4)*

> "Two other major engineering challenges in Layer 1 were **deduplication** and **idle state detection**.
>
> For deduplication, a naive rule removing duplicate `(Head, Count)` pairs globally would delete legitimate production whenever the counter restarts from zero after a reset. To solve this, we track a **reset segment ID**. A duplicate is strictly defined and removed only if it occurs **within the exact same segment**.
>
> In derived metrics, we compute the instantaneous capping speed in pieces per hour (`capping_speed_pph`) using the inferred closure count divided by the elapsed time, ensuring multi-closure bursts reflect true manufacturing cadence.
>
> For downtime, an idle period requires **all 36 heads to register `Status = 2` (No Load) simultaneously for at least 30 consecutive seconds**.
> However, we introduced a critical refinement: **handling sampling gaps**. If an observation gap exceeds an adaptive threshold—defined as `GAP_FACTOR` times the median sampling interval (exceeding 2 seconds)—we split the idle period. We cannot assume the machine was idle when communication was simply lost. This prevented misclassifying nearly **two hours and fifty-two minutes of missing packets as machine downtime**, refining our machine utilization to **33.63%**."

---

### [4:45 – 5:30] Wrap-up & Handover to Irene
*(Advance to Slide 5 as you hand over)*

> "Finally, Layer 1 serializes the normalized tables into **Apache Parquet** with Snappy compression—specifically `closure_events.parquet` (over 55 million rows across 16 columns) and `idle_periods.parquet` (3,493 downtime intervals)—along with an automated data quality summary in JSON and Markdown.
>
> Using a columnar binary format with column pruning reduces the storage footprint by over 80% and allows subsequent analytical queries to run in milliseconds.
>
> With our data foundation clean, deterministic, and validated, I will now hand over to Irene to present **Layer 2** and our analytical functions, followed by **Layer 3** and our AI agent architecture."

---

## 📋 Quick Cue Card (Key Points for Speaking Freely)

1. **Intro & Agenda (Slide 1–2, ~45s)**
   - Welcome Prof. Quer and audience.
   - 4 decoupled layers: Ingestion, Analytics, AI Agent, Bot Interface.
2. **Context & Raw Telemetry (Slide 3, ~1m 15s)**
   - 36 capping heads, continuous rotary machine, 1 Hz sampling from PLC (Programmable Logic Controller).
   - 89 CSVs, 4.6 GB, >77M rows.
   - 3 real-world deviations: grouped column layout by signal, US Daylight Saving Time shift in March, only 5 active status codes in reality.
3. **Streaming & Event Detection (Slide 4, ~1m 30s)**
   - Chunked streaming pipeline with `CarryState` across daily boundaries ($O(1)$ RAM, zero data loss).
   - Closure rule: $\Delta \text{Count} > 0$.
   - **Observed events** (sensor readings) vs. **Inferred closures** (full delta sum).
   - **Data quality tagging**: `single`, `aggregated` (high speed burst), `gap` (spans communication hole).
   - **Sensor flicker filter**: mathematical `post` vs `pre` check across 2,088 zero-runs. Mask *only* `Count` to keep real torque and status.
4. **Deduplication, Speed & Idle Refinement (Slide 4, ~1m 15s)**
   - Deduplication by `(Head, segment, Count)` to preserve valid closures after resets.
   - Instantaneous speed (`capping_speed_pph`) calculated with `inferred_closure_count`.
   - Idle definition: all 36 heads on `Status = 2` for $\ge 30$ seconds.
   - **Adaptive Sampling Gap**: missing packets $\neq$ downtime (saved 2h 52m of misclassified idle, utilization = 33.63%).
5. **Persistence & Handover (~30s)**
   - Columnar Apache Parquet format (`closure_events.parquet` with 16 columns, `idle_periods.parquet`).
   - Column pruning and Snappy compression.
   - Handover to Irene for Layers 2 & 3.

---

## 🛡️ Q&A Defense Sheet for Matteo (Prof. Quer's Potential Questions)

* **Q1: Why did you separate observed events from inferred closures?**
  * *Answer:* "A row where the counter increases by 5 provides only one snapshot of torque and status. Counting it as a single closure undercounts production; applying that single status to all 5 closures distorts quality statistics. We compute the success rate on observed events (99.9965%) and total production on inferred closures (55.95M)."

* **Q2: How do you handle counter resets without deleting valid closures?**
  * *Answer:* "We assign a monotonic `segment_id` that increments every time the counter drops. Deduplication checks `(head_id, segment_id, counter)`. When the counter restarts from zero in a new segment, those values are recognized as legitimate new closures."

* **Q3: How do you distinguish a real machine reset from sensor flicker?**
  * *Answer:* "We analyze the contiguous zero-run by comparing the first non-zero reading after the run (`post`) with the reading before (`pre`). If `post >= pre`, the hardware counter kept incrementing, so it is corrupted reporting; if `post << pre` (ratio < 0.05), it is a genuine reset. Verified across all 2,088 zero-runs in the archive with zero borderline cases. Furthermore, we mask only `Count`, keeping valid `AppTorque` and `Status` intact."

* **Q4: What is the `data_quality` attribute in your schema?**
  * *Answer:* "It labels the nature of each closure increment: `single` for normal 1-step increments at 1 Hz, `aggregated` when a head completes multiple closures within one second due to high machine speed, and `gap` when an increment spans a communication hole."

* **Q5: Why did you modify the idle detection logic with an adaptive gap threshold?**
  * *Answer:* "Initially, two idle readings separated by hours with no data were merged into one giant idle period. But missing data is not downtime. By setting an adaptive gap threshold at $\text{GAP\_FACTOR} \times \text{median interval}$ ($> 2\text{s}$), we split the idle run whenever data is missing, excluding 10,337 seconds (~2h 52m) of missing communication."

* **Q6: Why Parquet instead of SQLite or CSV?**
  * *Answer:* "Parquet is a columnar binary format with Snappy compression. It reduces 4.6 GB of raw CSVs to a few hundred megabytes, allows column pruning (reading only requested columns from disk) and predicate pushdown, and integrates natively with PyArrow, DuckDB, and Pandas for sub-second aggregations."
