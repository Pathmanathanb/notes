# SRE Mock Interview — Set 2
*Prepared for: Pathmanathan Boominathan | Experience: 6.5 years at PayPal*

---

## SECTION 1: Networking & Infrastructure

### Q1: What happens when you type a URL in a browser? (End to end)

**Answer:**

```
1. Browser checks local cache for DNS record
2. DNS resolution:
   - Check /etc/hosts → Local DNS cache → Recursive DNS resolver
   - → Root nameserver → TLD nameserver → Authoritative nameserver
   - Returns IP address
3. TCP 3-way handshake (SYN → SYN-ACK → ACK)
4. TLS handshake (if HTTPS):
   - Client hello → Server sends certificate
   - Client verifies cert → Session key exchanged
5. HTTP request sent
6. Server processes request → Response sent back
7. Browser renders HTML/CSS/JS
```

> At PayPal, understanding this flow was critical when debugging SSL cert issues causing payment page failures. We traced the issue to an expired intermediate cert in our chain — invisible to most tools but caught via `openssl s_client`.

---

### Q2: What is the difference between TCP and UDP? When would you use each?

**Answer:**

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented | Connectionless |
| Reliability | Guaranteed delivery | No guarantee |
| Order | In-order delivery | No ordering |
| Speed | Slower (overhead) | Faster |
| Use cases | HTTP, payment APIs, databases | DNS, video streaming, gaming |

> For payment systems at PayPal, we always use TCP — data integrity is non-negotiable. UDP is acceptable for internal metrics collection where losing a few data points doesn't matter.

---

### Q3: What is a load balancer? Difference between L4 and L7?

**Answer:**

A load balancer distributes traffic across multiple backend servers.

| | L4 (Transport Layer) | L7 (Application Layer) |
|--|----------------------|------------------------|
| Works at | TCP/UDP level | HTTP/HTTPS level |
| Routing based on | IP + Port | URL path, headers, cookies |
| Speed | Faster | Slightly slower (inspects payload) |
| Use case | Raw TCP traffic | Smart HTTP routing |

**L7 can do:**
- Route `/api/payments` to payment service
- Route `/api/accounts` to accounts service
- Sticky sessions (route same user to same server)
- SSL termination
- Canary deployments (route X% to new version)

> At PayPal, we used L7 load balancers to route traffic based on API paths and to handle canary deployments by sending 5% of traffic to new instances.

---

## SECTION 2: Database & Storage

### Q4: What is database replication? What is the difference between sync and async replication?

**Answer:**

Replication = copying data from primary DB to one or more replicas.

**Synchronous Replication:**
- Primary waits for replica to confirm write before acknowledging client
- Guarantees zero data loss
- Slower (waits for network round trip to replica)

**Asynchronous Replication:**
- Primary acknowledges client immediately, replicates in background
- Faster but risk of data loss if primary crashes before replica syncs
- Replica lag can cause stale reads

> At PayPal, we used synchronous replication for transaction records (zero data loss tolerance) and async replication for reporting replicas where slight staleness was acceptable.

---

### Q5: What is database connection pooling? Why does it matter?

**Answer:**

Creating a new DB connection for every request is expensive (TCP handshake, auth, memory allocation). Connection pooling maintains a pool of pre-opened connections that are reused.

**Without pooling:**
```
Request → Open connection → Query → Close connection (slow, expensive)
```

**With pooling:**
```
Request → Borrow connection from pool → Query → Return to pool (fast)
```

**Problems without connection pooling:**
- DB gets overwhelmed with connection requests under load
- Latency spikes during high traffic
- Can exhaust DB connection limits (e.g., PostgreSQL default: 100 connections)

> At PayPal, we had a service spiking connection counts to 5x normal during peak traffic. Root cause was a code change that accidentally bypassed the connection pool. Fixing it dropped DB CPU by 40%.

---

### Q6: What is the difference between SQL and NoSQL? When would you choose each?

**Answer:**

| | SQL (Relational) | NoSQL |
|--|-----------------|-------|
| Schema | Fixed schema | Flexible/schemaless |
| Scaling | Vertical (mostly) | Horizontal |
| Consistency | Strong (ACID) | Eventual (usually) |
| Transactions | Full ACID support | Limited (varies by DB) |
| Examples | MySQL, PostgreSQL | Cassandra, MongoDB, DynamoDB |

**Choose SQL when:**
- Data is structured and relational
- You need ACID transactions (e.g., payments, banking)
- Complex queries and joins are needed

**Choose NoSQL when:**
- High write throughput needed
- Schema changes frequently
- Massive scale (billions of records)
- Simple key-value or document lookups

> At PayPal, transaction records use relational DB (ACID required for payment integrity). User session data uses NoSQL (high throughput, simple lookups, short TTL).

---

## SECTION 3: Kubernetes Deep Dive

### Q7: What is the difference between liveness and readiness probes?

**Answer:**

| | Liveness Probe | Readiness Probe |
|--|----------------|-----------------|
| Purpose | Is the app alive? | Is the app ready to serve traffic? |
| Action on failure | Restart the container | Remove pod from load balancer |
| Use case | Detect deadlocks, hung processes | Startup time, temporary overload |

**Example configuration:**
```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

> At PayPal, we had a service that was "alive" (liveness OK) but couldn't serve traffic because its DB connection was broken. Adding a readiness probe that checked DB connectivity meant Kubernetes automatically pulled it from the load balancer until it recovered — zero user impact.

---

### Q8: What is a Kubernetes DaemonSet vs Deployment?

**Answer:**

| | Deployment | DaemonSet |
|--|------------|-----------|
| Replicas | You define how many | One pod per node (automatically) |
| Use case | Application workloads | Node-level agents |
| Scaling | Manual or HPA | Scales with nodes automatically |
| Examples | Web services, APIs, microservices | Log collectors, monitoring agents, security tools |

> At PayPal, our Datadog agent runs as a DaemonSet — every node automatically gets one agent for metrics collection. Our payment API runs as a Deployment with HPA scaling based on CPU and custom metrics.

---

### Q9: How does Kubernetes rolling update work? How do you roll back?

**Answer:**

**Rolling update process:**
```
Old: [v1][v1][v1][v1]
Step 1: [v2][v1][v1][v1]  ← new pod comes up, old pod removed
Step 2: [v2][v2][v1][v1]
Step 3: [v2][v2][v2][v1]
Step 4: [v2][v2][v2][v2]  ← complete
```

Kubernetes waits for new pod to pass readiness probe before removing old one.

**Key settings:**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1   # max pods that can be down at once
    maxSurge: 1         # max extra pods allowed during update
```

**Rollback commands:**
```bash
# Immediately rollback to previous version
kubectl rollout undo deployment/<name>

# Rollback to specific revision
kubectl rollout undo deployment/<name> --to-revision=3

# Check rollout history
kubectl rollout history deployment/<name>

# Monitor rollout status
kubectl rollout status deployment/<name>
```

> At PayPal, we had a bad deployment causing elevated errors. I ran `kubectl rollout undo` and traffic was back to normal within 90 seconds — well within our MTTR SLO.

---

## SECTION 4: Security & Reliability

### Q10: What is the principle of least privilege? How does it apply to SRE?

**Answer:**

Every system, service, or user should have only the minimum permissions needed to do their job — nothing more.

**SRE applications:**
- Service accounts should only access their own secrets/configs, not others
- Pods should not run as root
- Read-only DB access for reporting services (not write access)
- SRE production access should be audited and time-limited
- RBAC in Kubernetes — restrict which namespaces/resources each team can access

**Kubernetes security context example:**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
```

> At PayPal, we ran a security audit and found several services with admin-level DB access but only needing read access. We scoped down permissions — significantly reducing blast radius if any service was compromised.

---

### Q11: What is mTLS and why is it important in microservices?

**Answer:**

**TLS** = server proves identity to client (one-way authentication)
**mTLS (Mutual TLS)** = both client AND server prove identity to each other (two-way authentication)

**Why it matters in microservices:**
- Prevents unauthorized services from calling internal APIs
- Encrypts service-to-service traffic (even inside the cluster)
- Ensures only legitimate services can communicate

**How it works:**
```
Service A → presents its certificate → Service B verifies
Service B → presents its certificate → Service A verifies
Both certificates validated against trusted Certificate Authority
Encrypted communication channel established
```

**Tools:** Istio, Linkerd (service mesh handles mTLS automatically without app code changes)

> At PayPal, all internal service-to-service calls use mTLS enforced by our service mesh. This ensures even if an attacker gets inside the network, they can't impersonate a service without a valid certificate.

---

## SECTION 5: Performance & Scalability

### Q12: What is caching? What are the cache invalidation strategies?

**Answer:**

Caching stores frequently accessed data in fast storage (memory) to reduce latency and DB load.

**Caching layers:**
```
Client → CDN Cache → API Cache → Application Cache (Redis) → Database
```

**Cache invalidation strategies:**

| Strategy | How it works | Risk |
|----------|-------------|------|
| TTL (Time to Live) | Cache expires after X seconds | Stale data for TTL duration |
| Write-through | Update cache on every write | Slightly slower writes |
| Write-behind | Write to cache, async to DB | Risk of data loss |
| Cache-aside | App checks cache; miss → load from DB | Cache stampede on cold start |
| Event-driven | DB change triggers cache invalidation | Complex to implement |

**Cache stampede:**
- Cache expires → all requests simultaneously hit DB → DB overloaded
- Fix: Add jitter to TTL, use probabilistic early expiration, background refresh jobs

> At PayPal, we cached exchange rate data with a 60-second TTL. To prevent stampede, we used a background refresh job that refreshed the cache before expiry — so users always hit warm cache.

---

### Q13: How would you handle a traffic spike 10x normal load?

**Answer:**

**Before the spike (proactive):**
- Load test to know your breaking point before it happens
- Set up auto-scaling (HPA in Kubernetes based on CPU/custom metrics)
- Pre-warm caches and connection pools
- Pre-scale if spike is predictable (e.g., scheduled sale event)

**During the spike (reactive):**

```
1. Load shedding:
   - Rate limiting (reject excess requests with 429 Too Many Requests)
   - Circuit breakers to protect downstream services from cascading failure

2. Scaling out:
   - Horizontal scaling (add more pods/instances)
   - Scale DB read replicas to handle increased read traffic

3. Degraded mode:
   - Disable non-critical features via feature flags
   - Serve cached/stale responses where acceptable
   - Queue non-urgent writes, process asynchronously

4. Monitor in real-time:
   - Watch error rates, latency, DB connections, queue depths in Datadog
```

> At PayPal, before major sales events, we ran capacity planning exercises — projected 3x normal peak, pre-scaled to 2x, enabled auto-scale for the remaining buffer. We also had feature flags to disable non-critical features (e.g., product recommendations) to protect checkout during extreme load.

---

## SECTION 6: Advanced Scenarios

### Q14: Your deployment just caused a 10% increase in error rate. What do you do?

**Answer:**

```
Step 1 — IMMEDIATELY assess (first 60 seconds):
   - Is this getting worse or stable?
   - How many users are affected?
   - Is this above error budget threshold (P1/P2)?

Step 2 — Rollback decision (within 2-3 minutes):
   - If new deployment is confirmed cause → rollback IMMEDIATELY
   - Do NOT wait to investigate if users are impacted NOW
   - Mitigate first, understand later

Step 3 — Execute rollback:
   kubectl rollout undo deployment/<name>
   # OR redeploy previous artifact via CI/CD pipeline

Step 4 — Verify recovery:
   - Watch error rate drop in Datadog
   - Confirm rollback successful via health checks

Step 5 — Open incident if P1/P2:
   - Notify stakeholders
   - Post in incident channel with current status

Step 6 — Post-incident:
   - Investigate root cause in staging environment
   - Add test coverage for the failure scenario
   - Update deployment checklist to prevent recurrence
```

> At PayPal, my rule was: if a deployment causes >1% increase in errors, rollback first and ask questions later. We had a 2-minute rollback SLA for production deployments.

---

### Q15: How would you debug high memory usage in a production container?

**Answer:**

```bash
# 1. Check current memory usage
kubectl top pod <pod-name> -n <namespace>

# 2. Check if it's being OOMKilled
kubectl describe pod <pod-name> | grep -i oom
kubectl describe pod <pod-name> | grep -A5 "Last State"

# 3. Check memory trends over time in Datadog
# Look for: linear growth (leak) vs sawtooth pattern (normal GC)

# 4. Get heap dump (Java example)
kubectl exec -it <pod-name> -- jmap -heap <pid>
kubectl exec -it <pod-name> -- jmap -dump:format=b,file=/tmp/heap.hprof <pid>

# 5. Profile the application:
#    - Java: JVisualVM, Eclipse MAT, async-profiler
#    - Python: memory_profiler, tracemalloc, objgraph
#    - Go: pprof (net/http/pprof)
#    - Node.js: v8-profiler, heapdump

# 6. Common causes to check:
#    - Unbounded caches (no eviction policy set)
#    - Connection/resource leaks (not closing streams/connections)
#    - Large object accumulation in queues
#    - Memory fragmentation
#    - Thread-local storage accumulation
```

**Fix approaches:**
- Add eviction policy (max-size + TTL) to caches
- Fix resource leaks (use try-with-resources in Java)
- Tune JVM heap settings (-Xmx, GC algorithm)
- Increase memory limit as temporary mitigation while root cause is fixed

> At PayPal, we had a service growing 50MB/hour. Using heap dump analysis, I found a cache storing transaction objects without an eviction policy — growing unbounded. Adding max-size + TTL to the cache fixed the leak completely.

---

### Q16: What is the difference between a blue/green deployment and a canary deployment?

**Answer:**

**Blue/Green Deployment:**
```
Blue (current v1) → serving 100% traffic
Green (new v2)    → deployed, fully tested, idle

Switch:
Load balancer flips 100% traffic to Green instantly
Blue stays up as immediate rollback option (just flip back)
```

**Canary Deployment:**
```
v1 → 95% traffic
v2 → 5% traffic (canary)

Monitor for errors/latency on v2 for X minutes
Gradually increase: 5% → 25% → 50% → 100%
Roll back immediately if metrics degrade at any stage
```

| | Blue/Green | Canary |
|--|------------|--------|
| Traffic switch | All at once | Gradual |
| Rollback | Instant (flip back) | Gradual reduction |
| Risk | Medium (all users at once) | Low (small blast radius) |
| Infrastructure | 2x resources needed | Minimal extra resources |
| Best for | Well-tested, confident releases | High-risk or uncertain changes |

> At PayPal, we used canary deployments for payment flow changes — too risky to blue/green. We started with 1% of traffic, monitored transaction success rate for 30 minutes, then gradually rolled out. Caught 2 critical bugs in canary phase that would have impacted millions of transactions.

---

## SECTION 7: Advanced Rapid Fire Questions

| Question | Answer |
|----------|--------|
| What is a zombie process? | A process that finished execution but still has an entry in the process table because its parent hasn't read its exit status. Fix: fix the parent to call `wait()`, or kill the parent so init (PID 1) cleans it up |
| What is swap memory? When is it a problem? | Disk space used as overflow when RAM is full. Problem: disk is 100-1000x slower than RAM. High swap = memory-starved system, application latency will spike dramatically |
| What is a race condition? | When two processes access shared data simultaneously and the outcome depends on execution order. Fixed by locks, mutexes, or atomic operations |
| What is eventual consistency? | Data changes propagate to all nodes over time, but reads may return stale data temporarily. Acceptable for non-critical data (e.g., product catalog). NOT acceptable for payments (need strong consistency) |
| What is a service mesh? | Infrastructure layer handling service-to-service communication — provides mTLS, load balancing, observability, and traffic control without changing application code. Examples: Istio, Linkerd |
| What is RTO? | Recovery Time Objective — how quickly the system must be back online after a failure. Example: "restore within 1 hour" |
| What is RPO? | Recovery Point Objective — how much data loss is acceptable. Example: "max 5 minutes of transaction data loss" |
| What is rate limiting? How is it implemented? | Restricting how many requests a client can make in a time window. Responses with 429 Too Many Requests. Implemented via token bucket, leaky bucket, or sliding window algorithms |
| What is the CAP theorem? | A distributed system can only guarantee 2 of 3: Consistency (every read gets latest write), Availability (every request gets a response), Partition tolerance (works despite network splits). Since partitions happen in practice, you choose C or A |
| What is a memory leak? | Memory that is allocated but never freed/released, causing gradual memory growth over time until the process crashes (OOM) |
| What is connection draining? | Gracefully removing a backend from load balancer rotation — allows in-flight requests to complete before stopping the instance. Prevents dropped connections during deployments |
| What is backpressure? | Mechanism to slow down producers when consumers can't keep up. Prevents queue overflow and system overload. Implemented via bounded queues, rate limiting, or explicit signals |

---

## SECTION 8: System Design — Advanced

### Q17: How would you design a real-time alerting system for payment failures?

**Answer:**

```
Payment Service → emit events to Kafka topic (payment_results)
                      ↓
              Stream Processor (Flink/Spark Streaming)
              - Calculate error rate per minute per region
              - Compare against SLO thresholds
                      ↓
              Alert Engine
              - If error_rate > 1% for 3 consecutive minutes → fire alert
              - Deduplication (don't fire same alert multiple times)
              - Routing (P1 → PagerDuty, P3 → Slack)
                      ↓
              On-call SRE notified with:
              - Error rate
              - Affected region
              - Runbook link
              - Datadog dashboard link
```

**Key considerations:**
- Use streaming (not batch) for real-time detection
- Sliding window to reduce false positives
- Alert deduplication to avoid storm
- Auto-resolve when condition clears
- Correlation with recent deployments

---

### Q18: How would you implement a deployment pipeline with safety gates?

**Answer:**

```
Code merged to main
       ↓
Unit Tests + Linting (< 5 min)
       ↓
Build Docker image + push to registry
       ↓
Deploy to DEV → Run smoke tests
       ↓
Deploy to STAGING → Run integration tests + load tests
       ↓
[Safety Gate 1] — All tests pass? Error rate normal?
       ↓
Deploy to PRODUCTION — Canary (1%)
       ↓
[Safety Gate 2] — Monitor 15 min: error rate, latency, success rate
       ↓
Canary 10% → Monitor 15 min
       ↓
Canary 50% → Monitor 15 min
       ↓
Full rollout 100%
       ↓
[Safety Gate 3] — Monitor 30 min post full rollout
```

**Automatic rollback triggers:**
- Error rate increases > 1% vs baseline
- p99 latency increases > 20% vs baseline
- Any P0 alert fires

> At PayPal, we built automated rollback into the pipeline — if Datadog metrics degraded during canary, the pipeline would auto-rollback without human intervention.

---

## SECTION 9: Behavioral — Advanced Scenarios

### Q19: Tell me about a time you improved system reliability significantly.

**STAR Framework Answer:**

**Situation:** Our payment service had MTTR averaging 45 minutes. On-call engineers spent significant time manually diagnosing issues because runbooks were outdated and alerts had no context.

**Task:** As SRE lead, I was asked to improve MTTR to under 20 minutes.

**Action:**
- Audited all P1/P2 incidents from the past 6 months — identified top 5 recurring failure patterns
- Updated runbooks for each pattern with step-by-step diagnosis and fix procedures
- Added runbook links directly to Datadog alerts
- Built a diagnostic dashboard showing all key signals in one view
- Conducted 3 game days (simulated failures) to train the on-call rotation
- Implemented automated diagnosis scripts for the top 3 failure patterns

**Result:**
- MTTR reduced from 45 minutes to 17 minutes (62% improvement) in 3 months
- On-call satisfaction score improved significantly
- Two incidents were auto-mitigated before a human even paged

---

### Q20: Tell me about a time you had to make a difficult decision under pressure.

**STAR Framework Answer:**

**Situation:** During a production incident, our payment service was degraded. I identified that a rollback would fix it, but the last stable version was 3 days old — rolling back would revert a critical security patch.

**Task:** Decide: rollback and risk security exposure, or keep degraded service running and find another fix.

**Action:**
- Quickly assessed security risk vs customer impact
- Called in the security team lead for a 5-minute consultation
- Security team confirmed the risk window was low (specific conditions not present)
- Made the call to rollback to restore service, simultaneously creating an urgent ticket for the security team to fast-track a re-apply of the patch
- Documented the decision and rationale in the incident channel

**Result:**
- Payment service restored in 8 minutes
- Security patch re-applied in a targeted hotfix within 2 hours
- Wrote a postmortem recommending more granular deployment tagging so rollbacks don't need to revert multiple changes at once

---

## Key Interview Tips

| Tip | Detail |
|-----|--------|
| Quantify everything | "Reduced MTTR by 40%" is far stronger than "improved response time" |
| Lead with impact | Start your answer with the outcome, then explain how |
| Think out loud | Show your reasoning — interviewers want to see how you think, not just final answers |
| Use real PayPal examples | 6.5 years = rich material. Every answer should have a real story |
| Admit gaps | "I haven't used X but here's how I'd approach it..." shows intellectual honesty |
| Ask clarifying questions | In system design, always clarify requirements before designing |
| Structure answers | Problem → Approach → Result (for technical). STAR for behavioral |

---

## Recommended Study Resources

| Resource | Link |
|----------|------|
| Google SRE Book | https://sre.google/sre-book/table-of-contents/ (free online) |
| Site Reliability Workbook | https://sre.google/workbook/table-of-contents/ (free online) |
| Kubernetes Official Docs | https://kubernetes.io/docs/ |
| Datadog Learning Center | https://learn.datadoghq.com/ |
| Designing Data-Intensive Applications (book) | By Martin Kleppmann — excellent for distributed systems |
| LeetCode | Medium-level for scripting/systems problems |

---

*Good luck! Your 6.5 years at PayPal handling real production incidents at massive scale is your greatest advantage. Own it!*
