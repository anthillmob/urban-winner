# urban-winner

A Microsoft Fabric workspace for ingesting and enriching App Insights trace logs across Bronze and Silver Lakehouse layers.

## Repository Structure

```
urban-winner/
├── notebooks/
│   └── trace_logs_bronze_to_silver.ipynb          # PySpark notebook: Bronze → Silver transformation
└── fabric/
    ├── trace_logs_bronze_to_silver_pipeline.DataPipeline/
    │   ├── .platform                               # Fabric item metadata
    │   └── pipeline-content.json                  # Pipeline definition (activities, parameters, variables)
    └── triggers/
        └── trace_logs_daily_trigger.json          # Daily schedule trigger reference configuration
```

---

## Notebook: `trace_logs_bronze_to_silver`

Extracts Microsoft Log Analytics App Insights **trace** records from the Bronze Lakehouse, parses embedded key-value fields from the semi-structured `Message` column, and writes the enriched dataset to the Silver Lakehouse.

### Notebook Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `start_date` | `str` | `"2024-01-01"` | Inclusive start of the `event_time` window to process (ISO-8601 date/datetime string) |
| `end_date` | `str` | `"2024-12-31"` | Exclusive end of the `event_time` window to process (ISO-8601 date/datetime string) |
| `bronze_lakehouse` | `str` | `"BronzeLakehouse"` | Name of the Bronze Lakehouse |
| `silver_lakehouse` | `str` | `"SilverLakehouse"` | Name of the Silver Lakehouse |
| `bronze_table` | `str` | `"AppTraces"` | Source table name inside the Bronze Lakehouse |
| `silver_table` | `str` | `"AppTraces_Enriched"` | Destination table name inside the Silver Lakehouse |

### Design Principles

- **Idempotent** – Re-running with the same date range atomically replaces only the affected date partitions, preventing duplicates.
- **Parameterised** – `start_date` / `end_date` control which `event_time` window is processed; designed to be overridden at pipeline runtime.
- **Configurable field list** – Edit the *Configuration* cell to add or remove fields promoted from `Message` into first-class columns.

---

## Pipeline: `trace_logs_bronze_to_silver_pipeline`

A Fabric Data Factory Pipeline that executes the `trace_logs_bronze_to_silver` notebook on a recurring schedule and maps trigger time into the notebook's `start_date` / `end_date` parameters.

### Pipeline Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `windowStart` | `string` | `""` (empty) | Inclusive start of the event window (ISO-8601 datetime, e.g. `2024-06-01 00:00:00`). Leave empty to default to **24 hours before run time**. |
| `windowEnd` | `string` | `""` (empty) | Exclusive end of the event window (ISO-8601 datetime, e.g. `2024-06-02 00:00:00`). Leave empty to default to **current run time**. |

When both parameters are left empty (the default for the scheduled trigger), the pipeline automatically processes the previous 24-hour window on every run.

### Pipeline Activities

```
Set_effectiveWindowStart  ─┐
                            ├─► Execute_trace_logs_bronze_to_silver
Set_effectiveWindowEnd    ─┘
```

| Activity | Type | Description |
|---|---|---|
| `Set_effectiveWindowStart` | SetVariable | Evaluates `windowStart`: uses the supplied value or defaults to `utcNow() - 24h` |
| `Set_effectiveWindowEnd` | SetVariable | Evaluates `windowEnd`: uses the supplied value or defaults to `utcNow()` |
| `Execute_trace_logs_bronze_to_silver` | TridentNotebook | Runs the notebook with the resolved window; retries up to **2 times** on transient failure (30 s between attempts), 12-hour timeout |

### Importing the Pipeline into Fabric

1. In your Fabric workspace, go to **Data Factory** and create a new pipeline named `trace_logs_bronze_to_silver_pipeline`.
2. Open the pipeline's **JSON editor** and paste the contents of [`fabric/trace_logs_bronze_to_silver_pipeline.DataPipeline/pipeline-content.json`](fabric/trace_logs_bronze_to_silver_pipeline.DataPipeline/pipeline-content.json).
3. In the `Execute_trace_logs_bronze_to_silver` activity, replace the placeholder values:
   - `<NOTEBOOK_OBJECT_ID>` → the **Object ID** of the `trace_logs_bronze_to_silver` notebook in your workspace *(Settings → About)*
   - `<WORKSPACE_ID>` → your Fabric **Workspace ID** *(Workspace Settings → General)*
4. Save and validate the pipeline.

> **Tip – Fabric Git integration:** If your workspace is connected to this repository via Fabric Git integration, the pipeline item under `fabric/trace_logs_bronze_to_silver_pipeline.DataPipeline/` is automatically synchronised. You only need to update the two placeholder IDs once after the first sync.

---

## Scheduled Trigger: `trace_logs_daily_trigger`

The trigger configuration in [`fabric/triggers/trace_logs_daily_trigger.json`](fabric/triggers/trace_logs_daily_trigger.json) defines a **daily schedule at 00:00 UTC**.

### Configuring the Trigger in Fabric

1. Open `trace_logs_bronze_to_silver_pipeline` in the Fabric Data Factory editor.
2. Click **Add trigger → New/Edit**.
3. Create a new trigger with the following settings (or import from the JSON file as a reference):

   | Setting | Value |
   |---|---|
   | **Type** | Schedule |
   | **Recurrence** | Every 1 Day |
   | **Start time** | `2024-01-02 00:00 UTC` (or your desired start date) |
   | **Time zone** | UTC |
   | **windowStart** parameter | *(leave empty – defaults to now - 24 h)* |
   | **windowEnd** parameter | *(leave empty – defaults to now)* |

4. Activate the trigger. It will fire daily at midnight UTC, automatically processing the preceding 24-hour window.

### Custom Date Window (Manual Trigger)

To backfill or reprocess a specific date range, trigger the pipeline manually with explicit `windowStart` / `windowEnd` values:

1. Open the pipeline and click **Trigger → Trigger now**.
2. Set the parameters:
   - `windowStart`: `2024-05-15 00:00:00`
   - `windowEnd`: `2024-05-16 00:00:00`
3. Click **OK**. Only rows with `event_time` in `[2024-05-15 00:00:00, 2024-05-16 00:00:00)` will be processed.

Because the notebook uses Delta Lake `replaceWhere`, re-running the same window is safe and idempotent – existing data for those date partitions is atomically replaced.

---

## Robustness Notes

- **Idempotency:** Overlapping or repeated runs produce the same result without data duplication, thanks to Delta Lake `replaceWhere` partition replacement.
- **Retry on failure:** The notebook activity retries up to 2 times with a 30-second interval, handling transient Spark cluster or network issues.
- **Late-arriving data:** Extend the window (`windowStart` shifted earlier) when re-running to capture late-arriving events.
- **Parallelism:** Simultaneous runs over non-overlapping date windows are safe. Runs over the same window will contend on the same Delta partitions; schedule triggers accordingly to avoid overlap.