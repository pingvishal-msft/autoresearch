# Research Direction: VMware Workload Discovery & Migration Classification

## Goal
Discover the optimal combination of hypervisor and guest metadata signals
that maximises accurate workload identification and migration pattern
assignment across a VMware estate. Lower val_bpb = better generalisation
of classification rules to unseen VMs not in the training inventory.

## Setup First (Mandatory)
Before running autonomous experiments, enforce this fixed setup:
1. Build a baseline dataset split by VM identity (no VM leakage):
   - train: 70%
   - validation: 15%
   - test: 15%
2. Ensure all labels are from the catalogue below and include UNKNOWN.
3. Store only metadata fields explicitly allowed in this file.
4. Run a baseline hypervisor-only classifier before any guest-enriched run.
5. Keep confidence threshold floor at 0.65 (never lower).
6. Keep UNKNOWN preservation enabled (never force-map below threshold).

## Autonomous Execution Protocol
Run experiments in this strict sequence and keep/discard after each run:
1. Baseline A: vCenter-only (Layer 1 subset)
2. Baseline B: vCenter + NSX
3. Baseline C: vCenter + VMware Tools + NSX
4. Enriched D: Layer 1 + Layer 2 (SSH/WinRM)
5. Ablations around the current best run only

For every run:
- Train/evaluate on the same split definition.
- Log metrics and exact signal bundle.
- Keep only if all of the following are true:
  - Better val_bpb than current best
  - UNKNOWN precision does not regress by more than 1%
  - Known-workload recall does not regress by more than 1%
- Otherwise discard and revert to current best configuration.

## Confidence Scoring Rules
Use deterministic confidence scoring for reproducibility:
- Workload confidence:
  - 0.35 from process/package signature matches
  - 0.30 from listening-port matches
  - 0.20 from runtime/OS coherence
  - 0.15 from dependency-flow coherence
- Pattern confidence:
  - Average of matched migration signals
  - Require at least 2 independent signals for any non-undetermined assignment

Decision thresholds:
- CONFIRMED: confidence >= 0.85
- PROBABLE: 0.75-0.849
- PARTIAL: 0.65-0.749
- UNKNOWN: < 0.75 for workload mapping

## Experiment Budgeting
Use a fixed nightly budget:
- Max experiments per cycle: 20
- Early stop if no improvement after 6 consecutive discarded runs
- Every 5 runs, re-evaluate current best on test split before continuing

## Logging Format (Required)
Log each run as one append-only JSON line with these keys:
- run_id
- timestamp_utc
- signal_bundle
- guest_data_used
- workload_accuracy
- unknown_precision
- known_recall
- pattern_accuracy
- val_bpb
- keep_or_discard
- discard_reason
- promoted_as_best

Also log a best-so-far summary after each promoted run with:
- best_run_id
- best_val_bpb
- best_signal_bundle
- deltas_vs_previous_best

Success is measured by three compound outcomes:
1. Classification accuracy — correct workload type identified per VM
2. Pattern assignment confidence — rehost / replatform / rearchitect 
   recommendation backed by ≥2 independent metadata signals
3. Unknown flagging precision — unknowns flagged only when no known 
   catalogue entry matches above a confidence threshold of 0.75

---

## Metadata Sources Available

### Layer 1 — Hypervisor (No Guest Permission Required)
Collected via vSphere/vCenter API:
- VM name, guest OS type, OS version, hardware version
- vCPU count, memory allocation, memory active (vROps)
- Disk count, disk size, datastore type (VMFS/vSAN/NFS)
- Power state, uptime, snapshot count and age
- VMware Tools version and status
- VM tags and custom attributes

Collected via VMware Tools (guest introspection):
- Running process list (names only, NOT command-line arguments)
- Network interface count and IP addresses
- Disk I/O and CPU ready metrics
- Guest OS hostname and domain membership

Collected via NSX (network metadata):
- East-west traffic flows between VMs (source/dest IP, port, protocol)
- Micro-segmentation group membership
- Active listening ports per VM
- DNS query patterns (hostnames resolved, not content)

### Layer 2 — Guest (User Permission Required)
Collected via SSH (Linux) or WinRM (Windows):
- Installed packages / services: `rpm -qa`, `dpkg -l`, `sc query`, 
  `Get-Service`
- Active listening ports with owning process names: `ss -tlnp`, 
  `netstat -ano`
- Scheduled tasks and cron jobs (names only)
- JVM processes and versions: `java -version`
- Web server config file presence (not content): 
  `/etc/nginx/`, `/etc/apache2/`, `C:\inetpub\`
- Database data directory presence (not content):
  `/var/lib/mysql/`, `/var/lib/postgresql/`, 
  `C:\Program Files\Microsoft SQL Server\`
- Runtime version detection: Python, Node.js, .NET, Ruby, PHP

---

## Workload Catalogue

### Known Workloads — Match Against These
| Category | Signals to Match |
|---|---|
| Web/App Servers | nginx, apache2, IIS, tomcat, jboss, websphere, listening on 80/443/8080/8443 |
| Databases | mysql, postgresql, oracle, mssql, mongodb, redis, listening on 3306/5432/1521/1433/27017/6379 |
| Middleware/Integration | activemq, rabbitmq, kafka, mulesoft, tibco, websphere MQ, listening on 5672/9092/61616 |
| Legacy/EOL OS | Windows Server 2008/2012, RHEL 6/7, Ubuntu 14.04/16.04, VMware Tools version <10 |
| Java Workloads | JVM process present, heap >512MB, .jar/.war/.ear files detected |
| .NET Workloads | w3wp.exe, dotnet process, .NET Framework 4.x or .NET Core detected |

### Unknown Workload Flagging Rules
- Flag as UNKNOWN if: no catalogue entry matches with confidence ≥0.75
- Flag as PARTIAL if: 1 signal matches but ≥2 are required for confirmation
- Never suppress an unknown — always surface it with the raw signals 
  that were found

---

## Migration Pattern Assignment Rules

Test different signal combinations to assign migration patterns:

### Rehost (Lift and Shift)
Candidate signals:
- Stateless process list (no persistent data directory detected)
- Standard OS, no EOL components
- Low east-west dependency count (NSX flows < 5 unique destinations)
- No custom kernel modules detected
- No hardcoded IP references in config file presence check

### Replatform (Lift and Reshape)
Candidate signals:
- Database present but standard engine (MySQL, PostgreSQL, MSSQL)
- Java workload on Tomcat/JBoss (containerisable)
- Web server present but no custom modules in config path
- Moderate east-west dependencies (5–15 NSX flows)
- EOL OS but modern runtime (e.g. RHEL 6 running Node.js 18)

### Rearchitect
Candidate signals:
- EOL OS + EOL runtime combination
- Tightly coupled dependency cluster (NSX flows > 15 unique destinations)
- Monolithic .NET Framework 4.x on Windows Server 2008/2012
- Undocumented listening ports with no known catalogue match
- Snapshot age > 180 days (suggests fragile, unmaintained state)

---

## Constraints

- Do NOT modify prepare.py
- Do NOT collect or store command-line arguments from any process
- Do NOT read file contents — detect presence only (path existence)
- Do NOT collect usernames, passwords, environment variables, or 
  /proc/environ equivalents
- Do NOT flag a workload as Rearchitect based on a single signal alone
  — require ≥2 independent signals
- Guest layer (SSH/WinRM) enrichment is optional — all Layer 1 
  signals must be sufficient for a baseline classification
- An unknown flag must always be preservable — never force a 
  classification where confidence < 0.75

---

## Suggested Experiments

1. Test whether NSX listening port signals alone are sufficient for 
   80%+ workload classification without any guest access

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
