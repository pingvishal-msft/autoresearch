# Research Direction: Minimal Metadata for On-Premises Workload Discovery

## Goal

Determine the **smallest set of metadata signals** that can identify
what workloads are running in a customer's on-premises VMware environment
with **fair confidence (≥ 75% accuracy)**, while requiring the **least
intrusive permissions** from the customer.

This matters because customers are reluctant to grant deep access to
their infrastructure. We need to know: *how much can we learn with how
little access?*

## How to Read the Results

Run `python prepare.py` then `python train.py`. The output is in
**results.tsv** — open it in Excel or any spreadsheet tool. Each row
is one experiment that tests a specific hypothesis:

| Column | What It Tells You |
|---|---|
| experiment | Experiment label (e.g. E01, E02) |
| hypothesis | Plain-English statement being tested |
| permission_level | What customer access is needed (NONE / LOW / MEDIUM / HIGH) |
| signals_used | Comma-separated list of metadata fields used |
| vms_tested | How many VMs were classified |
| correct | How many were classified correctly |
| accuracy_pct | Percentage correct |
| unknowns_flagged | VMs flagged as UNKNOWN (unidentifiable) |
| false_positives | VMs classified as the wrong workload type |
| avg_confidence | Mean confidence score (0–1) across all VMs |
| hypothesis_validated | YES / NO / PARTIAL |
| key_finding | One-line takeaway from this experiment |

Check the **hypothesis_validated** column to see what worked and what
didn't. Compare accuracy_pct across permission_level to see the ROI
of asking for more access.

## Hypotheses to Test

| # | Hypothesis | Permission Level |
|---|---|---|
| H1 | VM name pattern matching alone identifies >30% of workloads | NONE |
| H2 | VM name + OS type identifies >45% of workloads | NONE |
| H3 | Hardware profile (CPU/RAM/disk) distinguishes DB vs web servers | NONE |
| H4 | VMware Tools version flags legacy/EOL VMs with >90% precision | LOW |
| H5 | Process list from VMware Tools is the single most impactful signal | LOW |
| H6 | NSX listening ports alone achieve >65% workload classification | LOW |
| H7 | Full Layer 1 (vCenter + Tools + NSX) achieves ≥80% accuracy | LOW |
| H8 | Guest-level packages add <10% incremental accuracy over Layer 1 | HIGH |
| H9 | Ports + process list alone are sufficient for >75% accuracy | LOW |
| H10 | Removing process list from Layer 1 drops accuracy by >15pp | LOW |
| H11 | NSX flow count distinguishes simple (rehost) from complex (rearchitect) VMs | LOW |
| H12 | Two-signal minimum rule prevents >90% of false workload assignments | LOW |

## Signal Layers & Customer Permission Requirements

### Layer 0 — Basic vCenter (No Special Permission)
Collected via standard vSphere/vCenter API read-only access:
- VM name, guest OS type, OS version, hardware version
- vCPU count, memory allocation
- Disk count, total disk size
- Power state, uptime
- Snapshot count and age
- VM tags and custom attributes

**Permission ask:** Read-only vCenter API access (standard for any
VMware admin). Customers almost always grant this.

### Layer 1a — VMware Tools Guest Introspection (Low Permission)
Requires VMware Tools installed and running in the guest (default on
most VMware deployments):
- Running process list (names only — NOT command-line arguments)
- Network interface count and IP addresses
- Guest OS hostname and domain membership
- VMware Tools version and status

**Permission ask:** None beyond what's already deployed. VMware Tools
is pre-installed on most VMs. No credentials needed.

### Layer 1b — NSX Network Metadata (Low Permission)
Collected via NSX Manager API (if NSX is deployed):
- Active listening ports per VM
- East-west traffic flows (source/dest IP, port, protocol)
- Micro-segmentation group membership
- DNS query patterns (hostnames resolved, not content)

**Permission ask:** Read-only NSX Manager API access. No guest access.

### Layer 2 — Guest OS Access (High Permission)
Collected via SSH (Linux) or WinRM (Windows) — requires credentials:
- Installed packages/services
- Listening ports with owning process names
- Scheduled tasks and cron jobs (names only)
- Runtime version detection (Java, .NET, Python, Node.js)
- Config directory presence (not file contents):
  `/etc/nginx/`, `/etc/apache2/`, `C:\inetpub\`
- Data directory presence (not file contents):
  `/var/lib/mysql/`, `/var/lib/postgresql/`,
  `C:\Program Files\Microsoft SQL Server\`

**Permission ask:** SSH/WinRM credentials per VM. This is the most
intrusive level — many customers will refuse or limit this heavily.

## Privacy & Security Constraints

- Do NOT collect command-line arguments from processes
- Do NOT read file contents — path existence only
- Do NOT collect usernames, passwords, environment variables
- Do NOT collect /proc/environ or registry secrets
- Do NOT flag Rearchitect from a single signal — require ≥2
- UNKNOWN must always be preservable — never force-classify below 0.75

## Workload Catalogue

| Category | Process Signatures | Port Signatures |
|---|---|---|
| Web Server | nginx, httpd, apache2, w3wp.exe, iisexpress | 80, 443, 8080, 8443 |
| Database | mysqld, postgres, sqlservr, mongod, redis-server | 3306, 5432, 1433, 27017, 6379 |
| Middleware | beam.smp, rabbitmq-server, kafka (java) | 5672, 15672, 9092, 61616 |
| Java App | java, catalina, jboss, wildfly | 8080, 8443, 8009, 9990 |
| .NET App | w3wp.exe, dotnet, dotnet.exe | 80, 443, 5000, 5001 |
| Legacy/EOL | (any) on Win2008/2012, RHEL6/7, Ubuntu 14/16 | (any) |

## Confidence Scoring (Deterministic)

Each classification uses up to four signal families. Scores are additive:

| Signal Family | Weight | What Contributes |
|---|---|---|
| Process/package match | 0.35 | Process name matches catalogue entry |
| Listening port match | 0.30 | Port matches known workload port |
| Runtime/OS coherence | 0.20 | OS type consistent with workload type |
| Dependency flow coherence | 0.15 | NSX flows consistent with workload role |

Decision thresholds:
- **CONFIRMED** — confidence ≥ 0.85
- **PROBABLE** — 0.75 to 0.849
- **PARTIAL** — 0.65 to 0.749
- **UNKNOWN** — confidence < 0.75

## Migration Pattern Rules

| Pattern | Candidate Signals (≥2 required) |
|---|---|
| **Rehost** | No data directory, standard OS, <5 NSX flow destinations, no EOL components |
| **Replatform** | Standard DB engine, Java on Tomcat/JBoss, moderate flows (5–15), EOL OS + modern runtime |
| **Rearchitect** | EOL OS + EOL runtime, >15 flow destinations, .NET Framework on Win2008/2012, unmapped ports, snapshot age >180d |

## Experiment Execution

Run these in order. Each experiment outputs one row to results.tsv:

```
python prepare.py          # Generate synthetic VM estate (one-time)
python train.py            # Run all experiments, write results.tsv
```

Then open results.tsv to review. The key question to answer:

**"What is the cheapest permission level that gives us ≥75% workload
identification accuracy?"**

If the answer is Layer 1 (no guest credentials needed), that's the
strategy to recommend to customers.

2. Test whether VMware Tools process list + NSX ports outperforms 
   guest SSH collection for identifying web/app workloads specifically

3. Test optimal confidence threshold for UNKNOWN flagging — 
   compare 0.65, 0.75, 0.85 against false positive rate

4. Test whether snapshot age and Tools version are meaningful 
   predictors of Rearchitect pattern assignment independent of 
   OS/runtime signals

5. Test east-west NSX flow count thresholds (5/10/15/20) against 
   migration pattern assignment accuracy — find the count that best 
   separates Rehost from Replatform candidates

6. Test whether a two-pass approach (hypervisor-only first, 
   guest-enriched second) produces meaningfully better results than 
   a single-pass combining all signals simultaneously

7. Test weighted confidence scoring versus unweighted vote scoring
  for UNKNOWN precision at fixed 0.75 threshold

8. Test strict two-signal minimum versus three-signal minimum for
  Rearchitect assignment and compare false positive rate

---

## Output Schema Per VM

Each classified VM should produce:
```json
{
  "vm_name": "string",
  "guest_os": "string",
  "workload_type": "web|database|middleware|legacy|java|dotnet|unknown",
  "workload_confidence": 0.0–1.0,
  "matched_catalogue_entry": "string|null",
  "unknown_signals": ["list of unmatched signals if flagged"],
  "migration_pattern": "rehost|replatform|rearchitect|undetermined",
  "pattern_confidence": 0.0–1.0,
  "pattern_evidence": ["signal_1", "signal_2"],
  "wave_candidate": "1|2|3|4|undetermined",
  "rightsizing_input": {
    "vcpu_allocated": int,
    "memory_allocated_gb": float,
    "peak_cpu_percent": float,
    "peak_memory_percent": float
  },
  "guest_data_used": true|false,
  "metadata_source": ["vcenter", "vmtools", "nsx", "ssh|winrm"]
}
```

---

## Keep / Discard Criteria

- Keep if val_bpb improves (better generalisation to unseen VMs)
- Keep if unknown flagging precision improves without reducing 
  known workload recall
- Discard if any experiment requires guest access to achieve 
  baseline classification — hypervisor layer must stand alone
- Discard if Rearchitect is assigned from a single signal
- Discard if confidence thresholds below 0.65 are used — too noisy

---

## NEVER STOP
Run autonomously overnight. Do not wait for human input between
experiments. Log every kept experiment with the specific signals
that drove the improvement.
