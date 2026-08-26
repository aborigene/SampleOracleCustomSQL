# Oracle Custom SQL Extension for Dynatrace

A step-by-step guide and ready-to-deploy Dynatrace Extension 2.0 that queries an Oracle database and sends results to Dynatrace as **logs** or **metrics**.

---

## Table of Contents

1. [Overview](#overview)
2. [Extension Structure](#extension-structure)
3. [Prerequisites](#prerequisites)
4. [Step 1 — Install the VS Code Extension](#step-1--install-the-vs-code-extension)
5. [Step 2 — Generate Developer Certificates](#step-2--generate-developer-certificates)
6. [Step 3 — Upload the Root CA Certificate to Dynatrace](#step-3--upload-the-root-ca-certificate-to-dynatrace)
7. [Step 4 — Copy the Root CA Certificate to ActiveGate](#step-4--copy-the-root-ca-certificate-to-activegate)
8. [Step 5 — Build the Extension](#step-5--build-the-extension)
9. [Step 6 — Deploy the Extension to Dynatrace](#step-6--deploy-the-extension-to-dynatrace)
10. [Step 7 — Create the Monitoring Configuration](#step-7--create-the-monitoring-configuration)
11. [Adding More Queries](#adding-more-queries)
12. [Ingesting Results as Logs](#ingesting-results-as-logs)
13. [Ingesting Results as Metrics](#ingesting-results-as-metrics)
14. [Verifying Data Collection](#verifying-data-collection)
15. [Troubleshooting](#troubleshooting)

---

## Overview

This extension:
- Connects to an Oracle database from an **ActiveGate** (no OneAgent required)
- Runs configurable SQL queries on a schedule (every 1–60 minutes)
- Sends each row of results to Dynatrace as a **log entry** _or_ aggregated results as **metrics**
- Works with any Oracle 11g+ database (on-premise or cloud)
- Requires **zero code changes** before building — all connection details are entered in the Dynatrace UI

**Included example queries:**
| Query Group | Ingest Type | Description |
|---|---|---|
| `active_sessions_log` | Log | Active Oracle sessions — one log per row |
| `session_count_metric` | Metric | `custom:oracle.sessions.active` gauge |
| `waiting_sessions_metric` | Metric | `custom:oracle.sessions.waiting` gauge |
| `long_running_queries_log` | Log | SQL statements running > 60 seconds |

---

## Extension Structure

```
SQLExtension/
├── extension/
│   └── extension.yaml   # Extension definition — queries, attributes, metrics
├── dist/                # Created automatically after building
│   └── custom-db.oracle.sql.extension-0.1.0.zip
└── README.md
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| **VS Code** | [Download](https://code.visualstudio.com/) |
| **ActiveGate** | A remote ActiveGate reachable from your Oracle database host |
| **Oracle monitoring user** | A dedicated DB user with the privileges listed below |
| **Java keytool** | Only needed if your Oracle DB uses SSL/TLS |
| **Dynatrace environment** | SaaS, version ≥ 1.331 |

### Minimum Oracle Privileges

Create a dedicated monitoring user and grant only what is needed:

```sql
-- Create the monitoring user
CREATE USER dt_monitor IDENTIFIED BY "your_secure_password";
GRANT CREATE SESSION TO dt_monitor;

-- Grant access to the views used by the built-in example queries
GRANT SELECT ON V_$SESSION  TO dt_monitor;
GRANT SELECT ON V_$SQL      TO dt_monitor;
GRANT SELECT ON V_$SYSSTAT  TO dt_monitor;

-- Grant access to your own application tables as needed
-- GRANT SELECT ON my_schema.my_table TO dt_monitor;
```

---

## Step 1 — Install the VS Code Extension

The **Dynatrace Extensions** VS Code extension provides certificate generation, building, and deployment commands — all from within VS Code.

1. Open VS Code.
2. Go to the **Extensions** panel (`Ctrl+Shift+X` / `Cmd+Shift+X`).
3. Search for **Dynatrace Extensions**.
4. Click **Install** on the extension published by *Dynatrace Platform Extensions*.

   > Direct link: [marketplace.visualstudio.com — Dynatrace Extensions](https://marketplace.visualstudio.com/items?itemName=DynatracePlatformExtensions.dynatrace-extensions)

5. Open the `SQLExtension` folder in VS Code:
   ```
   File → Open Folder → select the SQLExtension directory
   ```

6. Open Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`) and run:
   ```
   Dynatrace: Initialize workspace
   ```
   Enter your Dynatrace environment URL and an API token with the scopes:
   - `extensions.read`
   - `extensions.write`
   - `extensionConfigurations.read`
   - `extensionConfigurations.write`

---

## Step 2 — Generate Developer Certificates

Dynatrace Extension 2.0 packages must be **cryptographically signed** before Dynatrace and ActiveGate will accept them. The VS Code extension generates a self-signed CA certificate and a developer certificate automatically.

### Generate certificates

1. Open Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`).
2. Run:
   ```
   Dynatrace Extensions: Generate certificates
   ```
3. The command creates the following files in VS Code's workspace storage directory:

| File | Purpose |
|---|---|
| `ca.pem` | Root CA certificate (public) — **needs to go to Dynatrace and ActiveGate** |
| `ca.key` | Root CA private key — keep this secure, never share |
| `developer.pem` | Developer certificate (public) |
| `developer.key` | Developer private key |
| `developer.p12` | PKCS12 bundle used by VS Code to sign the `.zip` during build |

### Find the generated certificates

The certificates are stored under VS Code's workspace storage directory. Check the VS Code **Output** panel (channel: *Dynatrace Extensions*) during generation — the exact path is printed there.

Typical locations:

| OS | Path |
|---|---|
| **macOS** | `~/Library/Application Support/Code/User/workspaceStorage/<id>/DynatracePlatformExtensions.dynatrace-extensions/certificates/` |
| **Linux** | `~/.vscode-server/data/User/workspaceStorage/<id>/DynatracePlatformExtensions.dynatrace-extensions/certificates/` |
| **Windows** | `%APPDATA%\Code\User\workspaceStorage\<id>\DynatracePlatformExtensions.dynatrace-extensions\certificates\` |

To find the directory when you don't know `<id>`:

```bash
# macOS / Linux
find ~/Library/Application\ Support/Code/User/workspaceStorage \
  -name "ca.pem" 2>/dev/null

# Linux (VS Code Server / remote)
find ~/.vscode-server/data/User/workspaceStorage \
  -name "ca.pem" 2>/dev/null
```

> **Keep `ca.key` private.** Anyone with that file can sign extensions that Dynatrace will trust. The `ca.pem` file is public and safe to distribute.

---

## Step 3 — Upload the Root CA Certificate to Dynatrace

Dynatrace must know your root CA certificate so it can verify the signature on your extension when you upload it.

### Option A — Dynatrace UI (recommended for first-time setup)

1. In your Dynatrace environment go to:
   **Hub → Extensions 2.0 → Root certificates**
   (or navigate to `Settings → Extension certificates` depending on your version)
2. Click **Upload root certificate**.
3. Select the `ca.pem` file generated in Step 2.
4. Confirm the upload. The certificate fingerprint is shown for verification.

### Option B — Dynatrace API

```bash
# Replace placeholders:
#   <DT_ENV_URL>  → e.g. https://abc12345.live.dynatrace.com
#   <API_TOKEN>   → token with extensions.write scope
#   <CA_PEM_PATH> → full path to ca.pem

curl -X POST "<DT_ENV_URL>/api/v2/extensions/certificates" \
  -H "Authorization: Api-Token <API_TOKEN>" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @<CA_PEM_PATH>
```

---

## Step 4 — Copy the Root CA Certificate to ActiveGate

The ActiveGate that will run the extension also needs the root CA certificate so it can validate the extension package signature at runtime.

### Step 4.1 — Transfer `ca.pem` to the ActiveGate host

From your development machine, copy the file to the ActiveGate server:

```bash
# SCP example (Linux / macOS)
scp /path/to/ca.pem <user>@<activegate-host>:/tmp/ca.pem
```

Alternative transfer methods: WinSCP, SFTP, rsync, or any file copy tool.

### Step 4.2 — Install the certificate on the ActiveGate

SSH into the ActiveGate server:

```bash
ssh <user>@<activegate-host>
```

Copy the certificate to the ActiveGate certificates directory:

```bash
sudo cp /tmp/ca.pem \
  /var/lib/dynatrace/remotepluginmodule/agent/conf/certificates/ca.pem
```

Set the correct ownership and permissions:

```bash
sudo chown dtuserag:dtuserag \
  /var/lib/dynatrace/remotepluginmodule/agent/conf/certificates/ca.pem

sudo chmod 644 \
  /var/lib/dynatrace/remotepluginmodule/agent/conf/certificates/ca.pem
```

### Step 4.3 — Restart ActiveGate

```bash
sudo systemctl restart dynatracegateway
```

Verify the service restarted cleanly:

```bash
sudo systemctl status dynatracegateway
```

> **Note:** Only `ca.pem` is needed on the ActiveGate. The private key (`ca.key`) must never leave your development machine.

---

## Step 5 — Build the Extension

The `extension/extension.yaml` file is ready to build as-is — no edits required. Connection details are provided at configuration time inside Dynatrace, not inside the YAML.

### Build via VS Code

1. Open Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`).
2. Run:
   ```
   Dynatrace: Build Extension
   ```
3. VS Code signs the package with `developer.p12` and places the output in:
   ```
   dist/custom-db.oracle.sql.extension-0.1.0.zip
   ```

### Build via CLI (optional)

If you prefer the command line or need to build in CI:

```bash
# Install the Dynatrace Extensions CLI
pip install dt-extensions-sdk

# Build and sign
dt-sdk build --certificate /path/to/developer.p12 \
             --private-key /path/to/developer.key \
             extension/
```

---

## Step 6 — Deploy the Extension to Dynatrace

### Option A — Dynatrace Hub UI

1. Go to **Hub → Extensions 2.0**.
2. Click **Upload custom extension**.
3. Select `dist/custom-db.oracle.sql.extension-0.1.0.zip`.
4. Review the extension details and click **Upload**.

### Option B — Dynatrace API

```bash
curl -X POST "<DT_ENV_URL>/api/v2/extensions" \
  -H "Authorization: Api-Token <API_TOKEN>" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @dist/custom-db.oracle.sql.extension-0.1.0.zip
```

---

## Step 7 — Create the Monitoring Configuration

After deploying the extension, you create one **Monitoring Configuration** per database instance you want to monitor.

### Via Dynatrace UI

1. Go to **Hub → Extensions 2.0**.
2. Find **custom:db.oracle.sql.extension** and click it.
3. Click **Add monitoring configuration**.
4. Fill in the connection details:

| Field | Example value | Notes |
|---|---|---|
| **ActiveGate** | Select from dropdown | Must be the ActiveGate where you copied `ca.pem` |
| **Host** | `oracle-db.example.com` | Hostname or IP of the Oracle server |
| **Port** | `1521` | Default Oracle listener port |
| **Service name / SID** | `ORCL` | Oracle service name or SID |
| **Username** | `dt_monitor` | The dedicated monitoring user from Prerequisites |
| **Password** | `***` | Will be stored encrypted in the credential vault |

5. Click **Save**.

### Via API (example JSON)

```json
{
  "name": "Oracle Production DB",
  "extensionId": "custom:db.oracle.sql.extension",
  "version": "0.1.0",
  "activation": {
    "remote": {
      "unified": {
        "enabled": true
      }
    }
  },
  "properties": {
    "sqlOracle.host": "oracle-db.example.com",
    "sqlOracle.port": 1521,
    "sqlOracle.username": "dt_monitor",
    "sqlOracle.password": "your_secure_password",
    "sqlOracle.database": "ORCL"
  }
}
```

> **Tip:** For production, store the password in the Dynatrace Credential Vault:
> `Settings → Integration → Credential Vault → Add credentials`
> Then reference it in the monitoring configuration by selecting "Use credential vault" for the password field.

---

## Adding More Queries

Open `extension/extension.yaml` and add a new entry under `sqlOracle:`. There are two ingest types — pick the one that matches what you want to see in Dynatrace.

### Adding a log query (one Dynatrace log per result row)

```yaml
sqlOracle:

  # ... existing entries above ...

  - group: my_orders_log          # Unique name for this query group
    interval:
      minutes: 5                  # How often the query runs (1–60)
    ingest: log
    query: |
      SELECT
        ORDER_ID,
        CUSTOMER_ID,
        STATUS,
        TOTAL_AMOUNT,
        TO_CHAR(ORDER_DATE, 'YYYY-MM-DD HH24:MI:SS') AS ORDER_DATE,
        'new_order' AS event_type
      FROM MY_SCHEMA.ORDERS
      WHERE ORDER_DATE >= SYSDATE - INTERVAL '5' MINUTE
      ORDER BY ORDER_DATE DESC
    attributes:
      - key: content              # Drives the log message in Dynatrace
        value: col:event_type
      - key: order.id
        value: col:ORDER_ID
      - key: customer.id
        value: col:CUSTOMER_ID
      - key: order.status
        value: col:STATUS
      - key: order.amount
        value: col:TOTAL_AMOUNT
      - key: order.date
        value: col:ORDER_DATE
```

**Key rules for log queries:**
- The `query` must return at least one column.
- Each row in the result set becomes a separate log entry in Dynatrace.
- Map columns to log attributes using `value: col:<COLUMN_NAME>` (column names are case-sensitive and must match the query `SELECT` alias exactly).
- The `content` attribute drives the main log message — set it to a human-readable string such as an event type or description.
- Filter rows in SQL (e.g., `WHERE ORDER_DATE >= SYSDATE - INTERVAL '5' MINUTE`) so you only ingest new data on each run.

---

### Adding a metric query (aggregated numeric values)

```yaml
sqlOracle:

  # ... existing entries above ...

  - group: orders_per_status_metric
    interval:
      minutes: 5
    ingest: metric
    query: |
      SELECT
        STATUS,
        COUNT(*) AS order_count
      FROM MY_SCHEMA.ORDERS
      WHERE ORDER_DATE >= SYSDATE - INTERVAL '5' MINUTE
      GROUP BY STATUS
    metrics:
      - key: custom:oracle.orders.count   # Metric key visible in Metrics Explorer
        value: col:order_count            # Column containing the numeric value
        type: gauge                       # gauge | count | delta
        dimensions:
          - key: order.status             # Split metric by this dimension
            value: col:STATUS
```

**Key rules for metric queries:**
- The `value` column must contain a **number** (integer or float). Non-numeric columns cause the row to be skipped.
- `type` options:
  - `gauge` — absolute value at a point in time (e.g., active sessions count)
  - `count` — cumulative counter (monotonically increasing)
  - `delta` — difference since last collection
- Use `dimensions` to split one metric key into multiple time series (e.g., per status, per host).
- Metric keys must start with `custom:` when defined in a custom extension.
- After deployment, the metric appears in **Metrics Explorer** under its `custom:` key.

---

## Ingesting Results as Logs

Logs are the most flexible ingest type. Each result row becomes a structured log entry in Dynatrace Logs and can then be further processed with **OpenPipeline**.

### What you get in Dynatrace Logs

After the `active_sessions_log` group runs you will see log entries like:

```
content      : oracle_active_session
db.user      : SCOTT
db.session.id: 42
db.sql.id    : abcdef1234567
db.session.duration_seconds: 15
...
```

### Querying your logs with DQL

```
fetch logs
| filter matchesPhrase(content, "oracle_active_session")
| fields timestamp, db.user, db.session.id, db.sql.duration_seconds
| sort timestamp desc
| limit 100
```

### Processing logs with OpenPipeline

You can use OpenPipeline to transform logs into **Business Events**, remove sensitive data, or route data to different storage buckets.

To add an OpenPipeline pipeline to this extension, create a `pipeline.json` and add it to `extension.yaml`:

```yaml
openpipeline:
  pipelines:
    - displayName: Oracle Sessions to BizEvents
      pipelinePath: oracle-to-bizevents.pipeline.json
      configScope: logs
  sources:
    - displayName: Oracle Extension Source
      sourcePath: oracle-source.json
      configScope: logs
```

**Example `oracle-source.json`** — routes Oracle session logs into the pipeline:

```json
{
  "enabled": true,
  "pipelineType": "custom",
  "matcher": "content == \"oracle_active_session\" or content == \"oracle_long_running_query\"",
  "description": "Oracle monitoring events"
}
```

**Example `oracle-to-bizevents.pipeline.json`** — converts matching logs to Business Events:

```json
{
  "customId": "oracle-to-bizevents",
  "displayName": "Oracle Sessions to BizEvents",
  "dataExtraction": {
    "processors": [
      {
        "id": "extract-oracle-bizevents",
        "type": "bizevent",
        "enabled": true,
        "matcher": "true",
        "description": "Convert Oracle session data to business events",
        "bizevent": {
          "eventType": {
            "type": "constant",
            "value": "oracle.session"
          },
          "eventProvider": {
            "type": "constant",
            "value": "custom:db.oracle.sql.extension"
          },
          "fieldExtraction": {
            "type": "includeAll"
          }
        }
      }
    ]
  },
  "storage": {
    "processors": [
      {
        "id": "no-log-storage",
        "type": "noStorage",
        "enabled": true,
        "matcher": "true",
        "description": "Store as Business Events only, not logs"
      }
    ]
  }
}
```

---

## Ingesting Results as Metrics

Metrics are ideal for numeric values you want to chart, alert on, or feed into Davis AI anomaly detection.

### What you get in Dynatrace

After the `session_count_metric` group runs, the metric `custom:oracle.sessions.active` appears in:
- **Metrics Explorer** — chart it, set up SLOs
- **Davis AI** — anomaly detection out of the box (after a baseline period)
- **Alerting** — create metric events for threshold-based alerting

### Querying your metrics with DQL

```
timeseries avg(custom:oracle.sessions.active), by: {dt.entity.host}
| limit 100
```

Or a split by dimension:

```
timeseries sum(custom:oracle.orders.count), by: {order.status}
```

### Creating a Davis AI anomaly detection rule

After collecting a metric for ~1 week, Davis builds a baseline. You can also create manual threshold alerts:

1. Go to **Settings → Anomaly detection → Metric events**.
2. Click **Add metric event**.
3. Set:
   - **Metric:** `custom:oracle.sessions.active`
   - **Condition:** Static threshold or auto-adaptive baseline
   - **Severity:** `Performance` or `Resource`
4. Save. Davis will fire a Problem when the threshold is breached.

---

## Verifying Data Collection

### Check extension status

In the Dynatrace UI:
1. Go to **Hub → Extensions 2.0 → custom:db.oracle.sql.extension**.
2. Open your monitoring configuration.
3. The **Status** column shows `Running`, `Error`, or `No data`.

### Tail extension logs on the ActiveGate

```bash
sudo tail -f /var/lib/dynatrace/remotepluginmodule/log/extensions/custom_db_oracle_sql_extension/*.log
```

### Query logs in Dynatrace

```
fetch logs
| filter matchesPhrase(content, "oracle_active_session")
| sort timestamp desc
| limit 50
```

### Query metrics in Dynatrace

```
timeseries avg(custom:oracle.sessions.active)
```

### Query Business Events (if OpenPipeline is configured)

```
fetch bizevents
| filter event.provider == "custom:db.oracle.sql.extension"
| sort timestamp desc
| limit 50
```

---

## Troubleshooting

### Extension fails to upload — "Certificate not trusted"

- Confirm you uploaded `ca.pem` to Dynatrace (Step 3).
- Confirm the same `ca.pem` / `developer.p12` pair was used when building.
- Regenerate certificates and rebuild if you are unsure.

### Extension status shows "Error" — "Cannot connect to database"

- Verify the Oracle listener is reachable from the ActiveGate:
  ```bash
  # Run on the ActiveGate server
  nc -vz <oracle-host> 1521
  ```
- Confirm the service name or SID is correct (not both — use one or the other).
- Test the connection with sqlplus or any Oracle client from the ActiveGate host.

### Extension runs but no data appears in Dynatrace Logs

- Run the SQL query manually in your Oracle client to confirm it returns rows.
- Check that the `WHERE` clause is not filtering out all rows.
- Look at the ActiveGate extension log for query errors.

### Extension status shows "Error" — "Certificate not found on ActiveGate"

- Confirm `ca.pem` is at `/var/lib/dynatrace/remotepluginmodule/agent/conf/certificates/ca.pem`.
- Confirm ownership is `dtuserag:dtuserag` and permissions are `644`.
- Restart ActiveGate after copying the certificate.

### Metric does not appear in Metrics Explorer

- Metrics can take 2–5 minutes to appear after the first collection.
- Confirm `ingest: metric` is set (not `ingest: log`) for that query group.
- Confirm the `value` column returns a number in all rows.

---

## Additional Resources

- [Dynatrace Extension 2.0 Documentation](https://www.dynatrace.com/support/help/extend-dynatrace/extensions20)
- [Dynatrace VS Code Extension (Marketplace)](https://marketplace.visualstudio.com/items?itemName=DynatracePlatformExtensions.dynatrace-extensions)
- [SQL extensions reference (extension.yaml)](https://www.dynatrace.com/support/help/extend-dynatrace/extensions20/data-sources/sql-extensions)
- [OpenPipeline documentation](https://www.dynatrace.com/support/help/observe-and-explore/data-management/openpipeline)
- [Dynatrace Credential Vault](https://www.dynatrace.com/support/help/setup-and-configuration/integrations/credential-vault)
- [Oracle V$SESSION view reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SESSION.html)

---

## License

Proprietary — Copyright © 2026

## Author

Igor Osch Simões
