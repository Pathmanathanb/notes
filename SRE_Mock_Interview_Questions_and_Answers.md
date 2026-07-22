# SRE Mock Interview Questions & Answers
*Prepared for: Pathmanathan Boominathan | Experience: 6.5 years at PayPal*

---

## SECTION 1: Core SRE Concepts

### Q1: What is the difference between SLI, SLO, and SLA? Give an example.

**Answer:**
- **SLI** is the actual metric you measure. Example: "95th percentile latency of checkout API"
- **SLO** is your internal target for that metric. Example: "p95 latency must be < 500ms, 99.9% of the time"
- **SLA** is the contract with customers. Example: "We guarantee 99.5% availability or customers get a refund"

> At PayPal, we defined SLOs for our payment APIs — things like success rate, latency, and error rate. We tracked these in Datadog and used them to gate deployments when the error budget was running low.

---

### Q2: What is an error budget and how do you use it?

**Answer:**
Error budget = `1 - SLO`

If our SLO is 99.9% availability per month:
- Total minutes in a month = ~43,200
- Allowed downtime = 0.1% x 43,200 = ~43 minutes

**How we use it:**
- If budget is healthy → teams can deploy freely
- If budget is 50% consumed → increase caution, more testing
- If budget is exhausted → freeze non-critical deployments, focus on reliability

> At PayPal, during peak seasons like Black Friday, we froze risky changes 2 weeks before the event when the error budget dropped below 20%.

---

### Q3: What is toil? How did you reduce it in your team?

**Answer:**
Toil is manual, repetitive, operational work that scales with traffic but has no lasting engineering value.

**Examples of toil:**
- Manually restarting pods when they crash
- Manually checking logs to confirm deployments
- Copy-pasting configs across environments

**How to reduce it:**
- Automate runbooks with scripts
- Build self-healing systems
- Use CI/CD to eliminate manual deployments

> At PayPal, I wrote a script to detect OOM conditions and auto-restart a crashing service, then worked with the dev team to fix the memory leak. Saved ~2 hours of on-call toil per week.

---

## SECTION 2: Incident Management

### Q4: Walk me through how you handle a P1 incident.

**Answer:**

| Step | Action |
|------|--------|
| 1. Detect | Alert fires in Datadog/PagerDuty |
| 2. Acknowledge | On-call SRE acknowledges within SLA (e.g., 5 min) |
| 3. Assess severity | Check blast radius — how many users affected? |
| 4. Communicate | Post in incident channel, notify stakeholders |
| 5. Mitigate | Rollback, reroute traffic, scale up |
| 6. Resolve | Root cause fixed or stable workaround in place |
| 7. Postmortem | Blameless review within 48-72 hours |

**Key metrics:**
- MTTD — Mean Time to Detect
- MTTR — Mean Time to Resolve

> At PayPal, I identified a payment gateway latency spike via Datadog APM, rerouted traffic to a backup region within 8 minutes — reducing MTTR from a potential 45 minutes to under 15.

---

### Q5: What makes a good postmortem?

**Answer:**

A good postmortem is:
1. **Blameless** — focus on systems and processes, not individuals
2. **Timeline-driven** — exact chronological record of events
3. **Root cause focused** — use "5 Whys" technique
4. **Action-item driven** — every finding maps to an owner + deadline

**Structure:**
1. Summary (impact, duration, who was affected)
2. Timeline
3. Root Cause Analysis
4. Contributing Factors
5. Action Items (with DRI and due date)

> At PayPal, after a checkout outage, our postmortem revealed a config change bypassed staging validation. We automated mandatory staging sign-off in the CI/CD pipeline as a result.

---

## SECTION 3: Systems Design & Reliability

### Q6: How would you design a highly available payment processing system?

**Answer:**

```
Users → Global Load Balancer
           ↓
    Multi-region API Gateways (Active-Active)
           ↓
    Stateless Application Tier (Auto-scaled)
           ↓
    Distributed Cache (Redis Cluster)
           ↓
    Database (Primary + Read Replicas, cross-region failover)
           ↓
    Downstream Services (with circuit breakers)
```

**Key design decisions:**
- Active-Active multi-region — traffic served from 2+ regions simultaneously
- Circuit breakers — if downstream fails, fail fast instead of cascading
- Idempotent APIs — safe to retry without double-charging
- Queue-based processing — decouple components, absorb traffic spikes
- Health checks + auto-failover — remove unhealthy instances automatically

> At PayPal, we used weighted routing — 70% to primary, 30% to secondary — so failover was near-instant with zero cold start.

---

### Q7: What is a circuit breaker and why is it important?

**Answer:**

A circuit breaker prevents cascading failures by stopping calls to a failing service.

**States:**
- **Closed** — requests flow normally
- **Open** — service is failing; all requests fail fast
- **Half-Open** — allow a few test requests; if they succeed, close the circuit

> At PayPal, we used circuit breakers between our API layer and fraud detection service. When fraud detection was degraded, the circuit opened and we fell back to a lightweight rule-based check — keeping checkout functional.

---

## SECTION 4: Monitoring & Observability

### Q8: What is the difference between monitoring and observability?

**Answer:**

|   | Monitoring | Observability |
|---|------------|---------------|
| Focus | Known failure modes | Unknown unknowns |
| Approach | Pre-defined dashboards & alerts | Explore system state from outputs |
| Tools | Datadog alerts, thresholds | Distributed traces, structured logs |
| Question | "Is the system down?" | "Why is the system behaving this way?" |

**Three pillars of observability:**
- **Metrics** — what is the system doing? (Datadog, Prometheus)
- **Logs** — what happened? (CAL at PayPal, Splunk)
- **Traces** — where did this request spend time? (Datadog APM)

---

### Q9: What are the best practices for alerting?

**Answer:**

1. **Alert on symptoms, not causes**
   - Bad: "CPU > 80%"
   - Good: "Error rate > 1% for 5 minutes"
2. **Every alert must be actionable**
3. **Avoid alert fatigue** — too many alerts = engineers ignore them
4. **Use appropriate severity levels** (P0/P1: wake someone up, P3: next day ticket)
5. **Include runbook links** in every alert

> At PayPal, I ran an alert audit and eliminated duplicate/low-signal alerts, reducing noise by 60%. On-call quality improved significantly.

---

## SECTION 5: Networking & Infrastructure

### Q10: What happens when you type a URL in a browser?

**Answer:**

```
1. Browser checks local DNS cache
2. DNS resolution:
   /etc/hosts → Local DNS → Recursive resolver
   → Root nameserver → TLD → Authoritative nameserver → IP returned
3. TCP 3-way handshake (SYN → SYN-ACK → ACK)
4. TLS handshake (if HTTPS):
   Client hello → Server cert → Verify → Session key exchanged
5. HTTP request sent
6. Server processes request → Response returned
7. Browser renders HTML/CSS/JS
```

---

### Q11: What is the difference between TCP and UDP?

**Answer:**

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented | Connectionless |
| Reliability | Guaranteed delivery | No guarantee |
| Order | In-order delivery | No ordering |
| Speed | Slower (overhead) | Faster |
| Use cases | HTTP, payment APIs, DB | DNS, video streaming, gaming |

---

### Q12: What is a load balancer? Difference between L4 and L7?

**Answer:**

| | L4 (Transport Layer) | L7 (Application Layer) |
|--|----------------------|------------------------|
| Works at | TCP/UDP level | HTTP/HTTPS level |
| Routing based on | IP + Port | URL path, headers, cookies |
| Speed | Faster | Slightly slower |
| Use case | Raw TCP traffic | Smart HTTP routing, canary deploys |

---

## SECTION 6: Database & Storage

### Q13: What is database replication? Sync vs async?

**Answer:**

**Synchronous:**
- Primary waits for replica to confirm write before acknowledging client
- Zero data loss, slightly slower

**Asynchronous:**
- Primary acknowledges immediately, replicates in background
- Faster but risk of data loss if primary crashes before replication

> At PayPal, synchronous replication for transaction records, async for reporting replicas where slight staleness was acceptable.

---

### Q14: What is database connection pooling? Why does it matter?

**Answer:**

Maintains a pool of pre-opened connections that are reused instead of creating new ones per request.

**Without pooling:** Request → Open connection → Query → Close (slow, expensive)
**With pooling:** Request → Borrow from pool → Query → Return to pool (fast)

> At PayPal, a code change accidentally bypassed the connection pool causing 5x connection spike. Fixing it dropped DB CPU by 40%.

---

### Q15: SQL vs NoSQL — when to choose each?

**Answer:**

| | SQL | NoSQL |
|--|-----|-------|
| Schema | Fixed | Flexible |
| Scaling | Vertical | Horizontal |
| Consistency | Strong (ACID) | Eventual |
| Examples | MySQL, PostgreSQL | Cassandra, MongoDB, DynamoDB |

- **SQL:** Payments, banking, complex queries with joins
- **NoSQL:** High write throughput, simple lookups, massive scale, flexible schema

---

## SECTION 7: Kubernetes Deep Dive

### Q16: Liveness vs Readiness probes?

**Answer:**

| | Liveness Probe | Readiness Probe |
|--|----------------|-----------------|
| Purpose | Is the app alive? | Is the app ready to serve traffic? |
| Action on failure | Restart the container | Remove from load balancer |
| Use case | Detect deadlocks | Startup time, temporary overload |

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

### Q17: DaemonSet vs Deployment?

**Answer:**

| | Deployment | DaemonSet |
|--|------------|-----------|
| Replicas | You define how many | One pod per node |
| Use case | Application workloads | Node-level agents |
| Examples | Web services, APIs | Log collectors, monitoring agents |

---

### Q18: How does a Kubernetes rolling update work? How do you roll back?

**Answer:**

```
Old: [v1][v1][v1][v1]
Step 1: [v2][v1][v1][v1]
Step 2: [v2][v2][v1][v1]
Step 3: [v2][v2][v2][v1]
Step 4: [v2][v2][v2][v2] ← complete
```

```bash
# Rollback immediately
kubectl rollout undo deployment/<name>

# Rollback to specific revision
kubectl rollout undo deployment/<name> --to-revision=3

# Check history
kubectl rollout history deployment/<name>
```

> At PayPal, I ran `kubectl rollout undo` during a bad deployment — traffic restored within 90 seconds.

---

### Q19: A pod is in CrashLoopBackOff. What do you do?

**Answer:**

```bash
# 1. Check pod status
kubectl get pods -n <namespace>

# 2. Describe pod — look at Events section
kubectl describe pod <pod-name> -n <namespace>

# 3. Check logs of crashing container
kubectl logs <pod-name> -n <namespace> --previous
```

**Common causes:**
- OOMKilled → increase memory limit or fix memory leak
- Bad config → fix ConfigMap or Secret
- App crash → check logs, fix application bug
- Bad probe → fix liveness/readiness probe path/timing

---

## SECTION 8: Security & Reliability

### Q20: What is the principle of least privilege?

**Answer:**

Every system, service, or user should have only the minimum permissions needed — nothing more.

**SRE applications:**
- Service accounts only access their own secrets
- Pods should not run as root
- Read-only DB access for reporting services
- SRE prod access should be audited and time-limited

---

### Q21: What is mTLS and why is it important?

**Answer:**

**TLS** = server proves identity to client (one-way)
**mTLS** = both client AND server prove identity (two-way)

**Why it matters:**
- Prevents unauthorized services from calling internal APIs
- Encrypts service-to-service traffic
- Ensures only legitimate services can communicate

**Tools:** Istio, Linkerd (service mesh handles mTLS automatically)

---

## SECTION 9: Performance & Scalability

### Q22: What is caching? What are the cache invalidation strategies?

**Answer:**

**Cache invalidation strategies:**

| Strategy | How it works | Risk |
|----------|-------------|------|
| TTL | Cache expires after X seconds | Stale data for TTL duration |
| Write-through | Update cache on every write | Slightly slower writes |
| Write-behind | Write to cache, async to DB | Risk of data loss |
| Cache-aside | App checks cache; miss → load from DB | Cache stampede on cold start |
| Event-driven | DB change triggers cache invalidation | Complex to implement |

**Cache stampede fix:** Add jitter to TTL, use background refresh jobs.

---

### Q23: How would you handle a traffic spike 10x normal load?

**Answer:**

**Proactive:**
- Load test to know your breaking point
- Set up auto-scaling (HPA in Kubernetes)
- Pre-warm caches and connection pools
- Pre-scale if spike is predictable

**Reactive:**
1. Rate limiting (reject excess requests with 429)
2. Circuit breakers to protect downstream services
3. Horizontal scaling (add more pods/instances)
4. Degraded mode — disable non-critical features
5. Queue non-urgent writes

---

## SECTION 10: Advanced Scenarios

### Q24: Your deployment just caused a 10% increase in error rate. What do you do?

**Answer:**

```
1. Assess immediately:
   - Is this getting worse or stable?
   - How many users affected?
   - Is this above error budget threshold?

2. Rollback decision (within 2-3 min):
   - If new deployment is confirmed cause → rollback immediately
   - Don't wait to investigate if users are impacted NOW

3. Execute rollback:
   kubectl rollout undo deployment/<name>

4. Verify recovery:
   - Watch error rate drop in Datadog

5. Open incident if P1/P2

6. Post-incident:
   - Investigate root cause in staging
   - Add test coverage for the failure scenario
```

> At PayPal, rule was: if a deployment causes >1% increase in errors, rollback first and ask questions later. We had a 2-minute rollback SLA.

---

### Q25: How would you debug high memory usage in a production container?

**Answer:**

```bash
# 1. Check current memory usage
kubectl top pod <pod-name> -n <namespace>

# 2. Check if OOMKilled
kubectl describe pod <pod-name> | grep -i oom

# 3. Check for memory leak patterns:
#    - Memory grows continuously → likely leak
#    - Memory grows then GC drops it → normal (tune GC)
#    - Memory at limit → increase limit or fix leak

# 4. Profile the application:
#    - Java: JVisualVM, Eclipse MAT
#    - Python: memory_profiler, tracemalloc
#    - Go: pprof
```

**Common causes:**
- Unbounded caches (no eviction policy)
- Connection/resource leaks
- Large object accumulation in queues

---

### Q26: Blue/Green vs Canary Deployment?

**Answer:**

**Blue/Green:**
- Two identical environments
- Switch 100% traffic at once
- Easy rollback (flip back)
- Needs 2x infrastructure

**Canary:**
- Gradual traffic shift (1% → 5% → 25% → 100%)
- Monitor metrics at each step
- Low blast radius
- Minimal extra infrastructure

| | Blue/Green | Canary |
|--|------------|--------|
| Traffic switch | All at once | Gradual |
| Rollback | Instant | Gradual reduction |
| Risk | Medium | Low |
| Best for | Well-tested releases | High-risk changes |

> At PayPal, we used canary for payment flow changes — started at 1%, monitored for 30 minutes, caught 2 critical bugs before full rollout.

---

## SECTION 11: Rapid Fire Questions

| Question | Answer |
|----------|--------|
| Latency vs Throughput | Latency = time per request. Throughput = requests per second |
| What is p99 latency? | 99% of requests are faster than this value |
| Horizontal vs Vertical scaling | Vertical = bigger machine. Horizontal = more machines (preferred) |
| What is idempotency? | Same request produces same result regardless of how many times called |
| What is a deadlock? | Two processes waiting for each other to release a resource — both wait forever |
| What is a zombie process? | Finished process still in process table because parent hasn't read exit status |
| What is swap memory? | Disk used as overflow when RAM is full. High swap = severe latency |
| What is a race condition? | Two processes access shared data simultaneously; outcome depends on timing |
| What is eventual consistency? | Data converges to same value across nodes over time (may be stale temporarily) |
| What is a service mesh? | Infrastructure layer handling mTLS, load balancing, observability between services |
| RTO vs RPO | RTO = how quickly recover. RPO = how much data loss acceptable |
| What is rate limiting? | Restricting requests per time window. Returns 429 when exceeded |
| What is CAP theorem? | Distributed system can only guarantee 2 of: Consistency, Availability, Partition tolerance |

---

## SECTION 12: Behavioral Questions (STAR Method)

### Q27: Tell me about a major incident you led.

**STAR Framework:**
- **Situation:** What was the context?
- **Task:** What was your responsibility?
- **Action:** What specific steps did you take?
- **Result:** What was the measurable outcome?

**Sample Answer:**

> **Situation:** During peak traffic at PayPal, checkout success rate dropped from 99.8% to 94% — affecting millions of transactions.
>
> **Task:** As on-call SRE, restore service within 30-minute MTTR SLO.
>
> **Action:** Opened incident bridge, assigned IC/comms/tech lead roles. Used Datadog APM to trace failing requests — found errors on one payment processor. Identified a config change 20 min earlier with wrong timeout value (500ms instead of 5000ms). Coordinated rollback while updating stakeholders every 10 minutes.
>
> **Result:** Service restored in 22 minutes. Added automated config validation to CI/CD — prevented 2 similar incidents in the next 6 months.

---

### Q28: How do you handle disagreement with a developer team about reliability?

**Sample Answer:**

> **Situation:** A dev team wanted to ship a feature Friday before a major sales event. SRE analysis showed inadequate load testing.
>
> **Task:** Block release or find middle ground without damaging relationship.
>
> **Action:** Presented data showing projected failure rate. Proposed compromise: deploy behind feature flag, enable for 1% traffic, monitor 24 hours before full rollout. Set up monitoring and rollback plan together.
>
> **Result:** Feature launched safely at 1%, caught a connection pool exhaustion bug. Fixed before full rollout — zero customer impact. Dev team started involving SRE earlier in future releases.

---

### Q29: How do you prioritize during a major outage when multiple things are failing?

**Answer:**

Priority framework:
1. **Mitigate first, debug later** — restore service before root cause
2. **Blast radius first** — fix what affects most users first
3. **Quick wins first** — rollback (2 min) beats a fix (2 hours)
4. **Delegate** — assign roles: one investigates, one communicates, one executes

---

## Key Tips for the Interview

| Tip | Why it matters |
|-----|----------------|
| Think out loud | Shows reasoning, not just final answer |
| Use real examples | Proves practical experience |
| Quantify everything | "Reduced MTTR by 40%" beats "improved response time" |
| Ask clarifying questions | Shows maturity |
| Admit gaps honestly | "I haven't used X but I'd approach it by..." |
| Structure answers | Context → Approach → Result |

---

## Recommended Resources

- **Google SRE Book** — https://sre.google/sre-book/table-of-contents/ (free online)
- **The Site Reliability Workbook** — https://sre.google/workbook/table-of-contents/ (free online)
- **Kubernetes Docs** — https://kubernetes.io/docs/
- **Datadog Learning Center** — https://learn.datadoghq.com/
- **LeetCode** — Medium level for scripting/systems problems

---

*Good luck with your interview! Your 6.5 years at PayPal is your biggest advantage — quantify your impact and tell real stories.*
