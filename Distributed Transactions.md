# COMP0178 — Distributed Databases: Revision Notes
 
These notes supplement your lecture slides with content from Connolly & Begg, Chapters 22, 24, and 25. Organised to match your lecture structure.
 
---
 
## 1. What are Distributed Databases
 
### 1.1 Definitions
 
**Distributed database:** A logically interrelated collection of shared data (and a description of this data) physically distributed over a computer network.
 
**Distributed DBMS (DDBMS):** The software system that permits the management of the distributed database and makes the distribution transparent to users.
 
A DDBMS has these characteristics:
- A collection of logically related shared data
- Data is split into **fragments**
- Fragments may be **replicated**
- Fragments/replicas are **allocated** to sites
- Sites are linked by a communications network
- Data at each site is under the control of a local DBMS
- Each DBMS can handle **local applications** autonomously
- Each DBMS participates in at least one **global application**
Not every site needs its own local database — some sites may only access data stored elsewhere.
 
### 1.2 Contrast with Distributed Processing
 
**Distributed processing** is a centralized database that can be accessed over a computer network. The data itself lives at one site; multiple users simply connect to it remotely.
 
The key distinction: in a DDBMS, data is *physically distributed* across multiple sites. In distributed processing, data is centralised but *compute* is distributed. If data is central, it's distributed processing, not a distributed database — even if many remote users access it.
 
### 1.3 Contrast with Parallel DBMSs
 
A **parallel DBMS** runs across multiple processors and disks to execute operations in parallel for performance. Three architectures exist: shared memory (SMP), shared disk (clusters), and shared nothing (MPP).
 
Key differences from a DDBMS: parallel DBMS nodes are typically co-located at one site, under unified administration, with fast interconnects. DDBMS nodes are geographically distributed, separately administered, and connected by slower networks. Distribution in a parallel DBMS is driven purely by performance; in a DDBMS it reflects organisational structure.
 
### 1.4 Advantages of DDBMSs
 
1. **Reflects organisational structure** — data naturally lives where the organisational units that use it are located (e.g. each branch office keeps its own staff/property data).
2. **Improved shareability and local autonomy** — users at one site can access data at other sites, while each site retains local control and can enforce local policies.
3. **Improved availability** — failure at one site does not bring down the entire system; requests can potentially be rerouted.
4. **Improved reliability** — replicated data means a node or link failure doesn't necessarily make data inaccessible.
5. **Improved performance** — data is near the site of greatest demand; inherent parallelism across sites.
6. **Economics** — networks of smaller machines are cheaper than equivalently powerful mainframes.
7. **Modular growth** — new sites can be added without affecting existing operations.
8. **Integration** — can unify legacy systems and heterogeneous databases.
9. **Remaining competitive** — enables e-business, collaborative work, and workflow management.
### 1.5 Disadvantages of DDBMSs
 
1. **Complexity** — hiding distribution while maintaining performance, reliability, and availability is inherently harder than centralised management.
2. **Cost** — additional hardware (network), higher procurement and maintenance costs, communication costs, labour costs for distributed administration.
3. **Security** — must control access at multiple locations and secure the network itself.
4. **Integrity control more difficult** — enforcing constraints across distributed data incurs high communication and processing costs.
5. **Lack of standards** — communication and data access protocol standards are still evolving; no established tools for converting centralised to distributed.
6. **Lack of experience** — less industry experience compared to centralised systems.
7. **Database design more complex** — must consider fragmentation, allocation, and replication on top of normal design.
### 1.6 Homogeneous vs Heterogeneous DDBMSs
 
**Homogeneous:** All sites use the same DBMS product. Easier to design and manage; supports incremental growth.
 
**Heterogeneous:** Sites run different DBMS products, possibly with different data models (relational, network, object-oriented, etc.). Requires translations between different DBMSs — mapping data structures, query languages, hardware differences. Gateways are the typical solution, though they have limitations (often query-only, no cross-system transaction coordination).
 
### 1.7 Multidatabase Systems (MDBS)
 
A **multidatabase system** is a distributed DBMS where each site maintains complete autonomy. An MDBS sits transparently on top of existing database systems and presents a single database to users, maintaining only a global schema. Local DBMSs keep all user data and full control of their operations. The MDBS translates global queries into local queries, merges results, and coordinates commit/abort of global transactions.
 
**Federated** MDBSs have local users too (centralised for local users, distributed for global users). The global conceptual schema is a *subset* of local schemas — only what each site agrees to share — unlike a DDBMS where the GCS is the *union* of all local schemas.
 
---
 
## 2. Design Considerations
 
### 2.1 Why Fragment?
 
Four reasons to fragment rather than distributing whole relations:
1. **Usage** — applications typically work with views (subsets), not entire relations
2. **Efficiency** — data is stored close to where it's most frequently used; unneeded data isn't stored locally
3. **Parallelism** — fragments enable sub-queries to execute in parallel at different sites
4. **Security** — data not required by local applications is not stored locally and thus not available to unauthorised users
Two primary disadvantages:
- **Performance** — global queries needing data from fragments at different sites may be slower
- **Integrity** — enforcing constraints across fragments at different sites is harder
### 2.2 Correctness Rules for Fragmentation
 
Any fragmentation must satisfy three rules:
 
1. **Completeness** — every data item in the original relation R must appear in at least one fragment. No data loss.
2. **Reconstruction** — it must be possible to reconstruct R from the fragments using a relational operation (union for horizontal, natural join for vertical).
3. **Disjointness** — a data item should not appear in more than one fragment. Exception: primary key attributes must be repeated in vertical fragments for reconstruction.
For horizontal fragmentation, "data item" = tuple. For vertical fragmentation, "data item" = attribute.
 
### 2.3 Fragmentation Types
 
#### Horizontal Fragmentation
 
Subsets of **tuples**. Defined using the **Selection** (σ) operator.
 
Given relation R: σ_p(R) where p is a predicate on attributes of R.
 
**Example:** FragmentPropertyForRent by type:
- P1 = σ_{type='House'}(PropertyForRent)
- P2 = σ_{type='Flat'}(PropertyForRent)
Reconstruction: P1 ∪ P2 = PropertyForRent
 
#### Vertical Fragmentation
 
Subsets of **attributes**. Defined using the **Projection** (π) operator.
 
Given relation R: π_{a1,...,an}(R) where a1,...,an are attributes of R.
 
**Example:** Fragment Staff for payroll vs HR:
- S1 = π_{staffNo, position, sex, DOB, salary}(Staff)
- S2 = π_{staffNo, fName, lName, branchNo}(Staff)
Both fragments must include the **primary key** (staffNo) for reconstruction.
 
Reconstruction: S1 ⋈ S2 = Staff (natural join)
 
**Determining vertical fragments:** Create an attribute affinity matrix showing how frequently each pair of attributes is accessed together by transactions. Pairs with high affinity belong in the same fragment; pairs with low affinity can be separated.
 
#### Mixed (Hybrid) Fragmentation
 
A horizontal fragment is subsequently vertically fragmented, or vice versa. Defined using both σ and π.
 
**Example:** Vertically fragment Staff into S1 and S2, then horizontally fragment S2 by branch:
- S21 = σ_{branchNo='B003'}(S2)
- S22 = σ_{branchNo='B005'}(S2)
- S23 = σ_{branchNo='B007'}(S2)
Reconstruction: S1 ⋈ (S21 ∪ S22 ∪ S23) = Staff
 
#### Derived Horizontal Fragmentation
 
A horizontal fragment based on the fragmentation of a **parent** relation (the one whose primary key is referenced by a foreign key). Defined using the **Semijoin** (⋉) operator.
 
Given child relation R and parent S: R_i = R ⋉_f S_i
 
**Example:** Staff is horizontally fragmented by branchNo (S3, S4, S5). Fragment PropertyForRent to match:
- P_i = PropertyForRent ⋉_{staffNo} S_i
This ensures properties are co-located with the staff who manage them, making joins efficient.
 
If a relation has multiple foreign keys, choose the parent whose fragmentation is most frequently used or produces better join characteristics.
 
### 2.4 Data Allocation Strategies
 
| Strategy | Description |
|---|---|
| **Centralised** | Single database at one site, users distributed across network |
| **Fragmented (Partitioned)** | Disjoint fragments, each assigned to one site |
| **Complete Replication** | Complete copy of entire database at every site |
| **Selective Replication** | Combination: some data fragmented, some replicated, some centralised |
 
### 2.5 Comparison of Allocation Strategies
 
| Criterion | Centralised | Fragmented | Complete Replication | Selective Replication |
|---|---|---|---|---|
| **Locality of reference** | Lowest | High* | Highest | High* |
| **Reliability & availability** | Lowest | Low for item, high for system | Highest | Low for item, high for system |
| **Performance** | Unsatisfactory | Satisfactory* | Best for read | Satisfactory* |
| **Storage costs** | Lowest | Lowest | Highest | Average |
| **Communication costs** | Highest | Low* | High for update, low for read | Low* |
 
*Subject to good design.
 
Selective replication is the most commonly used strategy because of its flexibility — it aims to get all the advantages of the other approaches while minimising the disadvantages.
 
### 2.6 Fragmentation Design Summary
 
1. Design global relations normally (ER modelling → relational schema)
2. Examine system topology (sites at branch, city, or regional level)
3. Analyse important transactions to identify horizontal or vertical fragmentation opportunities
4. Decide which relations should NOT be fragmented (small, rarely updated → replicate everywhere)
5. Fragment one-side relations first; many-side relations are candidates for derived fragmentation
6. Check for situations where vertical or mixed fragmentation is appropriate
---
 
## 3. DDBMS Features & Transparency
 
The fundamental principle: **to the user, a distributed system should look exactly like a non-distributed system.**
 
### 3.1 Distribution Transparency
 
Four levels, ordered from most transparent to least:
 
#### Fragmentation Transparency (highest level)
User does not know data is fragmented. Queries are written against the global schema, identical to a centralised system:
```sql
SELECT fName, lName FROM Staff WHERE position = 'Manager';
```
 
#### Location Transparency (middle level)
User must know how data is fragmented (fragment names) but NOT where fragments are located:
```sql
SELECT fName, lName FROM S21
WHERE staffNo IN (SELECT staffNo FROM S1 WHERE position = 'Manager')
UNION ...
```
Advantage: database can be physically reorganised without impacting applications.
 
#### Replication Transparency
User is unaware that fragments are replicated. Implied by location transparency, but can exist independently.
 
#### Local Mapping Transparency (lowest level)
User must specify both fragment names AND locations:
```sql
SELECT fName, lName FROM S21 AT SITE 3
WHERE staffNo IN (SELECT staffNo FROM S1 AT SITE 5 WHERE position = 'Manager')
UNION ...
```
Unlikely to be acceptable to end-users due to complexity.
 
#### Naming Transparency
Each database object needs a globally unique name. Approaches:
- **Central name server** — ensures uniqueness but creates a bottleneck, reduces autonomy, and creates a single point of failure
- **Prefix with site identifier** (e.g. S1.Branch.F3.C2) — unique but destroys distribution transparency
- **Aliases/synonyms** — users use local names (e.g. "LocalBranch"); DDBMS maps to the systemwide name internally. Best approach.
R* uses a systemwide name with four components: Creator ID, Creator Site ID, Local Name, Birth-Site ID.
 
### 3.2 Transaction Transparency
 
#### Concurrency Transparency
Results of all concurrent transactions (distributed and non-distributed) must be logically consistent with some serial execution — same principle as centralised, but now the DDBMS must ensure both global and local transactions don't interfere, and all subtransactions of a global transaction are consistent.
 
Replication complicates concurrency: updates to one copy must propagate to all copies. Strategies range from synchronous (atomic with the original transaction — lower availability if a site is down) to asynchronous (delay of seconds to hours in regaining consistency).
 
#### Failure Transparency
In addition to centralised failure types (system crash, media failure, software error, etc.), the distributed environment adds: loss of a message, failure of a communication link, failure of a site, and network partitioning.
 
The DDBMS must ensure atomicity of the global transaction — all subtransactions commit or all abort. If subtransaction at S1 commits but S2 aborts, the database is inconsistent. This is solved by the **two-phase commit** protocol (Section 5).
 
### 3.3 Performance Transparency
 
The DDBMS should not suffer performance degradation due to the distributed architecture. The **distributed query processor (DQP)** must determine the most cost-effective execution strategy, considering:
- Which fragment to access
- Which copy of a replicated fragment to use
- Which location to use
Cost functions consider: access time (I/O), CPU time, and **communication cost** (often dominant in WANs). Optimisation may minimise total cost OR minimise response time (by maximising parallelism).
 
### 3.4 DBMS Transparency
 
Hides that local DBMSs at different sites may be different products. Only applicable to heterogeneous DDBMSs. The most difficult transparency to provide.
 
### 3.5 Date's 12 Rules for a DDBMS
 
**Fundamental principle:** To the user, a distributed system should look exactly like a non-distributed system.
 
1. **Local autonomy** — local data is locally owned and managed; local operations remain purely local
2. **No reliance on a central site** — no single site whose failure brings down the system (no central transaction manager, no central deadlock detection, etc.)
3. **Continuous operation** — no planned shutdowns needed for adding/removing sites or creating/deleting fragments
4. **Location independence** — users can access all data as if it were stored at their own site
5. **Fragmentation independence** — users can access data regardless of how it's fragmented
6. **Replication independence** — users should be unaware of replication; shouldn't access specific copies or manually update all copies
7. **Distributed query processing** — system must handle queries referencing data at multiple sites
8. **Distributed transaction processing** — transactions are the unit of recovery; both global and local transactions must satisfy ACID
9. **Hardware independence** — DDBMS runs on various hardware platforms
10. **Operating system independence** — runs on various operating systems
11. **Network independence** — runs on various communication networks
12. **Database independence** — supports heterogeneous local DBMSs (different data models)
Rules 9–12 are ideals; only partial compliance is expected in practice.
 
---
 
## 4. Distributed Query Optimisation
 
### 4.1 The Problem
 
In a centralised DBMS, the query processor finds an optimal execution strategy as an ordered sequence of operations. In a DDBMS, the DQP must additionally account for fragmentation, replication, and allocation schemas.
 
Communication cost is often dominant: Communication Time = C₀ + (bits_in_message / transmission_rate), where C₀ is the access delay per message.
 
**Key insight:** Sending data in bulk is much cheaper than sending individual records, because each individual transfer incurs the fixed access delay C₀. Minimising both the volume of data transmitted and the number of network transmissions is critical.
 
### 4.2 Worked Example (Rothnie & Goodman)
 
**Schema:**
- Property(propertyNo, city) — 10,000 records at London
- Client(clientNo, maxPrice) — 100,000 records at Glasgow
- Viewing(propertyNo, clientNo) — 1,000,000 records at London
**Query:** List properties in Aberdeen viewed by clients with maxPrice > £200,000.
 
**Assumptions:** 100 chars/tuple, transmission rate = 10,000 chars/sec, access delay = 1 sec, 10 clients with maxPrice > £200,000, 100,000 viewings for Aberdeen properties, computation time negligible.
 
| Strategy | Approach | Time |
|---|---|---|
| 1 | Move entire Client relation to London, process there | ≈ 16.7 minutes |
| 2 | Move Property + Viewing to Glasgow, process there | ≈ 28 hours |
| 3 | Join Property⋈Viewing at London, then for each Aberdeen tuple send individual query to Glasgow to check maxPrice | ≈ 2.3 days |
| 4 | Select maxPrice > 200K at Glasgow, for each (10 clients) send individual query to London to check for Aberdeen viewing | ≈ 20 seconds |
| 5 | Join Property⋈Viewing at London, select Aberdeen, project propertyNo+clientNo, move result to Glasgow for matching | ≈ 16.7 minutes |
| **6** | **Select maxPrice > 200K at Glasgow (10 tuples), move result to London for matching** | **≈ 1 second** |
 
**Key lessons:**
- Response times vary from 1 second to 2.3 days — choosing the wrong strategy is devastating
- Strategy 3 is terrible because it sends 100,000 individual messages (each incurring the 1-second access delay)
- Strategy 6 is best because it reduces the data *before* transmitting (only 10 tuples × 100 chars)
- **General principle:** reduce data volume at the source site first, then transmit the small result
### 4.3 The DQP Process
 
1. Parse the SQL query
2. Convert to a relational algebra tree
3. Replace leaf nodes with the appropriate fragments from the fragmentation schema
4. Eliminate redundant work (e.g. if a selection predicate matches the fragmentation predicate, some fragments can be pruned entirely)
5. Use statistics (table sizes, selectivities) to estimate costs of different strategies
6. Choose the strategy that minimises cost (total cost or response time)
7. Distribute sub-queries to the appropriate sites for execution
---
 
## 5. Distributed Transactions
 
A **distributed transaction** accesses data at more than one site. It is divided into **subtransactions**, one per site, each represented by an **agent**. Subtransactions at different sites can execute concurrently (inherent parallelism).
 
### 5.1 Concurrency Control Extensions
 
The DDBMS must ensure that:
- Both global and local transactions do not interfere with each other
- All subtransactions of a global transaction are consistent
- The consistency of replicated data is maintained
#### Distributed Locking
Extends 2PL to the distributed setting. Three approaches to managing lock tables:
 
1. **Centralised locking** — a single site manages all locks. Simple but creates a bottleneck and single point of failure. Violates "no reliance on a central site."
2. **Primary copy locking** — for replicated data, one copy is designated as the primary; locks are obtained at the primary copy's site. If data isn't replicated, locks are at the data's site.
3. **Distributed locking** — lock requests handled by the local lock manager at each site. For replicated data, locks must be obtained at *all* sites holding a copy (or a majority — "majority locking"). More resilient but higher overhead.
#### Distributed Timestamping
Each transaction is assigned a globally unique timestamp. Requires synchronised clocks or a globally agreed method of generating timestamps (e.g. local timestamp + site ID). Same basic and Thomas's write rule protocols apply, but across sites.
 
### 5.2 Challenges with Replication
 
When a replicated data item is updated, the update must propagate to all copies. Three strategies:
 
1. **Synchronous (eager)** — update all copies as part of the original transaction (atomic). Guarantees consistency but reduces availability (if any copy's site is unreachable, the transaction blocks or fails). Probability of success decreases exponentially with number of copies.
2. **Asynchronous (lazy)** — update propagated sometime after the original transaction commits. Higher availability but temporary inconsistency (eventual consistency). Delay can range from seconds to hours.
3. **Quorum-based** — read and write operations must access a quorum (majority) of copies. Ensures consistency without requiring all copies.
### 5.3 Distributed Deadlock
 
Deadlock detection is harder in a distributed setting because a wait-for graph must include transactions across all sites. A **global wait-for graph** must be constructed:
 
- **Centralised deadlock detection** — one site collects all local WFGs and constructs a global WFG. Single point of failure and bottleneck.
- **Distributed deadlock detection** — each site maintains a local WFG. Local WFGs are periodically transmitted and combined. Phantom deadlocks can occur if WFGs are out of sync.
- **Timeouts** — simplest practical approach. If a transaction waits longer than a threshold, assume deadlock and abort. May abort transactions that aren't actually deadlocked.
### 5.4 Two-Phase Commit (2PC)
 
2PC ensures atomicity of a global transaction — all subtransactions commit or all abort. One site acts as the **coordinator**; the others are **participants**.
 
#### Phase 1: Voting Phase
1. Coordinator writes **"begin_commit"** to its log
2. Coordinator sends **VOTE-REQUEST** (or PREPARE) to all participants
3. Each participant decides if it can commit its local subtransaction:
   - If yes: writes **"vote-commit"** to its log, sends **VOTE-COMMIT** (YES) to coordinator
   - If no: writes **"vote-abort"** to its log, sends **VOTE-ABORT** (NO) to coordinator, and can abort immediately
#### Phase 2: Decision Phase
4. Coordinator collects all votes:
   - **If ALL participants voted COMMIT:** coordinator writes **"global-commit"** to its log, sends **GLOBAL-COMMIT** to all participants
   - **If ANY participant voted ABORT:** coordinator writes **"global-abort"** to its log, sends **GLOBAL-ABORT** to all participants
5. Each participant receives the decision:
   - On GLOBAL-COMMIT: writes "commit" to log, commits locally, sends **ACK** to coordinator
   - On GLOBAL-ABORT: writes "abort" to log, rolls back locally, sends **ACK** to coordinator
6. Once coordinator receives all ACKs, it writes **"end_transaction"** to its log
#### 2PC Recovery — Coordinator Failures
 
| Last log entry | Recovery action |
|---|---|
| No record of transaction | Start abort (transaction hadn't begun voting) |
| **begin_commit** | Restart voting phase — resend VOTE-REQUEST to all participants |
| **global-commit** | Resend GLOBAL-COMMIT to all participants that haven't acknowledged |
| **global-abort** | Resend GLOBAL-ABORT to all participants that haven't acknowledged |
| **end_transaction** | No action needed — transaction completed |
 
#### 2PC Recovery — Participant Failures
 
| Last log entry | Recovery action |
|---|---|
| No record of transaction | Must have voted NO or not voted yet — abort locally |
| **vote-commit** | Participant voted YES but doesn't know the outcome — must ask the coordinator for the decision. This is the **uncertainty period**. |
| **vote-abort** | Participant already voted NO — abort locally |
| **commit** | Already committed — no action needed (redo if necessary) |
| **abort** | Already aborted — no action needed |
 
#### 2PC as a Blocking Protocol
 
2PC is a **blocking protocol**: if the coordinator fails after sending VOTE-REQUEST but before sending the decision, participants who voted YES are stuck in an **uncertain state**. They cannot commit (the coordinator might have decided abort) or abort (the coordinator might have decided commit). They must wait until the coordinator recovers.
 
During the uncertainty period, participants hold their locks, blocking other transactions. This is the main weakness of 2PC.
 
#### Three-Phase Commit (3PC)
 
3PC adds an extra phase to eliminate the blocking problem. After collecting all YES votes, the coordinator sends a **PRE-COMMIT** message before sending the final COMMIT. This ensures that if the coordinator fails, participants know that no abort can have been issued after the pre-commit phase, so they can safely commit.
 
3PC is non-blocking but more expensive (extra round of messages) and harder to implement. It assumes no network partitions. In practice, **2PC is used far more commonly** because network partition failures are rare and the extra overhead of 3PC is usually not justified.
 
### 5.5 Recovery with Logging in Distributed Systems
 
Each site maintains its own **local log file**. The coordinator maintains a separate log tracking the 2PC protocol. The **write-ahead log protocol** still applies: log records must be written before the corresponding database changes.
 
For the coordinator's log:
- begin_commit, global-commit or global-abort, end_transaction
For each participant's log:
- vote-commit or vote-abort, commit or abort
On recovery, the recovery manager at each site examines its local log. Committed transactions are redone, active (uncommitted) transactions are undone. If a participant finds a vote-commit without a subsequent commit/abort, it contacts the coordinator to learn the outcome.
 
### 5.6 DRDA Transaction Classification
 
IBM's Distributed Relational Database Architecture defines four levels:
 
1. **Remote request** — single SQL statement sent to a single remote site
2. **Remote unit of work** — all SQL statements in a transaction sent to a single remote site; local site decides commit/rollback
3. **Distributed unit of work** — SQL statements in a transaction sent to multiple remote sites, but each statement executes at only one site
4. **Distributed request** — SQL statements can access data from multiple sites (e.g. a join across sites)
---
 
## 6. Case Study: Oracle DDBMS
 
### 6.1 Oracle's Approach to Distribution
 
Oracle does **not** support automatic fragmentation of tables across sites. Instead, it uses **database links** — named connection definitions that allow SQL statements at one site to reference tables at another site using the syntax `table@dblink`.
 
This provides **location transparency** when combined with **synonyms**: a local synonym can be created for a remote table, hiding the database link from application code:
```sql
CREATE SYNONYM Staff FOR Staff@london_branch;
```
 
### 6.2 Types of Distributed Transactions in Oracle
 
Mapping to the DRDA classification:
 
- **Remote SQL** — a single SQL statement that references a remote table (via a database link). Equivalent to a remote request.
- **Distributed SQL** — a single SQL statement that references tables at multiple sites (e.g. a join across sites). Equivalent to a distributed request.
- **Remote transaction** — a transaction containing multiple SQL statements, all targeting one remote site.
- **Distributed transaction** — a transaction containing SQL statements that reference tables at multiple sites.
### 6.3 Oracle's Concurrency Control
 
Oracle uses a **multiversion read consistency** protocol rather than the standard 2PL described in the textbook:
 
- Oracle maintains **undo segments** storing before-images of modified data
- A **System Change Number (SCN)** provides logical timestamps for operation ordering
- Read operations **never block** write operations — readers see a consistent snapshot as of their statement or transaction start time
- Two isolation levels: READ COMMITTED (default, statement-level) and SERIALIZABLE (transaction-level)
- Row-level locking is used; lock information is stored within the data block itself (not in a separate lock table)
- Oracle **never escalates** locks (from row to table), avoiding the reduced concurrency and deadlock risks of escalation
- Oracle automatically detects deadlocks and resolves them by rolling back one of the involved statements
### 6.4 Oracle's 2PC Support
 
Oracle fully supports the **two-phase commit protocol** for distributed transactions. When a distributed transaction spans multiple database links:
 
- Oracle automatically designates one site as the **commit point site** (coordinator)
- The 2PC protocol executes transparently — the application simply issues COMMIT
- If a failure occurs during 2PC, Oracle's recovery process (RECO background process) automatically resolves in-doubt transactions when connectivity is restored
### 6.5 Integrity Constraints and Triggers
 
Because Oracle doesn't fragment tables, referential integrity across sites must be enforced at the application level, typically using **database triggers**. Standard declarative foreign key constraints cannot reference tables at remote sites.
 
### 6.6 Heterogeneous Gateways
 
Oracle provides **Heterogeneous Services** (and specific gateways like Oracle Transparent Gateway for SQL Server, DB2, etc.) that allow Oracle to access non-Oracle databases through database links. This gives Oracle-based systems a degree of **DBMS transparency** for heterogeneous environments, though with limitations on transaction support and available SQL features.
 
### 6.7 Oracle Recovery
 
- **Instance recovery** — on restart after a crash, Oracle uses redo logs to roll forward committed changes and roll back uncommitted ones
- **Checkpoints** — Oracle writes modified buffers to disk at configurable intervals
- **Point-in-time recovery** — restore datafiles from backups and apply redo logs up to a specific time or SCN
- **Standby database** — a standby database at a remote location receives and applies redo logs for failover
- **Flashback Technology** — rewind the database or individual tables to a past point without traditional backup/restore
---
 
## Exam-Relevant Quick Reference
 
### Key Formulas
 
**Communication Time** = C₀ + (number_of_bits / transmission_rate)
 
When sending N records individually: N × [C₀ + (bits_per_record / rate)]
When sending N records in bulk: C₀ + (N × bits_per_record / rate)
 
Bulk is almost always better because you pay C₀ only once.
 
### Fragmentation at a Glance
 
| Type | Operator | Splits by | Reconstruction |
|---|---|---|---|
| Horizontal | σ (Selection) | Tuples (rows) | Union (∪) |
| Vertical | π (Projection) | Attributes (columns) | Natural Join (⋈) |
| Mixed | σ + π | Both | Union + Natural Join |
| Derived | ⋉ (Semijoin) | Tuples, based on parent | Union |
 
### 2PC Decision Table
 
| All vote YES? | Decision |
|---|---|
| Yes | GLOBAL-COMMIT |
| Any NO | GLOBAL-ABORT |
 
### Answering "Two Strategies" Questions (Exam Q4 pattern)
 
These appear frequently. The two strategies to compare are usually:
 
**Strategy A — Centralise then compute:** Ship all relevant data to one site and execute the query there. High data transfer, but one round of communication.
 
**Strategy B — Compute locally then combine:** Each site performs local computation (partial aggregates, local selections), then sends only the small results to a coordinating site for final combination. Lower data transfer.
 
Strategy B is almost always more efficient because it reduces data volume before transmission. When arguing, discuss: (1) volume of data transmitted, (2) number of messages, (3) parallelism (local computations happen concurrently).
