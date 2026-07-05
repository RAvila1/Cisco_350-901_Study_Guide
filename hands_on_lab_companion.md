# 350-901 Hands-On Lab Companion
### Built for: EVE-NG (IOS-XE / NX-OS / IOS-XR / IOL) + CWS (VS Code, Python, Ansible, Terraform) + separate Linux VM (Docker: GitLab CE + runner)

This pairs with the concept study guide. Each lab is scoped to be doable in your environment, in roughly 30–90 minutes, and produces an artifact (playbook, pipeline, script) you can reuse and expand. Do these in order — later domains (Ops, AI) reuse code from Domain 1/2 labs.

---

## Lab 0 — Environment Wiring (do this once)

1. **EVE-NG topology**: Build one lab with at least 2× IOS-XE (Cat8000v/CSR), 1× NX-OS (Nexus9kv), 1× IOS-XR (XRv9k), 2× IOL (cheap L2/L3 filler devices). Give each a mgmt interface reachable from CWS (you said this is done — confirm you can `ssh` to each from CWS right now before proceeding).
2. **Enable API surfaces on-device** (needed for 1.3/1.4/2.7):
   - IOS-XE: `netconf-yang` + `restconf` under global config, plus `ip http secure-server`.
   - NX-OS: `feature nxapi` for NX-API (REST), `feature netconf` if available on your image.
   - IOS-XR: `netconf-yang agent ssh` (XR's RESTCONF/NETCONF enablement differs by release — check your image's config guide).
3. **GitLab CE on the Linux VM**:
   ```bash
   docker run --detach --hostname gitlab.local --publish 443:443 --publish 80:80 --publish 2222:22 \
     --name gitlab --restart always --volume gitlab-config:/etc/gitlab \
     --volume gitlab-logs:/var/log/gitlab --volume gitlab-data:/var/opt/gitlab \
     gitlab/gitlab-ce:latest
   ```
   Then register a **shell or docker executor runner** (Admin Area → CI/CD → Runners → registration token):
   ```bash
   docker run --rm -it -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner register
   ```
4. **CWS toolchain check**: `python3 -m venv venv && pip install ansible netmiko napalm ncclient requests pyats[library] genie --break-system-packages` (or inside the venv, drop the flag). Install `terraform` CLI. Install VS Code extensions: YAML, Ansible, GitLens.
5. **Push a "network-automation" repo to both GitHub and your GitLab CE instance** — you'll use GitHub for pure Git mechanics (2.1) and GitLab CE for pipeline labs (2.2/2.3).

---

# Domain 1.0 Labs — Network Automation

### Lab 1.1 — Ansible: VLANs, OSPF, interfaces, ACLs
1. Build inventory (`inventory.yml`) with your IOS-XE and NX-OS hosts, `ansible_network_os` set correctly per host.
2. Write `vlans.yml` playbook using `ios_vlans` to create VLANs 10/20/30 declaratively. Run it **twice** — confirm the second run reports `changed: false` (idempotency proof — screenshot/save this for your own confidence check).
3. Add a task using `ios_ospfv2` to enable OSPF area 0 on two interfaces between your IOL devices.
4. Add an `ios_acls` task creating a numbered ACL denying one host, applied inbound on an interface.
5. **Break it on purpose**: manually change a VLAN name on-device via CLI, rerun the playbook, confirm Ansible detects and corrects the drift. This is the single best exercise for internalizing 1.1 + 3.3 together.
6. Repeat step 2's VLAN task using the equivalent `nxos_vlans` module against your Nexus node — note syntax differences in facts/module naming (`nxos_*` vs `ios_*` family).

### Lab 1.2 — Terraform
1. Find/install a Cisco IOS-XE Terraform provider (`CiscoDevNet/iosxe`) or, if unavailable/unstable in your version, use the **RESTCONF-backed generic `http` provider** as a fallback to hit your IOS-XE RESTCONF endpoint directly from `.tf` — either approach teaches the same plan/apply/state lifecycle.
2. Write a `main.tf` declaring an interface description + VLAN resource.
3. `terraform init`, `terraform plan`, `terraform apply`.
4. **Drift exercise**: SSH to the device, manually change the interface description, then `terraform plan` again — observe the diff it proposes. This is your best real-world proof of the 1.2 drift concept.
5. `terraform destroy` and rebuild from scratch to internalize the full lifecycle.

### Lab 1.3 — RESTCONF + YANG
1. From CWS, use `curl` or VS Code's REST Client extension against your IOS-XE RESTCONF endpoint:
   ```
   GET https://<device>/restconf/data/ietf-interfaces:interfaces
   Accept: application/yang-data+json
   ```
2. Explore `ietf-yang-library:yang-library` to see supported models on your specific image/version — compare IOS-XE vs IOS-XR support (XR's RESTCONF support and model set differs — document what you find, this is genuinely useful exam intuition).
3. **PATCH** a single new ACL entry into an existing ACL list without disturbing others — confirm with a follow-up GET.
4. **POST** a brand-new VLAN resource, then POST it again — confirm you get the 409 Conflict the study guide describes, in your own terminal.

### Lab 1.4 — Python (Netmiko / NAPALM / ncclient)
1. Write a Netmiko script: connect to an IOL device, `send_config_set()` to build an ACL, wrapped in `try/except NetmikoTimeoutException`.
2. Write a NAPALM script doing the full safe-change sequence (`open → load_merge_candidate → compare_config → commit_config → close`) against IOS-XE. Print the diff from `compare_config()` before committing — read it before you approve the commit, don't just script through it blindly.
3. Write an `ncclient` script doing a NETCONF `get_config()` then `edit_config()` against the same device — compare the raw XML you're pushing to the JSON you PATCHed in Lab 1.3 (same YANG model, different encoding — this comparison is genuinely clarifying).

### Lab 1.5 — Approach selection (thought exercise, no lab needed)
Write a one-paragraph justification for which of your Domain 1 tools (Ansible/Terraform/RESTCONF-Python) you'd choose for three scenarios: (a) a helpdesk tech needs to reset a port to a standard access config, (b) your team needs Git-reviewed, repeatable multi-site VLAN rollout, (c) a custom app needs to pull inventory from three different systems and reconcile it. Explaining your reasoning out loud is the Feynman check for this subtopic.

### Lab 1.6 — REST API consumption edge cases
1. If you don't have a real paginated/rate-limited API handy, stand up a tiny Flask app on your Linux VM that fakes one (returns paginated JSON, a `next` field, and occasionally a 429). This is worth building — it's the only way to *practice* backoff/pagination code against something that actually behaves that way.
2. Write a Python client with: token caching (store/reuse a bearer token), pagination-following loop, and exponential backoff on 429.

---

# Domain 2.0 Labs — Infrastructure as Code

### Lab 2.1 — Git (on GitHub, as planned)
Do this as one continuous scenario in a scratch repo:
1. Create `main`, branch to `feature/vlan-update`, commit changes to your Lab 1.1 playbook.
2. Deliberately create a merge conflict (edit the same line differently on `main` and your feature branch), resolve it, commit.
3. `git cherry-pick` a single commit from `feature/vlan-update` onto a third branch `hotfix/urgent`.
4. Make 3 commits, then `git reset --mixed HEAD~1` and inspect `git status` (staged vs unstaged) — then redo with `--soft` and `--hard` to feel the difference firsthand.
5. `git revert` a pushed commit and confirm history is preserved (`git log --oneline --graph`).
6. `git checkout <old-commit> -- file.yml` to pull one file's old version without switching branches.

### Lab 2.2 / 2.3 — GitLab CE CI/CD pipeline
1. Push your Lab 1.1 Ansible project to your self-hosted GitLab CE instance.
2. Write `.gitlab-ci.yml` with stages `build`, `prevalidation`, `deploy`, `postvalidation`:
   - **build**: `pip install -r requirements.txt`, lint the playbook (`ansible-lint`).
   - **prevalidation**: run a `pyats`/Genie pre-check (e.g., confirm devices reachable, snapshot current VLAN state) — make this job **fail on purpose once** (point it at a wrong IP) and confirm the pipeline halts before deploy.
   - **deploy**: run `ansible-playbook` against your EVE-NG devices.
   - **postvalidation**: re-run the Genie snapshot, `genie diff` against the pre-check snapshot, fail the job if unexpected deltas appear.
3. Intentionally break each failure category from 2.2 and observe the log signature:
   - Remove a package from `requirements.txt` → missing dependency error.
   - Pin an incompatible library version → version conflict error.
   - Push a config expected to fail a post-validation check → failed test.

### Lab 2.4 — CML mention / EVE-NG equivalent
You're using EVE-NG, not CML — conceptually identical for the exam (disposable virtual topology for pre-prod testing). Since the exam names CML specifically, at minimum: read up on CML's YAML topology export format and its `virl2_client` Python API so you can recognize CML-specific syntax/terms on the exam, even though your hands-on reps are happening in EVE-NG. Optionally, add an EVE-NG topology start/stop script triggered from a CI/CD stage to mirror what a CML-integrated pipeline would do.

### Lab 2.5 — Docker Compose
1. On the Linux VM, write a `docker-compose.yml` spinning up: a Flask "mock source of truth" API, a Postgres or SQLite-backed volume for it, and your GitLab runner — all on one user-defined network.
2. Test `depends_on` behavior: make the SoT API intentionally slow to start, and observe that a dependent container can start before the API is actually ready to serve — confirms the 2.5 gotcha firsthand.

### Lab 2.6 — Source of Truth
1. Stand up NetBox via its official `docker-compose` on your Linux VM (or use your Lab 2.5 mock SoT).
2. Populate it with your EVE-NG devices' names/mgmt IPs/roles.
3. Write an Ansible **dynamic inventory** plugin config (`netbox.yml`) pulling hosts from NetBox instead of a static `inventory.yml` — rerun Lab 1.1's playbook against the dynamic inventory.

### Lab 2.7 — YANG → JSON/YAML
Using the YANG models you explored in Lab 1.3, hand-write a JSON payload for a new list entry (e.g., a new interface or ACE) purely from reading the YANG module (via `pyang` tree output or the yang-library GET), *then* validate it by POSTing/PATCHing it and seeing if it's accepted — the fastest possible feedback loop for this skill.

---

# Domain 3.0 Labs — Operations

### Lab 3.1 — Model-driven telemetry
1. On IOS-XE, configure a gRPC dial-out telemetry subscription streaming interface counters to a collector.
2. Stand up **Telegraf** (Docker on the Linux VM) configured with the `cisco_telemetry_gnmi` or `grpc` input plugin, writing to **InfluxDB**, visualized in **Grafana** (all as Docker Compose services — reuses 2.5 skills).
3. Compare dial-in vs dial-out practically: also configure a dial-in subscription and pull it from a Python gNMI client — feel the difference in who-initiates-the-connection.

### Lab 3.2 — Logging strategy
1. Add structured JSON logging to your Lab 1.4 Python scripts (use Python's `logging` + a JSON formatter) — log every config change attempt, result, and timestamp, but assert in code review that no token/password ever gets logged.
2. Stand up a syslog server (`rsyslog` container) and point your IOS-XE lab devices at it; trigger a few severity levels intentionally (e.g., a bad config attempt vs a routine change) and watch them land.
3. Write a webhook notifier: on pipeline (2.3) deploy success/failure, POST a JSON payload to a Webex space (or a mock endpoint if you don't have a Webex bot token) via `requests`.

### Lab 3.3 — Diagnosing from logs
Deliberately generate each failure signature from the study guide and save the real log output as your own reference: an expired/wrong SSH credential (401-equivalent), a wrong mgmt IP (timeout), a deliberately malformed CLI push (`% Invalid input`), and a non-idempotent Ansible task (comment out a `state:` parameter and watch `changed: true` repeat). Keep a personal "signature cheat sheet" of real output — far stronger than memorizing mine.

### Lab 3.4 — pyATS/Genie change validation
1. Write a testbed YAML for your EVE-NG devices.
2. `pyats learn ospf` before and after a deliberate OSPF change; `genie diff` the two learned outputs.
3. Wire this exact learn/diff pair into your Lab 2.3 pipeline's prevalidation/postvalidation stages if you haven't already.

### Lab 3.5 — CA-signed certs
1. Stand up a simple internal CA (`openssl` on the Linux VM, or step-ca in Docker for a more realistic ACME-style flow).
2. Generate a keypair + CSR for your IOS-XE's RESTCONF HTTPS listener, sign it with your internal CA, install the cert (`crypto pki` commands on IOS-XE), and confirm your Lab 1.3 RESTCONF calls now trust it instead of hitting a self-signed cert warning.

### Lab 3.6 — Secure coding
1. Take your Lab 1.1/1.4 scripts and refactor: move all credentials into `ansible-vault`-encrypted vars or environment variables (never hardcoded), add input validation on any user/API-sourced VLAN ID or interface name (reject out-of-range values before they touch a command string).
2. Run `ansible-vault encrypt group_vars/all/secrets.yml`, commit it to your repo, and confirm the file is unreadable in Git history without the vault password.

---

# Domain 4.0 Labs — AI in Automation

### Lab 4.1/4.2 — Risk awareness (exercise, not code)
Deliberately paste a **sanitized/fake** running-config into a public AI chat tool and note what you'd have leaked if it were real (a useful, safe way to feel the 4.1/4.2 risk viscerally without actually exposing real secrets).

### Lab 4.3 — FastMCP server
1. On CWS, `pip install fastmcp`.
2. Write an MCP server exposing tools wrapping your Lab 1.4 Python functions: `get_interface_status(device_id)`, `get_vlan_list(device_id)` — using NAPALM/Netmiko under the hood, with input validation on `device_id` (must match a known inventory entry, ties to 3.6).
3. Run the server, connect an MCP-compatible client (e.g., Claude Desktop's MCP config, if available to you) and ask it network-state questions — watch it call your tools instead of guessing.

### Lab 4.4 — Conversational agent
1. Extend Lab 4.3: build a small script that takes a natural-language question, sends it to an LLM API with your MCP tools registered as function-calling schemas, executes whichever tool the model selects, and returns a formatted answer.
2. Add a guardrail: any tool that would *change* device state (not just read) requires a manual `y/n` confirmation printed to console before executing — implement this yourself so the "human-in-the-loop" concept is muscle memory, not just a bullet point.

### Lab 4.5 — Evaluating AI recommendations
Ask an AI assistant for a config recommendation on a feature you haven't configured yet (e.g., HSRP, or a specific IOS-XR feature), then verify it against official Cisco documentation and actually test it in EVE-NG before trusting it — do this once deliberately, noting anywhere the AI's answer was subtly wrong or version-mismatched. That lived experience is worth more than any flashcard for 4.5.

---

## Suggested Order / Timeline
Given your setup is fully ready, a realistic pace:
- **Week 1**: Lab 0 (environment), Domain 1 labs (1.1–1.6)
- **Week 2**: Domain 2 labs (2.1–2.7) — this is the most infrastructure-heavy week (GitLab CE, NetBox, Docker Compose)
- **Week 3**: Domain 3 labs (3.1–3.6)
- **Week 4**: Domain 4 labs (4.1–4.5) + a full review pass through the study guide's Q&A using what you now have hands-on scars from
- **Final days**: timed practice quiz (say the word and I'll build one), re-run your weakest labs from memory without notes

If a specific lab stalls (provider install issues, IOS-XR RESTCONF quirks, GitLab runner registration, etc.), bring me the actual error output — I can debug it with you in real time, which will teach you more than the lab instructions alone.