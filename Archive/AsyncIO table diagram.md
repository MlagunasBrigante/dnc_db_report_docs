- [ ] ```mermaid
flowchart TB
    subgraph INPUT["📥 External Sources"]
        API["Law Ruler API<br/>(Custom Report)"]
        CSV["CSV File<br/>(Manual Upload)"]
        SCRUB_API["DNC Scrub API"]
    end

    subgraph ODS_DB["🗄️ ODS Database (scrub_ods.db)"]
        RAW_DATA[("raw_data<br/>─────────<br/>Lead ID, Phone,<br/>First Name, Last Name,<br/>Status, Tags...")]
    end

    subgraph STAGING_DB["🗄️ Staging Database (scrub_staging.db)"]
        CLEANED[("cleaned_phone_numbers<br/>─────────<br/>Lead ID, Phone,<br/>is_phone_valid,<br/>batch_id")]
        BATCH[("batch_status<br/>─────────<br/>batch_id, status,<br/>updated_at")]
        SCRUB_RAW[("scrub_results<br/>─────────<br/>batch_id,<br/>response_json")]
        SCRUB_TRANS[("scrub_results_translated<br/>─────────<br/>Phone, ResultCode,<br/>Reason")]
        UPDATE_STATUS[("update_status<br/>─────────<br/>leadId, status,<br/>updated_at")]
        UPDATE_RESULTS[("update_results<br/>─────────<br/>leadId,<br/>response_json")]
        LEADS_TAGS[("leads_with_tags<br/>─────────<br/>leadId, Phone,<br/>names (Tag),<br/>ResultCode")]
    end

    subgraph OUTPUT["📤 Output"]
        CSV_EXPORT["CSV Export<br/>leads_with_tags_{date}.csv"]
        LAWRULER["Law Ruler API<br/>(Update Tags)"]
    end

    subgraph PIPELINES["⚙️ Processing Pipelines"]
        P1["1️⃣ DataIngestionPipeline<br/>extract() → clean()"]
        P2["2️⃣ ScrubbingPipeline<br/>run()"]
        P3["3️⃣ ScrubbedDataProcessor<br/>run() → save_idempotent()"]
        P4["4️⃣ UpdateLawRulerPipeline<br/>_translate_tags() → run()"]
    end

    %% Flow connections
    API -->|"GetCustomReport()"| P1
    CSV -->|"Alternative input"| P1
    
    P1 -->|"Save raw data"| RAW_DATA
    P1 -->|"Clean & validate phones"| CLEANED
    
    CLEANED -->|"Batch phones"| P2
    P2 -->|"Track batches"| BATCH
    P2 -->|"Call API"| SCRUB_API
    SCRUB_API -->|"JSON responses"| SCRUB_RAW
    
    SCRUB_RAW -->|"Parse JSON"| P3
    P3 -->|"Interpret codes"| SCRUB_TRANS
    
    RAW_DATA -->|"Load leads"| P4
    SCRUB_TRANS -->|"Load scrub results"| P4
    P4 -->|"Merge & translate tags"| LEADS_TAGS
    P4 -->|"Track updates"| UPDATE_STATUS
    P4 -->|"Store responses"| UPDATE_RESULTS
    
    LEADS_TAGS -->|"Export"| CSV_EXPORT
    P4 -->|"Update leads<br/>(if enabled)"| LAWRULER

    %% Styling
    classDef inputStyle fill:#e1f5fe,stroke:#01579b
    classDef odsStyle fill:#fff3e0,stroke:#e65100
    classDef stagingStyle fill:#f3e5f5,stroke:#7b1fa2
    classDef outputStyle fill:#e8f5e9,stroke:#2e7d32
    classDef pipelineStyle fill:#fce4ec,stroke:#c2185b
    
    class API,CSV,SCRUB_API inputStyle
    class RAW_DATA odsStyle
    class CLEANED,BATCH,SCRUB_RAW,SCRUB_TRANS,UPDATE_STATUS,UPDATE_RESULTS,LEADS_TAGS stagingStyle
    class CSV_EXPORT,LAWRULER outputStyle
    class P1,P2,P3,P4 pipelineStyle
```
---
# Table Schemas
```sql
CREATE TABLE IF NOT EXISTS "scrub_results_translated" (
	"Phone" TEXT,
	"ResultCode" TEXT,
	"Reserved" TEXT,
	"Reason" TEXT,
	"RegionAbbrev" TEXT,
	"Country" TEXT,
	"Locale" TEXT,
	"CarrierInfo" TEXT,
	"NewReassignedAreaCode" TEXT,
	"TZCode" TEXT,
	"CallingWindow" TEXT,
	"UTCOffset" TEXT,
	"DoNotCallToday" TEXT,
	"CallingTimeRestrictions" TEXT,
	"EBRType" TEXT,
	"IsWirelessOrVoIP" TEXT,
	"LineType" TEXT,
	PRIMARY KEY ("Phone")
);
```