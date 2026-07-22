# SRE Mock Interview — Set 1
*Prepared for: Pathmanathan Boominathan | Experience: 6.5 years at PayPal*

---

## SECTION 1: Core SRE Concepts

### Q1: What is the difference between SLI, SLO, and SLA? Give an example.

**Answer:**
- **SLI (Service Level Indicator)** is the actual metric you measure.
  - Example: "95th percentile latency of checkout API"
- **SLO (Service Level Objective)** is your internal target for that metric.
  - Example: "p95 latency must be < 500ms, 99.9% of the time"
- **SLA (Service Level Agreement)** is the contract with customers.
  - Example: "We guarantee 99.5% availability or customers get a refund"

> At PayPal, we defined SLOs for our payment APIs — things like success rate, latency, and error rate. We tracked these in Datadog and used them to gate deployments when the error budget was running low.

---

### Q2: What is an error budget and how do you use it?

**Answer:**
Error budget = `1 - SLO`

If our SLO is 99.9% availability per month:
- Total minutes in a month = ~43,200
- Allowed downtime = 0.1% × 43,200 = **~43 minutes**

**How we use it:**
- If budget is healthy → teams can deploy freely
- If budget is 50% consumed → increase caution, more testing
- If budget is exhausted → freeze non-critical deployments, focus on reliability

> At PayPal, during peak seasons like Black Friday, we were extra conservative with error budgets. We would freeze risky changes 2 weeks before the event.

---

### Q3: What is toil? How did you reduce it in your team?

**Answer:**
Toil is manual, repetitive, operational work that scales with traffic but has no lasting engineering value.

**Examples of toil:**
- Manually restarting pods when they crash
- Manually checking logs to confirm deployments
- Copy-pasting configs across environments

**How to reduce toil:**
- Automate runbooks with scripts
- Build self-healing systems (auto-restart, auto-scaling)
- Use CI/CD to eliminate manual deployments
- Create dashboards so humans don't need to dig through logs

> At PayPal, we had engineers manually restarting a service that crashed under high load every few weeks. I wrote a script to detect the OOM condition and auto-restart it, and then worked with the dev team to fix the memory leak. This saved ~2 hours of on-call toil per week.

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
- **MTTD** — Mean Time to Detect
- **MTTR** — Mean Time to Resolve
- **Blameless postmortem** — focus on systems, not people

> At PayPal, I was on-call when our payment gateway latency spiked to 10x normal. I quickly identified via Datadog APM that one downstream service was timing out. We rerouted traffic to the backup region within 8 minutes, reducing MTTR from what could have been 45 minutes to under 15.

---

### Q5: What makes a good postmortem?

**Answer:**

A good postmortem is:
1. **Blameless** — focus on systems and processes, not individuals
2. **Timeline-driven** — exact chronological record of events
3. **Root cause focused** — "5 Whys" technique to find real cause
4. **Action-item driven** — every finding maps to an owner + deadline

**Structure:**
1. Summary (impact, duration, who was affected)
2. Timeline
3. Root Cause Analysis
4. Contributing Factors
5. Action Items (with DRI and due date)

**Key principle:** "The system failed, not the person."

> At PayPal, after a major checkout outage, our postmortem revealed the root cause was a config change that bypassed our staging validation. The action item was to enforce mandatory staging sign-off before production — which we automated in the CI/CD pipeline.

---

## SECTION 3: Systems Design & Reliability

### Q6: How would you design a highly available payment processing system?

**Answer:**

Key principle: **redundancy at every layer**

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
- **Active-Active multi-region** — traffic served from 2+ regions simultaneously
- **Circuit breakers** — if downstream fails, fail fast instead of cascading
- **Idempotent APIs** — safe to retry without double-charging
- **Queue-based processing** — decouple components, absorb traffic spikes
- **Health checks + auto-failover** — remove unhealthy instances automatically

> At PayPal, our payment service was deployed across multiple data centers. We used weighted routing — 70% to primary, 30% to secondary — so failover was near-instant with zero cold start.

---

### Q7: What is a circuit breaker and why is it important?

**Answer:**

A circuit breaker prevents cascading failures by stopping calls to a failing service.

**States:**
- **Closed** — requests flow normally
- **Open** — service is failing; all requests fail fast (no waiting)
- **Half-Open** — allow a few test requests; if they succeed, close the circuit

**Why it matters:**
Without circuit breakers, one slow service can cause threads to pile up, memory to exhaust, and the entire system to crash.

> At PayPal, we used circuit breakers between our API layer and fraud detection service. When fraud detection was degraded, the circuit opened and we fell back to a lightweight rule-based check instead — keeping checkout functional for users.

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
- **Traces** — where did this request spend time? (Datadog APM, Jaeger)

> At PayPal, we used Datadog APM to trace payment transactions end-to-end. When latency spiked, I could pinpoint exactly which microservice and which database query was the bottleneck — without guessing.

---

### Q9: What are the best practices for alerting?

**Answer:**

1. **Alert on symptoms, not causes**
   - Bad: "CPU > 80%"
   - Good: "Error rate > 1% for 5 minutes"
2. **Every alert must be actionable**
   - If an engineer can't do anything about it, it shouldn't page them
3. **Avoid alert fatigue**
   - Too many alerts → engineers start ignoring them → missed real incidents
4. **Use appropriate severity levels**
   - P0/P1: wake someone up immediately
   - P2/P3: next business day ticket
5. **Include runbook links in alerts**
   - Engineer should know exactly what to do when alert fires

> At PayPal, we had a problem with noisy alerts — over 200 alerts firing daily, most auto-resolving. I ran an alert audit, eliminated duplicate and low-signal alerts, and reduced noise by 60%. On-call quality improved significantly.

---

## SECTION 5: Linux & Kubernetes

### Q10: A production server is running slow. How do you troubleshoot it?

**Answer:**

I follow a systematic top-down approach:

```bash
# 1. Check system load
top
htop

# 2. Check CPU usage per process
ps aux --sort=-%cpu | head -20

# 3. Check memory
free -h
# Look for swap usage — if swap is high, system is memory-starved

# 4. Check disk I/O
iostat -x 1
iotop

# 5. Check network
netstat -s
ss -tp

# 6. Check disk space
df -h
du -sh /var/log/*

# 7. Check for zombie processes
ps aux | grep Z

# 8. Check application logs
tail -f /var/log/app/application.log
```

> At PayPal, we had a service randomly going slow. Using `iostat` I found disk I/O was saturated. The cause was a log rotation job running during business hours writing gigabytes of compressed logs. Moving it to off-peak hours resolved the issue.

---

### Q11: A pod in Kubernetes is in CrashLoopBackOff. What do you do?

**Answer:**

```bash
# 1. Check pod status
kubectl get pods -n <namespace>

# 2. Describe the pod — look at Events section
kubectl describe pod <pod-name> -n <namespace>

# 3. Check logs of the crashing container
kubectl logs <pod-name> -n <namespace> --previous

# 4. Common causes:
#    - App crashing on startup (check logs for stack trace)
#    - Liveness probe failing (bad health check config)
#    - OOMKilled (container hitting memory limit)
#    - Missing config/secret (env var not set)
#    - Wrong image tag
```

**Fix approach by cause:**
- OOMKilled → increase memory limit or fix memory leak
- Bad config → fix ConfigMap or Secret
- App crash → check logs, fix application bug
- Bad probe → fix liveness/readiness probe path/timing

> At PayPal, I debugged a CrashLoopBackOff where our service was OOMKilled every 30 minutes. The logs showed a memory leak in a background thread accumulating cached data indefinitely. We added a cache eviction policy and the issue was resolved.

---

## SECTION 6: Behavioral (STAR Method)

### Q12: Tell me about a major incident you led.

**Answer (STAR):**

**Situation:** During a peak traffic period at PayPal, our checkout success rate dropped from 99.8% to 94% — affecting millions of transactions.

**Task:** As the on-call SRE, I needed to identify the root cause and restore service within our MTTR SLO of 30 minutes.

**Action:**
- Immediately opened incident bridge and assigned roles (IC, comms, tech lead)
- Used Datadog APM to trace failing requests — found errors concentrated on one payment processor integration
- Checked recent deployments — a config change had been pushed 20 minutes earlier with a wrong timeout value (500ms instead of 5000ms)
- Coordinated rollback of the config change while keeping stakeholders informed every 10 minutes

**Result:**
- Service restored in 22 minutes (within SLO)
- Root cause: insufficient config validation in deployment pipeline
- Action item: added automated config value range checks to CI/CD — prevented similar incidents twice in the next 6 months

---

### Q13: How do you handle disagreement with a developer team about reliability?

**Answer (STAR):**

**Situation:** A development team at PayPal wanted to ship a new feature on the Friday before a major sales event. Our SRE analysis showed the feature had inadequate load testing and could impact checkout reliability.

**Task:** I needed to either block the release or find a middle ground without damaging the relationship.

**Action:**
- Presented data: showed the load test results and projected failure rate under peak load
- Proposed a compromise: deploy behind a feature flag, enable for only 1% of traffic, monitor for 24 hours before full rollout
- Worked with the dev team to add additional monitoring and a quick rollback plan

**Result:**
- Feature launched safely at 1%, caught a connection pool exhaustion bug under load
- Fixed before full rollout — zero customer impact
- Built trust with the dev team; they started involving SRE earlier in their release process

---

### Q14: How do you prioritize during a major outage when multiple things are failing?

**Answer:**

I use this priority framework:

1. **Mitigate first, debug later** — restore service before finding root cause
2. **Blast radius first** — fix what affects the most users first
3. **Quick wins first** — if a rollback takes 2 minutes vs a fix that takes 2 hours, rollback first
4. **Delegate** — assign roles (one person investigates, one communicates, one executes)

> At PayPal, during a multi-service outage, we had 3 things failing simultaneously. I triaged: database failover (affecting 100% of users) was priority 1, a degraded fraud service (causing 5% slowdown) was priority 2, and a logging pipeline failure was priority 3. We resolved in order — checkout was restored in 18 minutes while the other issues were handled by separate engineers in parallel.

---

## SECTION 7: Rapid Fire Questions

| Question | Answer |
|----------|--------|
| Latency vs Throughput? | Latency = time to complete one request (e.g., 200ms). Throughput = number of requests per second (e.g., 10,000 TPS) |
| What is p99 latency? | 99th percentile latency — 99% of requests are faster than this value. Captures worst-case user experience |
| What is a rolling deployment? | Gradually replace old instances with new ones, a few at a time. Traffic shifts to new instances as they become healthy |
| What is idempotency and why is it critical for payments? | An idempotent operation produces the same result no matter how many times called. Critical to prevent double-charging on retries |
| Horizontal vs Vertical scaling? | Vertical = bigger machine (more CPU/RAM). Horizontal = more machines. Horizontal preferred — no single point of failure |
| What is a deadlock? | Two processes each waiting for the other to release a resource — both wait forever. Resolved by timeout, lock ordering, or deadlock detection |

---

## Tips for Your Interview

| Tip | Why it matters |
|-----|----------------|
| Lead with PayPal impact | Quantify everything (MTTR, uptime %, scale) |
| Think out loud | Interviewers want to see your reasoning process |
| Ask clarifying questions | Especially in system design — shows maturity |
| Admit what you don't know | Then explain how you'd find out |
| Prepare 5 STAR stories | Covering: incident, improvement, conflict, failure, collaboration |
| Structure your answer | Context → Approach → Result |

---

*Your 6.5 years at PayPal is your biggest asset. Quantify your impact and tell real stories!*
