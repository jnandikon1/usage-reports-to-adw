# Usage2ADW for Oracle Cloud@Customer (ExaCC) - Implementation Plan

## Context

Your organization runs **Oracle Cloud@Customer (ExaCC)** with Exadata infrastructure, Autonomous Databases, APEX servers, OEM, and multiple Oracle instances. You need **cost reporting and usage reports** to gain visibility into infrastructure spending, enable department chargebacks, and track resource consumption trends.

This repository (**Usage2ADW** by Adi Zohar, Oracle open-source, v25.10.01) is an Oracle-provided tool that extracts OCI cost and usage reports from the OCI tenant's Object Storage and loads them into an **Autonomous Data Warehouse (ADW)** with **APEX dashboards** for visualization. It is directly applicable to your ExaCC environment since Cloud@Customer cost reports are generated the same way as public OCI - in the Oracle-managed "bling" Object Storage bucket.

> **Disclaimer**: This is NOT an official Oracle billing tool. Use OCI's built-in Cost Analysis for official utilization calculations. Usage2ADW is for trend analysis, reporting, and chargeback.

---

## What This Repository Can Do for Your Organization

### 1. Out-of-the-Box Capabilities

| Capability | Description | Relevant Files |
|-----------|-------------|----------------|
| **Cost Data Extraction** | Pulls daily cost CSVs from OCI Object Storage ("bling" bucket) containing all ExaCC service costs | `usage2adw.py` (lines 1055-1224) |
| **ADW Data Warehouse** | Stores cost data in `OCI_COST` table with full historical retention for trend analysis | `usage2adw.py` (lines 817-910) |
| **APEX Dashboards** | 6 built-in report types: Current State, CPU Over Time, Storage Over Time, Cost Analysis, Cost Over Time, Rate Card | `usage2adw_demo_apex_app.sql` (2.4MB) |
| **Tag-Based Cost Allocation** | 4 "special" tag columns (`TAG_SPECIAL` through `TAG_SPECIAL4`) for department/project chargeback | `usage2adw.py` (lines 317-320, 883-905) |
| **ExaCC Resource Inventory** | ShowOCI integration loads 88+ tables including **`OCI_SHOWOCI_DATABASE_EXA_CC_VMS`** with ExaCC VM Cluster details (OCPUs, memory, storage, GI version, maintenance windows) | `usage2adw_showoci_csv2adw.py` (lines 919-968) |
| **Daily Email Reports** | HTML email with 5 tables: daily cost, monthly cost, OCPU daily, storage daily, OCPU by service | `shell_scripts/run_daily_report.sh` |
| **CSV Exports** | Cost exports by compartment/service/SKU for external tools or ERP integration | `shell_scripts/run_report_compart_service_daily_to_csv.sh` |
| **Multi-Tenant** | Load cost data from multiple OCI tenancies into one ADW for consolidated reporting | `shell_scripts/run_multi_daily_usage2adw.sh` |
| **Public Rate Card** | Compares your actual costs against public PAYG pricing | `usage2adw.py` (OCI_PRICE_LIST table) |
| **FOCUS Reports (Beta)** | FinOps standard cost format for multi-cloud reporting | `focus2adw/focus2adw.py` |

### 2. ExaCC-Specific Value

- **Fixed infrastructure cost tracking**: ExaCC rack costs appear as recurring line items - track actual consumption vs. committed capacity
- **VM Cluster monitoring**: `OCI_SHOWOCI_DATABASE_EXA_CC_VMS` captures shape, cpu_core_count, shape_ocpus, db_storage_gb, memory_gb, node_count, GI version, maintenance windows
- **ADB-D cost tracking**: Per-ADB instance costs on Dedicated Exadata Infrastructure
- **Compartment-based chargeback**: Cost data includes `PRD_COMPARTMENT_NAME` and `PRD_COMPARTMENT_PATH` for org-level allocation
- **Database inventory**: `OCI_SHOWOCI_DATABASES` and `OCI_SHOWOCI_DATABASES_PDBS` tables capture all database instances and PDBs

### 3. Database Tables Created

**Core tables**: `OCI_COST`, `OCI_COST_STATS`, `OCI_COST_TAG_KEYS`, `OCI_COST_REFERENCE`, `OCI_PRICE_LIST`, `OCI_LOAD_STATUS`

**ExaCC-relevant ShowOCI tables** (subset of 88+ total):
- `OCI_SHOWOCI_DATABASE_EXA_INFRA` - Exadata rack details
- `OCI_SHOWOCI_DATABASE_EXA_CC_VMS` - ExaCC VM Clusters (OCPU, memory, storage, maintenance)
- `OCI_SHOWOCI_DATABASES` - All database instances
- `OCI_SHOWOCI_DATABASES_PDBS` - PDB inventory
- `OCI_SHOWOCI_DATABASES_ADB` - Autonomous Database instances
- `OCI_SHOWOCI_DATABASE_BACKUPS` - Backup inventory
- `OCI_SHOWOCI_COMPUTE` - Compute instances
- `OCI_SHOWOCI_NETWORK_VCN` / `OCI_SHOWOCI_NETWORK_SUBNET` - Network topology
- `OCI_SHOWOCI_MONITOR_DB_MANAGEMENT` - DB Management status

---

## Architecture

```
+------------------------------------------------------------------+
|                   Customer Data Center (On-Premises)              |
|                                                                   |
|  +-------------------------+    +-----------------------------+   |
|  | ExaCC Infrastructure    |    | (Optional) On-Prem VM       |   |
|  | - Exadata Racks         |    | - For on-prem deployment    |   |
|  | - VM Clusters           |    | - Uses User API Keys        |   |
|  | - ADB-D Instances       |    +-----------------------------+   |
|  | - Oracle Databases      |                                      |
|  +-------------------------+                                      |
|  +-------------------------+                                      |
|  | OEM Server              |    (Complementary to Usage2ADW)      |
|  +-------------------------+                                      |
+------------------------------------------------------------------+
          |  FastConnect / VPN
          v
+------------------------------------------------------------------+
|                     OCI Cloud (Control Plane)                     |
|                                                                   |
|  +-------------------------+    +-----------------------------+   |
|  | Object Storage          |    | Usage2ADW VM (OL8)          |   |
|  | "bling" bucket          |--->| - Python 3.9 + OCI SDK      |   |
|  | (Cost CSV files)        |    | - usage2adw.py              |   |
|  +-------------------------+    | - Instance Principals auth   |   |
|                                 +------------|----------------+   |
|  +-------------------------+                 |                    |
|  | KMS Vault               |                 v                    |
|  | - DB Password Secret    |    +-----------------------------+   |
|  +-------------------------+    | Autonomous Data Warehouse    |   |
|                                 | - OCI_COST tables            |   |
|  +-------------------------+    | - OCI_SHOWOCI_* tables       |   |
|  | OCI Email Delivery      |    | - APEX Workspace + App       |   |
|  | - Daily email reports   |    +-----------------------------+   |
|  +-------------------------+         ^                            |
|                                      | (APEX access via LB       |
|                                      |  or Private Endpoint)     |
+------------------------------------------------------------------+
```

**Recommended deployment**: VM + ADW in OCI public cloud (same region as ExaCC control plane), with Instance Principals authentication. APEX accessed from corporate network via Load Balancer or VPN. This is what the Terraform at `terraform/` automates entirely.

---

## Implementation Roadmap

### Phase 0: Prerequisites (Week 1)

1. **Validate network connectivity** - Confirm ExaCC control plane region, verify VCN exists with NAT Gateway for outbound OCI API access
2. **Create IAM resources** - Dynamic Group matching the Usage2ADW VM, Policy with required statements (see `terraform/modules/iam/main.tf` lines 40-47):
   ```
   define tenancy usage-report as ocid1.tenancy.oc1..aaaaaaaaned4fkpkisbwjlr56u7cj63lf3wffbilvqknstgtvzub7vhqkggq
   endorse dynamic-group UsageDownloadGroup to read objects in tenancy usage-report
   Allow dynamic-group UsageDownloadGroup to inspect compartments in tenancy
   Allow dynamic-group UsageDownloadGroup to inspect tenancies in tenancy
   Allow dynamic-group UsageDownloadGroup to read autonomous-databases in compartment <APPCOMP>
   Allow dynamic-group UsageDownloadGroup to read secret-bundles in compartment <APPCOMP>
   ```
3. **Create KMS Vault + Secret** - Store the ADW admin password in OCI Vault
4. **Define tag strategy** - Decide TAG_SPECIAL mappings (e.g., CostCenter, Department, Environment, Project) and ensure ExaCC resources are tagged

### Phase 1: Core Deployment (Week 2)

**Using Terraform (Recommended)**:
1. Upload repo ZIP to OCI Resource Manager
2. Set Working Directory to `usage-reports-to-adw-main/terraform`
3. Configure: compartment, VCN/subnet, ADW (Private Endpoint), Secret OCID, VM shape, tag special keys, extract start date
4. Apply stack - creates ADW, VM, network, IAM (~10 min bootstrap)
5. SSH to VM, verify: `cat /home/opc/boot.log`

**Key files**: `terraform/main.tf`, `terraform/variables.tf`, `terraform/modules/*/main.tf`

### Phase 2: Verification (Week 2-3)

1. Run connectivity check: `python3 usage2adw_check_connectivity.py` (tests all OCI API access)
2. Verify data load: check `/home/opc/usage_reports_to_adw/report/local/*.txt` for "Rows Inserted"
3. Login to APEX (Workspace=Usage, User=Usage) - verify ExaCC costs appear
4. Filter by service = "Database" / "Exadata" to confirm ExaCC-specific line items

### Phase 3: ShowOCI Resource Inventory (Week 3-4)

1. Add IAM policy: `Allow dynamic-group UsageDownloadGroup to read all-resources in tenancy`
2. Install ShowOCI on VM
3. Run initial extract: `/home/opc/showoci/run_daily_report.sh`
4. Load CSVs to ADW: `shell_scripts/run_load_showoci_csv_to_adw.sh`
5. Verify `OCI_SHOWOCI_DATABASE_EXA_CC_VMS` table is populated with your ExaCC VM Clusters

### Phase 4: Email Reports and Alerting (Week 4-5)

1. Set up OCI Email Delivery (Approved Sender + SMTP credentials)
2. Install Postfix on VM, configure SMTP relay
3. Update `shell_scripts/run_daily_report.sh`: set `MAIL_FROM_EMAIL` and `MAIL_TO`
4. Schedule crontab (see Operational Procedures below)
5. Optional: configure APEX email subscriptions for self-service report delivery

### Phase 5: Customization (Week 5-8)

- **Tag-based chargebacks**: Configure `-ts CostCenter -ts2 Department -ts3 Environment -ts4 Project`
- **Custom APEX dashboards**: ExaCC infrastructure page, department cost allocation, capacity planning
- **Custom SQL views**: Per-database cost allocation by distributing VM Cluster costs proportionally:
  ```sql
  -- Example: Allocate ExaCC costs by per-DB OCPU share
  SELECT d.name AS database_name,
         d.cpu_core_count AS db_cpus,
         vm.cpu_core_count AS cluster_cpus,
         c.COST_MY_COST * (d.cpu_core_count / NULLIF(vm.cpu_core_count, 0)) AS allocated_cost
  FROM OCI_COST c
  JOIN OCI_SHOWOCI_DATABASE_EXA_CC_VMS vm ON ...
  JOIN OCI_SHOWOCI_DATABASES d ON ...
  WHERE c.PRD_SERVICE = 'DATABASE';
  ```
- **CSV exports**: Clone `run_report_compart_service_daily_to_csv.sh` for ExaCC-specific cost exports

### Phase 6: Production Hardening (Week 6-8)

- Switch ADW to Private Endpoint (if not already)
- Implement Load Balancer for APEX access
- Set up VM monitoring (OCI Monitoring Agent)
- Configure backup strategy
- Create additional APEX users per `step_by_step_howto.md` section 1
- Document operational runbook

---

## Operational Procedures

### Crontab Schedule
```bash
# Cost data load - midnight daily
0 0 * * * timeout 6h /home/opc/usage_reports_to_adw/shell_scripts/run_multi_daily_usage2adw.sh > /home/opc/usage_reports_to_adw/cron_run_multi_tenants_crontab_run.txt 2>&1

# ShowOCI extract - midnight daily
0 0 * * * timeout 23h /home/opc/showoci/run_daily_report.sh > /home/opc/showoci/run_daily_report_crontab_run.txt 2>&1

# ShowOCI CSV load to ADW - 8am daily
00 8 * * * timeout 2h /home/opc/usage_reports_to_adw/shell_scripts/run_load_showoci_csv_to_adw.sh > /home/opc/usage_reports_to_adw/cron/run_load_showoci_csv_to_adw.sh_run.txt 2>&1

# Email report - 9am daily
0 9 * * * timeout 6h /home/opc/usage_reports_to_adw/shell_scripts/run_daily_report.sh > /home/opc/usage_reports_to_adw/shell_scripts/run_daily_report_crontab_run.txt 2>&1

# Gather stats - Sunday midnight
30 0 * * 0 timeout 6h /home/opc/usage_reports_to_adw/shell_scripts/run_gather_stats.sh > /home/opc/usage_reports_to_adw/run_gather_stats_run.txt 2>&1
```

### Monitoring
- **Daily**: Check load logs for errors, verify APEX Data Statistics page, confirm email delivery
- **Weekly**: Check ADW storage via `run_table_size_info.sh`, review ShowOCI data freshness
- **Monthly**: Review cost trends, check VM disk space, upgrade app if new version available

---

## Integration with Existing Infrastructure

| Existing System | Integration Approach |
|----------------|---------------------|
| **OEM** | Complementary - OEM handles real-time performance monitoring; Usage2ADW handles financial reporting. Create OEM metric extensions that query `OCI_COST` for cost-per-performance analysis |
| **APEX Servers** | Use ADW's built-in APEX (recommended) - no need for separate servers. OR import `usage2adw_demo_apex_app.sql` into existing APEX if on-prem data residency required |
| **Oracle Databases** | Can use existing DB instead of ADW (README: "DbaaS can be used as well"). Create user with `connect, resource, dwrole` grants, use standard TNS connectivity |
| **Oracle Analytics Cloud** | Terraform supports optional OAC deployment (`terraform/modules/oac/main.tf`) for advanced analytics on top of Usage2ADW data |

---

## Limitations and Workarounds

| Limitation | Workaround |
|-----------|------------|
| No real-time data (24hr latency) | Use OEM/OCI Monitoring for real-time; Usage2ADW for trend analysis |
| ExaCC costs lack per-database breakdown | Create custom allocation views distributing costs by OCPU/storage share |
| No built-in budget alerts | Modify `run_daily_report.sh` to check thresholds, or use OCI Budgets |
| No per-PDB cost breakdown | Use `OCI_SHOWOCI_DATABASES_PDBS` inventory + custom allocation logic |
| No cost forecasting | Export to Oracle Analytics Cloud for predictive analysis |
| ShowOCI needs broad read policy | Scope `read all-resources` to specific compartments if possible |

---

## Verification

After deployment, verify end-to-end:

1. **Connectivity**: `python3 usage2adw_check_connectivity.py` - all 6 checks must pass
2. **Data load**: Check `OCI_LOAD_STATUS` table shows files loaded with recent timestamps
3. **APEX**: Login to APEX workspace, verify Cost Analysis shows ExaCC services
4. **ShowOCI**: Query `SELECT count(*) FROM OCI_SHOWOCI_DATABASE_EXA_CC_VMS` - should return your ExaCC VM Cluster count
5. **Email**: Trigger `run_daily_report.sh` manually, verify HTML email received
6. **Tags**: Verify `TAG_SPECIAL` columns populated in `OCI_COST` for tagged resources

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `usage2adw.py` | Core extraction engine - OCI cost/usage to ADW |
| `usage2adw_setup.sh` | Installation automation (-setup_full, -upgrade_app, -create_tables) |
| `usage2adw_check_connectivity.py` | Pre-flight OCI API connectivity test |
| `usage2adw_showoci_csv2adw.py` | ShowOCI CSV loader - 88+ infrastructure tables |
| `usage2adw_demo_apex_app.sql` | APEX visualization application |
| `usage2adw_download_adb_wallet.py` | ADW wallet generation |
| `usage2adw_retrieve_secret.py` | KMS Vault secret retrieval |
| `shell_scripts/run_multi_daily_usage2adw.sh` | Daily orchestration script |
| `shell_scripts/run_daily_report.sh` | Email report generation |
| `shell_scripts/run_load_showoci_csv_to_adw.sh` | ShowOCI data load |
| `terraform/main.tf` | Infrastructure-as-Code deployment |
| `terraform/modules/iam/main.tf` | Required IAM policy statements |
| `focus2adw/focus2adw.py` | FOCUS format reports (beta) |
| `step_by_step_installation.md` | Manual installation guide |
| `step_by_step_howto.md` | Operations manual (users, upgrades, email, scheduling) |
| `step_by_step_terraform.md` | Terraform deployment guide |
