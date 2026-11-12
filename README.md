# Distributed Computing Group 4

## Milestone 5: Distributed transactions

This milestone demonstrates distributed slice creation with two-phase commit (2PC) and three-phase commit (3PC). PostgreSQL stores each participant's transaction state and slice reservation durably. The host coordinator keeps a durable decision log in SQLite and exposes coordinator decisions over HTTP for participant recovery.

### Transaction states

```text
2PC: PREPARING -> PREPARED -> COMMIT_DECIDED -> COMMITTED -> COMPLETED
                         \-> ABORT_DECIDED -> ABORTED
3PC: PREPARING -> PREPARED -> PRECOMMITTED -> COMMIT_DECIDED -> COMMITTED -> COMPLETED
```

The coordinator writes COMMIT to its durable log before sending participant commits. It records ABORT before asking participants to release reservations. If a coordinator stops after deciding COMMIT, rerun its recovery or run `transaction_coordinator.recover()`; recovery retries the saved decision. If a participant restarts, its durable rows remain available. Run its `/recover` endpoint with the coordinator's reachable base URL to fetch decisions from the coordinator.

### Configure and start

Install worker dependencies from `worker_vms/requirements.txt` on each participant, and ensure PostgreSQL is running with database `telecom_slices` created. Configure `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`, and `DB_PORT` in the environment. The participant creates the Milestone 5 tables on startup; `worker_vms/setup_d.sql` is also provided for manual setup.

Start the participant process on each VM with the matching protocol:

```powershell
$env:TRANSACTION_PROTOCOL = "2pc"
python participant_db.py 6001
```

For 3PC use `TRANSACTION_PROTOCOL=3pc` and launch `participant_3pc.py`. Provide the participant's port as the script argument. Both participant entry points use the same durable service implementation.

On the host, install `host_pc/requirements.txt` and configure:

```powershell
$env:TRANSACTION_PARTICIPANTS = "http://192.168.56.104:6001,http://192.168.56.107:6002"
$env:COORDINATOR_LOG = "transactions.sqlite3"
$env:COORDINATOR_DECISION_PORT = "6100"
```

The coordinator decision port must be reachable from participant VMs when using participant-triggered `/recover`. If participants use different protocol configurations, start the matching coordinator and participant variant together.

Run a small demonstration with `python coordinator_2pc.py` or `python coordinator_3pc.py` from `host_pc`. For a combined measurement and failure run:

```powershell
python transaction_experiments.py --protocol both --count 20 --failure
```

The failure option substitutes an unreachable second participant for one transaction after the first participant has prepared. It should cause an abort and release the first participant's reservation. The output includes attempts, successes, failures, mean successful transaction latency, throughput, and success rate. Values depend on the VM/network environment; this repository does not include fabricated benchmark results.

### Recovery

The host runs an HTTP decision endpoint on port 6100 while coordinating. The endpoint `/transactions/<tx_id>` returns the durable decision. On a participant VM, after the host coordinator is available, request:

```text
POST http://<participant-host>:<participant-port>/recover
Content-Type: application/json

{"decision_url":"http://<host-reachable-address>:6100"}
```

Coordinator recovery (`recover()` and the experiment script) retries decisions saved in SQLite. Keep the SQLite log file and the PostgreSQL data durable; deleting either removes recovery information. An `UNDECIDED` coordinator entry is resolved to ABORT when the coordinator catches an error before recording COMMIT.

### ACID and consistency notes

- **Atomicity:** each participant's committed slice, transaction state, and reservation release are changed in one PostgreSQL transaction. The coordinator's durable decision drives retries after partial commit delivery. Distributed atomicity depends on eventual connectivity and retaining both logs.
- **Consistency:** participant reservations use a primary key on `slice_id`, preventing two prepared transactions from reserving the same slice on one participant. The committed `slices` table also has a primary key. Compare `/transactions/<tx_id>` and slice rows on every participant after normal, failed, and recovered runs.
- **Isolation:** PostgreSQL uniqueness constraints serialize competing reservations for the same slice. Different slices can prepare concurrently. This demo does not model arbitrary multi-row business invariants.
- **Durability:** participant state and slice data are stored in PostgreSQL; coordinator decisions are stored in SQLite with WAL mode. Protect and back up both stores.

3PC adds a PRECOMMITTED state between prepare and commit. It improves progress behavior under stated timing and failure assumptions, but this small HTTP simulation is not a production consensus protocol: network partitions, coordinator log loss, and arbitrary clock assumptions can still prevent progress or violate the assumptions behind 3PC.

### Evaluation and report template

Run the experiment on the same VM/network configuration for both protocols. Record the generated JSON, node count, participant locations, database configuration, and failure scenario. Use `throughput_tx_per_s` and `mean_latency_ms` to compare successful operations; report `availability_success_rate` and failures separately. Include the following evidence in the submitted analysis:

| Scenario | Protocol | Attempts | Successes | Failures | Mean latency (ms) | Throughput (tx/s) | Availability | Participant states agree? |
|---|---|---:|---:|---:|---:|---:|---:|---|
| Normal load | 2PC | | | | | | | |
| Normal load | 3PC | | | | | | | |
| Participant unavailable during prepare | 2PC | | | | | | | |
| Participant unavailable during prepare | 3PC | | | | | | | |
| Coordinator stopped after durable commit decision; then recovered | each | | | | | | | |

The built-in failure option exercises prepare failure and abort cleanup. To evaluate coordinator recovery, stop the coordinator after a `COMMIT_DECIDED` entry exists and before all participants finish, restart it with the same SQLite log, and invoke `recover()`. Verify that both participants reach `COMPLETED` and have the same slice. Record actual outcomes rather than treating the expected behavior as an observed result.

## Milestone 6: Concurrency and deadlock management

`worker_vms/concurrent_worker.py` is a separate concurrent resource service. Each CPU and memory unit is represented as a single-instance resource in a resource-allocation graph. Allocations use `Rj -> Pi` edges; blocked acquisitions use `Pi -> Rj` edges. A graph cycle triggers recovery by aborting the youngest process in the cycle and releasing all its resources. Per-resource FIFO wait queues provide a basic fairness policy. Tasks that exceed their wait timeout are reported as `TIMED_OUT`.

Start the service on **Mode1 (edge)**, from the `worker_vms` folder. Use one simulated CPU unit and one memory unit to make the deadlock experiment predictable:

```powershell
pip install -r requirements.txt
python concurrent_worker.py Mode1-edge 1 1 7001 8000
```

Start the second service on **node2 (core)**, also from `worker_vms`:

```powershell
pip install -r requirements.txt
python concurrent_worker.py node2-core 1 1 7001 8000
```

From the host PC's `host_pc` folder, start the opposite-order deadlock experiment across both VMs:

```powershell
pip install -r requirements.txt
python trigger_concurrency.py --scenario deadlock --tasks 2 --cpu 1 --memory 1
```

The pair of tasks on each VM requests CPU then memory and memory then CPU, respectively. The output reports submitted/completed/aborted tasks, per-node deadlock detection/recovery counts, wait time, resource-allocation graph edges, and sampled peak/current CPU and memory utilization. The VM terminal logs print detected cycles and their selected victims. The expected model outcome is that a cycle is found, one task is aborted to free resources, and the remaining task can complete; use the actual JSON output as the experiment result.

Run a contention test without opposite acquisition orders:

```powershell
python trigger_concurrency.py --scenario contention --tasks 6 --cpu 1 --memory 1 --hold 2
```

The worker exposes `GET /api/status` and `GET /api/resources` for capacities, allocations, utilization, wait queues, task states, and graph edges; `GET /api/tasks` returns task outcomes; `GET /api/deadlock_check` reports any current cycle and the graph; and `POST /api/recover` attempts victim recovery if a cycle exists at that time. Prometheus metrics are exposed on each VM's configured metrics port (8000 by default).

The simulator's resource units are logical capacities, not measurements of actual VM CPU cores or memory consumption. The host driver's utilization values describe those simulated units and use periodic sampling, so very short peaks may be missed. For the submitted analysis, record the output JSON, VM capacities, scenario, task count, terminal task states, deadlock count, recovery count, mean wait, and peak utilization. Compare contention and deadlock scenarios and state whether the observed outcomes match the expected behavior. The fairness policy prevents later requests from bypassing an earlier waiter for the same resource, but this small in-memory simulator does not prove starvation freedom under process crashes or VM restarts.

## Prometheus and Grafana dashboard

The repository includes a Prometheus and Grafana Compose stack. Prometheus scrapes the metrics exporter on both VMs (`192.168.56.104:8000` for Mode1 edge and `192.168.56.107:8000` for node2 core); edit `worker_vms/prometheus.yml` if the VM addresses differ. Ensure those addresses are reachable from the host PC running Docker and that each VM allows inbound TCP port 8000.

Start the dashboard stack from the repository root:

```powershell
docker compose up -d
```
Open Grafana at `http://localhost:3000` and sign in with `admin` / `admin` on first run, then set a new password. The **Distributed Computing → Distributed Computing — Node & Concurrency Metrics** dashboard and Prometheus datasource are provisioned automatically. Prometheus is also available at `http://localhost:9090`. To stop the stack, run `docker compose down`; persistent Prometheus/Grafana data remains in named Docker volumes. `docker compose down -v` removes those volumes and their stored data.

Dashboard panels show Prometheus scrape health, simulated CPU/memory utilization, Milestone 6 deadlock and recovery counters, concurrent task throughput, and existing worker process/migration and API latency metrics. Deadlock/task panels populate while `concurrent_worker.py` is running. Existing worker-node panels populate while `worker_node.py` is running. Both programs expose metrics on port 8000 by default, so run only one metrics exporter per VM at a time, or change the exporter ports and corresponding Prometheus targets.
