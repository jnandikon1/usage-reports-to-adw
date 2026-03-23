# Usage2ADW for Oracle Cloud@Customer (ExaCC) - Complete Implementation Plan

## Context

**Problem**: Your organization runs Oracle Cloud@Customer (ExaCC) with Exadata infrastructure, ADB, APEX servers, OEM, and Oracle instances across **2 tenancies (Prod & Dev)** in a **private domain data center**. You need cost reporting and usage visibility for budget tracking, department chargebacks, and capacity planning.

**Solution**: Deploy **Usage2ADW** (Oracle open-source, v25.10.01) to extract OCI cost/usage reports into an **Autonomous Data Warehouse (ADW-S)** in OCI Cloud with **APEX dashboards**, automated via **Azure DevOps self-hosted agent**, with daily email reports.

**Architecture Decisions** (per your input):
- **VM**: OCI Cloud VM with Instance Principals authentication
- **Database**: OCI Cloud ADW-S (Shared) with built-in APEX
- **Automation**: Azure DevOps self-hosted agent on OCI VM for CI/CD pipelines (health checks, upgrades, config management, monitoring)
- **Tenancies**: 2 (Prod + Dev) loaded into single ADW for consolidated reporting
- **On-Prem Servers**: RHEL 9.0+ and Windows (for accessing APEX dashboards)

> **Disclaimer**: This is NOT official Oracle billing. Use OCI Cost Analysis for official calculations.

---

## Architecture

```
+------------------------------------------------------------------+
|              Your Data Center (Private Domain)                    |
|                                                                   |
|  +-------------------------+  +-------------------------------+   |
|  | ExaCC Infrastructure    |  | RHEL 9 / Windows Servers      |   |
|  | - Exadata Racks         |  | - Browser access to APEX      |   |
|  | - VM Clusters           |  | - OCI CLI configured          |   |
|  | - ADB-D Instances       |  +-------------------------------+   |
|  | - Oracle Databases      |                                      |
|  +-------------------------+                                      |
|  +-------------------------+                                      |
|  | OEM Server              |  (Complementary monitoring)          |
|  +-------------------------+                                      |
+------------------------------------------------------------------+
          |  FastConnect / VPN / Private Peering
          v
+------------------------------------------------------------------+
|                  OCI Cloud (Prod Tenancy)                         |
|                                                                   |
|  +-------------------------+  +-------------------------------+   |
|  | Object Storage          |  | Usage2ADW VM (Oracle Linux 8)  |  |
|  | "bling" bucket (Prod)   |->| - Python 3.9 + OCI SDK         |  |
|  +-------------------------+  | - usage2adw.py                 |  |
|                               | - Instance Principals           |  |
|  +-------------------------+  | - Postfix (email)              |  |
|  | Object Storage          |  | - Azure DevOps Self-Hosted Agt |  |
|  | "bling" bucket (Dev)    |->+--------------|----------------+   |
|  +-------------------------+  |              v                    |
|                               | +-----------------------------+  |
|  +-------------------------+  | | ADW-S (2 ECPU, 1TB, 23ai)  |  |
|  | KMS Vault               |  | | - OCI_COST tables           |  |
|  | - DB Password Secret    |  | | - OCI_SHOWOCI_* tables      |  |
|  +-------------------------+  | | - APEX Workspace + App      |  |
|                               | +-----------------------------+  |
|  +-------------------------+  |        ^                         |
|  | OCI Email Delivery      |  |        | HTTPS (443)             |
|  +-------------------------+  | +-----------------------------+  |
|                               | | Load Balancer (Public)      |  |
|                               | | -> Private Endpoint ADW     |  |
|                               | +-----------------------------+  |
+------------------------------------------------------------------+
```

---

## Excel Task Tracker - Complete Step-by-Step

> **Excel columns**: Task ID | Phase | Task Description | Owner | Exact Command/Action | Depends On | Verification | Status | Notes

---

### PHASE 0: PREREQUISITES & PLANNING (Week 1)

| ID | Task | Exact Action / Command | Depends On | Verification |
|----|------|----------------------|------------|--------------|
| P0-001 | Identify OCI Prod tenancy OCID | OCI Console > Administration > Tenancy Details > Copy OCID (`ocid1.tenancy.oc1..xxx`) | - | OCID noted |
| P0-002 | Identify OCI Dev tenancy OCID | OCI Console > Administration > Tenancy Details > Copy OCID | - | OCID noted |
| P0-003 | Identify Prod tenancy home region | OCI Console > Administration > Tenancy Details > Home Region (e.g., `us-ashburn-1`) | P0-001 | Region noted |
| P0-004 | Identify ExaCC control plane region | Same region as ExaCC was provisioned in | P0-003 | Matches Prod home region |
| P0-005 | Verify FastConnect/VPN connectivity | Confirm on-prem servers can reach OCI API endpoints via private network | - | `curl -I https://identity.<region>.oraclecloud.com` from on-prem |
| P0-006 | Choose OCI Compartment for Usage2ADW | OCI Console > Identity > Compartments > Create Compartment (e.g., `Usage2ADW`) | P0-001 | Compartment OCID noted |
| P0-007 | Choose VCN and Subnet for VM | OCI Console > Networking > VCNs > Select existing VCN with NAT Gateway + Service Gateway | P0-006 | VCN OCID + Subnet OCID noted |
| P0-008 | Choose Subnet for Load Balancer | Select a public subnet in the same VCN (for APEX access from on-prem) | P0-007 | LB Subnet OCID noted |
| P0-009 | Generate SSH key pair | `ssh-keygen -t rsa -b 4096 -f ~/.ssh/usage2adw_key` (on admin workstation) | - | Public key file ready |
| P0-010 | Define tag strategy for cost allocation | Decide 4 tag keys: TAG_SPECIAL=`CostCenter`, TAG_SPECIAL2=`Department`, TAG_SPECIAL3=`Environment`, TAG_SPECIAL4=`Project` | - | 4 tag key names documented |
| P0-011 | Create OCI Tag Namespace | OCI Console > Governance > Tag Namespaces > Create (e.g., `CostTracking`) | P0-010 | Namespace created |
| P0-012 | Create OCI Tag Keys | Create keys: `CostCenter`, `Department`, `Environment`, `Project` under namespace | P0-011 | 4 tag keys created |
| P0-013 | Tag all ExaCC resources | Apply tags to VM Clusters, ADB instances, databases in both Prod & Dev | P0-012 | Resources tagged in OCI Console |
| P0-014 | Choose ADW admin password | Generate password: 12-30 chars, 1 upper, 1 lower, 1 number, only `#` or `_` for symbols | - | Password securely stored |
| P0-015 | Create KMS Vault | OCI Console > Security > Vault > Create Vault (in Usage2ADW compartment) | P0-006 | Vault OCID noted |
| P0-016 | Create Master Encryption Key | Vault > Master Encryption Keys > Create Key (AES, 256-bit) | P0-015 | Key OCID noted |
| P0-017 | Create Secret for ADW password | Vault > Secrets > Create Secret > paste ADW admin password | P0-016, P0-014 | Secret OCID noted (critical - save this) |
| P0-018 | Choose extract start date | Decide how far back to load cost data (format: `YYYY-MM`, e.g., `2024-01`) | - | Date documented |
| P0-019 | Create OCI API key for Dev tenancy | OCI Console (Dev) > Identity > Users > API Keys > Add API Key > Download PEM | P0-002 | Config file snippet saved with user OCID, fingerprint, tenancy OCID, region, key_file path |
| P0-020 | Document all collected OCIDs | Create a spreadsheet with: Prod Tenancy OCID, Dev Tenancy OCID, Compartment OCID, VCN OCID, VM Subnet OCID, LB Subnet OCID, Secret OCID, Dev API Key path | All above | Master OCID reference sheet complete |

---

### PHASE 1: OCI INFRASTRUCTURE DEPLOYMENT (Week 2)

**Option A: Terraform via OCI Resource Manager (Recommended)**

| ID | Task | Exact Action / Command | Depends On | Verification |
|----|------|----------------------|------------|--------------|
| P1-001 | Download repository ZIP | Download from GitHub: `https://github.com/oracle-samples/usage-reports-to-adw` > Code > Download ZIP | - | ZIP file downloaded |
| P1-002 | Login to OCI Console (Prod) | Navigate to OCI Console with admin credentials | P0-001 | Logged in |
| P1-003 | Create Resource Manager Stack | OCI Console > Developer Services > Resource Manager > Stacks > Create Stack > Upload ZIP | P1-001 | Stack created |
| P1-004 | Set Terraform Working Directory | Set to: `usage-reports-to-adw-main/terraform` | P1-003 | Directory set |
| P1-005 | Configure Stack - Compartment | Select compartment: `Usage2ADW` (from P0-006) | P1-004 | Compartment selected |
| P1-006 | Configure Stack - Tags | Set freeform tags: `Project=Usage2ADW` | P1-004 | Tags configured |
| P1-007 | Configure Stack - IAM | Select: "New IAM Dynamic Group and Policy will be created". Set names: `Usage2ADW_DynGroup`, `Usage2ADW_Policy` | P1-004 | IAM option selected |
| P1-008 | Configure Stack - Network | Select VCN (P0-007), Subnet (P0-007). Option: "Provision Public Load Balancer". LB Subnet (P0-008). LB Name: `Usage2ADW_LB` | P1-004 | Network configured |
| P1-009 | Configure Stack - Database | Option: "Private Endpoint". DB Name: `ADWCUSG`. License: `BRING_YOUR_OWN_LICENSE` (if you have licenses) or `LICENSE_INCLUDED`. Secret Compartment + Secret OCID (P0-017) | P1-004 | ADW config set: 2 ECPU, 1TB, 23ai |
| P1-010 | Configure Stack - Compute | Shape: `VM.Standard.E4.Flex` (1 OCPU, 15GB RAM). SSH Key: paste public key from P0-009. Name: `Usage2ADW-VM` | P1-004 | Compute configured |
| P1-011 | Configure Stack - Extraction | Extract From Date: value from P0-018. Tag Special 1: `CostCenter`. Tag Special 2: `Department`. Tag Special 3: `Environment`. Tag Special 4: `Project` | P1-004, P0-010 | Tags configured |
| P1-012 | Review Terraform Plan | Click "Plan" > Review plan output for resources to be created | P1-005 to P1-011 | Plan shows: 1 ADW, 1 VM, 1 NSG, 1 LB, 1 Dynamic Group, 1 Policy |
| P1-013 | Apply Terraform Stack | Click "Apply" > Confirm | P1-012 | Job status: SUCCEEDED |
| P1-014 | Record Terraform Outputs | Copy all outputs: APEX URLs, LB IP, VM Private IP, VM Public IP, DB Secret ID | P1-013 | All outputs recorded in spreadsheet |
| P1-015 | Wait for bootstrap completion | Bootstrap takes ~10 minutes after Terraform completes (80s IAM wait + setup_full) | P1-013 | Wait 15 min |
| P1-016 | SSH into Usage2ADW VM | `ssh -i ~/.ssh/usage2adw_key opc@<VM_PUBLIC_IP>` | P1-014 | Connected successfully |
| P1-017 | Verify bootstrap log | `cat /home/opc/boot.log` - check for "completed successfully" | P1-016 | No errors in boot.log |
| P1-018 | Verify config.user created | `cat /home/opc/usage_reports_to_adw/config.user` | P1-016 | Shows DATABASE_USER=USAGE, DATABASE_NAME, SECRET_ID, TAG_SPECIAL values |
| P1-019 | Verify Python packages | `python3 -c "import oci; import oracledb; import requests; print('OK')"` | P1-016 | Prints "OK" |
| P1-020 | Verify Oracle Instant Client | `/usr/lib/oracle/current/client64/bin/sqlplus -V` | P1-016 | Shows version 23.x |
| P1-021 | Verify ADW wallet | `ls -la /home/opc/ADWCUSG/` | P1-016 | Contains: cwallet.sso, tnsnames.ora, sqlnet.ora, etc. |
| P1-022 | Verify crontab installed | `crontab -l` | P1-016 | Shows run_multi_daily_usage2adw.sh and run_gather_stats.sh entries |

---

### PHASE 2: VERIFICATION & INITIAL DATA LOAD (Week 2-3)

| ID | Task | Exact Action / Command | Depends On | Verification |
|----|------|----------------------|------------|--------------|
| P2-001 | Run connectivity check | `cd /home/opc/usage_reports_to_adw && python3 usage2adw_check_connectivity.py` | P1-016 | All 6 checks pass: Identity, Tenancy, Regions, Home Region, Compartments, Object Storage |
| P2-002 | Verify initial data load ran | `ls -la /home/opc/usage_reports_to_adw/report/local/` | P1-017 | Report files exist with today's date |
| P2-003 | Check load results | `cat /home/opc/usage_reports_to_adw/report/local/*.txt \| grep -E "Rows Inserted\|Total.*Files"` | P2-002 | Shows "Rows Inserted" > 0 and "Total X Cost Files Loaded" |
| P2-004 | Check for errors | `cat /home/opc/usage_reports_to_adw/report/local/*.txt \| grep -i error` | P2-002 | No errors found |
| P2-005 | Login to APEX (via LB) | Browser: `https://<LB_IP>/ords/f?p=100:LOGIN_DESKTOP` Workspace: `USAGE`, User: `USAGE`, Password: (your secret from P0-014) | P1-014 | APEX dashboard loads |
| P2-006 | Verify Cost Analysis page | APEX > Cost Analysis > Check data appears | P2-005 | Cost data visible with chart/table |
| P2-007 | Filter ExaCC costs | Cost Analysis > Filter by Service = "Database" or "Exadata" | P2-006 | ExaCC infrastructure costs appear |
| P2-008 | Check compartment breakdown | Cost Analysis > Filter by Compartment | P2-006 | Compartments match your org structure |
| P2-009 | Verify Data Statistics page | APEX > Data Statistics page | P2-005 | Shows OCI_LOAD_STATUS with loaded files, timestamps |
| P2-010 | Verify Rate Card | APEX > Rate Card page | P2-005 | Shows pricing data for your SKUs |
| P2-011 | Verify tag columns | APEX > Cost Analysis > Check TAG_SPECIAL filters appear | P2-005 | Tag-based filtering works |
| P2-012 | Run manual data load | `/home/opc/usage_reports_to_adw/shell_scripts/run_multi_daily_usage2adw.sh` | P2-001 | Completes without errors |
| P2-013 | Check table sizes | `/home/opc/usage_reports_to_adw/shell_scripts/run_table_size_info.sh` | P2-012 | OCI_COST table shows data size |

---

### PHASE 3: ADD DEV TENANCY (Week 3)

| ID | Task | Exact Action / Command | Depends On | Verification |
|----|------|----------------------|------------|--------------|
| P3-001 | Copy Dev API key to VM | `scp -i ~/.ssh/usage2adw_key dev_api_private_key.pem opc@<VM_IP>:/home/opc/.oci/` | P0-019 | Key file on VM |
| P3-002 | Create OCI config for Dev | Create `/home/opc/.oci/config` with `[dev]` profile section containing: user, fingerprint, tenancy, region, key_file for Dev tenancy | P3-001 | Config file with [dev] profile |
| P3-003 | Create IAM Policy on Dev tenancy | On Dev tenancy OCI Console, create policy: `define tenancy usage-report as ocid1.tenancy.oc1..aaaaaaaaned4fkpkisbwjlr56u7cj63lf3wffbilvqknstgtvzub7vhqkggq` + `endorse group <DevGroup> to read objects in tenancy usage-report` + `Allow group <DevGroup> to inspect compartments in tenancy` + `Allow group <DevGroup> to inspect tenancies in tenancy` | P0-002 | Policy created on Dev tenancy |
| P3-004 | Test Dev connectivity | `python3 usage2adw_check_connectivity.py -t dev -c /home/opc/.oci/config` | P3-002, P3-003 | All checks pass for Dev tenancy |
| P3-005 | Edit multi-tenant run script | Edit `/home/opc/usage_reports_to_adw/shell_scripts/run_multi_daily_usage2adw.sh` - Add line at bottom: `run_report dev CostCenter Department Environment Project` | P3-004 | Script updated |
| P3-006 | Run multi-tenant load | `/home/opc/usage_reports_to_adw/shell_scripts/run_multi_daily_usage2adw.sh` | P3-005 | Both Prod (local) and Dev tenants load |
| P3-007 | Verify Dev data in APEX | APEX > Cost Analysis > Filter by Tenant = Dev tenant name | P3-006 | Dev tenancy costs visible |
| P3-008 | Verify consolidated view | APEX > Cost Over Time > View both tenants | P3-007 | Both Prod and Dev data in single dashboard |

---

### PHASE 4: SHOWOCI RESOURCE INVENTORY (Week 3-4)

| ID | Task | Exact Action / Command | Depends On | Verification |
|----|------|----------------------|------------|--------------|
| P4-001 | Add ShowOCI IAM policy (Prod) | OCI Console (Prod) > Identity > Policies > Edit Usage2ADW_Policy > Add: `Allow dynamic-group Usage2ADW_DynGroup to read all-resources in tenancy` | P1-013 | Policy updated |
| P4-002 | Install ShowOCI on VM | SSH to VM: `bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-python-sdk/master/examples/showoci/showoci_upgrade.sh)"` | P4-001 | ShowOCI installed at `/home/opc/showoci/` |
| P4-003 | Set ShowOCI permissions | `chmod +x /home/opc/showoci/run_daily_report.sh` | P4-002 | Executable |
| P4-004 | Create cron output dir | `mkdir -p /home/opc/usage_reports_to_adw/cron` | P4-002 | Directory exists |
| P4-005 | Run ShowOCI initial extract | `/home/opc/showoci/run_daily_report.sh` (takes 1-4 hours for large tenancies) | P4-003 | CSV files generated at `/home/opc/showoci/report/local/csv/local/` |
| P4-006 | Verify ShowOCI CSV files | `ls -la /home/opc/showoci/report/local/csv/local/` | P4-005 | Files include: `database_db_exacc.csv`, `database_db_exa_infra.csv`, `compute.csv`, etc. |
| P4-007 | Download ShowOCI CSV loader | `wget https://raw.githubusercontent.com/oracle-samples/usage-reports-to-adw/main/shell_scripts/run_load_showoci_csv_to_adw.sh -O /home/opc/usage_reports_to_adw/shell_scripts/run_load_showoci_csv_to_adw.sh && chmod +x /home/opc/usage_reports_to_adw/shell_scripts/run_load_showoci_csv_to_adw.sh` | P4-004 | Script downloaded |
| P4-008 | Load ShowOCI CSVs to ADW | `/home/opc/usage_reports_to_adw/shell_scripts/run_load_showoci_csv_to_adw.sh` | P4-006, P4-007 | Completes without errors |
| P4-009 | Verify ExaCC infra table | SSH to VM: `/home/opc/usage_reports_to_adw/shell_scripts/run_sqlplus_usage.sh` then: `SELECT count(*) FROM OCI_SHOWOCI_DATABASE_EXA_INFRA;` | P4-008 | Count > 0 (matches your Exadata rack count) |
| P4-010 | Verify ExaCC VM Clusters table | `SELECT name, shape, cpu_core_count, db_storage_gb, memory_gb, node_count FROM OCI_SHOWOCI_DATABASE_EXA_CC_VMS;` | P4-008 | Shows your ExaCC VM Clusters |
| P4-011 | Verify databases table | `SELECT count(*) FROM OCI_SHOWOCI_DATABASES;` | P4-008 | Count matches your database count |
| P4-012 | Verify PDBs table | `SELECT count(*) FROM OCI_SHOWOCI_DATABASES_PDBS;` | P4-008 | Count matches your PDB count |
| P4-013 | Verify APEX ShowOCI page | APEX > ShowOCI Data page > Check infrastructure inventory displays | P4-008 | ExaCC data visible in dashboards |
| P4-014 | Add ShowOCI crontab entries | `crontab -e` and add: `0 0 * * * timeout 23h /home/opc/showoci/run_daily_report.sh > /home/opc/showoci/run_daily_report_crontab_run.txt 2>&1` and `00 8 * * * timeout 2h /home/opc/usage_reports_to_adw/shell_scripts/run_load_showoci_csv_to_adw.sh > /home/opc/usage_reports_to_adw/cron/run_load_showoci_csv_to_adw.sh_run.txt 2>&1` | P4-008 | `crontab -l` shows both entries |

---

### PHASE 5: EMAIL REPORTS (Week 4-5)

| ID | Task | Exact Action / Command | Depends On | Verification |
|----|------|----------------------|------------|--------------|
| P5-001 | Create Approved Sender in OCI | OCI Console > Solutions and Platform > Email Delivery > Approved Senders > Create: `costreport@yourdomain.com` | - | Sender approved |
| P5-002 | Generate SMTP credentials | OCI Console > Identity > Users > select user > SMTP Credentials > Generate. **Save username and password immediately** | P5-001 | SMTP username + password saved securely |
| P5-003 | Note SMTP endpoint for your region | Look up endpoint (e.g., Ashburn: `smtp.us-ashburn-1.oraclecloud.com`, Phoenix: `smtp.us-phoenix-1.oraclecloud.com`) | P5-001 | SMTP endpoint noted |
| P5-004 | Install Postfix on VM | SSH to VM: `sudo dnf install -y postfix` | P1-016 | Package installed |
| P5-005 | Configure firewall for SMTP | `sudo firewall-cmd --zone=public --add-service=smtp --permanent && sudo firewall-cmd --reload` | P5-004 | SMTP port open |
| P5-006 | Remove sendmail if present | `sudo dnf remove -y sendmail 2>/dev/null; sudo alternatives --set mta /usr/sbin/sendmail.postfix` | P5-004 | Sendmail removed |
| P5-007 | Install mailx | `sudo dnf install -y mailx` | P5-004 | Installed |
| P5-008 | Configure Postfix main.cf | Edit `/etc/postfix/main.cf` - add: `smtp_tls_security_level = may`, `smtp_sasl_auth_enable = yes`, `smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd`, `smtp_sasl_security_options =`, `relayhost = <SMTP_ENDPOINT>:587` | P5-003 | Config updated |
| P5-009 | Configure SMTP credentials file | Create `/etc/postfix/sasl_passwd` with: `<SMTP_ENDPOINT>:587 <SMTP_USERNAME>:<SMTP_PASSWORD>` | P5-002, P5-003 | File created |
| P5-010 | Secure SMTP credentials | `sudo chown root:root /etc/postfix/sasl_passwd && sudo chmod 600 /etc/postfix/sasl_passwd && sudo postmap hash:/etc/postfix/sasl_passwd` | P5-009 | Permissions set, hash map created |
| P5-011 | Start Postfix | `sudo systemctl enable postfix && sudo postfix start && sudo postfix reload` | P5-008, P5-010 | Service running |
| P5-012 | Test email delivery | `echo "Test from Usage2ADW" \| mail -s "Test" -r "costreport@yourdomain.com" your.email@company.com` | P5-011 | Test email received |
| P5-013 | Configure daily report script | Edit `/home/opc/usage_reports_to_adw/shell_scripts/run_daily_report.sh` - Set: `MAIL_FROM_EMAIL="costreport@yourdomain.com"`, `MAIL_TO="team-dl@company.com"` | P5-012 | Variables updated |
| P5-014 | Test daily report manually | `/home/opc/usage_reports_to_adw/shell_scripts/run_daily_report.sh` | P5-013, P2-012 | HTML email received with 5 report tables |
| P5-015 | Add daily report to crontab | `crontab -e` - add: `0 7 * * * timeout 6h /home/opc/usage_reports_to_adw/shell_scripts/run_daily_report.sh > /home/opc/usage_reports_to_adw/shell_scripts/run_daily_report_crontab_run.txt 2>&1` | P5-014 | Crontab entry verified |
| P5-016 | Configure tenant usage report (optional) | Edit `shell_scripts/run_tenant_usage_report.sh` - add entries: `run_report <prod_name> <prod_id> "prod-team@company.com"` and `run_report <dev_name> <dev_id> "dev-team@company.com"` | P5-012, P3-006 | Per-tenant emails configured |
| P5-017 | Configure APEX email (optional) | Connect as ADMIN to ADW via sqlplus: `BEGIN APEX_INSTANCE_ADMIN.SET_PARAMETER('SMTP_HOST_ADDRESS','<SMTP_ENDPOINT>'); APEX_INSTANCE_ADMIN.SET_PARAMETER('SMTP_USERNAME','<SMTP_USER>'); APEX_INSTANCE_ADMIN.SET_PARAMETER('SMTP_PASSWORD','<SMTP_PASS>'); COMMIT; END;` | P5-011 | APEX can send subscription emails |

---

### PHASE 6: AZURE DEVOPS SELF-HOSTED AGENT AUTOMATION (Week 5-6)

| ID | Task | Exact Action / Command | Depends On | Verification |
|----|------|----------------------|------------|--------------|
| P6-001 | Create Azure DevOps project | Azure DevOps > New Project > Name: `Usage2ADW-OCI` > Visibility: Private | - | Project created |
| P6-002 | Create Agent Pool | Azure DevOps > Project Settings > Agent Pools > Add Pool > Self-hosted > Name: `Usage2ADW-Pool` > Grant access to all pipelines | P6-001 | Pool created |
| P6-003 | Generate Personal Access Token (PAT) | Azure DevOps > User Settings > Personal Access Tokens > New Token > Scopes: Agent Pools (Read & Manage), Build (Read & Execute) > Expiration: 1 year | P6-001 | PAT saved securely |
| P6-004 | Download agent on Usage2ADW VM | SSH to VM: `mkdir -p /home/opc/azagent && cd /home/opc/azagent && curl -fkSL -o vsts-agent-linux-x64.tar.gz https://vstsagentpackage.azureedge.net/agent/3.248.0/vsts-agent-linux-x64-3.248.0.tar.gz && tar xzf vsts-agent-linux-x64.tar.gz` | P1-016, P6-003 | Agent files extracted |
| P6-005 | Configure agent | `cd /home/opc/azagent && ./config.sh --unattended --url https://dev.azure.com/<YOUR_ORG> --auth pat --token <PAT> --pool Usage2ADW-Pool --agent Usage2ADW-VM --acceptTeeEula --replace` | P6-004 | Agent configured, `.agent` and `.credentials` files created |
| P6-006 | Install agent as systemd service | `cd /home/opc/azagent && sudo ./svc.sh install opc && sudo ./svc.sh start` | P6-005 | `sudo ./svc.sh status` shows running |
| P6-007 | Verify agent online in Azure DevOps | Azure DevOps > Agent Pools > Usage2ADW-Pool > Agents tab | P6-006 | Agent shows "Online" with green indicator |
| P6-008 | Create Git repo for pipelines | Azure DevOps > Repos > Initialize with README. Push pipeline YAML files (see below) | P6-001 | Repo with `pipelines/` directory |
| P6-009 | Create pipeline - Health Check | Create `pipelines/healthcheck.yml` (see YAML below). Azure DevOps > Pipelines > New Pipeline > Azure Repos Git > select repo > Existing YAML > path: `pipelines/healthcheck.yml` | P6-008 | Pipeline created, manual run succeeds |
| P6-010 | Create pipeline - Config Management | Create `pipelines/configure.yml`: Pipeline with variables for DATABASE_NAME, SECRET_ID, TAG keys. Steps: template `config.user`, update `run_multi_daily_usage2adw.sh` tenant list, update `run_daily_report.sh` email settings | P6-008 | Pipeline manages config files |
| P6-011 | Create pipeline - Upgrade App | Create `pipelines/upgrade.yml`: Step runs `bash /home/opc/usage_reports_to_adw/usage2adw_setup.sh -upgrade_app` with timeout 30min | P6-008 | Pipeline upgrades app on demand |
| P6-012 | Create pipeline - Wallet Refresh | Create `pipelines/refresh-wallet.yml`: Step runs `bash /home/opc/usage_reports_to_adw/usage2adw_setup.sh -download_wallet` | P6-008 | Pipeline refreshes ADW wallet |
| P6-013 | Create pipeline - Monitoring | Create `pipelines/monitor.yml`: Steps check log files for errors, verify last load timestamp < 24h, check disk space > 20% free, publish test results for visibility | P6-008 | Pipeline detects issues and reports status |
| P6-014 | Schedule monitoring pipeline | Edit `pipelines/monitor.yml` > Add `schedules:` trigger with `cron: '0 */6 * * *'` (every 6 hours) | P6-013 | Scheduled runs appear in pipeline history |
| P6-015 | Schedule wallet refresh pipeline | Edit `pipelines/refresh-wallet.yml` > Add `schedules:` trigger with `cron: '0 0 1 */6 *'` (1st of every 6th month) | P6-012 | Scheduled every 6 months |
| P6-016 | Create Variable Group for secrets | Azure DevOps > Pipelines > Library > Variable Groups > New: `Usage2ADW-Config`. Add: `DATABASE_NAME`, `DATABASE_SECRET_ID`, `MAIL_FROM_EMAIL`, `MAIL_TO`. Link to Azure Key Vault if available | P6-001 | Variable group created, linked to pipelines |
| P6-017 | Test all pipelines | Run each pipeline manually from Azure DevOps | P6-009 to P6-015 | All pipelines execute successfully on self-hosted agent |
| P6-018 | Set up pipeline notifications | Azure DevOps > Project Settings > Notifications > New Subscription > Pipeline run failed > Send to team email | P6-017 | Team notified on pipeline failures |

#### Pipeline YAML Reference: Health Check (`pipelines/healthcheck.yml`)

```yaml
trigger: none
schedules:
  - cron: '0 */6 * * *'
    displayName: 'Every 6 hours'
    branches:
      include: [main]
    always: true

pool: Usage2ADW-Pool

steps:
  - script: |
      echo "=== Python & Dependencies ==="
      python3 -c "import oci; import oracledb; import requests; print('OK')"

      echo "=== Oracle Instant Client ==="
      /usr/lib/oracle/current/client64/bin/sqlplus -V

      echo "=== ADW Wallet ==="
      ls -la /home/opc/ADWCUSG/cwallet.sso

      echo "=== Crontab ==="
      crontab -l | grep -c usage_reports_to_adw

      echo "=== Disk Space ==="
      DISK_PCT=$(df -h / | awk 'NR==2 {gsub(/%/,""); print $5}')
      echo "Disk usage: ${DISK_PCT}%"
      if [ "$DISK_PCT" -gt 80 ]; then
        echo "##vso[task.logissue type=warning]Disk usage above 80%"
      fi

      echo "=== Last Cost Load ==="
      LAST_LOG=$(ls -t /home/opc/usage_reports_to_adw/log/run_multi_daily_usage2adw_crontab_run.txt 2>/dev/null)
      if [ -n "$LAST_LOG" ]; then
        LAST_MOD=$(stat -c %Y "$LAST_LOG")
        NOW=$(date +%s)
        HOURS_AGO=$(( (NOW - LAST_MOD) / 3600 ))
        echo "Last load: ${HOURS_AGO} hours ago"
        if [ "$HOURS_AGO" -gt 24 ]; then
          echo "##vso[task.logissue type=error]Cost load is more than 24 hours behind"
          exit 1
        fi
      fi
    displayName: 'Usage2ADW Health Check'
```

#### Pipeline YAML Reference: Monitoring (`pipelines/monitor.yml`)

```yaml
trigger: none
schedules:
  - cron: '0 */6 * * *'
    displayName: 'Every 6 hours'
    branches:
      include: [main]
    always: true

pool: Usage2ADW-Pool

steps:
  - script: |
      echo "=== Checking cost load logs for errors ==="
      LOG_DIR="/home/opc/usage_reports_to_adw/log"
      REPORT_DIR="/home/opc/usage_reports_to_adw/report"
      ERRORS=0

      for LOG in "$LOG_DIR"/*.txt "$REPORT_DIR"/local/*.txt; do
        if [ -f "$LOG" ]; then
          if grep -qi "error\|exception\|traceback" "$LOG" 2>/dev/null; then
            echo "##vso[task.logissue type=warning]Errors found in: $LOG"
            grep -i "error\|exception" "$LOG" | tail -5
            ERRORS=$((ERRORS + 1))
          fi
        fi
      done

      echo "=== Checking ShowOCI logs ==="
      SHOWOCI_LOG="/home/opc/showoci/run_daily_report_crontab_run.txt"
      if [ -f "$SHOWOCI_LOG" ]; then
        if grep -qi "error\|exception" "$SHOWOCI_LOG"; then
          echo "##vso[task.logissue type=warning]Errors in ShowOCI log"
          ERRORS=$((ERRORS + 1))
        fi
      fi

      if [ "$ERRORS" -gt 0 ]; then
        echo "##vso[task.logissue type=error]Found $ERRORS log files with errors"
        exit 1
      fi
      echo "All logs clean."
    displayName: 'Check Logs for Errors'

  - script: |
      echo "=== Disk Space Check ==="
      df -h / /home/opc
      DISK_PCT=$(df / | awk 'NR==2 {gsub(/%/,""); print $5}')
      if [ "$DISK_PCT" -gt 90 ]; then
        echo "##vso[task.logissue type=error]CRITICAL: Disk usage at ${DISK_PCT}%"
        exit 1
      elif [ "$DISK_PCT" -gt 80 ]; then
        echo "##vso[task.logissue type=warning]Disk usage at ${DISK_PCT}%"
      fi
    displayName: 'Check Disk Space'

  - script: |
      echo "=== ADW Connectivity Check ==="
      cd /home/opc/usage_reports_to_adw
      python3 usage2adw_check_connectivity.py 2>&1 | tail -20
    displayName: 'Verify ADW Connectivity'
```

#### Pipeline YAML Reference: Upgrade (`pipelines/upgrade.yml`)

```yaml
trigger: none
pool: Usage2ADW-Pool

steps:
  - script: |
      cd /home/opc/usage_reports_to_adw
      echo "=== Current version ==="
      grep -i "version" usage2adw.py | head -3

      echo "=== Running upgrade ==="
      bash usage2adw_setup.sh -upgrade_app

      echo "=== New version ==="
      grep -i "version" usage2adw.py | head -3
    displayName: 'Upgrade Usage2ADW Application'
    timeoutInMinutes: 30
```

---

### PHASE 7: CSV EXPORTS & CUSTOM REPORTS (Week 5-6)

| ID | Task | Exact Action / Command | Depends On | Verification |
|----|------|----------------------|------------|--------------|
| P7-001 | Test compartment/service CSV export | `/home/opc/usage_reports_to_adw/shell_scripts/run_report_compart_service_daily_to_csv.sh` | P2-012 | CSV file at `report/daily/daily_compartment_service_YYYYMMDD.csv` |
| P7-002 | Test compartment/service/SKU CSV export | `/home/opc/usage_reports_to_adw/shell_scripts/run_report_compart_service_sku_daily_to_csv.sh` | P2-012 | CSV with tenant, date, compartment, service, sku, desc, total |
| P7-003 | Schedule CSV exports in crontab | Add: `0 9 * * * /home/opc/usage_reports_to_adw/shell_scripts/run_report_compart_service_daily_to_csv.sh` | P7-001 | Daily CSV files generated |
| P7-004 | Create ExaCC cost allocation SQL view | Connect to ADW as USAGE user. Create `VIEW VW_EXACC_COST_BY_DB` joining OCI_COST + OCI_SHOWOCI_DATABASE_EXA_CC_VMS + OCI_SHOWOCI_DATABASES to allocate costs by OCPU share | P4-010 | View returns per-database cost allocation |
| P7-005 | Create monthly chargeback SQL view | Create `VIEW VW_MONTHLY_CHARGEBACK` grouping OCI_COST by TAG_SPECIAL (CostCenter), TAG_SPECIAL2 (Department), month | P2-011 | View returns department-level monthly costs |
| P7-006 | Create APEX custom page (optional) | APEX > App Builder > Create Page > Report on VW_MONTHLY_CHARGEBACK | P7-005 | Custom chargeback page accessible |

---

### PHASE 8: PRODUCTION HARDENING (Week 6-8)

| ID | Task | Exact Action / Command | Depends On | Verification |
|----|------|----------------------|------------|--------------|
| P8-001 | Verify ADW Private Endpoint | OCI Console > ADW > Network > Confirm Private Endpoint enabled, NSG attached with ports 1522 + 443 | P1-013 | Private Endpoint active |
| P8-002 | Verify Load Balancer health | OCI Console > Networking > Load Balancers > Check backend health = OK | P1-013 | Backend healthy, LB IP accessible |
| P8-003 | Test APEX from on-prem browser | From Windows/RHEL desktop: `https://<LB_IP>/ords/f?p=100:LOGIN_DESKTOP` | P8-002, P0-005 | APEX loads from on-prem network |
| P8-004 | Create additional APEX users | APEX > Administration > Manage Users and Groups > Create User for each team member (Finance, Ops, Management) | P2-005 | Users created with appropriate access |
| P8-005 | Update OCI_TENANT display names | APEX > Tenant Display Update page > Set friendly names for Prod and Dev tenants | P3-007 | Tenant names show as "Production" and "Development" in reports |
| P8-006 | Verify ADW auto-backup | OCI Console > ADW > Backups > Confirm automatic backups enabled | P1-013 | Backups listed |
| P8-007 | Set up OCI Monitoring for VM | OCI Console > Monitoring > Alarms > Create alarm for VM CPU > 80%, Disk > 90% | P1-013 | Alarms configured |
| P8-008 | Set up OCI Notifications | OCI Console > Notifications > Create Topic + Subscription (email) for alarm notifications | P8-007 | Email alerts configured |
| P8-009 | Document runbook | Create operations document covering: daily checks, error recovery, wallet refresh, upgrade procedure, password rotation, troubleshooting | All phases | Runbook document complete |
| P8-010 | Conduct team training | Walk through APEX dashboards, email reports, and CSV exports with Finance/Ops teams | P8-004 | Team trained |

---

### PHASE 9: ONGOING OPERATIONS (Recurring)

| ID | Task | Frequency | Exact Action | Verification |
|----|------|-----------|-------------|--------------|
| P9-001 | Verify daily cost load | Daily | Check `tail -20 /home/opc/usage_reports_to_adw/log/run_multi_daily_usage2adw_crontab_run.txt` for "Completed at" | No errors |
| P9-002 | Verify ShowOCI extract | Daily | Check `tail -5 /home/opc/showoci/run_daily_report_crontab_run.txt` | Completed successfully |
| P9-003 | Verify daily email received | Daily | Check inbox for "Cost Usage Report" email | Email received with 5 tables |
| P9-004 | Check ADW storage | Weekly | Run `run_table_size_info.sh` | Storage within limits |
| P9-005 | Check VM disk space | Weekly | `df -h` on VM | >20% free |
| P9-006 | Review cost anomalies | Weekly | APEX > Cost Over Time > Look for spikes | No unexpected cost increases |
| P9-007 | Run gather stats | Auto (Sunday) | Crontab runs `run_gather_stats.sh` | Verify Monday morning: no ORA- errors |
| P9-008 | Upgrade Usage2ADW app | As released | `bash -c "export usage2adw_param=-upgrade_app; $(curl -L https://raw.githubusercontent.com/oracle-samples/usage-reports-to-adw/main/usage2adw_setup.sh)"` | New version confirmed |
| P9-009 | Refresh ADW wallet | Every 6 months | `usage2adw_setup.sh -download_wallet` | New wallet extracted |
| P9-010 | Rotate ADW password | Per security policy | Change in KMS Vault Secret, then: `ALTER USER USAGE IDENTIFIED BY <new_pass>;` in sqlplus as ADMIN | Load still works |
| P9-011 | Review tag compliance | Monthly | Check new resources are tagged with CostCenter/Department/Environment/Project | TAG_SPECIAL columns populated |

---

## Complete Crontab Reference (VM: /home/opc)

```bash
# Usage2ADW cost data load - every 4 hours
0 */4 * * * timeout 6h /home/opc/usage_reports_to_adw/shell_scripts/run_multi_daily_usage2adw.sh > /home/opc/usage_reports_to_adw/log/run_multi_daily_usage2adw_crontab_run.txt 2>&1

# ShowOCI infrastructure extract - midnight daily
0 0 * * * timeout 23h /home/opc/showoci/run_daily_report.sh > /home/opc/showoci/run_daily_report_crontab_run.txt 2>&1

# ShowOCI CSV load to ADW - 8am daily (after ShowOCI completes)
00 8 * * * timeout 2h /home/opc/usage_reports_to_adw/shell_scripts/run_load_showoci_csv_to_adw.sh > /home/opc/usage_reports_to_adw/cron/run_load_showoci_csv_to_adw.sh_run.txt 2>&1

# Daily email cost report - 9am
0 9 * * * timeout 6h /home/opc/usage_reports_to_adw/shell_scripts/run_daily_report.sh > /home/opc/usage_reports_to_adw/shell_scripts/run_daily_report_crontab_run.txt 2>&1

# CSV export - 9:30am daily
30 9 * * * timeout 2h /home/opc/usage_reports_to_adw/shell_scripts/run_report_compart_service_daily_to_csv.sh > /home/opc/usage_reports_to_adw/log/run_csv_export_run.txt 2>&1

# Gather database stats - Sunday 00:30
30 0 * * 0 timeout 6h /home/opc/usage_reports_to_adw/shell_scripts/run_gather_stats.sh > /home/opc/usage_reports_to_adw/log/run_gather_stats_run.txt 2>&1
```

---

## Troubleshooting Quick Reference

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Error obtaining instance principals certificate" | IAM policy not propagated | Wait 10 min, verify Dynamic Group matching rule |
| "Error retrieving Secret" | Missing `read secret-bundles` policy | Add policy for secret compartment |
| "Error manipulating database" | Wallet expired or wrong password | Run `usage2adw_setup.sh -download_wallet` |
| "Total 0 cost files found" | Not running from home region | Check `config['region']` matches home region |
| APEX login fails | USAGE user locked | `ALTER USER USAGE ACCOUNT UNLOCK;` (as ADMIN) |
| "usage2adw.py is already running" | Previous run still active | `ps -ef \| grep usage2adw.py`, kill stale process |
| No email received | Postfix misconfigured | Check `/var/log/maillog`, verify SMTP credentials |
| Dev tenant shows no data | API key or policy issue | Run `usage2adw_check_connectivity.py -t dev` |

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `usage2adw.py` | Core engine - extracts cost CSVs from OCI Object Storage, loads to ADW |
| `usage2adw_setup.sh` | Setup automation: `-setup_full`, `-upgrade_app`, `-create_tables`, `-download_wallet` |
| `usage2adw_check_connectivity.py` | Pre-flight test: Identity, Tenancy, Regions, Compartments, Object Storage, Rates API |
| `usage2adw_showoci_csv2adw.py` | Loads 88+ ShowOCI infrastructure tables including ExaCC-specific tables |
| `usage2adw_demo_apex_app.sql` | APEX application with 13 pages: cost analysis, trends, rate card, ShowOCI |
| `usage2adw_download_adb_wallet.py` | Downloads and extracts ADW mTLS wallet |
| `usage2adw_retrieve_secret.py` | Retrieves password from OCI KMS Vault |
| `shell_scripts/run_multi_daily_usage2adw.sh` | Multi-tenant daily orchestration (Prod + Dev) |
| `shell_scripts/run_daily_report.sh` | HTML email report: daily cost, monthly cost, OCPU, storage, by-service |
| `shell_scripts/run_load_showoci_csv_to_adw.sh` | Loads ShowOCI CSV files into ADW tables |
| `shell_scripts/run_gather_stats.sh` | Weekly database statistics gathering |
| `shell_scripts/run_report_compart_service_daily_to_csv.sh` | CSV export: cost by compartment/service/day |
| `shell_scripts/run_table_size_info.sh` | Reports database object sizes |
| `shell_scripts/run_sqlplus_usage.sh` | Interactive SQL*Plus connection to ADW as USAGE user |
| `terraform/` | Full IaC: ADW + VM + Network + IAM + Load Balancer |
| `config.user` (created at runtime) | Stores: DATABASE_USER, DATABASE_NAME, SECRET_ID, EXTRACT_DATE, TAG_SPECIAL keys |

---

## What You Get When Done

1. **APEX Dashboard** - 13-page web app accessible from on-prem browsers via Load Balancer
2. **Consolidated Cost View** - Prod + Dev tenancy costs in single dashboard
3. **ExaCC Infrastructure Inventory** - VM Clusters, databases, PDBs, Exadata rack details
4. **Daily Email Reports** - Cost trends, OCPU usage, storage usage delivered to your inbox
5. **Tag-Based Chargeback** - Filter costs by CostCenter, Department, Environment, Project
6. **CSV Exports** - Daily cost files for finance/ERP integration
7. **Azure DevOps Pipelines** - Self-hosted agent with pipelines for health checks, upgrades, config management, monitoring, and scheduled wallet refresh
8. **Rate Card Comparison** - Your actual costs vs. public PAYG pricing
