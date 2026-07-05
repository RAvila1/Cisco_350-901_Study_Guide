# Cisco 350-901 (DEVCOR / AUTOCOR) Study Guide
### Designing, Deploying and Managing Network Automation Systems v2.0

**How to use this guide**

1. **Feynman pass first.** For every subtopic, read the "Explain it simply" block, then close the file and explain that concept out loud to an imaginary colleague with no automation background. If you stumble, that's your signal to re-read.
2. **Spaced repetition.** Turn each Q&A into a flashcard (Anki/Quizlet). Tag by domain (1.1, 2.3, etc.) so you can filter weak areas. Review missed cards same-day, then 1-day, 3-day, 7-day, 14-day intervals.
3. **Distractor awareness.** Cisco loves near-miss answers — a command that's *almost* right, a module that does something *adjacent*, a status code that's close but wrong. Each MCQ below includes distractors with a note on *why* they're wrong — study the "why," not just the correct letter.
4. **Ordering questions.** These mirror Cisco's "put these in order" format (e.g., OSI layers). Practice reconstructing sequences from memory, not recognition.

---

# DOMAIN 1.0 — Network Automation (30%)

## 1.1 Ansible for Network Configuration

**Explain it simply:** Ansible is a tool that reads a "recipe" (a YAML playbook) describing the state you want a device to be in — like "this interface should be a trunk" — and pushes the commands to get there. It doesn't need an agent installed on the device; it connects over SSH (for network gear) or via APIs, using **modules** as the building blocks and an **inventory** file to know which devices to touch.

**Key facts to lock in:**
- Ansible is **agentless**, **idempotent**, uses **YAML** playbooks.
- Network modules are typically **network_cli** (SSH) or **httpapi** (REST) connection types, set in `ansible.cfg` or inventory vars (`ansible_connection`, `ansible_network_os`).
- `ios_config`, `ios_facts`, `ios_vlans`, `ios_interfaces`, `ios_ospfv2`, `ios_acls` are **resource modules** — they manage a specific config resource declaratively (you describe desired state, Ansible calculates the diff).
- **Jinja2** templates (`.j2`) generate device-specific config from variables; combined with `template` module or resource module `config` state.
- `group_vars` / `host_vars` hold variable data (e.g., VLAN IDs per site).

**Q1 (MCQ):** Which Ansible module would you use to declaratively manage VLAN state on an IOS device without hand-crafting CLI commands?
A) `ios_command`
B) `ios_config`
C) `ios_vlans`
D) `ios_facts`

**Answer: C.** `ios_vlans` is a resource module — you declare the desired VLAN list/state and Ansible computes and applies the diff. `ios_config` (B) is a distractor because it *can* push VLAN CLI lines, but it's imperative (raw lines), not idempotent-by-resource — Cisco exam favors the resource-module answer when "declaratively manage" is the phrasing. `ios_command` (A) only runs exec-mode show commands, no config changes. `ios_facts` (D) only gathers state, doesn't change anything.

**Q2 (True/False):** Ansible requires an agent to be installed on the target network device to push configuration.
**Answer: False.** Ansible is agentless — it connects via SSH/NETCONF/RESTCONF using connection plugins; no persistent software runs on the device.

**Q3 (Fill in the blank):** In an Ansible inventory, the variable `_______` tells Ansible which OS-specific module family to use (e.g., `ios`, `nxos`, `eos`).
**Answer:** `ansible_network_os`

**Q4 (MCQ):** A playbook task fails with "connection type ssh is not valid for this module." What is the most likely cause?
A) The module requires `ansible_connection: network_cli` or `httpapi`, but the inventory has it set to `ssh`
B) The device is unreachable
C) The YAML has a syntax error
D) The wrong Python version is installed

**Answer: A.** Network resource modules require `network_cli` (or `httpapi`/`netconf`) connection plugins, not the generic `ssh` (used for Linux/`command`/`shell` modules). This is a classic exam trap — "ssh" *sounds* right since network devices use SSH, but Ansible's *connection plugin name* must match the module family.

**Q5 (Ordering):** Order the typical Ansible playbook execution flow:
`[ ] Task executes module against target host` `[ ] Ansible parses inventory & playbook` `[ ] Facts gathered (if enabled)` `[ ] Result returned & idempotency check compared against reported device state`
**Answer:** Parse inventory & playbook → Gather facts → Task executes module → Result returned/idempotency comparison.

---

## 1.2 Terraform for Network Configuration

**Explain it simply:** Terraform is also declarative infrastructure-as-code, but its core idea is a **state file** — a record of what it believes exists. You write `.tf` files describing desired resources (VLANs, interfaces), Terraform diffs your desired state against its state file, and produces an **execution plan** before applying anything.

**Key facts:**
- Written in **HCL** (HashiCorp Configuration Language).
- Workflow: `terraform init` → `terraform plan` → `terraform apply` → `terraform destroy`.
- Uses **providers** (e.g., Cisco's `ciscodevnet/iosxe`, `CiscoDevNet/aci` providers) to translate HCL resources into API calls.
- **State file** (`terraform.tfstate`) tracks real-world resource mapping; can be stored remotely (e.g., Terraform Cloud, S3 backend) for team locking.
- **Drift** = when real device config no longer matches the state file (someone made a manual change) — `terraform plan` will detect and show it.
- `terraform import` brings an existing, unmanaged resource under Terraform management.

**Q1 (MCQ):** A network engineer manually changes an interface description outside of Terraform. What will `terraform plan` show on the next run?
A) Nothing, Terraform only tracks resources it created
B) A drift — the plan shows a diff between the state file and the real device, and proposes reverting to the declared config
C) An error requiring `terraform destroy`
D) Terraform automatically updates its state file to match, silently

**Answer: B.** Terraform refreshes state against real infrastructure before planning; if the live value diverges from the `.tf` declaration, it flags drift and proposes reconciling toward the declared (desired) state. D is the trap — Terraform *does* refresh state data, but it does not silently accept the drift as the new desired state; it still proposes a corrective plan unless you change the `.tf` file itself.

**Q2 (Fill in the blank):** The Terraform command that shows *what would change* without applying it is `_______`.
**Answer:** `terraform plan`

**Q3 (True/False):** Terraform state files can safely be edited by hand as the normal way to fix state drift.
**Answer: False.** Manual state edits are risky and discouraged; `terraform import`, `terraform state rm`, or `terraform refresh`/`plan`-driven reconciliation are the supported paths.

**Q4 (Ordering):** Order the standard Terraform workflow:
`terraform apply` / `terraform init` / `terraform plan` / write `.tf` config
**Answer:** Write `.tf` config → `terraform init` → `terraform plan` → `terraform apply`.

**Q5 (MCQ):** Which file/mechanism allows a team to collaborate on the same Terraform-managed infrastructure without stepping on each other's applies?
A) `.gitignore`
B) Remote backend with state locking
C) `terraform fmt`
D) Local `.tfvars` file

**Answer: B.** A remote backend (e.g., Terraform Cloud, S3+DynamoDB) stores shared state and locks it during an apply so two engineers can't corrupt it simultaneously. Distractor D (`.tfvars`) just supplies variable values, unrelated to collaboration/state safety.

---

## 1.3 RESTCONF + YANG

**Explain it simply:** YANG is a *modeling language* that describes the structure of configuration/state data (like a schema). RESTCONF (RFC 8040) is a protocol that exposes that YANG-modeled data over HTTP, using standard verbs (GET/POST/PUT/PATCH/DELETE) and JSON or XML payloads — so you can manage config the same way you'd call any REST API, but the data shape is strictly defined by YANG.

**Key facts:**
- RESTCONF root resource is conventionally `/restconf/data/`.
- Content types: `application/yang-data+json` or `+xml`.
- HTTP methods map to NETCONF-like operations: `GET` (read), `POST` (create, non-idempotent — fails if resource exists), `PUT` (create or fully replace, idempotent), `PATCH` (merge partial update), `DELETE` (remove).
- YANG data is organized into **containers**, **lists** (keyed, can have multiple entries), **leaves** (single values), **leaf-lists**.
- Native/OpenConfig YANG models both exist; OpenConfig is vendor-neutral, native is Cisco-specific (e.g., `Cisco-IOS-XE-native`).
- Discover supported capabilities via `/restconf/data/ietf-yang-library:yang-library` or `.well-known/host-meta`.

**Q1 (MCQ):** You want to add a single new ACL entry to an existing list without disturbing other entries. Which HTTP method should you use in RESTCONF?
A) PUT
B) POST
C) PATCH
D) DELETE

**Answer: C.** PATCH performs a merge — it adds/updates the given fields while leaving the rest of the list untouched. PUT (A) is a trap: PUT *replaces* the entire targeted resource, so if you PUT just one ACE it could wipe siblings unless you PUT at the correct list-item URI. POST (B) creates a new child resource but fails with 409 Conflict if the exact resource already exists (not ideal for "add one entry to an existing list" phrasing when target already exists in some form).

**Q2 (Fill in the blank):** The RESTCONF root container for configuration and operational data is `/restconf/_______/`.
**Answer:** `data`

**Q3 (True/False):** RESTCONF and NETCONF can be used against the same underlying YANG datastore concurrently.
**Answer: True.** Both protocols operate on the same YANG-modeled datastores; the device reconciles access, though best practice is to avoid mixing simultaneous conflicting edits.

**Q4 (MCQ):** A RESTCONF POST to create a new VLAN returns HTTP 409. What does this indicate?
A) Authentication failure
B) The resource already exists (conflict)
C) Malformed JSON payload
D) Method not allowed

**Answer: B.** 409 Conflict = resource already exists. A (auth) would be 401/403. C (bad JSON) would typically be 400. D (method not allowed) is 405.

**Q5 (Ordering):** Order the steps to configure a device feature via RESTCONF using a YANG model:
`[ ] Send PATCH/PUT with JSON payload matching model` `[ ] Identify correct YANG module & container path` `[ ] Authenticate to RESTCONF endpoint` `[ ] Verify with GET`
**Answer:** Identify correct YANG module & container path → Authenticate to RESTCONF endpoint → Send PATCH/PUT with JSON payload → Verify with GET.

---

## 1.4 Python for Network Configuration

**Explain it simply:** Python gives you full programmatic control — instead of a fixed playbook syntax, you write logic: loops, conditionals, error handling. For network automation you typically pair Python with libraries like **Netmiko** (SSH CLI automation), **NAPALM** (multi-vendor abstraction layer with `get_facts`, `load_merge_candidate`, `commit_config`), **ncclient** (NETCONF), or `requests`/`requests_toolbelt` for REST/RESTCONF.

**Key facts:**
- **Netmiko**: wraps Paramiko for CLI automation across vendors; `ConnectHandler()`, `send_config_set()`, `send_command()`.
- **NAPALM**: vendor-abstraction; supports `get_facts()`, `get_interfaces()`, `load_merge_candidate()`, `compare_config()`, `commit_config()`, `discard_config()` — this candidate/commit model matters for exam questions about safe config changes.
- **ncclient**: Python NETCONF client; `manager.connect()`, `get_config()`, `edit_config()`, `commit()`.
- Exception handling: wrap device calls in `try/except` (e.g., `NetmikoTimeoutException`, `NetmikoAuthenticationException`) for resiliency.
- Context managers (`with ConnectHandler(...) as conn:`) ensure sessions close properly.

**Q1 (MCQ):** Which NAPALM method lets you stage a configuration change and review the diff before committing it?
A) `load_replace_candidate()` + `commit_config()` only
B) `load_merge_candidate()` then `compare_config()` then `commit_config()`
C) `get_config()` then `push_config()`
D) `open()` then `commit_config()` directly

**Answer: B.** This is the canonical NAPALM safe-change workflow: stage a merge candidate, diff it with `compare_config()`, and only then commit. A is a distractor because `load_replace_candidate` is valid but the answer omits the crucial `compare_config()` review step implied by "review the diff." C/D skip staging entirely.

**Q2 (Fill in the blank):** In Netmiko, the exception raised when a device doesn't respond within the timeout window is `_______`.
**Answer:** `NetmikoTimeoutException`

**Q3 (True/False):** NAPALM provides a vendor-neutral abstraction, so the same Python code can retrieve facts from a Cisco IOS-XE and an Arista EOS device with minimal changes.
**Answer: True** — that's NAPALM's core design goal (via driver plugins per vendor).

**Q4 (MCQ):** You need to push a raw multi-line CLI configuration block to a Cisco IOS device over SSH from Python. Which method is most appropriate?
A) `napalm.load_template()`
B) `netmiko.send_config_set(config_commands)`
C) `ncclient.edit_config()`
D) `requests.post()`

**Answer: B.** Netmiko's `send_config_set()` is built exactly for pushing a list of CLI config-mode commands over SSH. `ncclient.edit_config()` (C) is the NETCONF-based equivalent — a distractor because it's also valid config-pushing but requires NETCONF/XML, not raw CLI. `requests.post()` (D) implies REST, not applicable to raw CLI over SSH.

**Q5 (Ordering):** Order a typical Python/NAPALM safe-change workflow:
`[ ] commit_config()` `[ ] open() connection` `[ ] load_merge_candidate(config)` `[ ] compare_config()` `[ ] close() connection`
**Answer:** `open()` → `load_merge_candidate(config)` → `compare_config()` → `commit_config()` → `close()`.

---

## 1.5 Selecting the Right Automation Approach

**Explain it simply:** Not every problem needs custom Python. Sometimes a low-code/no-code tool (like Cisco Catalyst Center workflows, or a simple Ansible playbook) gets the job done faster and is easier for the team to maintain. The exam wants you to reason about trade-offs: complexity, skill of the team, need for custom logic/integration, speed of delivery, and long-term maintainability.

**Key decision factors:**
- **IaC framework (Ansible/Terraform)**: best for repeatable, declarative, version-controlled infra changes across many devices; good team collaboration via Git.
- **Low-code/no-code (e.g., Catalyst Center automation templates, Meraki dashboard, Webex Workflows)**: best for simple, well-defined tasks, faster time-to-value, less engineering overhead, but less flexible for edge cases/custom logic.
- **Custom application (Python + APIs)**: best when you need custom business logic, integration with multiple systems (ITSM, source of truth, AI), or a UI/self-service portal.

**Q1 (MCQ):** A business wants a simple, one-time provisioning task done by non-engineers via a form-based interface with no custom logic. Which is the best-fit approach?
A) Custom Python application
B) Terraform
C) Low-code/no-code platform
D) Raw RESTCONF scripting

**Answer: C.** Simple, form-driven, non-engineer usage is exactly what low-code/no-code targets. Custom Python (A) and raw RESTCONF (D) add unnecessary engineering overhead for a "no custom logic" ask. Terraform (B) is a distractor — powerful for IaC, but overkill/wrong fit for non-engineers doing a one-off form task.

**Q2 (True/False):** Low-code/no-code solutions are always preferable because they reduce technical debt.
**Answer: False.** They reduce *initial* engineering effort but can accumulate their own technical debt/lock-in and lack flexibility for complex or evolving requirements — a nuance the exam likes to test.

**Q3 (Fill in the blank):** When a requirement involves integrating multiple systems (source of truth, ticketing, AI agent) with custom decision logic, the most appropriate approach is typically a `_______`.
**Answer:** custom application

---

## 1.6 Consuming REST APIs (Advanced)

**Explain it simply:** Real-world APIs aren't just "GET and done." You must handle **pagination** (getting data in pages via `offset`/`limit` or a `next` link), **complex auth** (OAuth2 token refresh flows, not just a static API key), **rate limiting** (backing off when you get HTTP 429), robust **error handling** (retry logic, distinguishing 4xx client errors from 5xx server errors), and **persistent authentication** (reusing/refreshing tokens instead of re-authenticating every call).

**Key facts:**
- **Pagination** patterns: offset/limit, cursor-based (`next` link in response), page-number based.
- **Rate limiting**: HTTP `429 Too Many Requests`; response often includes `Retry-After` header — implement exponential backoff.
- **OAuth2** flows: client credentials (machine-to-machine, common for network APIs like Meraki/DNAC), authorization code (user-involved), token refresh via refresh_token before access_token expiry.
- **Persistent authentication**: cache/reuse bearer tokens; refresh proactively before expiry rather than on failure only.
- **Idempotency**: design retries so repeated calls (e.g., after a timeout) don't create duplicate resources — use idempotency keys where supported.

**Q1 (MCQ):** An API call returns HTTP 429. What is the best automated response?
A) Immediately retry the same request in a tight loop
B) Fail permanently and alert
C) Read the `Retry-After` header (or apply exponential backoff) then retry
D) Switch to a different API endpoint

**Answer: C.** 429 explicitly signals rate limiting; the correct behavior is backing off (using `Retry-After` if provided, or exponential backoff) then retrying. A is the classic wrong answer — hammering the API worsens the problem and may get you blocked longer.

**Q2 (Fill in the blank):** The OAuth2 grant type most commonly used for server-to-server network automation (no interactive user login) is `_______`.
**Answer:** client credentials (grant)

**Q3 (True/False):** It's best practice to authenticate fresh for every single API call to guarantee token validity.
**Answer: False.** Best practice is persistent/cached authentication — store and reuse the token, refreshing it proactively before expiry, to reduce overhead and avoid unnecessary auth-server load.

**Q4 (MCQ):** You're pulling 10,000 device records from an inventory API that returns 100 records per page with a `next` field in the JSON response. What's the correct pattern?
A) Call the endpoint once with `limit=10000`
B) Loop, following the `next` link/cursor until it's null/absent, accumulating results
C) Make 100 simultaneous parallel calls to page 1–100
D) Only request the first page since exam scope caps at 100

**Answer: B.** This is the standard cursor/pagination-following loop. A ignores the API's stated page size constraint. C risks hitting rate limits and is not the documented pattern for cursor-based pagination.

**Q5 (Ordering):** Order the steps in a robust REST API consumption workflow:
`[ ] Handle pagination to collect all data` `[ ] Authenticate (obtain token)` `[ ] Handle errors/retries with backoff` `[ ] Make the API call` `[ ] Cache/reuse token for subsequent calls`
**Answer:** Authenticate (obtain token) → Make the API call → Handle errors/retries with backoff → Handle pagination to collect all data → Cache/reuse token for subsequent calls.

---

# DOMAIN 2.0 — Infrastructure as Code (30%)

## 2.1 Git Version Control Operations

**Explain it simply:** Git tracks every change to your code as a chain of commits. Different commands let you move/copy/undo changes in different ways: `merge` combines branch histories, `cherry-pick` copies one specific commit onto another branch, `reset` rewrites history/moves the branch pointer, `checkout` switches branches or restores files, and `revert` undoes a commit by creating a new "opposite" commit (safe for shared history).

**Key facts:**
- **Merge**: combines two branches; can create a merge commit or fast-forward. **Squash merge** condenses all commits from a feature branch into one commit on the target branch (clean history). Conflicts occur when the same lines were changed differently — resolved by editing conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) and committing.
- **git cherry-pick `<commit>`**: applies the changes from one specific commit onto the current branch, creating a new commit with a new hash. Useful for backporting a hotfix without merging the whole branch.
- **git reset**: moves the branch pointer to a prior commit. `--soft` (keeps changes staged), `--mixed` (default, unstages but keeps working directory changes), `--hard` (discards changes entirely). **Rewrites history** — dangerous on shared/pushed branches.
- **git checkout**: switches branches (`git checkout <branch>`) or restores specific files from a commit (`git checkout <commit> -- file`). Doesn't rewrite history.
- **git revert `<commit>`**: creates a **new commit** that undoes the changes of a target commit — safe for public/shared branches since it doesn't rewrite history (unlike reset).

**Q1 (MCQ):** You need to undo a bad commit that has already been pushed and pulled by teammates, without rewriting shared history. What do you use?
A) `git reset --hard`
B) `git revert`
C) `git checkout`
D) `git cherry-pick`

**Answer: B.** `revert` creates a new commit undoing the change — safe for shared history. `reset --hard` (A) rewrites history and is dangerous once pushed/pulled by others — classic exam trap since both "undo" something, but only revert is safe for shared branches.

**Q2 (Fill in the blank):** The command to apply a single specific commit from another branch onto your current branch is `git _______`.
**Answer:** `cherry-pick`

**Q3 (True/False):** A squash merge preserves every individual commit from the feature branch in the target branch's history.
**Answer: False.** Squash merge condenses all commits into a single commit on the target branch — individual commit granularity from the feature branch is lost (though still recoverable in the original branch/reflog if not deleted).

**Q4 (MCQ):** During a merge, Git reports a conflict in `vlans.yml`. What is the correct next step?
A) Run `git merge --abort` and give up
B) Manually edit the file to resolve the `<<<<<<<`/`=======`/`>>>>>>>` markers, then `git add` and commit
C) Delete the file and recreate it from scratch
D) Run `git reset --hard origin/main`

**Answer: B.** Standard conflict resolution workflow. D is a trap since it would discard your work entirely rather than resolve the conflict.

**Q5 (MCQ):** Which command moves the current branch pointer to an earlier commit and unstages changes but **keeps them in your working directory** for review?
A) `git reset --hard HEAD~1`
B) `git reset --soft HEAD~1`
C) `git reset --mixed HEAD~1` (or bare `git reset HEAD~1`)
D) `git checkout HEAD~1`

**Answer: C.** `--mixed` is the default reset mode: moves HEAD/branch pointer, unstages, but leaves working directory files intact. `--soft` (B) keeps changes staged (not unstaged) — a common distractor since both B and C "keep changes," but only mixed unstages them.

**Q6 (Ordering):** Order the resolution of a merge conflict:
`[ ] git add <resolved files>` `[ ] git commit` `[ ] git merge <branch>` `[ ] Edit files to remove conflict markers`
**Answer:** `git merge <branch>` → Edit files to remove conflict markers → `git add <resolved files>` → `git commit`.

---

## 2.2 Diagnosing GitLab CE CI/CD Pipeline Failures

**Explain it simply:** A CI/CD pipeline is a series of automated stages (defined in `.gitlab-ci.yml`) that build, test, and deploy your code every time you push. When it fails, you read the **job log** to find *where* it broke: missing dependency (a package/library not installed in the runner image), version incompatibility (e.g., Python 3.9 code run on a 3.11 runner missing a removed feature, or a pinned library version conflicting with another), or failing tests (actual logic/config errors caught by pyATS/pytest).

**Key facts:**
- `.gitlab-ci.yml` defines `stages:`, `jobs`, and each job specifies an `image:`, `script:`, `before_script:`.
- **Missing dependency** symptoms: `ModuleNotFoundError`, `command not found` in job log → fix via `requirements.txt`/`before_script: pip install` or better base image.
- **Incompatible versions**: symptoms like `SyntaxError` from newer syntax on an old interpreter, or a library requiring a newer API than what's pinned → fix by pinning compatible versions in `requirements.txt` or upgrading the runner image.
- **Failed tests**: the test stage job exits non-zero; check test framework output (pyATS/pytest assertion failures) to find the specific configuration/logic issue.
- GitLab Runners execute jobs; a **runner not registered/tagged** properly for a job also causes a stuck ("pending") pipeline — a different failure mode than a script error.
- `artifacts:` pass files between stages; if a later stage can't find a file, check that the artifact was declared/retained from the prior stage.

**Q1 (MCQ):** A pipeline job fails with `ModuleNotFoundError: No module named 'napalm'`. What is the most likely root cause?
A) A YANG model mismatch
B) The runner's environment doesn't have `napalm` installed and it's missing from `requirements.txt`/install step
C) The GitLab license expired
D) An expired TLS certificate

**Answer: B.** Straightforward missing-dependency signature — matches the exam's stated failure category exactly.

**Q2 (True/False):** A pipeline stuck in "pending" indefinitely is typically a script/test failure, not a runner availability issue.
**Answer: False.** A pipeline stuck pending usually means no available/matching runner (wrong tags, no runners registered) — this is a distinct failure mode from a script error (which shows as "failed," not "pending").

**Q3 (Fill in the blank):** Files (like build outputs or test reports) passed between pipeline stages are declared using the `_______` keyword in `.gitlab-ci.yml`.
**Answer:** `artifacts`

**Q4 (MCQ):** The test stage fails with a pyATS assertion mismatch showing the deployed VLAN differs from the expected value in the testbed. This is best categorized as:
A) Missing dependency
B) Incompatible component versions
C) A failed test (logic/config discrepancy)
D) Runner misconfiguration

**Answer: C.** The pipeline and dependencies ran fine; the actual configuration state didn't match expectations — a genuine test failure, not an infrastructure/dependency issue.

---

## 2.3 Constructing a GitLab CE CI/CD Pipeline

**Explain it simply:** A good network-automation pipeline mirrors safe change management: **build** (assemble/package code, install deps, maybe render templates), **prevalidation** (check things are safe/sane *before* touching production — e.g., syntax check, pyATS pre-checks, linting), **deploy** (actually push the config via Ansible/Terraform/Python), **post-validation** (verify the change worked — pyATS post-checks, comparing intended vs actual state).

**Key facts:**
- Stage order in `.gitlab-ci.yml`: `stages: [build, prevalidation, deploy, postvalidation]` — jobs execute in this order by default, with jobs inside a stage running in parallel unless dependencies (`needs:`) are set.
- **Prevalidation** should be able to **halt the pipeline** (via job failure / `allow_failure: false`) before deploy occurs — this is critical: prevalidation is a *gate*.
- **Post-validation failure** should ideally trigger **rollback** (either automatically via a subsequent job, or by alerting for manual rollback) — exam tests whether you understand post-validation isn't just "log and move on."
- pyATS/Genie is commonly used in both pre- and post-validation stages to snapshot and compare device state.

**Q1 (MCQ):** In a well-designed pipeline, if the **prevalidation** stage fails, what should happen?
A) The deploy stage still runs, but with a warning
B) The pipeline halts before deploy runs
C) The pipeline skips straight to post-validation
D) The build stage re-runs automatically

**Answer: B.** Prevalidation exists specifically as a gate to prevent unsafe deploys — a failed prevalidation job (without `allow_failure: true`) stops the pipeline.

**Q2 (Fill in the blank):** The stage that verifies the deployed configuration actually matches the intended state after deployment is called `_______`.
**Answer:** post-validation (post-deployment validation)

**Q3 (Ordering):** Order the four pipeline stages for a network automation deployment:
`[ ] deploy` `[ ] build` `[ ] post-validation` `[ ] prevalidation`
**Answer:** build → prevalidation → deploy → post-validation.

**Q4 (True/False):** It's acceptable for the deploy stage to run even if prevalidation checks (e.g., pyATS pre-checks confirming device reachability and current state) fail, as long as post-validation will catch any issues afterward.
**Answer: False.** This defeats the purpose of prevalidation as a gate; catching problems *after* a bad push to production defeats safe change management — the whole point of prevalidation is to prevent the bad push in the first place.

---

## 2.4 Cisco Modeling Labs (CML) for Network Simulation

**Explain it simply:** CML lets you build a virtual topology of routers/switches (using real Cisco OS images) to safely test your automation scripts/playbooks before touching production. You define the topology (nodes, links) — often via a YAML topology file — start it, and point your automation at the simulated devices' management IPs, exactly as you would real hardware.

**Key facts:**
- CML topologies can be defined/exported as **YAML** files, allowing them to be version-controlled and spun up as part of a CI/CD pipeline (e.g., a "deploy test topology" job before prevalidation).
- CML provides a REST API / Python client (`virl2_client`) to programmatically start/stop labs, useful for automated testing pipelines.
- Supports node types running actual Cisco images (IOSv, IOS-XRv, CSR1000v/Cat8000v, NX-OSv) plus external connectors.
- Common exam angle: using CML in a CI/CD **prevalidation/testing stage** to spin up a disposable topology, run the automation against it, tear it down — providing a safe test environment without touching production.

**Q1 (MCQ):** Why would a team integrate CML into their CI/CD pipeline rather than testing automation directly against production?
A) CML is required for GitLab to function
B) CML provides an isolated, disposable virtual topology to validate automation safely before it touches real devices
C) CML replaces the need for pyATS
D) CML only works with Terraform, not Ansible

**Answer: B.** This is the core purpose — safe, repeatable pre-production testing. C and D are false — CML complements pyATS (testing framework) and works with any automation tool that can reach the simulated devices' management interfaces.

**Q2 (True/False):** CML topologies can be defined in YAML and version-controlled alongside your automation code.
**Answer: True.**

---

## 2.5 Interpreting Docker Compose Files

**Explain it simply:** Docker Compose lets you define a multi-container application in one YAML file: which container images to run (**services**), how they talk to each other (**networks**), what data persists across restarts (**volumes**), and — in older syntax — explicit connections between services (**links**, largely superseded by user-defined networks but still tested).

**Key facts:**
- `services:` — each key is a container definition (`image:`, `build:`, `ports:`, `environment:`, `depends_on:`).
- `networks:` — user-defined networks let services reach each other by service name (DNS-based discovery); default bridge network is more limited.
- `volumes:` — named volumes or bind mounts persist data outside the container lifecycle (e.g., a database's data directory, or a source-of-truth file).
- `links:` (legacy) — explicitly declares one-way visibility between containers; mostly replaced by putting services on the same user-defined network, but can appear in exam questions about legacy Compose files.
- `depends_on:` controls **start order**, not readiness — a container can start before its dependency is actually ready to serve traffic (a common gotcha/distractor).

**Q1 (MCQ):** In a Docker Compose file, what does `depends_on` guarantee?
A) The dependent service's application is fully ready to accept connections before the current service starts
B) Only that the dependent container has started (not necessarily "ready")
C) Network isolation between the two services
D) Persistent storage between the two services

**Answer: B.** Classic exam distractor — `depends_on` controls **container start order**, not application readiness; you'd need healthchecks or retry logic in the app for true readiness gating.

**Q2 (Fill in the blank):** To allow two Compose services to persist and share data beyond the container's lifecycle, you define a `_______`.
**Answer:** volume

**Q3 (True/False):** Services on the same user-defined Docker network can reach each other using the service name as a hostname.
**Answer: True.** Docker's embedded DNS resolves service names to container IPs on user-defined networks.

---

## 2.6 Integrating a Source of Truth

**Explain it simply:** A **source of truth (SoT)** is the single authoritative place describing what your network *should* look like — device inventory, IPs, VLANs, roles (e.g., **NetBox**, or a simpler YAML/CSV/CMDB). Your automation reads from the SoT to know what to configure, rather than hardcoding values in scripts — so the SoT drives automation, and ideally automation also updates the SoT (or validates it) to keep it accurate.

**Key facts:**
- Common SoT tools: **NetBox** (IPAM/DCIM), Git-based YAML files, ServiceNow CMDB.
- Automation pulls data from SoT via API (e.g., NetBox REST/GraphQL API) to dynamically build Ansible inventory or Python config data — this is called **dynamic inventory** in Ansible.
- Best practice: SoT is the single point of truth — avoid configuration drift by ensuring changes always go: SoT update → automation run → device config, not the reverse.
- Risk exam tests: what happens if SoT and live network diverge (drift) — automation should detect/report, not blindly overwrite without validation, and processes should keep SoT and live state reconciled.

**Q1 (MCQ):** In a source-of-truth-driven pipeline, what is the correct order of change?
A) Change the device directly, then update the SoT to match
B) Update the SoT first, then run automation to push the change to devices
C) Devices and SoT should be updated independently and reconciled manually each quarter
D) The SoT is only for documentation and shouldn't drive automation

**Answer: B.** This is the fundamental principle — SoT-driven automation means the SoT is authoritative and changes flow from it outward to devices, preventing drift.

**Q2 (Fill in the blank):** Using NetBox's API to dynamically generate an Ansible inventory (instead of a static `hosts` file) is called `_______ inventory`.
**Answer:** dynamic

---

## 2.7 YAML/JSON Representation from a YANG Model

**Explain it simply:** Given a YANG model (the schema), you need to be able to write the actual data instance — in JSON or YAML — that conforms to it: correct container/list nesting, correct key naming (including module prefixes when needed), correct data types.

**Key facts:**
- YANG **containers** map to JSON objects; YANG **lists** map to JSON arrays of objects (each needs its key leaf present); YANG **leaf** maps to a single JSON key-value.
- Module namespace prefixing: top-level elements are often qualified as `module-name:element` (e.g., `"ietf-interfaces:interfaces"`) especially in RESTCONF payloads.
- Data types matter: YANG `uint8`, `string`, `enumeration`, `boolean` (`true`/`false`, lowercase in JSON) — a common trap is capitalizing booleans (`True`) which is valid Python but invalid JSON.

**Q1 (MCQ):** Given a YANG list `interface` keyed by `name`, which JSON structure correctly represents two interface entries?
A) `{"interface": {"name": "Gi0/1"}, {"name": "Gi0/2"}}`
B) `{"interface": [{"name": "Gi0/1"}, {"name": "Gi0/2"}]}`
C) `{"interface": ["Gi0/1", "Gi0/2"]}`
D) `{"interfaces": {"Gi0/1": {}, "Gi0/2": {}}}`

**Answer: B.** YANG lists map to JSON arrays of objects, each containing its key leaf and other leaves — A is invalid JSON syntax (a classic "looks plausible" trap), C loses the object structure, D isn't how YANG lists serialize.

**Q2 (True/False):** In JSON representations of YANG boolean leaves, the value must be lowercase `true`/`false`, not `True`/`False`.
**Answer: True.** JSON spec requires lowercase booleans; Python's `True`/`False` would break strict JSON parsing if not serialized properly (e.g., forgetting to use `json.dumps()`).

---

# DOMAIN 3.0 — Operations (20%)

## 3.1 Model-Driven Telemetry (MDT) Architecture

**Explain it simply:** Instead of you *polling* a device every few minutes for stats (SNMP-style), model-driven telemetry has the device **push** structured, YANG-modeled data to a collector continuously/on-change — much more scalable and near-real-time.

**Key facts:**
- Two subscription types: **dial-in** (collector initiates a session, subscribes to a sensor path, e.g., via gRPC/NETCONF) and **dial-out** (device initiates the connection to the collector — useful when collector is behind NAT/firewall or device can't be reached inbound).
- Transport: **gRPC** (with **GPB** — Google Protocol Buffers — or JSON encoding) is the most common modern transport; NETCONF notifications also possible.
- Data is defined by **YANG models** — the same models used for config can often be subscribed to for streaming telemetry (operational state).
- Architecture pieces: **Publisher (device)** → **Telemetry Receiver/Collector** (e.g., Telegraf, Pipeline) → **Time-series database** (e.g., InfluxDB, Prometheus) → **Visualization** (e.g., Grafana).
- **Cadence-based** (periodic, e.g., every 30s) vs **event/on-change** subscriptions — on-change only sends updates when the value actually changes, reducing overhead.

**Q1 (MCQ):** A collector cannot initiate a connection to a device because the device sits behind a NAT boundary with no inbound path. Which telemetry mode should be used?
A) Dial-in
B) Dial-out
C) SNMP polling
D) Syslog

**Answer: B.** Dial-out means the device initiates the outbound session to the collector, solving the NAT/inbound-reachability problem. Dial-in (A) requires the collector to reach the device — the opposite of what's needed here.

**Q2 (Fill in the blank):** The transport most commonly used for modern model-driven telemetry, often encoded with Google Protocol Buffers, is `_______`.
**Answer:** gRPC

**Q3 (True/False):** On-change telemetry subscriptions send updates on a fixed timer regardless of whether the value changed.
**Answer: False.** That describes cadence/periodic subscriptions. On-change sends updates only when the subscribed value actually changes.

**Q4 (Ordering):** Order the model-driven telemetry data path:
`[ ] Time-series database stores data` `[ ] Device streams YANG-modeled data via gRPC` `[ ] Dashboard visualizes data` `[ ] Collector receives and parses telemetry`
**Answer:** Device streams YANG-modeled data via gRPC → Collector receives and parses telemetry → Time-series database stores data → Dashboard visualizes data.

---

## 3.2 Logging Strategy (Syslog / Webhooks)

**Explain it simply:** Your automation needs to tell humans (and systems) what happened — successes, failures, changes made. **Syslog** is the traditional standard (severity levels 0–7, sent to a central syslog server). **Webhooks** are HTTP callbacks — your automation POSTs a JSON payload to a URL (e.g., a Webex space, a Slack channel, or an ITSM system) when an event occurs.

**Key facts:**
- Syslog severity levels (0=Emergency ... 7=Debug) — memorize: 0 Emergency, 1 Alert, 2 Critical, 3 Error, 4 Warning, 5 Notice, 6 Informational, 7 Debug.
- Structured logging (JSON logs) is preferred over free-text for automation, since it's machine-parseable for correlation/search (e.g., ELK stack).
- Webhooks are push-based, event-driven, HTTP POST — good for near-real-time human notification (e.g., posting to a **Webex Messaging** space when a pipeline deploy completes or fails).
- Logging strategy should include: what to log (change made, who/what triggered it, success/fail, timestamp), where (syslog server, webhook target), and retention.
- Sensitive data (passwords, tokens) must **never** be logged — ties into 3.6 secure coding.

**Q1 (MCQ):** Which syslog severity level indicates a condition that renders the system unusable?
A) 0 – Emergency
B) 3 – Error
C) 5 – Notice
D) 7 – Debug

**Answer: A.** Emergency (0) is the most severe — system unusable. Error (3) indicates an error condition but not necessarily system-wide unusability — a common confusion point on the exam.

**Q2 (Fill in the blank):** Sending an automated pipeline failure notification as an HTTP POST with a JSON payload to a Webex space is an example of a `_______`.
**Answer:** webhook

**Q3 (True/False):** It is acceptable to include the API token used for authentication in a log message for troubleshooting purposes, as long as log access is restricted.
**Answer: False.** Secrets should never be logged, regardless of access restrictions — this is a secure coding/logging best practice tested explicitly under 3.6 as well.

**Q4 (Ordering, severity — highest severity/most critical first):** Order these syslog levels from most to least severe: `Warning` / `Emergency` / `Error` / `Critical`
**Answer:** Emergency → Critical → Error → Warning.

---

## 3.3 Diagnosing Problems from Logs/Output

**Explain it simply:** Given a chunk of log output or an error trace, you need to identify *what* went wrong and *why* — auth failure vs. timeout vs. config syntax error vs. API rate limit vs. a YANG validation error — each has distinctive signatures.

**Key facts (signature recognition):**
- **401/403 in API logs** → authentication/authorization failure (bad token, expired token, insufficient scope).
- **Timeout / "no response" in SSH logs** → device unreachable, wrong IP, ACL blocking, or device too slow to respond within configured timeout.
- **"Invalid input detected"** in device CLI output → syntax error in pushed configuration.
- **429** → rate limiting.
- **YANG validation error** (e.g., "value not in range", "instance-required") → payload didn't conform to the model constraints.
- **Idempotency check failures in Ansible (`changed: true` every run)** → suggests the module/task isn't correctly detecting existing state, or config is being reapplied unnecessarily (a config-drift or module misuse symptom).

**Q1 (MCQ):** An Ansible playbook run against the same unchanged device reports `changed: true` on every single run. What does this most likely indicate?
A) The device is down
B) A module/task is not idempotent as configured (e.g., using `ios_config` with lines that always register as a change) or there's config drift
C) The playbook has a YAML syntax error
D) The API rate limit was hit

**Answer: B.** This is a classic idempotency diagnostic question — repeated "changed" states point to non-idempotent task design or actual drift, not a syntax or connectivity issue.

**Q2 (MCQ):** A device CLI log shows `% Invalid input detected at '^' marker`. What's the most likely cause?
A) Rate limiting
B) A syntax error in the pushed configuration command
C) An expired TLS certificate
D) A YANG range violation

**Answer: B.** This is IOS's classic CLI syntax error message.

---

## 3.4 Change Validation with pyATS CLI Tools

**Explain it simply:** pyATS (with the Genie library) lets you snapshot device state (parsed structured data, not raw text) before and after a change, then diff them to confirm the change did what you expected and nothing else broke.

**Key facts:**
- `pyats` CLI tools: `pyats learn` (learns/parses feature state, e.g., `pyats learn ospf`), `genie diff` (compares two snapshots/learned states).
- Common workflow: **learn** device state (pre-change) → make the change → **learn** again (post-change) → **diff** the two structured outputs to validate only the intended delta occurred.
- Testbed file (YAML) defines devices/credentials for pyATS/Genie to connect to.
- `genie parse` — converts raw `show` command output into structured Python data (dict) using Genie parsers, useful even outside full pyATS jobs.
- Integrates naturally into CI/CD as pre/post-validation stage jobs (ties to 2.3).

**Q1 (MCQ):** Which pyATS/Genie capability best supports validating "only the intended configuration changed, nothing else drifted" between before/after a deployment?
A) `genie parse`
B) `pyats learn` snapshots compared with `genie diff`
C) Syslog severity filtering
D) Terraform state comparison

**Answer: B.** This is precisely the learn-then-diff change-validation pattern pyATS/Genie is built for.

**Q2 (Fill in the blank):** The pyATS/Genie command that converts raw CLI `show` command text into structured Python data is `genie _______`.
**Answer:** parse

**Q3 (True/False):** A pyATS testbed file defines the devices, connection details, and credentials pyATS will use to connect for learn/validation tasks.
**Answer: True.**

---

## 3.5 Obtaining and Deploying CA-Signed TLS Certificates

**Explain it simply:** For a device/app to present a certificate trusted by clients (not a self-signed warning), you: generate a private key, create a **CSR** (Certificate Signing Request) containing your public key + identity info, submit the CSR to a **Certificate Authority (CA)**, receive back a **signed certificate**, then install (bind) that cert + the CA's chain onto the device/service.

**Key facts:**
- Steps: Generate key pair → Generate CSR (includes CN/SAN, org info) → Submit CSR to CA (internal enterprise CA or public CA) → CA validates identity/domain ownership → CA issues signed cert → Install cert + intermediate/root chain on device/server → Configure service to use it (e.g., bind to HTTPS/RESTCONF listener) → Renew before expiry.
- **CSR** never contains the private key — only the public key and identity fields; private key stays local.
- Automated issuance protocols: **ACME** (e.g., Let's Encrypt) automates request/validation/renewal; enterprise CAs may use **SCEP** or **EST** for device enrollment.
- Renewal must happen **before expiry** — automation often monitors cert expiry dates and triggers renewal workflows (ties into ops/monitoring).

**Q1 (MCQ):** What does a Certificate Signing Request (CSR) contain?
A) The private key and public key
B) The public key and identity information (e.g., CN, SAN), but not the private key
C) Only the CA's root certificate
D) The signed certificate itself

**Answer: B.** The private key must never leave the requesting device — only the public key + identity info goes in the CSR.

**Q2 (Ordering):** Order the CA-signed certificate lifecycle:
`[ ] Install signed cert + chain on the device` `[ ] Generate private/public key pair` `[ ] CA validates and issues signed certificate` `[ ] Generate and submit CSR to CA`
**Answer:** Generate private/public key pair → Generate and submit CSR to CA → CA validates and issues signed certificate → Install signed cert + chain on the device.

**Q3 (Fill in the blank):** The protocol commonly used to automate certificate issuance and renewal (popularized by Let's Encrypt) is `_______`.
**Answer:** ACME

---

## 3.6 Secure Coding Practices

**Explain it simply:** Your automation code is itself an attack surface. You must validate all input (never trust user/API input blindly), handle authentication properly (don't hardcode credentials), and manage secrets safely (never commit them to Git, use a vault/secrets manager).

**Key facts:**
- **Input validation**: sanitize/validate all external input (API payloads, user CLI args, YAML variables) before using it to build commands or queries — prevents injection attacks and malformed config pushes.
- **Authentication**: use strong, non-default credentials; prefer token-based/OAuth2 over embedded static passwords; enforce least privilege (service accounts scoped to only what they need).
- **Secret management**: never hardcode secrets in source code or commit them to Git; use environment variables, `.gitignore`'d files, or dedicated secrets managers (HashiCorp Vault, Ansible Vault, CyberArk); rotate secrets periodically.
- **Ansible Vault** specifically encrypts sensitive variable files (`ansible-vault encrypt vars.yml`) so secrets can live in Git safely (encrypted at rest).
- Logging discipline (from 3.2) intersects here: never log secrets.

**Q1 (MCQ):** A playbook stores the device admin password in plaintext inside `group_vars/all.yml`, which is committed to a public Git repo. What is the correct remediation?
A) Add the file to `.gitignore` only going forward, leaving history untouched
B) Encrypt the sensitive variables using Ansible Vault (and rotate the exposed credential, and scrub Git history)
C) Base64-encode the password before committing
D) Move the password into a code comment for visibility

**Answer: B.** Ansible Vault is the correct built-in mechanism, and — critically — since it was already exposed, the credential must be considered compromised and rotated, plus history scrubbed if truly public. C is a trap: **Base64 is encoding, not encryption** — trivially reversible, a classic exam distractor.

**Q2 (True/False):** Base64 encoding a secret before storing it in a config file is an acceptable substitute for encryption.
**Answer: False.** Base64 is reversible encoding, not encryption — provides no real confidentiality protection.

**Q3 (Fill in the blank):** The Ansible feature used to encrypt sensitive variable files so they can be safely stored in version control is `_______`.
**Answer:** Ansible Vault

**Q4 (MCQ):** Which practice best demonstrates proper input validation in a network automation script accepting a VLAN ID from an API request?
A) Directly interpolate the value into the CLI command string
B) Validate the value is an integer within 1–4094 (and not a reserved ID) before using it in any command construction
C) Trust the value since it came from an internal API
D) Log the raw input for later review, then proceed regardless

**Answer: B.** Explicit range/type validation before use is the correct secure-coding pattern; A is a classic injection-risk pattern, C violates the "never trust input" principle even for internal sources.

---

# DOMAIN 4.0 — AI in Automation (20%)

## 4.1 Benefits and Risks of AI-Assisted Code Development

**Explain it simply:** AI coding assistants (Copilot-style tools, LLM chat assistants) can speed up writing playbooks/scripts significantly, but they introduce risks: your proprietary network data/config might leave your environment (data privacy), it's unclear who "owns" AI-generated code (IP ownership), and AI-generated code can be subtly wrong or insecure — so it still needs human review (code validation).

**Key facts:**
- **Data privacy**: pasting real device configs/topology/secrets into a public AI tool may expose sensitive data to a third party — enterprises often require using enterprise-tier/private AI deployments, or scrubbing data first.
- **IP ownership**: legal ambiguity/policy questions around whether AI-generated code can be copyrighted, and whether training data used by the AI vendor creates licensing risk — organizations typically need internal policy on this.
- **Code validation**: AI-generated code can "hallucinate" APIs/parameters that don't exist, or be functionally correct but insecure — human review, testing (ties into pyATS/CI pipeline), and static analysis are still required; never blindly trust and deploy.
- Benefit: faster first-draft code, boilerplate generation, documentation help — net productivity gain when paired with proper validation.

**Q1 (MCQ):** An engineer pastes a full running-config (including SNMP community strings and passwords) into a public AI chatbot to ask for a refactor suggestion. What is the primary risk?
A) IP ownership dispute
B) Data privacy exposure of sensitive network/security information to a third party
C) Rate limiting
D) YANG model incompatibility

**Answer: B.** This is a textbook data-privacy risk — sensitive config/secrets sent to an external service.

**Q2 (True/False):** Code generated by an AI assistant can be safely deployed to production without additional human review or testing, since modern LLMs rarely make mistakes in network configuration syntax.
**Answer: False.** AI-generated code must still go through validation/review/testing (e.g., CI pipeline, pyATS, peer review) — hallucinated commands/parameters are a known risk.

---

## 4.2 Security Risks in AI-Based Network Automation Solutions

**Explain it simply:** When you plug an LLM/AI agent into your network automation (e.g., letting it call APIs or generate/execute commands), you introduce new attack surfaces: prompt injection (malicious input tricking the AI into doing something unintended), over-permissioned agents (an AI with too much access executing destructive changes), and data leakage through model context/logs.

**Key facts:**
- **Prompt injection**: malicious content (e.g., embedded in a device banner, a ticket description, or a log line the AI reads) manipulates the AI's instructions to perform unintended/unauthorized actions.
- **Excessive agency/permission**: an AI agent granted broad API scopes (e.g., full config-write access) can cause much larger blast radius if manipulated or simply wrong — least-privilege scoping and human-in-the-loop approval for risky actions are mitigations.
- **Output validation**: AI-suggested commands must be validated/sandboxed before execution — never let an agent execute arbitrary generated CLI/API calls directly against production without a review/validation gate.
- **Data leakage**: conversation context/logs sent to an LLM API (especially third-party/cloud) may retain sensitive data depending on vendor data-retention policy.

**Q1 (MCQ):** An AI agent that reads incoming support tickets to help troubleshoot network issues is manipulated when an attacker submits a ticket containing hidden text instructing the AI to "also disable all firewall ACLs." This is an example of:
A) Rate limiting
B) Prompt injection
C) A YANG validation failure
D) A CI/CD pipeline failure

**Answer: B.** Classic prompt injection — malicious instructions embedded in otherwise-normal input data manipulate the AI's behavior.

**Q2 (MCQ):** What is the best mitigation for an AI agent that has been granted overly broad configuration-write permissions across the network?
A) Increase the AI's context window
B) Apply least-privilege scoping and require human approval/gating for high-risk actions
C) Disable logging to improve performance
D) Switch to a larger LLM model

**Answer: B.** Least privilege + human-in-the-loop approval directly reduces blast radius from a manipulated or erroneous agent.

**Q3 (True/False):** It is safe for an AI agent to execute any command it generates directly against production devices, as long as the underlying LLM is a well-known, reputable model.
**Answer: False.** Model reputation doesn't eliminate the need for output validation, scoping, and approval gates before executing generated changes.

---

## 4.3 Constructing an MCP Server with Python FastMCP

**Explain it simply:** MCP (**Model Context Protocol**) is a standard way to expose "tools" and data to an AI agent so it can call real functions (like "get interface status" or "get VLAN list") instead of just guessing from training data. **FastMCP** is a Python framework that makes it easy to define these tools as decorated Python functions, which an MCP-compatible AI client can then discover and call.

**Key facts:**
- Core FastMCP pattern: `from fastmcp import FastMCP`, instantiate `mcp = FastMCP("server-name")`, then decorate functions with `@mcp.tool()` to expose them as callable tools; `@mcp.resource()` to expose data resources.
- MCP separates **tools** (actions the AI agent can invoke, e.g., a function that runs `show ip interface brief` and returns parsed data) from **resources** (readable data/context, e.g., a device inventory) and **prompts** (reusable prompt templates).
- Tool functions should have clear docstrings/type hints — the AI agent uses the function signature/description to decide when/how to call it, so accurate descriptions matter a lot (a poorly documented tool leads to wrong AI usage).
- The MCP server runs as a process the AI client connects to (via stdio or HTTP/SSE transport); it exposes a discoverable list of tools via the protocol's handshake.
- Security-wise, tools exposed via MCP should still enforce input validation and least privilege (ties into 3.6 and 4.2) — the AI agent is effectively an untrusted-ish caller from a security-design standpoint.

**Q1 (MCQ):** In FastMCP, which decorator exposes a Python function as an invocable action for an AI agent?
A) `@mcp.resource()`
B) `@mcp.tool()`
C) `@mcp.prompt()`
D) `@app.route()`

**Answer: B.** `@mcp.tool()` is the FastMCP decorator for exposing callable actions. `@mcp.resource()` (A) exposes readable data/context instead of an invocable action — a close distractor since both are FastMCP decorators but serve different roles. `@app.route()` (D) is a Flask pattern, not MCP.

**Q2 (Fill in the blank):** In MCP terminology, a piece of exposed readable data/context (as opposed to a callable action) is called a `_______`.
**Answer:** resource

**Q3 (True/False):** The description/docstring of an MCP tool function is purely for human documentation and has no effect on how the AI agent uses the tool.
**Answer: False.** The AI agent relies on the tool's name, description, and parameter schema to decide when and how to invoke it — poor documentation leads to incorrect or missed tool usage.

**Q4 (MCQ):** An MCP server exposes a `get_interface_status(device_id: str)` tool with no input validation on `device_id`. What is the primary risk?
A) The AI agent won't be able to find the tool
B) An attacker (via prompt injection) or a malfunctioning agent could pass an unexpected/malicious value, potentially causing unintended access or errors
C) FastMCP requires validation to start the server at all
D) MCP tools cannot accept string parameters

**Answer: B.** Same input-validation principle as 3.6, now applied to the AI-facing tool layer — a key exam crossover point between domains 3 and 4.

---

## 4.4 Constructing a Conversational Agent Leveraging LLMs

**Explain it simply:** A conversational agent takes natural language ("show me interfaces that are down on router1"), uses an LLM to interpret intent, decides which tool/API to call (often via MCP or direct function-calling), executes it, and returns a natural-language answer — essentially a chat interface wrapped around your automation tooling.

**Key facts:**
- Core architecture: **User input → LLM (with function/tool-calling capability) → Tool/API execution (network automation backend) → LLM formats result → User-facing response**.
- **Function calling / tool use**: modern LLMs can be given a schema of available functions/tools (name, description, parameters) and will output a structured call (e.g., JSON) indicating which to invoke and with what arguments — the application code then actually executes it and returns the result to the LLM for final response formatting.
- **Context/memory**: conversational agents need to manage conversation history/state (especially for multi-turn troubleshooting) — trade-off between context window size/cost and continuity.
- **Grounding**: agents should be grounded in real device data (via tool calls) rather than letting the LLM "guess" network state from training data alone — reduces hallucination for factual network queries.
- **Guardrails**: input/output validation, scoping which tools are available to the agent (least privilege), and human confirmation before destructive actions (e.g., "disable interface") are standard design requirements.

**Q1 (MCQ):** A conversational agent is asked "What's the current OSPF neighbor state on R1?" Which design approach avoids the LLM simply fabricating a plausible-sounding but incorrect answer?
A) Let the LLM answer directly from its training data
B) Ground the response by having the LLM call a tool (e.g., an MCP tool or API) that retrieves live OSPF neighbor state from the device, then format that real data into the answer
C) Increase the model's temperature setting for more creative answers
D) Cache a static answer from the last time this question was asked, regardless of age

**Answer: B.** This is the core "grounding" principle — real-time factual network state must come from actual tool/API calls, not LLM memory, to avoid hallucination.

**Q2 (True/False):** A well-designed conversational agent should be able to execute destructive commands (like shutting down an interface) immediately upon a user's natural-language request, without any confirmation step, to maximize responsiveness.
**Answer: False.** Best practice requires a human-confirmation/guardrail step before destructive/high-risk actions, regardless of how "responsive" that makes the agent.

**Q3 (Fill in the blank):** The LLM capability that allows it to output a structured request to invoke a specific function/tool with specific parameters (rather than just free text) is called `_______` (two words).
**Answer:** function calling (a.k.a. tool calling/tool use)

---

## 4.5 Evaluating the Accuracy of AI Recommendations

**Explain it simply:** Before trusting an AI-generated automation recommendation (a playbook, a config snippet, a troubleshooting suggestion), you need a systematic way to check it: does it match documented vendor behavior, does it pass validation/testing (pyATS), is it internally consistent, and would a subject-matter expert sign off on it?

**Key facts:**
- Evaluation methods: **cross-reference against authoritative documentation** (vendor docs, RFCs, YANG models); **test in a safe environment** (CML lab, CI/CD prevalidation) before trusting/deploying; **peer/expert review**; check for **hallucinated syntax/APIs** (commands or parameters that don't actually exist on the platform).
- AI recommendations should be treated as a **draft/starting point**, not an authoritative source — same principle as 4.1's "code validation" but applied specifically to accuracy evaluation.
- Confidence signals to watch for: does the AI cite verifiable sources/versions, is the suggested config idiomatic/consistent with known best practice, does it handle edge cases (or silently ignore them)?
- Red flag pattern: AI confidently presents outdated or version-specific syntax as universal — always verify against the specific platform/version in use.

**Q1 (MCQ):** An AI assistant recommends a Cisco IOS-XE command that doesn't actually exist on the target platform/version. What is the best process to have caught this before deployment?
A) Trust the AI since it's a well-known model
B) Validate the recommendation against official documentation and test it in a lab (e.g., CML) or CI/CD prevalidation stage before deploying
C) Deploy it and see if it works
D) Ask the AI a second time to confirm its own answer

**Answer: B.** Independent verification against documentation plus safe testing is the correct evaluation process — D is a trap, since asking the same (or even a different) LLM to "confirm itself" doesn't constitute real validation; hallucinations can be repeated confidently.

**Q2 (True/False):** If an AI-generated recommendation is well-formatted and confidently stated, that is a reliable indicator of its technical accuracy.
**Answer: False.** LLM confidence/fluency of output has no inherent correlation with factual correctness — this is a core AI-literacy point the exam tests directly.

---

# Quick-Reference Cram Sheets

**Git command cheat sheet:**
| Command | Effect | Rewrites shared history? |
|---|---|---|
| `merge` | Combines branches | Adds merge commit (safe) |
| `cherry-pick` | Copies one commit to current branch | Safe (new commit) |
| `reset --soft/mixed/hard` | Moves branch pointer | **Yes — dangerous if pushed** |
| `checkout` | Switch branch / restore file | No |
| `revert` | New commit undoing a prior commit | Safe (new commit) |

**HTTP methods in RESTCONF:**
| Method | Behavior | Idempotent? |
|---|---|---|
| GET | Read | Yes |
| POST | Create (fails if exists, 409) | No |
| PUT | Create or full replace | Yes |
| PATCH | Partial merge update | Yes (typically) |
| DELETE | Remove | Yes |

**Syslog severities (0=worst):** Emergency, Alert, Critical, Error, Warning, Notice, Informational, Debug.

**CI/CD stage order:** Build → Prevalidation → Deploy → Post-validation.

**NAPALM safe-change order:** `open()` → `load_merge_candidate()` → `compare_config()` → `commit_config()` → `close()`.

**Cert lifecycle:** Generate keypair → Generate CSR → Submit to CA → CA issues signed cert → Install cert+chain → Renew before expiry.

---

*Tip for Feynman review: pick one row from each cheat sheet above and explain, out loud, why the order/behavior is what it is — not just what it is.*