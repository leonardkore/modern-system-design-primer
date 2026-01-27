# Modern System Design Primer

> **A 2026 update to system design fundamentals.** Cloud-native patterns, AI/ML integration, and hard-won lessons from operating at scale.

This guide assumes you're building systems today, not studying for interviews. Concepts link to practical decisions. Theory serves implementation.

---

## Table of Contents

### Foundations
- [CAP and PACELC Theorems](#cap-and-pacelc-theorems)
- [Consistency Models](#consistency-models)
- [Latency vs Throughput](#latency-vs-throughput)
- [Replication](#replication)
- [Partitioning](#partitioning)
- [Consensus](#consensus)

### Compute
- [Containers and Orchestration](#containers-and-orchestration)
- [Platform Engineering](#platform-engineering)
- [Serverless Patterns](#serverless-patterns)
- [Edge Computing](#edge-computing)

### Data
- [Database Selection](#database-selection)
- [Caching Strategies](#caching-strategies)
- [Streaming vs Batch](#streaming-vs-batch)
- [Data Modeling](#data-modeling)

### Communication
- [API Design](#api-design)
- [API Security](#api-security)
- [Event-Driven Architecture](#event-driven-architecture)
- [Service Mesh](#service-mesh)
- [Resilience Patterns](#resilience-patterns)

### AI/ML Systems
- [ML Infrastructure](#ml-infrastructure)
- [Vector Databases](#vector-databases)
- [LLM Integration](#llm-integration)
- [RAG Architecture](#rag-architecture)

### Operations
- [Observability](#observability)
- [Incident Management](#incident-management)
- [Deployment Strategies](#deployment-strategies)
- [Cost Optimization](#cost-optimization)
- [Zero-Trust Architecture](#zero-trust-architecture)
- [Disaster Recovery](#disaster-recovery)

### Appendix
- [Latency Numbers Every Engineer Should Know](#latency-numbers-every-engineer-should-know)
- [Tools Reference](#tools-reference)
- [Case Studies](#case-studies)

---

## How to Use This Guide

**Building something specific?** Jump to the relevant section. Each topic stands alone.

**Learning system design?** Read Foundations first. The concepts there—consistency, partitioning, replication—appear everywhere else.

**Preparing for interviews?** This isn't an interview prep guide, but it covers the concepts interviewers expect you to know. Focus on tradeoffs, not memorization.

---

# Foundations

The distributed systems concepts everything else builds on. Master these and the rest of system design becomes pattern recognition.

---

## CAP and PACELC Theorems

### Core Concept

Every distributed system faces a fundamental constraint: when the network partitions, you must choose between availability and consistency. This is the CAP theorem, and it's not a menu—it's what happens when things go wrong.

The classic "pick two of three" framing misleads engineers. CA systems don't exist in practice because network partitions are unavoidable. Your datacenter switch will fail. Your cloud provider will have an outage. The question isn't whether to handle partitions, but how.

PACELC extends CAP to cover normal operation: if there's a **P**artition, choose **A**vailability or **C**onsistency; **E**lse (normal operation), choose **L**atency or **C**onsistency. This captures what databases actually do.

| Configuration | During Partition | Normal Operation | Examples |
|--------------|------------------|------------------|----------|
| PA/EL | Availability | Low latency | Cassandra, DynamoDB |
| PA/EC | Availability | Consistency | MongoDB (default) |
| PC/EC | Consistency | Consistency | CockroachDB, Spanner |

### Practical Guidance

**Default to eventual consistency with conflict resolution.** Most applications tolerate stale reads and can merge conflicting writes. Shopping carts, social feeds, and analytics dashboards rarely need strong consistency.

**Use strong consistency for coordination and money.** Inventory counts that can go negative, account balances, and distributed locks require linearizability. Accept the latency cost.

**Understand your database's actual behavior.** Marketing materials claim "strong consistency" while defaulting to weaker modes. Read the documentation on consistency levels. Cassandra's QUORUM reads don't give you linearizability. MongoDB's "majority" write concern doesn't prevent stale reads.

> **Common mistake:** Treating CAP as a design-time choice. CAP describes runtime behavior during failures. You choose consistency levels per-operation, not per-system.

### Going Deeper

- [Brewer's CAP Theorem](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/) — Brewer's 2012 retrospective clarifying common misconceptions
- [PACELC: The Extended CAP](https://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf) — Abadi's paper introducing the PACELC model
- [Jepsen: Consistency Models](https://jepsen.io/consistency) — Kyle Kingsbury's visual guide to consistency guarantees

### TL;DR

- CAP isn't a menu; it's what happens when networks fail
- PACELC adds the latency/consistency tradeoff during normal operation
- Default to eventual consistency; reserve strong consistency for coordination
- Check your database's actual consistency behavior, not its marketing

---

## Consistency Models

### Core Concept

Consistency models define what values a read can return. They're contracts between the database and your application: "If you write X, here's what subsequent reads might see."

**Linearizability** (strong consistency) makes a distributed system behave like a single machine. Every operation appears to happen instantaneously at some point between invocation and response. If you write a value and then read it, you see your write—and so does everyone else.

**Eventual consistency** guarantees that if no new updates occur, all replicas will eventually converge to the same value. The "eventually" is unbounded. A read might return stale data for milliseconds, seconds, or longer depending on replication lag.

**Causal consistency** sits between them: operations that are causally related appear in order, but concurrent operations may appear in different orders to different observers. If user A posts a message and user B replies, everyone sees the post before the reply—but two unrelated posts may appear in different orders to different users.

### Practical Guidance

**Choose the weakest consistency model your application tolerates.** Stronger consistency costs latency and availability. A social feed showing posts out of order is annoying. A banking system showing incorrect balances is a lawsuit.

**Use read-your-writes consistency for user sessions.** Users expect to see their own changes immediately. Route session requests to the same replica, or use quorum reads after writes.

**Understand how your consistency model composes.** Two eventually consistent operations don't give you causal consistency. If you read a user's profile and then their posts, you might see posts from a "future" version of the profile.

> **Common mistake:** Assuming eventual consistency means "a few milliseconds." Replication lag can spike during failures or high load. Design for the worst case, not the average.

### Going Deeper

- [Designing Data-Intensive Applications, Ch. 9](https://dataintensive.net/) — Kleppmann's definitive treatment of consistency models
- [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — Why strong consistency matters for coordination
- [Cassandra's consistency levels](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html#tunable-consistency) — Practical example of tunable consistency

### TL;DR

- Linearizability: behaves like one machine, highest cost
- Eventual consistency: replicas converge "eventually," lowest cost
- Causal consistency: related operations ordered, concurrent operations not
- Pick the weakest model your application can tolerate
- Read-your-writes consistency prevents user confusion

---

## Latency vs Throughput

### Core Concept

Latency measures how long a single request takes. Throughput measures how many requests complete per unit time. Optimizing one often hurts the other.

A system processing requests sequentially has low throughput but predictable latency. Batch processing increases throughput—handle 100 requests together instead of one at a time—but the first request waits for the batch to fill, increasing its latency.

The relationship isn't linear. Queueing theory shows that as utilization approaches 100%, latency grows exponentially. A server running at 90% utilization has much higher tail latency than one at 70%.

### Practical Guidance

**Measure percentiles, not averages.** The average response time hides outliers. P99 latency (the slowest 1% of requests) often determines user experience. If P99 is 10 seconds, 1 in 100 users waits 10 seconds.

**Keep utilization below 70% for latency-sensitive systems.** The exponential curve means headroom matters more than efficiency. Provision for peak load, not average load.

**Batch for throughput, stream for latency.** Writing 1,000 records individually gives low latency per write. Batching them gives high throughput. Choose based on your requirements.

**Understand Little's Law: L = λW.** The average number of requests in a system (L) equals arrival rate (λ) times average time in system (W). If you process 100 requests/second and each takes 0.1 seconds, you have 10 concurrent requests. This tells you how many connections your database needs.

> **Common mistake:** Adding caching to reduce latency without considering throughput. Cache misses now go to an undersized backend that can't handle the load.

### Going Deeper

- [Latency Numbers Every Programmer Should Know](https://gist.github.com/jboner/2841832) — Jeff Dean's classic reference (see our [updated version](#latency-numbers-every-engineer-should-know))
- [The Tail at Scale](https://research.google/pubs/pub40801/) — Google's paper on managing tail latency
- [Little's Law and its application](https://www.johndcook.com/blog/2009/01/30/littles-law/) — Practical explanation with examples

### TL;DR

- Latency: time per request. Throughput: requests per time.
- Optimizing one often hurts the other
- Measure P99 latency, not averages
- Keep utilization below 70% for latency-sensitive paths
- Little's Law: concurrent requests = arrival rate × latency

---

## Replication

### Core Concept

Replication copies data across multiple machines for durability and availability. If one machine fails, another has the data. If traffic spikes, multiple replicas share the load.

**Leader-follower replication** (primary-secondary) routes writes to one node that propagates changes to followers. Followers serve reads. Simple to reason about, but the leader is a bottleneck and single point of failure.

**Multi-leader replication** allows writes to multiple nodes, each propagating to others. Better write availability and geographic distribution, but concurrent writes can conflict. You need conflict resolution: last-writer-wins, vector clocks, or application-level merging.

**Leaderless replication** sends writes to multiple replicas directly. Read from multiple replicas and reconcile. Cassandra and DynamoDB use this model. No single point of failure, but consistency is harder to reason about.

### Practical Guidance

**Start with leader-follower replication.** Most databases default to it. It's the simplest model and handles most use cases. Switch to multi-leader only when you need writes in multiple regions.

**Synchronous replication trades availability for durability.** If the leader waits for followers to acknowledge, a follower failure stalls writes. Asynchronous replication risks data loss if the leader fails before propagating. Semi-synchronous (one synchronous follower) is a common compromise.

**Monitor replication lag.** Followers behind the leader serve stale data. Alert when lag exceeds your consistency requirements. In leader-follower setups, route time-sensitive reads to the leader.

**Plan for leader failure.** Automatic failover promotes a follower when the leader dies. But the promoted follower might be behind—do you accept data loss or halt writes until you recover the old leader? Decide in advance.

> **Common mistake:** Routing all reads to the leader "to ensure consistency" and negating the read scaling benefit of replication.

### Going Deeper

- [Designing Data-Intensive Applications, Ch. 5](https://dataintensive.net/) — Comprehensive treatment of replication
- [How Discord Stores Trillions of Messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) — Cassandra replication at scale
- [Vitess: YouTube's database scaling](https://vitess.io/docs/concepts/replication/) — MySQL replication at Google scale

### TL;DR

- Leader-follower: simple, leader is bottleneck
- Multi-leader: write availability, needs conflict resolution
- Leaderless: no single point of failure, complex consistency
- Monitor replication lag—stale reads cause bugs
- Plan for leader failure before it happens

---

## Partitioning

### Core Concept

Partitioning (sharding) splits data across multiple machines. Each partition holds a subset of the data. Unlike replication where every node has all data, partitioning divides data so each node handles less.

**Hash partitioning** assigns records by hashing a key. User ID 12345 hashes to partition 3. Distributes load evenly but destroys key ordering—range queries scan all partitions.

**Range partitioning** assigns records by key ranges. Users A-M go to partition 1, N-Z to partition 2. Enables efficient range queries but risks hot spots—if most users' names start with S, one partition overloads.

**Consistent hashing** minimizes data movement when partitions change. Adding a node moves only keys in its range, not all keys. Essential for systems that scale dynamically.

### Practical Guidance

**Choose partition keys carefully.** A good key distributes load evenly and keeps related data together. User ID is good for user-scoped queries. Timestamp is bad—all writes go to the "now" partition.

**Avoid cross-partition queries when possible.** A query touching one partition is fast. A query touching all partitions is slow and expensive. Design schemas to keep related data in the same partition.

**Plan for hot partitions.** Some keys are accessed more than others. A viral tweet, a celebrity's profile. Strategies: add random suffixes to spread load (read from multiple partitions), cache hot keys, or use separate storage for high-traffic entities.

**Consider composite partition keys.** Partition by (tenant_id, date) to keep each tenant's daily data together. Queries for one tenant's day hit one partition; queries across days hit multiple but fewer than all.

> **Common mistake:** Partitioning too early. A single well-indexed database handles more load than engineers expect. Partition when you've exhausted vertical scaling and query optimization.

### Going Deeper

- [Amazon Dynamo Paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — Foundational paper on consistent hashing and partitioning
- [How Figma scaled to multiple databases](https://www.figma.com/blog/how-figma-scaled-to-multiple-databases/) — Practical sharding story
- [Vitess: Sharding YouTube's MySQL](https://vitess.io/docs/concepts/sharding/) — Sharding middleware approach

### TL;DR

- Hash partitioning: even distribution, no range queries
- Range partitioning: efficient ranges, risk of hot spots
- Consistent hashing: minimal data movement when scaling
- Choose partition keys for even distribution and query locality
- Avoid cross-partition queries; plan for hot partitions

---

## Consensus

### Core Concept

Consensus lets distributed nodes agree on a value even when some nodes fail or messages are lost. It's the foundation of leader election, distributed transactions, and replicated state machines.

The problem is harder than it sounds. Nodes can fail silently or send conflicting messages. Networks can delay, duplicate, or drop messages. The Fischer-Lynch-Paterson (FLP) impossibility result proves that no deterministic consensus algorithm guarantees progress in an asynchronous network with even one faulty node.

Practical consensus algorithms like Raft and Paxos work around FLP by using timeouts to detect failures. They sacrifice guaranteed progress for practical reliability—a node that times out might not be dead, just slow, but the system proceeds anyway.

**Raft** structures consensus as leader election plus log replication. A leader proposes entries; followers accept if they're consistent with their log. If the leader fails, followers elect a new one. The algorithm is designed for understandability—its paper explicitly prioritizes clarity over optimization.

### Practical Guidance

**Don't implement consensus yourself.** Use etcd, Consul, or ZooKeeper for coordination. Use a database with built-in consensus (CockroachDB, TiDB) for distributed transactions. Consensus bugs are subtle and catastrophic.

**Understand the cost.** Consensus requires multiple round trips and disk fsyncs. A Raft write waits for a majority of nodes to persist to disk. This costs tens of milliseconds minimum. Don't use consensus for data that doesn't need it.

**Use consensus for coordination, not data.** Store leader election state, distributed locks, and configuration in a consensus system. Store your actual application data in databases optimized for your access patterns.

**Size clusters for failure tolerance, not throughput.** A 3-node cluster tolerates 1 failure. A 5-node cluster tolerates 2. More nodes don't add read throughput in most consensus systems—they add durability and availability.

> **Common mistake:** Running a single etcd/Consul node "for simplicity." You've lost all fault tolerance. A 3-node cluster is the minimum for production.

### Going Deeper

- [In Search of an Understandable Consensus Algorithm (Raft)](https://raft.github.io/raft.pdf) — The Raft paper, designed for readability
- [Raft Visualization](https://raft.github.io/) — Interactive Raft demonstration
- [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) — Lamport's accessible Paxos explanation
- [The Raft Consensus Algorithm](https://www.youtube.com/watch?v=vYp4LYbnnW8) — Diego Ongaro's conference talk

### TL;DR

- Consensus: nodes agree on values despite failures
- FLP proves guaranteed progress is impossible; algorithms use timeouts
- Raft: leader election + log replication, designed for understandability
- Don't implement consensus yourself; use etcd, Consul, or ZooKeeper
- Minimum 3 nodes for fault tolerance; 5 for higher availability

---

# Compute

Where your code runs and how it scales. In 2026, containers are infrastructure, not a topic—this section covers how to use them effectively.

---

## Containers and Orchestration

### Core Concept

Containers package an application with its dependencies into a portable unit. Unlike VMs, they share the host kernel, making them lightweight and fast to start. Docker standardized the format; Kubernetes standardized orchestration.

Kubernetes schedules containers across a cluster, handles failures, manages networking, and provides service discovery. It's complex because distributed systems are complex—Kubernetes surfaces that complexity rather than hiding it.

The ecosystem has matured significantly. Most engineers interact with Kubernetes through abstractions: GitOps tools that sync desired state from Git, internal developer platforms that hide YAML, and managed services that handle the control plane.

### Practical Guidance

**Use a managed Kubernetes service.** EKS, GKE, and AKS handle control plane operations, upgrades, and availability. Running your own Kubernetes control plane requires dedicated expertise. Most teams shouldn't do it.

**Know when Kubernetes is overkill.** K8s shines at 10+ services with dedicated platform expertise. For smaller deployments, simpler options work better:

| Situation | Better choice than K8s |
|-----------|------------------------|
| < 5 services, small team | ECS, Cloud Run, or even VMs behind a load balancer |
| Single application | App Engine, Heroku, Railway |
| Batch jobs only | AWS Batch, Cloud Run Jobs |
| No ops expertise on team | Managed PaaS (Render, Fly.io) |

The complexity tax of Kubernetes—networking, RBAC, upgrades, observability—requires ongoing investment. Don't pay it unless you're getting value back.

**Treat Kubernetes configuration as code.** Store manifests in Git. Use GitOps tools (ArgoCD, Flux) to sync cluster state from repositories. Changes flow through pull requests with review, rollback is a git revert.

**Understand resource requests and limits.** This is where most Kubernetes pain originates.

```yaml
resources:
  requests:
    memory: "256Mi"    # Scheduler uses this for placement
    cpu: "250m"        # 250 millicores = 0.25 CPU
  limits:
    memory: "512Mi"    # Hard ceiling—exceed this and get OOMKilled
    cpu: "500m"        # Throttled, not killed, when exceeded
```

**Requests** are what your container needs to run. The scheduler places pods on nodes with enough unrequested resources. Set requests too low and pods land on overcommitted nodes. Set them too high and you waste cluster capacity.

**Limits** are ceilings. Memory limits are hard—exceed them and Kubernetes kills your container (OOMKilled). CPU limits are soft—exceed them and you're throttled, not terminated.

The **OOMKilled spiral**: Container hits memory limit → killed → restarted → loads data back into memory → killed again. Fix by increasing limits or fixing memory leaks. Don't just keep restarting.

```bash
# Diagnose OOMKilled pods
kubectl describe pod <name> | grep -A5 "Last State"
kubectl top pod <name>  # Current memory usage
```

**Use Deployments for stateless workloads.** Most applications should use Deployments with external state (databases, caches, object storage). Deployments handle rolling updates, rollbacks, and scaling without caring about pod identity.

**Reserve StatefulSets for true stateful needs.** StatefulSets provide stable network identities and persistent storage ordering. Use them for:
- Databases you're running yourself (don't)
- Distributed systems that need stable peer addressing (Kafka, Elasticsearch)
- Applications that must process data in order

If you're considering StatefulSets, first ask: can I use a managed service instead? Managing stateful workloads on Kubernetes is expert-level operations.

**Choose a namespace strategy and stick to it.**

| Strategy | Pros | Cons |
|----------|------|------|
| By environment (dev/staging/prod) | Simple, clear boundaries | Cross-team visibility issues |
| By team | Team autonomy, clear ownership | Environment sprawl |
| By service | Fine-grained quotas | Hundreds of namespaces to manage |

Most teams do well with environment-based namespaces and RBAC for team isolation. Start simple.

**Pick Kustomize or Helm, not both.**

- **Kustomize**: Built into kubectl, patches base manifests. Good for environment-specific overrides without templating.
- **Helm**: Full templating with values files and package management. Good when you need conditional logic or are distributing charts.

Opinion: Use Kustomize for your own services (simpler), Helm for third-party software (it's how they're distributed).

**Implement health checks.** Liveness probes restart stuck containers. Readiness probes remove unhealthy containers from load balancers. Without them, Kubernetes can't distinguish a crashed container from a healthy one.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10    # Give app time to start
  periodSeconds: 10
  failureThreshold: 3        # Kill after 3 consecutive failures

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
  failureThreshold: 2        # Remove from LB after 2 failures
```

> **Common mistake:** Setting liveness probe timeouts too aggressive. A garbage collection pause or slow dependency shouldn't kill your pod. Start with generous timeouts (30s) and tighten based on actual behavior.

### Going Deeper

- [Kubernetes Documentation: Concepts](https://kubernetes.io/docs/concepts/) — Official concepts guide
- [Production-ready Kubernetes](https://www.oreilly.com/library/view/production-kubernetes/9781492092292/) — Comprehensive operations guide
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/) — GitOps for Kubernetes
- [Kubernetes Failure Stories](https://k8s.af/) — Learn from others' incidents

### TL;DR

- Use managed Kubernetes (EKS, GKE, AKS) unless you have dedicated platform expertise
- Know when K8s is overkill—smaller deployments have simpler options
- Requests are for scheduling; limits are ceilings. Memory limits kill; CPU limits throttle.
- Use Deployments for stateless, StatefulSets only for true stateful needs
- Implement health checks with generous initial timeouts

---

## Platform Engineering

### Core Concept

Platform engineering builds internal developer platforms (IDPs) that abstract infrastructure complexity. Instead of every team learning Kubernetes, Terraform, and CI/CD pipelines, they interact with a self-service platform that enforces standards while enabling autonomy.

The goal is paved roads, not guardrails. A paved road makes the right thing easy: "deploy a service" becomes a form submission, not a 500-line YAML file. Teams can still go off-road when needed, but most won't need to.

This inverts the traditional DevOps model. Instead of embedding operations knowledge in every team, platform teams build products that encode that knowledge. Application teams consume the platform; platform teams maintain it.

### Practical Guidance

**Start with developer pain points, not technology.** Survey your engineers. What takes too long? What requires tickets to another team? Build platforms that eliminate those bottlenecks. Technology choice follows from problems, not vice versa.

**A golden path looks like this:**

```
Developer runs: platform create-service my-api --template=python-fastapi

Behind the scenes:
1. Creates Git repo from template (Dockerfile, CI config, K8s manifests)
2. Provisions namespace and RBAC
3. Sets up CI/CD pipeline
4. Creates monitoring dashboards
5. Configures secrets management
6. Adds service to catalog

Result: Production-ready service in one command, PR to add features
```

The golden path isn't about control—it's about removing toil. Developers who want custom setups can still have them. Most won't need to.

**Know when platform engineering is premature.** Building internal platforms is expensive. You need:
- 20+ engineers to justify the investment
- Repeated patterns (if every service is unique, there's nothing to standardize)
- Platform team capacity (2-4 dedicated engineers minimum)

At smaller scale, "platform engineering" is a fancy name for "shared scripts and documentation." That's fine. Graduate to real platforms when the pain justifies it.

**Provide golden paths, not mandates.** A golden path is the recommended way to do something—a template, a wizard, a CLI command. Make it easier than the alternative and teams will adopt it voluntarily. Mandates create workarounds.

**Treat your platform as a product.** Platform teams have customers (internal developers) and should operate like product teams: gather feedback, prioritize features, measure adoption. A platform nobody uses isn't a platform.

**Use Backstage or similar for a developer portal.** Backstage (open-sourced by Spotify) provides a unified interface for service catalogs, documentation, templates, and plugins. It's the "front door" to your platform.

**Invest in self-service.** Every manual step that requires platform team involvement is a bottleneck. Automate environment creation, secret rotation, database provisioning. The platform team should build capabilities, not execute requests.

> **Common mistake:** Building a platform before understanding what developers need. Interview your teams first. Many "platform" projects become shelfware because they solve problems engineers don't have.

### Going Deeper

- [Team Topologies](https://teamtopologies.com/) — Organizing for platform teams
- [Backstage Documentation](https://backstage.io/docs/) — Spotify's developer portal
- [Platform Engineering on Kubernetes](https://www.manning.com/books/platform-engineering-on-kubernetes) — Building internal platforms with Kubernetes
- [The CNCF Platforms White Paper](https://tag-app-delivery.cncf.io/whitepapers/platforms/) — Industry consensus on platform engineering

### TL;DR

- Platform engineering builds self-service developer tools
- Golden paths remove toil: one command from zero to production-ready
- Premature before 20+ engineers—start with shared scripts
- Treat your platform as a product with developer customers
- Automate everything that requires a ticket to another team

---

## Serverless Patterns

### Core Concept

Serverless computing abstracts server management entirely. You deploy functions; the provider handles scaling, availability, and infrastructure. You pay for execution time, not idle capacity.

The name misleads—servers exist, you just don't manage them. The key shift is the operational model: no capacity planning, no patching, no scaling configuration. The tradeoff is less control and potential cold start latency.

**Functions as a Service (FaaS)** runs code in response to events. AWS Lambda, Cloud Functions, and Azure Functions are the major providers. Functions are stateless, short-lived, and triggered by events (HTTP requests, queue messages, database changes).

**Backend as a Service (BaaS)** provides managed capabilities: authentication (Auth0, Firebase Auth), databases (Firestore, DynamoDB), file storage (S3, Cloud Storage). Serverless applications compose these services instead of building from scratch.

### Practical Guidance

**Understand cold start reality.** When a function hasn't run recently, the provider must initialize it. Real-world numbers:

| Runtime | Cold Start | With dependencies |
|---------|------------|-------------------|
| Lambda (Python) | 100-200ms | 200-500ms |
| Lambda (Java) | 500-800ms | 1-3s |
| Lambda (Node.js) | 100-150ms | 150-400ms |
| Cloud Run | 0-300ms | Depends on image size |
| Cloud Run (min instances=1) | ~0ms* | Near-zero |

*With min instances, cold start is eliminated but internal routing adds 5-20ms. True zero requires warm instances serving traffic.

For user-facing APIs where P99 latency matters, cold starts hurt. Mitigations:
- **Provisioned concurrency** (Lambda): keep instances warm, pay for idle
- **Min instances** (Cloud Run): same idea, different pricing model
- **Smaller dependencies**: faster initialization

**Use serverless for the right workloads.** Serverless excels at:
- **Scheduled jobs (cron)**: Run nightly reports, cleanup tasks, data syncs. No idle cost.
- **Event processing**: Image resize on upload, webhook handlers, queue consumers.
- **Variable traffic APIs**: Marketing landing pages, internal tools with sporadic use.

Serverless struggles with:
- **Consistent high traffic**: Containers are cheaper at sustained load
- **Latency-sensitive paths**: Cold starts are unpredictable
- **Long-running processes**: Function timeouts (15 min Lambda, 60 min Cloud Run)

**Watch the cost cliff.** Serverless pricing is per-invocation and per-GB-second. At low volume, this is cheap. At high volume, it's not.

```
Example: API handling 100 req/sec, 200ms avg, 256MB memory

Lambda cost (us-east-1, 2026 pricing):
- Requests: 100/sec × 86,400 sec/day × 30 days = 259.2M requests/month
- Request cost: 259.2M × $0.20/1M = ~$52/month
- GB-seconds: 259.2M × 0.2s × 0.25GB = 12.96M GB-sec/month
- Compute cost: 12.96M × $0.0000166667 = ~$216/month
- Total: ~$270/month

Equivalent container (2 instances, 0.5 vCPU, 1GB each):
- ~$60/month on ECS Fargate
- ~$45/month on GKE Autopilot
```

The crossover point varies, but roughly: if your function runs more than 20% of the time, containers are probably cheaper.

**Serverless for cron is almost always right.** A nightly job that runs for 5 minutes costs pennies on Lambda. The equivalent always-on container costs dollars. This is serverless's killer use case.

```python
# Perfect serverless use case: nightly data cleanup
def cleanup_expired_sessions(event, context):
    # Always use parameterized queries—even when values are safe
    # This builds habits that prevent SQL injection when you add user input later
    deleted = db.execute(
        "DELETE FROM sessions WHERE expires_at < %s",
        (datetime.utcnow(),)
    )
    return {"deleted": deleted}
```

**Keep functions small and focused.** Large functions with many dependencies have slower cold starts. The Unix philosophy applies: do one thing well.

> **Common mistake:** Using serverless for everything. Serverless isn't cheaper or simpler for all workloads. Model your costs and latency requirements before committing.

### Going Deeper

- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html) — Official guidance
- [The Serverless Framework](https://www.serverless.com/framework/docs) — Multi-cloud serverless deployment
- [Cold Starts in AWS Lambda](https://mikhail.io/serverless/coldstarts/aws/) — Detailed cold start analysis
- [Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) — Find optimal memory/cost balance

### TL;DR

- Cold starts are real: 100-500ms typical, 1-3s for Java with dependencies
- Serverless for cron jobs is almost always the right choice
- Cost cliff: at ~20%+ utilization, containers become cheaper
- Keep functions focused with minimal dependencies
- Model costs before committing to serverless for high-traffic paths

---

## Edge Computing

### Core Concept

Edge computing runs code geographically close to users. Instead of a request traveling to us-east-1 and back, it executes at the nearest edge location—dozens or hundreds of milliseconds closer.

**CDN edge functions** (Cloudflare Workers, Lambda@Edge, Vercel Edge) run at CDN points of presence. They handle personalization, A/B testing, authentication, and request routing without round-trips to origin servers.

**IoT edge** runs computation on devices or local gateways. Manufacturing sensors process data locally, sending summaries to the cloud. Autonomous vehicles run ML models on-device because cloud latency is unacceptable at 60mph.

The tradeoff is complexity. Edge locations have limited compute, limited storage, and limited connectivity. Applications must handle partial failures and eventual consistency with central systems.

### Practical Guidance

**Use edge for authentication.** This is the highest-ROI edge pattern. JWT validation at the edge:
- Rejects unauthorized requests before they hit your infrastructure
- Reduces origin load by 10-30% (depending on auth failure rate)
- Adds zero latency for valid requests (validation happens during TLS handshake time anyway)

```javascript
// Cloudflare Worker: JWT validation at edge
import { jwtVerify } from 'jose';

export default {
  async fetch(request, env) {
    const authHeader = request.headers.get('Authorization');
    if (!authHeader?.startsWith('Bearer ')) {
      return new Response('Unauthorized', { status: 401 });
    }
    const token = authHeader.split(' ')[1];

    try {
      const { payload } = await jwtVerify(token, env.JWT_SECRET, {
        algorithms: ['RS256'],           // Explicit allowlist—never allow 'none'
        issuer: 'https://auth.example.com',
        audience: 'https://api.example.com',
        requiredClaims: ['sub', 'exp'],
      });

      // Add user context to origin request
      const headers = new Headers(request.headers);
      headers.set('X-User-ID', payload.sub);
      return fetch(request, { headers });
    } catch (err) {
      return new Response('Invalid token', { status: 403 });
    }
  }
}
```

> **Security note:** This example shows minimum viable JWT validation. Production systems also need token revocation checks, key rotation handling, and rate limiting on auth failures.

**Use edge for geolocation routing.** Redirect users to region-specific content or comply with data residency requirements:

```javascript
// Route EU users to EU-specific page
const country = request.cf?.country;
if (['DE', 'FR', 'IT', 'ES'].includes(country)) {
  return Response.redirect('https://eu.example.com' + url.pathname);
}
```

**Cache at the edge, compute when necessary.** Start with static caching—it's free and fast. Add edge compute only for:
- Dynamic variations that can't be cached (user-specific headers)
- Request transformations (rewriting URLs, adding headers)
- Light personalization (A/B test assignment, feature flags)

**Understand edge runtime limitations.** Edge functions are not Node.js:

| Limitation | Cloudflare Workers | Lambda@Edge |
|------------|-------------------|-------------|
| CPU time | 10-50ms | 5s (viewer), 30s (origin) |
| Memory | 128MB | 128MB (viewer), 10GB (origin) |
| Bundle size | 1MB compressed | 1MB (viewer), 50MB (origin) |
| Network | Limited APIs | Full Node.js |

**Handle edge-to-origin failures gracefully.** The edge might not reach your origin. Serve stale content, show degraded experiences, or queue requests for later.

> **Common mistake:** Moving business logic to the edge. Edge is for request-level decisions (route, authenticate, personalize), not for complex operations. If you're querying databases from edge functions, you've gone too far.

### Going Deeper

- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/) — Edge computing platform
- [Vercel Edge Functions](https://vercel.com/docs/functions/edge-functions) — Edge runtime for Next.js
- [Edge Computing Patterns](https://martinfowler.com/articles/patterns-of-distributed-systems/edge-compute.html) — Fowler's distributed systems patterns

### TL;DR

- Edge for authentication is high-ROI: validate JWTs before requests hit origin
- Geolocation routing and A/B tests are natural edge fits
- Start with caching; add compute only for dynamic variations
- Edge runtimes have strict limits—keep logic simple
- Business logic belongs at origin, not edge

---

# Data

Storing, processing, and moving data at scale. The choices here constrain everything else in your system.

---

## Database Selection

### Core Concept

Database selection isn't about finding the "best" database—it's about matching characteristics to requirements. Every database makes tradeoffs, and the wrong fit creates problems that workaround after workaround can't fix.

**Relational databases** (PostgreSQL, MySQL) store structured data in tables with relationships. They provide ACID transactions, complex queries via SQL, and decades of optimization. Most applications should start here.

**Document databases** (MongoDB, DynamoDB) store semi-structured data as JSON-like documents. They sacrifice joins and complex transactions for flexible schemas and horizontal scaling. Good for varying document structures or extreme write throughput.

**Wide-column stores** (Cassandra, ScyllaDB) optimize for write-heavy workloads with predictable queries. Data is denormalized—you model queries, not entities. Essential for time-series data at scale, but awkward for ad-hoc analysis.

**Graph databases** (Neo4j, Amazon Neptune) model relationships as first-class citizens. Queries traverse relationships efficiently: "find friends of friends who like X" is natural in a graph, painful in SQL.

### Practical Guidance

**Start with PostgreSQL.** Postgres handles more use cases than engineers expect: JSON documents (JSONB), full-text search, geospatial queries, time-series data (with TimescaleDB extension). Add specialized databases when Postgres becomes a bottleneck for a specific workload.

**The decision matrix with teeth:**

| If you need... | Use | Not |
|----------------|-----|-----|
| ACID + complex queries | PostgreSQL | MongoDB "for flexibility" |
| Document flexibility + ACID | PostgreSQL JSONB | MongoDB |
| Massive write throughput (100k+ writes/sec) | Cassandra, ScyllaDB | PostgreSQL with replicas |
| Key-value at scale with simple access | DynamoDB, Redis | PostgreSQL as KV store |
| Vector search (< 10M vectors) | pgvector | Pinecone (overkill) |
| Vector search (> 100M vectors) | Pinecone, Qdrant | pgvector (won't scale) |
| Time-series with SQL | TimescaleDB | InfluxDB (limited query language) |
| Relationship traversal | Neo4j, Neptune | Recursive SQL CTEs |

**Pushback responses—the ammunition you need:**

"MongoDB scales better"
> Horizontal scaling is a problem you don't have yet. A single PostgreSQL instance handles 10,000+ queries per second with proper indexing. When you actually need horizontal scaling, you'll have the revenue to solve it properly.

"We need DynamoDB for serverless"
> Only if your access patterns are known, simple, and won't change. Single-table design looks clever until you need a query you didn't plan for. DynamoDB's pricing also surprises teams—read the fine print on read/write capacity units.

"We need schema flexibility"
> PostgreSQL JSONB gives you flexibility with ACID transactions. You can query JSON fields, index them, and still join with relational tables. Pure document databases give flexibility but take away query power you'll miss later.

"[Famous company] uses [exotic database]"
> They also have dedicated database teams, custom tooling, and problems at scales you won't reach for years. Match your database choice to your current team and scale, not your aspirations.

**Understand the migration reality.** Switching databases is a 6-month project minimum. You need dual writes, data migration, query translation, testing, and rollback planning. Choose based on where you'll be in 3 years, not where you are today. The pain of a suboptimal choice now is less than the pain of migration later.

**Choose based on query patterns, not data volume.** A 10TB relational database is manageable with good indexing. A 1GB database with the wrong access pattern is a nightmare. Model your queries before picking technology.

**Understand your consistency requirements.** Financial transactions need strong consistency. Social feeds tolerate eventual consistency. The wrong choice causes either bugs or unnecessary latency.

**Plan for operational burden.** Managed services (RDS, Cloud SQL, DynamoDB) cost more but include backups, patching, failover, and monitoring. Self-managed databases require dedicated expertise. Factor this into your decision.

**Don't prematurely optimize for scale.** A single Postgres instance handles thousands of requests per second. Sharding introduces complexity you might never need. Vertical scaling (bigger machine) is underrated.

> **Common mistake:** Choosing a database because it's trendy or because a famous company uses it. Netflix uses Cassandra because they have Netflix-scale problems. You probably don't.

### Going Deeper

- [Designing Data-Intensive Applications](https://dataintensive.net/) — Kleppmann's comprehensive guide
- [PostgreSQL Documentation](https://www.postgresql.org/docs/) — Official docs are excellent
- [Things I Wished More Developers Knew About Databases](https://rakyll.medium.com/things-i-wished-more-developers-knew-about-databases-2d0178464f78) — Common misconceptions
- [How to Choose the Right Database](https://www.prisma.io/dataguide/intro/comparing-database-types) — Decision framework

### TL;DR

- Start with PostgreSQL; add specialized databases for specific bottlenecks
- PostgreSQL JSONB handles most "flexibility" needs with ACID
- Database migration is a 6+ month project—choose for 3 years out
- Match database to your team's expertise, not famous companies' choices
- Don't shard until you've exhausted vertical scaling

---

## Caching Strategies

### Core Concept

Caching stores frequently accessed data in faster storage to reduce latency and backend load. The cache sits between clients and the database, serving requests that would otherwise hit slower storage.

**Cache-aside (lazy loading)** is the most common pattern: the application checks the cache first, queries the database on miss, then populates the cache. The cache contains only requested data, minimizing memory use.

```python
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    if user is None:
        user = db.query("SELECT * FROM users WHERE id = %s", user_id)
        cache.set(f"user:{user_id}", user, ttl=300)
    return user
```

**Write-through** writes to cache and database together. The cache always has fresh data, but writes are slower. Use when read latency matters more than write latency.

**Write-behind (write-back)** writes to cache immediately and asynchronously flushes to the database. Fast writes, but risks data loss if the cache fails before flushing.

### Practical Guidance

**Use Redis as your default cache.** It's fast, well-documented, and handles more than key-value: sorted sets, pub/sub, Lua scripting. Memcached is simpler but less capable.

**Set TTLs on everything.** Time-to-live prevents stale data from persisting forever. Even "permanent" data should have long TTLs (24 hours) to handle edge cases where invalidation fails.

**Choose cache invalidation carefully.** Cache invalidation is genuinely hard. Options:

| Strategy | Pros | Cons | Best for |
|----------|------|------|----------|
| TTL only | Simple, no coordination | Serves stale data | Content that can be stale for minutes |
| Invalidate on write | Fresh data | Complex in distributed systems | Single-service ownership |
| Event-driven invalidation | Decoupled | Eventually consistent | Microservices, high scale |
| Versioned keys | No invalidation needed | Key proliferation | Immutable data references |

**Cache stampede prevention patterns:**

1. **Locking** — One request rebuilds, others wait:
```python
def get_user_safe(user_id):
    user = cache.get(f"user:{user_id}")
    if user is None:
        lock_acquired = cache.set(f"lock:user:{user_id}", "1", nx=True, ex=10)
        if lock_acquired:
            try:
                user = db.query("SELECT * FROM users WHERE id = %s", user_id)
                cache.set(f"user:{user_id}", user, ttl=300)
            finally:
                cache.delete(f"lock:user:{user_id}")
        else:
            time.sleep(0.1)  # Wait and retry
            return get_user_safe(user_id)
    return user
```

2. **Probabilistic early expiration** — Refresh before TTL expires:
```python
def get_with_early_refresh(key, ttl=300, beta=1.0):
    value, expiry = cache.get_with_ttl(key)
    if value is None:
        return refresh_and_cache(key, ttl)

    # Probabilistically refresh before expiration
    remaining = expiry - time.time()
    if remaining < ttl * random.random() * beta:
        refresh_and_cache(key, ttl)  # Background refresh

    return value
```

3. **Background refresh** — Best for high-traffic keys. A background job refreshes the cache before expiration. The cache never actually expires for popular keys.

**What to cache (and what not to):**

| Data type | Cache? | Notes |
|-----------|--------|-------|
| Computation results | Yes | Expensive calculations, ML inference |
| Database query results | Carefully | Watch cardinality and invalidation |
| Session data | Yes | Use Redis with persistence |
| User-specific data | Carefully | High cardinality can exhaust memory |
| Configuration | Yes | Refresh on deploy or via pub/sub |
| Rate limit counters | Yes | Redis INCR with TTL |

**Handle cache failures gracefully.** If Redis dies, fall back to the database—don't return errors for cache misses. Design systems where cache improves performance but isn't required for correctness.

**Cache at the right layer.** Options from fastest to slowest: in-process cache (local memory), distributed cache (Redis), CDN (edge), database query cache. Each layer adds complexity. Start with fewer caches and add as needed.

> **Common mistake:** Caching everything. Caches consume memory and add complexity. Cache hot paths where latency matters, not cold paths accessed once a day.

### Going Deeper

- [Caching Best Practices](https://aws.amazon.com/caching/best-practices/) — AWS caching patterns
- [Redis Documentation](https://redis.io/docs/) — Official Redis docs
- [Scaling Memcache at Facebook](https://research.facebook.com/publications/scaling-memcache-at-facebook/) — Large-scale caching
- [A Hitchhiker's Guide to Caching Patterns](https://hazelcast.com/blog/a-hitchhikers-guide-to-caching-patterns/) — Pattern comparison

### TL;DR

- Cache-aside is the default: check cache, miss to database, populate cache
- Use Redis; set TTLs on everything
- Prevent stampedes with locking, probabilistic refresh, or background jobs
- Cache computation results and hot paths; skip cold data
- Handle cache failures gracefully—fall back to database

---

## Streaming vs Batch

### Core Concept

Data processing divides into two paradigms: batch processes large datasets periodically; streaming processes records continuously as they arrive.

**Batch processing** runs on schedules—hourly, daily, or triggered by data availability. It handles large volumes efficiently, transforming gigabytes or terabytes in a single job. Good for analytics, reporting, and ETL.

**Stream processing** handles unbounded data flows in real-time. Events are processed within seconds or milliseconds of occurring. Good for alerting, real-time dashboards, and event-driven architectures.

The line blurs with *micro-batching*: small batches processed frequently (every few seconds). Spark Streaming and Kafka Streams use this model, providing streaming semantics with batch-like execution.

### Practical Guidance

**Default to batch for analytics.** Batch is simpler, cheaper, and easier to debug. Most analytics can tolerate hourly or daily refresh. Real-time analytics add complexity—justify it with actual requirements.

**The "real-time" audit—questions to ask:**

| Claim | Question | Usually true answer |
|-------|----------|---------------------|
| "We need real-time analytics" | What's the business cost of 5-minute delay? | Negligible |
| "Events must process immediately" | What breaks if there's a 30-second delay? | Nothing |
| "Users expect instant updates" | Do they, or do they expect it within a refresh? | Within a refresh |
| "We need streaming for scale" | Can batch process your daily volume in an hour? | Yes |

**When streaming is actually required:**

- **Fraud detection** — Sub-second matters. A fraudulent transaction approved is money lost.
- **Live dashboards users are actively watching** — Rare. Most dashboards are glanced at, not monitored.
- **Event-driven architecture backbone** — Kafka as the source of truth, services react to events.
- **Alerting on anomalies** — Detect spikes in error rates, latency, or resource usage.
- **User-facing notifications** — "Your order shipped" should arrive in seconds, not hours.

**When batch wins:**

- **Analytics and reporting** — Aggregations over historical data.
- **ML model training** — Batch over training datasets.
- **Data warehouse ETL** — Transform and load on schedule.
- **Anything aggregated** — Daily summaries, weekly reports, monthly billing.

Batch is 10x simpler to operate. No consumer offset management, no exactly-once semantics, no backpressure handling. Debug by rerunning the job.

**Kafka is the default streaming platform.** Apache Kafka handles high-throughput event streams with durability and replay. Alternatives exist (Pulsar, Redpanda, Kinesis), but Kafka has the largest ecosystem.

**Understand delivery guarantees:**

| Guarantee | Behavior | Use case |
|-----------|----------|----------|
| At-most-once | Fire and forget; may lose messages | Metrics, logs where loss is tolerable |
| At-least-once | May duplicate; consumer handles idempotency | Most applications |
| Exactly-once | No loss, no duplicates; highest complexity | Financial transactions |

**Design for idempotency.** At-least-once delivery (the practical default) means consumers may process the same message twice. Idempotent operations produce the same result regardless of repetition—essential for correctness.

```python
def handle_order_created(event):
    order_id = event['order_id']

    # Check if already processed (idempotency)
    if db.exists(f"processed:{order_id}"):
        return  # Already handled, skip

    # Process the order
    process_order(event)

    # Mark as processed (with TTL for cleanup)
    db.set(f"processed:{order_id}", True, ex=86400 * 7)  # 7 days
```

**Separate streaming infrastructure from application logic.** Kafka, Flink, and their ilk are infrastructure. Application teams shouldn't manage Kafka clusters. Provide managed streaming as a platform service.

> **Common mistake:** Building real-time pipelines because "batch feels old." Real-time is harder, costlier, and often unnecessary. Ask whether users need seconds-fresh data or if minutes-fresh suffices.

### Going Deeper

- [Designing Data-Intensive Applications, Ch. 10-11](https://dataintensive.net/) — Batch and stream processing
- [Kafka: The Definitive Guide](https://www.confluent.io/resources/kafka-the-definitive-guide-v2/) — Comprehensive Kafka reference
- [Streaming Systems](https://www.oreilly.com/library/view/streaming-systems/9781491983867/) — Stream processing fundamentals
- [The Log: What every software engineer should know](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) — LinkedIn's foundational post

### TL;DR

- Ask "what breaks with a 5-minute delay?" — usually nothing
- Streaming for: fraud detection, event-driven architecture, alerting
- Batch for: analytics, ML training, reporting, anything aggregated
- Kafka is the default streaming platform
- Design for idempotency; at-least-once is the practical reality

---

## Data Modeling

### Core Concept

Data modeling defines how information is structured, stored, and related. Good models make queries fast and code simple; bad models create workarounds that compound over time.

**Normalization** eliminates redundancy by splitting data into related tables. A user's name is stored once, referenced by ID elsewhere. Updates happen in one place, consistency is automatic.

**Denormalization** duplicates data for read performance. Store the user's name alongside their orders to avoid joins. Writes become complex (update in multiple places), but reads are fast.

Neither extreme works universally. Normalize transactional data where consistency matters. Denormalize read-heavy paths where latency matters. Most systems use both.

### Practical Guidance

**Model queries, not entities.** Start with the queries your application will run. What data appears together on screen? Store it together. Entity-relationship diagrams help understand the domain, but queries determine the schema.

**Normalize your core domain.** Users, products, orders—the entities your business depends on—should be normalized. Inconsistency here causes real problems: shipping to wrong addresses, double-charging customers.

**Denormalize for read paths.** If a dashboard shows user name, order count, and last order date, consider a denormalized "user summary" table. One query instead of three joins.

**When to denormalize:**

| Signal | Denormalization might help |
|--------|---------------------------|
| Same join appears in every query | Copy the joined data |
| Read:write ratio > 100:1 | Reads benefit; writes tolerate cost |
| Query latency is critical | Avoid joins at read time |
| Relationship is stable | User's name rarely changes |

**When to stay normalized:**

| Signal | Keep normalized |
|--------|-----------------|
| Data changes frequently | Updates would cascade everywhere |
| Consistency is critical | Financial data, inventory |
| Storage is expensive | Don't duplicate unnecessarily |
| Schema is evolving | Denormalized schemas are harder to change |

**Index strategy—the rules from PostgreSQL docs:**

Multicolumn B-tree indexes are most efficient when queries have **equality constraints on leading columns**. The exact rule: equality constraints on leading columns, plus any inequality constraint on the first column without an equality constraint, limits the index scan.

```sql
-- Index on (a, b, c)
CREATE INDEX idx_orders_compound ON orders (user_id, status, created_at);

-- Excellent: equality on leading columns
SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';

-- Good: equality on lead, inequality on next
SELECT * FROM orders WHERE user_id = 123 AND created_at > '2026-01-01';

-- Poor: missing leading column
SELECT * FROM orders WHERE status = 'pending';  -- Scans entire index

-- Useless: only trailing column
SELECT * FROM orders WHERE created_at > '2026-01-01';  -- Full table scan
```

**Partial indexes** index only rows matching a condition. Smaller, faster, cheaper:

```sql
-- Only index active orders (skip 95% of rows)
CREATE INDEX idx_active_orders ON orders (user_id, created_at)
WHERE status = 'active';
```

**The "index everything" trap:** Every index slows writes (index must be updated) and consumes storage. Three or more columns in an index is usually a sign of over-indexing. Profile actual slow queries before adding indexes.

**Handle time carefully.** Store timestamps as `TIMESTAMPTZ` (with timezone information), not `TIMESTAMP`. Historical data needs "as-of" timestamps—when did this price apply? When did this relationship exist?

```sql
-- Always TIMESTAMPTZ for timestamps
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

-- For historical data, track validity periods
CREATE TABLE price_history (
    product_id INTEGER NOT NULL,
    price NUMERIC(10,2) NOT NULL,
    valid_from TIMESTAMPTZ NOT NULL,
    valid_to TIMESTAMPTZ,  -- NULL = current
    CONSTRAINT price_history_pkey PRIMARY KEY (product_id, valid_from)
);
```

**Design for evolution.** Schemas change as requirements evolve. Avoid designs that make changes difficult: stringly-typed fields, God tables with 100 columns, deep polymorphic hierarchies.

> **Common mistake:** Over-normalizing for theoretical purity. A join that appears in every query should probably be denormalized. Pragmatism beats theory.

### Going Deeper

- [Database Design for Mere Mortals](https://www.informit.com/store/database-design-for-mere-mortals-a-hands-on-guide-to-9780136788041) — Relational modeling fundamentals
- [SQL Antipatterns](https://pragprog.com/titles/bksqla/sql-antipatterns/) — Common modeling mistakes
- [PostgreSQL Multicolumn Indexes](https://www.postgresql.org/docs/17/indexes-multicolumn.html) — Official index documentation
- [Use The Index, Luke](https://use-the-index-luke.com/) — SQL indexing deep dive

### TL;DR

- Model queries, not entities—structure data for how it's accessed
- Normalize core domain; denormalize stable, read-heavy paths
- Compound indexes: equality on leading columns, inequality on first non-equality
- Partial indexes for frequently filtered subsets (e.g., active records)
- Always use TIMESTAMPTZ, never TIMESTAMP

---

# Communication

How services talk to each other. API design determines developer experience; messaging patterns determine reliability.

---

## API Design

### Core Concept

APIs are contracts between services. The format you choose affects developer experience, performance, tooling, and evolution. Three patterns dominate: REST for simplicity, GraphQL for flexibility, gRPC for performance.

**REST** uses HTTP verbs and URLs to model resources. It's simple, cacheable, and universally understood. Most APIs should start here.

```
GET    /users/123        → Fetch user
POST   /users            → Create user
PUT    /users/123        → Replace user
PATCH  /users/123        → Update user fields
DELETE /users/123        → Delete user
```

**GraphQL** lets clients request exactly the data they need. One endpoint, flexible queries, strong typing. Good for complex frontends with varying data needs.

```graphql
query {
  user(id: "123") {
    name
    orders(last: 5) {
      total
      status
    }
  }
}
```

**gRPC** uses Protocol Buffers for binary serialization and HTTP/2 for multiplexing. It's faster than JSON over HTTP but requires code generation and has limited browser support.

```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
}
```

### Practical Guidance

**Default to REST.** It's simple, well-understood, and works everywhere. HTTP caching, load balancers, and debugging tools all work out of the box. Choose alternatives when REST's limitations become concrete problems.

**Use GraphQL for complex client needs.** Mobile apps fetching varying data, dashboards aggregating multiple resources, or public APIs with diverse consumers benefit from GraphQL. Don't use it for simple CRUD or server-to-server communication.

**Use gRPC for internal service-to-service communication.** When services exchange data frequently and performance matters, gRPC's binary protocol and streaming reduce latency. But keep REST endpoints for external consumers and debugging.

| Pattern | Choose when... | Avoid when... |
|---------|---------------|---------------|
| REST | Simple CRUD, public APIs, caching matters | Complex nested resources, many round trips |
| GraphQL | Varied client needs, complex queries, federation | Simple APIs, server-to-server, real-time |
| gRPC | High-throughput internal APIs, streaming | Browser clients, simple services, debugging |

**Worked example—when to use what:**

```
Scenario: E-commerce platform with web, mobile, and internal services

Public product API (third-party integrations):
→ REST. Universal compatibility, cacheable, debuggable with curl.

Mobile app fetching user dashboard:
→ REST with sparse fieldsets, or GraphQL if needs are truly varied.
   Most mobile apps don't need GraphQL—they have 3-5 views, not 50.

Order service → Inventory service (internal, high frequency):
→ gRPC. Binary protocol, streaming for batch updates, ~10x faster.

Admin tool calling order service (internal, low frequency):
→ REST. Debuggability matters more than performance.

Same team, same repo, same process:
→ Function calls. Don't add network overhead for local communication.
```

**GraphQL red flags—when "flexibility" is oversold:**

| Claim | Reality check |
|-------|---------------|
| "Flexibility for frontend" | REST with sparse fieldsets (`?fields=id,name`) does this |
| "Reduce round trips" | REST compound endpoints do this |
| "Strong typing" | OpenAPI/Swagger gives you types with REST |
| "Multiple clients with different needs" | Do you actually have multiple clients, or one web app? |

GraphQL makes sense when: you have genuinely diverse clients (web, mobile, partners) with different data needs, and a dedicated team to maintain the schema. That's maybe 5% of applications.

**Design resources, not endpoints.** REST works best when URLs represent nouns (resources), not verbs (actions). `/users/123/activate` is a code smell—consider `PATCH /users/123 {status: "active"}` instead.

**Version your APIs explicitly.** Use URL versioning (`/v1/users`) or header versioning (`Accept: application/vnd.api+json; version=1`). URL versioning is simpler; header versioning is purer. Either works—just be consistent.

**Use consistent error formats.** Define an error schema and stick to it. Include error codes, human-readable messages, and field-level details for validation errors.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": [
      {"field": "email", "message": "Invalid email format"}
    ]
  }
}
```

> **Common mistake:** Choosing GraphQL because it's modern. GraphQL adds complexity: query parsing, N+1 prevention, caching strategies, and authorization per field. Justify the complexity with actual requirements.

### Going Deeper

- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines) — Comprehensive REST guidance
- [GraphQL Best Practices](https://graphql.org/learn/best-practices/) — Official GraphQL patterns
- [gRPC Introduction](https://grpc.io/docs/what-is-grpc/introduction/) — Protocol Buffers and HTTP/2
- [API Design Patterns](https://www.manning.com/books/api-design-patterns) — Manning book on API design

### TL;DR

- Default to REST; it works everywhere
- GraphQL for complex, varying client data needs
- gRPC for high-throughput internal services
- Design resources, not endpoints
- Version APIs explicitly and use consistent error formats

---

## API Security

### Core Concept

Every API is an attack surface. Security isn't a feature you add later—it's built into design decisions from the start. The basics: authenticate requests, validate input, rate limit abuse, and configure CORS correctly.

### Practical Guidance

**Rate limit everything.** Unauthenticated endpoints need aggressive limits (10-100 req/min). Authenticated endpoints need per-user limits. Use token bucket or sliding window algorithms. Return `Retry-After` headers so clients can back off gracefully.

**Validate input at the boundary.** Never trust client data. Use schema validation (Zod, JSON Schema) to reject malformed requests before they reach business logic. Prefer rejection over sanitization—sanitizing can introduce subtle bugs.

**Configure CORS narrowly.** Don't use `*` for allowed origins in production. Explicitly list allowed origins. Credentials require explicit origin (not `*`). Preflight caching (`Access-Control-Max-Age`) reduces OPTIONS request overhead.

**Set security headers.** At minimum: `Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`. Use a framework or middleware that sets these by default.

**GraphQL-specific concerns.** Limit query depth and complexity to prevent resource exhaustion. Disable introspection in production. Field-level authorization is harder than REST—every resolver must check permissions.

> **Common mistake:** Adding authentication but skipping authorization. Knowing *who* made the request isn't enough; you must verify they're *allowed* to perform the action.

### TL;DR

- Rate limit all endpoints; return `Retry-After` headers
- Validate input with schemas; reject don't sanitize
- CORS: explicit origins, no wildcards with credentials
- Set security headers via middleware
- GraphQL: limit depth, disable introspection, authorize per field

---

## Event-Driven Architecture

### Core Concept

Event-driven architecture decouples services through asynchronous messaging. Instead of service A calling service B directly, A publishes an event; B subscribes and reacts. This reduces coupling, improves resilience, and enables independent scaling.

**Events** are immutable records of something that happened: "OrderCreated", "PaymentProcessed", "UserRegistered". Events describe facts, not commands.

**Commands** are requests for action: "CreateOrder", "ProcessPayment". Commands expect a response; events do not.

**Event brokers** (Kafka, RabbitMQ, AWS SNS/SQS) transport events between producers and consumers. They handle delivery, ordering, and durability.

The core benefit is temporal decoupling. Service A publishes without knowing or caring which services consume. New consumers can subscribe without changing producers. Services can process at their own pace.

### Practical Guidance

**Start with synchronous, move to asynchronous when needed.** Async adds complexity: eventual consistency, debugging difficulty, ordering challenges. Use it when you need to decouple services, handle varying processing speeds, or build audit trails—not by default.

**Design events as immutable facts.** "OrderCreated" with order details, not "CreateOrder" as a command. Events should make sense to any consumer, including future consumers you haven't imagined.

```json
{
  "eventType": "OrderCreated",
  "timestamp": "2026-01-26T10:30:00Z",
  "data": {
    "orderId": "ord_123",
    "userId": "usr_456",
    "items": [...],
    "total": 99.99
  }
}
```

**Include enough context in events.** Consumers shouldn't need to call back to the producer for basic information. Include relevant data at the time of the event—user's email when they signed up, not just user ID.

**Handle ordering carefully.** Events for the same entity should be processed in order. Partition by entity ID (Kafka partition key, SQS message group) to ensure ordering within a stream while allowing parallelism across streams.

**Build idempotent consumers.** Events may be delivered more than once. Consumers must handle duplicates gracefully—check if already processed before taking action.

**Event schema evolution—what happens when events need to change:**

Events are contracts. Changing them breaks consumers. Handle evolution carefully:

| Change type | Safe? | Approach |
|-------------|-------|----------|
| Add optional field | Yes | Consumers ignore unknown fields |
| Add required field | No | Make it optional with default, or version the event |
| Remove field | No | Deprecate, stop populating, remove after all consumers updated |
| Rename field | No | Add new field, deprecate old, remove later |
| Change field type | No | New field with new type, deprecate old |

**Always additive, never breaking:**

```json
// Version 1
{"eventType": "OrderCreated", "orderId": "123", "total": 99.99}

// Version 2 - GOOD: added optional field
{"eventType": "OrderCreated", "orderId": "123", "total": 99.99, "currency": "USD"}

// Version 2 - BAD: removed field (breaks consumers expecting total)
{"eventType": "OrderCreated", "orderId": "123", "currency": "USD", "amount": {"value": 99.99, "currency": "USD"}}

// If breaking change required, version the event type
{"eventType": "OrderCreated.v2", ...}
```

**The "exactly-once" myth.** You don't get exactly-once delivery. You get at-least-once with idempotent consumers. Even Kafka's "exactly-once" is actually "effectively exactly-once"—it deduplicates at the broker but consumers can still see duplicates during rebalances.

Build idempotency into every consumer:
- Store processed event IDs with TTL
- Use database constraints (unique on event_id)
- Design operations to be naturally idempotent (set to value, not increment)

**Use the outbox pattern for reliability.** Don't publish events directly from application code. Write events to an "outbox" table in the same database transaction as your data changes, then publish from the outbox. This ensures events are only published if the transaction commits.

```
1. Begin transaction
2. Update orders table
3. Insert into outbox table (event payload)
4. Commit transaction
5. Background process: read outbox, publish to Kafka, mark as published
```

> **Common mistake:** Publishing events before the database transaction commits. If the transaction rolls back, consumers process events for data that doesn't exist.

### Going Deeper

- [Designing Event-Driven Systems](https://www.confluent.io/resources/ebook/designing-event-driven-systems/) — Confluent's free book
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) — Classic patterns reference
- [The Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html) — Reliable event publishing
- [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) — Storing state as events

### TL;DR

- Events are immutable facts; commands are requests
- Start synchronous; add async for specific benefits
- Include enough context in events for consumers
- Partition by entity ID for ordering
- Use the outbox pattern for reliable publishing

---

## Service Mesh

### Core Concept

A service mesh handles cross-cutting concerns—observability, security, traffic management—at the infrastructure layer rather than in application code. Proxies (sidecars) intercept all network traffic, applying policies consistently across services.

**Sidecar proxies** (Envoy, Linkerd-proxy) run alongside each service instance. They handle mTLS encryption, retries, circuit breaking, and telemetry without application changes.

**Control planes** (Istio, Linkerd) configure the proxies. They distribute certificates, define traffic policies, and aggregate telemetry.

The promise is powerful: mTLS everywhere, consistent retry policies, traffic shifting, and distributed tracing—all without changing application code.

### Practical Guidance

**Most teams don't need a service mesh.** Service meshes solve problems of scale and complexity. If you have fewer than 20 services, the mesh's operational overhead likely exceeds its benefits. Libraries and frameworks can handle observability and resilience for smaller deployments.

**Linkerd is simpler than Istio.** Istio is powerful but complex—many teams spend more time operating Istio than benefiting from it. Linkerd focuses on simplicity and performance. Start with Linkerd if you need a mesh.

**Evaluate operational cost honestly.** Service meshes add latency (proxy hops), resource consumption (sidecar memory), and operational complexity (upgrades, debugging). Measure these costs against the benefits for your specific situation.

**Start with specific features, not the whole mesh.** Need mTLS? Consider Linkerd's auto-mTLS. Need observability? Consider standalone tools (Jaeger, Prometheus) before adopting a full mesh. The mesh should solve problems you actually have.

**Ensure your team can operate it.** A service mesh is infrastructure. Debugging requires understanding proxy configuration, control plane health, and certificate rotation. If your team can't operate it, the mesh becomes a liability.

**A concrete trigger—when a mesh actually helped:**

> "We adopted Linkerd when we hit 40 services and couldn't debug cross-service latency without distributed tracing baked in. Before that, application-level retries and a load balancer were fine. The trigger wasn't scale—it was observability. We needed to see why requests were slow without instrumenting every service manually."

The pattern: service meshes solve observability and security at scale. If your pain is something else (deployment, scaling, development velocity), the mesh won't help.

| Feature | Library/Framework Approach | Service Mesh Approach |
|---------|---------------------------|----------------------|
| Retries | Application code or HTTP client | Sidecar proxy |
| mTLS | Certificate management per service | Auto-rotation, mesh-wide |
| Tracing | Instrumentation per service | Automatic span injection |
| Traffic control | Load balancer rules | Fine-grained, dynamic |

> **Common mistake:** Adopting a service mesh because large companies use them. Google and Uber have thousands of services and dedicated platform teams. A mesh that helps them may burden you.

### Going Deeper

- [Linkerd Documentation](https://linkerd.io/docs/) — Simple service mesh
- [Istio Documentation](https://istio.io/latest/docs/) — Feature-rich but complex
- [Service Mesh Comparison](https://servicemesh.io/) — Feature comparison
- [Do You Really Need a Service Mesh?](https://www.nginx.com/blog/do-i-need-a-service-mesh/) — Critical evaluation

### TL;DR

- Service meshes handle mTLS, retries, and observability at infrastructure layer
- Most teams don't need one—evaluate honestly
- Linkerd is simpler than Istio
- Consider standalone tools before adopting a full mesh
- Ensure your team can operate whatever you adopt

---

## Resilience Patterns

### Core Concept

Distributed systems fail. Networks partition, services crash, databases slow down. Resilience patterns help services degrade gracefully rather than cascade failures across the system.

**Circuit breakers** stop calling failing services. After a threshold of failures, the circuit "opens" and requests fail immediately rather than waiting for timeouts. After a cooldown, the circuit allows test requests to check if the service recovered.

```
Closed (normal) → Failures exceed threshold → Open (failing fast)
                                               ↓
                                          Wait timeout
                                               ↓
Half-Open (testing) → Success → Closed
                   → Failure → Open
```

**Retries with backoff** handle transient failures. Retry failed requests with increasing delays (exponential backoff) and random variation (jitter) to prevent thundering herds.

**Bulkheads** isolate failures. Separate thread pools or connection pools for different dependencies prevent one slow service from exhausting resources needed by others.

**Timeouts** bound waiting. Every external call needs a timeout. Without them, slow dependencies block threads indefinitely, eventually exhausting capacity.

### Practical Guidance

**Set timeouts on everything.** Every HTTP client, database connection, and external service call needs a timeout. A missing timeout is a latent outage waiting to happen. Default to aggressive timeouts (1-5 seconds) and adjust based on actual latency.

**Use exponential backoff with jitter.** Linear backoff causes thundering herds when services recover. Exponential backoff with random jitter spreads retry load over time.

```python
def retry_with_backoff(operation, max_retries=3):
    for attempt in range(max_retries):
        try:
            return operation()
        except TransientError:
            if attempt == max_retries - 1:
                raise
            delay = min(2 ** attempt + random.uniform(0, 1), 30)
            time.sleep(delay)
```

**Distinguish transient and permanent failures.** Retry transient failures (network timeouts, 503s). Don't retry permanent failures (400s, 404s). Retrying permanent failures wastes resources and delays error reporting.

**Implement circuit breakers for critical dependencies.** If a payment service fails, orders should degrade gracefully (queue for later) rather than failing entirely. Circuit breakers prevent wasting resources on known-failing services.

**Use bulkheads for isolation.** If your service calls three external APIs, use separate connection pools. One slow API shouldn't exhaust connections needed by the others.

```python
payment_pool = ConnectionPool(max_connections=10)
inventory_pool = ConnectionPool(max_connections=10)
shipping_pool = ConnectionPool(max_connections=10)
```

**Implement health checks that reflect dependencies.** A service might be running but unable to serve requests because its database is down. Health endpoints should check critical dependencies, not just return 200.

**Test failure modes.** Chaos engineering validates resilience patterns actually work. Netflix's Chaos Monkey randomly terminates instances. Smaller teams can start with controlled failure injection in staging.

> **Common mistake:** Implementing retries without backoff. If a service is struggling, immediate retries make things worse. Always use exponential backoff with jitter.

### Going Deeper

- [Release It!](https://pragprog.com/titles/mnee2/release-it-second-edition/) — Stability patterns in depth
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html) — Fowler's explanation
- [Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) — AWS best practices
- [Bulkhead Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/bulkhead) — Microsoft's documentation

### TL;DR

- Set timeouts on every external call
- Use exponential backoff with jitter for retries
- Distinguish transient vs permanent failures—only retry transient
- Circuit breakers and bulkheads isolate failures and dependencies
- Health checks should verify critical dependencies

---

# AI/ML Systems

Machine learning in production systems. The challenge isn't building models—it's serving them reliably at scale.

---

## ML Infrastructure

### Core Concept

ML infrastructure bridges the gap between data science experiments and production systems. Models trained in notebooks must somehow serve millions of predictions with consistent latency and quality.

**Feature stores** provide consistent feature computation across training and serving. During training, you compute features from historical data. During serving, you compute the same features in real-time. Feature stores ensure these computations are identical, preventing training-serving skew.

**Model registries** version and track models. They store model artifacts, metadata (training parameters, performance metrics), and deployment history. When a model degrades, you need to know what changed.

**Model serving** loads models and handles prediction requests. Options range from embedding models in application code to dedicated inference servers (TensorFlow Serving, Triton) to managed services (SageMaker, Vertex AI).

### Practical Guidance

**Start simple.** Many ML systems fail not from model complexity but from infrastructure overhead. A scikit-learn model serialized with joblib and served from a Flask endpoint works for many use cases. Add infrastructure as you hit specific bottlenecks.

**Separate training and serving environments.** Training needs GPUs, large datasets, and experiment flexibility. Serving needs low latency, high availability, and predictable resources. Don't run training jobs on serving infrastructure.

**Use feature stores when feature complexity justifies it.** If you have a few features computed with simple SQL, feature stores add overhead. If you have hundreds of features, complex transformations, and multiple models sharing features, a feature store (Feast, Tecton, AWS Feature Store) prevents drift and duplication.

**Version everything.** Version models, training data, feature definitions, and serving configurations. When model quality degrades, you need to trace back to what changed. MLflow and similar tools provide this lineage.

```yaml
# Model registry metadata
model:
  name: fraud_detection_v3
  version: 3.2.1
  training_data: s3://bucket/fraud_data_2026_01
  features: feature_set_v2.yaml
  metrics:
    auc: 0.94
    precision_at_95_recall: 0.72
  deployed_at: 2026-01-15T10:00:00Z
```

**Monitor model performance in production.** Accuracy on test sets doesn't guarantee production performance. Monitor prediction distributions, feature distributions, and business metrics. Alert on drift before users complain.

**Plan for model updates.** Models need retraining as data changes. Build pipelines that can retrain, validate, and deploy automatically. Shadow deployment (running new models in parallel) validates performance before cutover.

> **Common mistake:** Building elaborate ML infrastructure before validating the model solves the business problem. Infrastructure should scale with proven value, not speculative potential.

### Going Deeper

- [Designing Machine Learning Systems](https://www.oreilly.com/library/view/designing-machine-learning/9781098107956/) — Chip Huyen's comprehensive guide
- [MLOps: Continuous delivery for ML](https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning) — Google's MLOps guide
- [Feast Documentation](https://docs.feast.dev/) — Open-source feature store
- [Made With ML](https://madewithml.com/) — MLOps course and patterns

### TL;DR

- Start simple; add infrastructure as you hit bottlenecks
- Separate training and serving environments
- Feature stores prevent training-serving skew at scale
- Version models, data, and features for traceability
- Monitor production performance; test accuracy isn't enough

---

## Vector Databases

### Core Concept

Vector databases store and search high-dimensional vectors efficiently. They power similarity search: finding items "like" a query based on semantic meaning rather than keyword matching.

**Embeddings** are vector representations of data. Text, images, and audio can be converted to dense vectors (typically 384-1536 dimensions) where similar items have similar vectors. "King" and "queen" are close in embedding space; "king" and "refrigerator" are far apart.

**Approximate nearest neighbor (ANN)** algorithms find similar vectors without comparing against every vector in the database. HNSW (Hierarchical Navigable Small World), IVF (Inverted File Index), and PQ (Product Quantization) trade small accuracy losses for dramatic speed improvements.

**Vector databases** (Pinecone, Weaviate, Milvus, Qdrant, pgvector) combine storage, indexing, and search. They handle the complexity of building and maintaining ANN indexes.

### Practical Guidance

**Start with pgvector if you're already using PostgreSQL.** Adding a vector column to existing tables avoids new infrastructure. pgvector handles millions of vectors with good performance. Only move to dedicated vector databases when you need billions of vectors or specific features.

**Choose embedding models carefully.** Embedding quality matters more than database choice. OpenAI's embeddings, Cohere, and open models (sentence-transformers) have different tradeoffs in quality, cost, and latency. Test on your actual data.

| Model | Dimensions | Quality | Cost | Latency |
|-------|------------|---------|------|---------|
| OpenAI text-embedding-3-large | 3072 | High | Per-token | ~100ms |
| Cohere embed-v3 | 1024 | High | Per-token | ~100ms |
| sentence-transformers (local) | 384-768 | Good | Infrastructure | ~10ms |

**Understand the accuracy-speed tradeoff.** ANN indexes sacrifice some accuracy for speed. An index might return the 95th-best match instead of the best match. For most applications, this is acceptable. For critical applications, test recall rates.

**Combine vector search with traditional filtering.** "Find similar products under $50 in the electronics category" combines vector similarity with metadata filters. Most vector databases support hybrid queries. Design your schema to support both.

```python
results = index.query(
    vector=query_embedding,
    filter={"category": "electronics", "price": {"$lt": 50}},
    top_k=10
)
```

**Plan for index updates.** Adding new vectors is cheap. Updating or deleting vectors may require index rebuilding depending on the algorithm. Understand your database's update characteristics before committing.

**Batch embedding generation.** Embedding APIs are often the bottleneck. Batch requests (10-100 texts per call) reduce latency and cost. Pre-compute embeddings for static content.

> **Common mistake:** Choosing a vector database based on benchmarks. Benchmarks test specific configurations. Your query patterns, update frequency, and scale may produce different results. Test with your actual workload.

### Going Deeper

- [What are Vector Databases?](https://www.pinecone.io/learn/vector-database/) — Pinecone's explanation
- [HNSW Algorithm Explained](https://www.pinecone.io/learn/hnsw/) — How ANN indexing works
- [pgvector Documentation](https://github.com/pgvector/pgvector) — PostgreSQL vector extension
- [Embedding Models Comparison](https://huggingface.co/spaces/mteb/leaderboard) — MTEB benchmark leaderboard

### TL;DR

- Embeddings represent semantic meaning as vectors
- Start with pgvector if using PostgreSQL
- Embedding model quality matters more than database choice
- ANN trades small accuracy loss for major speed gains
- Combine vector search with metadata filtering
- Batch embedding generation to reduce latency

---

## LLM Integration

### Core Concept

Large language models (LLMs) generate text, answer questions, and perform tasks described in natural language. Integrating them into production systems requires managing latency, cost, reliability, and output quality.

**Prompts** instruct the model. Prompt engineering—crafting inputs that produce desired outputs—is often more important than model choice. Small prompt changes can dramatically affect quality.

**Tokens** are the units LLMs process. Roughly 4 characters per token for English. Input and output tokens are priced separately; longer prompts and responses cost more.

**Context windows** limit how much text the model can process at once. GPT-4 handles 128K tokens; Claude handles 200K. Fitting relevant information in the context window is a key design challenge.

### Practical Guidance

**Use the smallest model that works.** GPT-4 is more capable than GPT-3.5-turbo, but 20-30x more expensive. Many tasks—classification, extraction, simple Q&A—work fine with smaller, cheaper models. Test before defaulting to the largest model.

| Model Tier | Cost (approx) | Use When |
|------------|---------------|----------|
| Small (GPT-3.5, Claude Haiku) | $0.50/M tokens | Classification, extraction, simple tasks |
| Medium (GPT-4o, Claude Sonnet) | $5/M tokens | Complex reasoning, nuanced understanding |
| Large (GPT-4, Claude Opus) | $15-60/M tokens | Hardest tasks, highest quality required |

**Design for latency.** LLM responses take 500ms-30s depending on model and output length. Stream responses for interactive applications. Use async processing for non-interactive workflows. Set user expectations appropriately.

**Handle failures gracefully.** LLM APIs have rate limits, occasional outages, and variable latency. Implement retries with backoff, circuit breakers, and fallback behaviors. A chatbot that shows "service unavailable" is better than one that hangs.

**Validate and constrain outputs.** LLMs can produce incorrect, inconsistent, or harmful outputs. Validate structured outputs against schemas. Filter or moderate content. Never trust LLM output for security-critical decisions.

```python
from pydantic import BaseModel

class ProductRecommendation(BaseModel):
    product_id: str
    reason: str
    confidence: float

# Parse LLM output into structured format
response = llm.generate(prompt)
recommendation = ProductRecommendation.model_validate_json(response)
```

**Log prompts and responses.** Debugging LLM systems requires seeing what went in and what came out. Log prompts, responses, and metadata (model, latency, tokens used). This data also powers improvement through prompt iteration.

**Consider fine-tuning last.** Fine-tuning is expensive and requires data pipelines. Most applications can achieve good results with prompt engineering. Only fine-tune when you have clear evidence that prompts are insufficient and you have quality training data.

> **Common mistake:** Putting sensitive data in prompts without considering where it goes. LLM providers may log requests for training or abuse detection. For sensitive data, use providers with appropriate data handling agreements or self-hosted models.

### Going Deeper

- [Prompt Engineering Guide](https://www.promptingguide.ai/) — Comprehensive prompting techniques
- [OpenAI Cookbook](https://cookbook.openai.com/) — Practical LLM patterns
- [Anthropic's Claude Documentation](https://docs.anthropic.com/) — Claude-specific guidance
- [LLM Security Best Practices](https://owasp.org/www-project-top-10-for-large-language-model-applications/) — OWASP LLM security guide

### TL;DR

- Use the smallest model that produces acceptable results
- Design for latency: stream responses, use async processing
- Implement retries and fallbacks; LLM APIs fail
- Validate and constrain outputs; never trust blindly
- Log prompts and responses for debugging and improvement
- Prompt engineering before fine-tuning

---

## RAG Architecture

### Core Concept

Retrieval-Augmented Generation (RAG) combines search with LLMs. Instead of relying solely on what the model learned during training, RAG retrieves relevant documents and includes them in the prompt. This grounds responses in specific, current information.

**The RAG pipeline:**

1. **Query** — User asks a question
2. **Retrieve** — Search for relevant documents (vector search, keyword search, or hybrid)
3. **Augment** — Add retrieved documents to the prompt as context
4. **Generate** — LLM produces a response grounded in the retrieved context

```
User: "What's our refund policy?"
     ↓
Retrieve: [policy_doc_v3.txt, faq_refunds.txt]
     ↓
Prompt: "Based on these documents: {docs}. Answer: What's our refund policy?"
     ↓
Response: "Our refund policy allows returns within 30 days..."
```

RAG solves key LLM limitations: it provides access to private data the model wasn't trained on, current information beyond the training cutoff, and verifiable sources for claims.

### Practical Guidance

**Chunk documents appropriately.** Documents must be split into chunks that fit in the context window. Chunks too small lose context; chunks too large dilute relevance. Start with 500-1000 tokens per chunk with 10-20% overlap.

```python
# Simple chunking with overlap
def chunk_text(text, chunk_size=500, overlap=50):
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start = end - overlap
    return chunks
```

**Retrieval quality determines response quality.** If you retrieve irrelevant documents, the LLM produces irrelevant answers. Invest in retrieval: test different embedding models, tune chunk sizes, consider hybrid search (combining vector and keyword search).

**Include metadata in chunks.** Knowing a chunk came from "policy_v3.pdf, page 5" enables citations, filtering by source, and debugging. Store metadata alongside vectors.

**Rerank retrieved results.** Initial retrieval (ANN search) optimizes for speed. Reranking with a cross-encoder model improves precision by more carefully comparing query and document. The cost is additional latency.

```
1. Vector search: top 20 candidates (fast, approximate)
2. Reranker: select top 5 (slower, more accurate)
3. Generate from top 5 documents
```

**Handle "no relevant documents" gracefully.** Sometimes retrieval finds nothing relevant. The LLM should acknowledge this rather than hallucinating. Include instructions in the prompt: "If the provided documents don't contain the answer, say so."

**Evaluate end-to-end.** RAG systems have many failure modes: bad chunking, poor embeddings, insufficient retrieval, incorrect generation. Build evaluation sets that test the full pipeline, not just individual components.

| Failure Mode | Symptom | Fix |
|--------------|---------|-----|
| Bad chunking | Context split mid-sentence | Adjust chunk boundaries |
| Poor retrieval | Irrelevant docs returned | Better embeddings, hybrid search |
| Insufficient context | Partial answers | Retrieve more documents |
| Hallucination | Claims not in docs | Prompt engineering, citations |

> **Common mistake:** Building RAG without evaluation data. You can't improve what you can't measure. Create question-answer pairs from your documents and test retrieval recall and answer quality.

### Going Deeper

- [RAG Survey Paper](https://arxiv.org/abs/2312.10997) — Academic overview of RAG techniques
- [LangChain RAG Tutorial](https://python.langchain.com/docs/tutorials/rag/) — Practical implementation
- [Anthropic's Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) — Advanced RAG techniques
- [Evaluation of RAG Systems](https://www.trulens.org/) — TruLens evaluation framework

### TL;DR

- RAG retrieves relevant documents and includes them in LLM prompts
- Chunk documents at 500-1000 tokens with overlap
- Retrieval quality determines response quality—invest here
- Reranking improves precision at the cost of latency
- Handle "no relevant documents" explicitly
- Build evaluation sets to measure the full pipeline

---

# Operations

Running systems in production. Building software is the easy part; operating it reliably is the challenge.

---

## Observability

### Core Concept

Observability is the ability to understand a system's internal state from its external outputs. When things break—and they will—observability determines whether you spend minutes or hours finding the cause.

The **three pillars of observability** are complementary signals:

**Logs** are discrete events with timestamps and context. "User 123 logged in at 10:30:05." Logs answer "what happened" but become overwhelming at scale.

**Metrics** are numeric measurements over time. Request count, error rate, latency percentiles. Metrics answer "how much" and "how fast" and work well for alerting.

**Traces** follow requests across services. A single user action might touch 10 services; traces connect those calls into a coherent story. Traces answer "where did time go" and "what path did this request take."

Each pillar has strengths and weaknesses. Used together, they provide comprehensive visibility.

### Practical Guidance

**Instrument from the start.** Adding observability to a running system is painful. Embed logging, metrics, and tracing in your development workflow. Every new service should emit signals from day one.

**Structure your logs.** Unstructured logs ("ERROR: something went wrong") are nearly useless at scale. Structured logs (JSON with consistent fields) enable filtering, aggregation, and correlation.

```json
{
  "timestamp": "2026-01-26T10:30:05Z",
  "level": "error",
  "service": "payment-api",
  "trace_id": "abc123",
  "user_id": "usr_456",
  "message": "Payment failed",
  "error_code": "INSUFFICIENT_FUNDS",
  "amount": 99.99
}
```

> **PII in logs:** Log user IDs, not emails or names. Never log passwords, tokens, or full credit card numbers. If you need PII for debugging, ensure you have explicit purpose, retention limits, and access controls. GDPR and similar regulations apply to log data.

**Use consistent identifiers.** Trace IDs, request IDs, and user IDs should flow through all logs, metrics, and traces. When investigating an incident, you need to correlate signals across services.

**Alert on symptoms, not causes.** Users experience symptoms: errors, latency, unavailability. Alert on these. "Error rate > 1%" is better than "CPU > 80%"—high CPU might not affect users, and user-affecting problems might not show in CPU.

**Measure the four golden signals.** Google's SRE book recommends: latency (how long requests take), traffic (how many requests), errors (how many fail), and saturation (how full are resources). These cover most user-impacting issues.

| Signal | What it measures | Example metrics |
|--------|-----------------|-----------------|
| Latency | Request duration | p50, p95, p99 response time |
| Traffic | Request volume | Requests per second |
| Errors | Failure rate | Error count, error percentage |
| Saturation | Resource utilization | CPU, memory, queue depth |

**Sample traces in production.** Tracing every request is expensive at scale. Sample a percentage (1-10%) of requests. For debugging specific issues, enable 100% tracing temporarily with filters.

**Use OpenTelemetry.** OpenTelemetry provides vendor-neutral instrumentation for all three pillars. Instrument once, send to any backend (Datadog, Honeycomb, Jaeger, Grafana). Avoid vendor lock-in.

**Combat alert fatigue ruthlessly.** Alert fatigue is the #1 observability failure. When on-call engineers ignore alerts, you have no alerting.

Signs of alert fatigue:
- Alerts acknowledged without investigation
- "Expected" alerts that nobody fixes
- On-call dreading their rotation
- Pages for non-actionable issues

Fixes:
- **SLO-based alerts**: Alert on user impact, not system metrics. "Error rate above SLO" beats "CPU > 80%"
- **Delete alerts that don't lead to action**: If the runbook says "wait and see," delete the alert
- **Fix noisy alerts immediately**: A flaky alert is worse than no alert
- **Review alert volume weekly**: Trending up = problem

Every page should be: urgent, actionable, and user-impacting. Everything else is noise.

> **Common mistake:** Collecting data without using it. Terabytes of logs are worthless if nobody queries them. Start with questions you need to answer, then collect the data to answer them.

### Going Deeper

- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) — Four golden signals
- [Observability Engineering](https://www.oreilly.com/library/view/observability-engineering/9781492076438/) — Comprehensive guide
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/) — Vendor-neutral instrumentation
- [Distributed Tracing in Practice](https://www.oreilly.com/library/view/distributed-tracing-in/9781492056621/) — Tracing deep dive

### TL;DR

- Logs, metrics, and traces are complementary—use all three
- Structure logs as JSON; flow trace IDs through all services
- Alert on symptoms (error rate, latency) not causes (CPU)
- Measure the four golden signals: latency, traffic, errors, saturation
- Use OpenTelemetry for vendor-neutral instrumentation

---

## Incident Management

### Core Concept

Incidents are unplanned interruptions to service. How you handle them determines both the immediate user impact and whether you'll face the same incident again.

**Incident response** coordinates people and actions during an outage. Clear roles prevent chaos: someone declares the incident, someone coordinates response, someone communicates status.

**Postmortems** (or retrospectives) analyze incidents after resolution. The goal isn't blame—it's understanding what happened and preventing recurrence. Good postmortems are blameless, thorough, and produce concrete action items.

**On-call** rotations ensure someone is always available to respond. On-call works when it's sustainable: reasonable frequency, clear escalation paths, and compensation for the burden.

### Practical Guidance

**Define incident severity levels.** Not all incidents are equal. A brief slowdown differs from a complete outage. Clear severity definitions enable appropriate response.

| Severity | Definition | Response |
|----------|------------|----------|
| SEV1 | Complete outage, all users affected | All hands, immediate |
| SEV2 | Major feature broken, many users affected | On-call + backup, urgent |
| SEV3 | Minor feature broken, some users affected | On-call, business hours |
| SEV4 | Degradation, minimal user impact | Normal prioritization |

**Assign clear incident roles.** At minimum: Incident Commander (coordinates), Communications Lead (updates stakeholders), and technical responders. In small teams, one person may hold multiple roles, but roles should still be explicit.

**Communicate proactively.** Users accept outages better when informed. Status pages, proactive emails, and honest timelines build trust. "We're investigating" followed by silence erodes it.

**Document as you go.** During an incident, someone should keep a timeline: what was observed, what actions were taken, what the results were. This timeline becomes the postmortem's foundation.

**Write blameless postmortems.** Humans make mistakes; systems should prevent those mistakes from causing outages. Focus on systemic failures, not individual blame. "The deployment process allowed untested code" not "Alice deployed bad code."

Postmortem template:
```markdown
## Incident Summary
One-paragraph description of what happened.

## Timeline
- 10:30 - Alert fired for increased error rate
- 10:35 - On-call acknowledged, began investigation
- 10:45 - Root cause identified: database connection exhaustion
- 11:00 - Mitigation applied: increased connection pool
- 11:15 - All clear

## Root Cause
What actually caused the incident.

## Impact
Users affected, duration, business impact.

## Lessons Learned
What went well, what went poorly.

## Action Items
Specific, assigned, tracked improvements.
```

**Follow up on action items.** Postmortems without follow-through are theater. Track action items, assign owners, and review completion. The same incident recurring is a leadership failure.

**Make on-call sustainable.** Frequent pages burn out engineers. Noisy alerts desensitize responders. Invest in alert quality: every page should be actionable, every false positive should be fixed.

> **Common mistake:** Blaming individuals in postmortems. If one engineer could cause an outage, your systems have bigger problems than that engineer. Focus on systemic improvements.

### Going Deeper

- [Google SRE Book: Postmortem Culture](https://sre.google/sre-book/postmortem-culture/) — Blameless postmortems
- [PagerDuty Incident Response](https://response.pagerduty.com/) — Comprehensive incident guide
- [Atlassian Incident Management](https://www.atlassian.com/incident-management) — Process and templates
- [FireHydrant](https://firehydrant.com/) — Incident management platform

### TL;DR

- Define severity levels with clear criteria and response expectations
- Assign explicit incident roles: commander, communications, technical
- Communicate proactively through status pages and updates
- Blameless postmortems with tracked action items—same incident twice is failure
- Make on-call sustainable: fix noisy alerts, reasonable rotation

---

## Deployment Strategies

### Core Concept

Deployment strategies control how new code reaches production. The goal is delivering changes quickly while minimizing risk. Different strategies trade off speed, safety, and complexity.

**Rolling deployment** gradually replaces old instances with new ones. At any moment, both versions run. Simple and supported by default in Kubernetes.

**Blue-green deployment** maintains two identical environments. "Blue" runs current production; "green" gets the new version. After validation, traffic switches from blue to green. Fast rollback (switch back to blue), but requires double infrastructure.

**Canary deployment** routes a small percentage of traffic to the new version first. If metrics look good, gradually increase the percentage. Catches problems before they affect all users.

**Feature flags** separate deployment from release. Code ships to production but features activate conditionally. Enables A/B testing, gradual rollouts, and instant kill switches.

### Practical Guidance

**Start with rolling deployments.** They're simple, built into container orchestrators, and sufficient for many teams. Add sophistication when you have specific problems to solve.

**Use canary for high-risk changes.** Changes to critical paths, significant refactors, or new features benefit from canary validation. Route 1-5% of traffic to canary, monitor error rates and latency, then expand.

```yaml
# Kubernetes canary with Argo Rollouts
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 5
      - pause: {duration: 10m}
      - setWeight: 20
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 10m}
```

**Automate rollback triggers.** Canary deployments should automatically roll back when error rates spike or latency increases. Manual monitoring doesn't scale; automated analysis does.

**Feature flags for decoupling deploy and release.** Ship code frequently (multiple times per day). Release features deliberately (when ready for users). Flags enable this separation.

```python
def get_recommendations(user):
    if feature_flags.is_enabled("new_recommendation_algo", user):
        return new_algorithm(user)
    return old_algorithm(user)
```

**Clean up old feature flags.** Feature flags accumulate if not removed. Dead flags are technical debt: confusing code, potential bugs. Set expiration dates and review regularly.

**Test rollback procedures.** Rollback under pressure isn't the time to learn the process. Practice rolling back in staging. Ensure your team knows the commands, has the permissions, and understands the impact.

| Strategy | Pros | Cons | Use when |
|----------|------|------|----------|
| Rolling | Simple, minimal resources | Both versions run together | Default choice |
| Blue-green | Fast rollback, isolated testing | Double infrastructure | Fast rollback critical |
| Canary | Validates with real traffic | Complex setup, slower | High-risk changes |
| Feature flags | Flexible, instant disable | Code complexity | Gradual feature rollout |

> **Common mistake:** Complex deployment strategies without the observability to support them. Canary deployments require metrics that can detect problems. Without good observability, you're just hoping.

### Going Deeper

- [Kubernetes Deployment Strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) — Rolling deployments
- [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) — Progressive delivery for Kubernetes
- [Feature Flags Best Practices](https://launchdarkly.com/blog/feature-flags-best-practices/) — LaunchDarkly's guidance
- [Continuous Delivery](https://continuousdelivery.com/) — Foundational concepts

### TL;DR

- Rolling deployments are the default; add complexity for specific needs
- Canary routes small traffic percentage to validate before full rollout
- Feature flags separate deployment (ship code) from release (enable feature)
- Automate rollback triggers based on error rates and latency
- Clean up old feature flags—they're technical debt
- Test rollback procedures before you need them

---

## Cost Optimization

### Core Concept

Cloud costs grow with scale—and often faster than revenue. Cost optimization isn't about spending less; it's about spending efficiently: getting maximum value from every dollar.

**Unit economics** measure cost per business outcome. Cost per request, cost per user, cost per transaction. Absolute costs matter less than cost efficiency as you scale.

**Reserved capacity** (Reserved Instances, Savings Plans, committed use) discounts steady-state workloads. You commit to usage; providers discount in return. 30-60% savings are typical.

**Spot/preemptible instances** offer deep discounts (60-90%) for interruptible workloads. The provider can reclaim instances with minimal notice. Good for batch processing, CI/CD, and fault-tolerant workloads.

### Practical Guidance

**Establish cost visibility first.** You can't optimize what you can't see. Tag all resources by team, service, and environment. Use cost allocation tools (AWS Cost Explorer, GCP Billing) to understand where money goes.

**Right-size before reserving.** Reserved capacity for oversized instances locks in waste. Analyze actual utilization, right-size instances, then commit. Most instances are larger than needed.

```
Step 1: Tag resources (who owns what)
Step 2: Analyze utilization (CPU, memory, network)
Step 3: Right-size (smaller instances where appropriate)
Step 4: Reserve (commit to discounted pricing)
```

**Use spot instances for fault-tolerant workloads.** Batch jobs, stateless services behind load balancers, and CI/CD pipelines can tolerate interruption. Mix spot and on-demand for balance.

**Shut down non-production environments.** Development and staging environments often run 24/7 but are used 8-10 hours per day. Automatic shutdown during nights and weekends cuts non-prod costs 60-70%.

**Optimize data transfer costs.** Data egress (leaving cloud provider) is expensive. Keep data processing close to storage. Use CDNs for global distribution—they're often cheaper than egress.

**Monitor and alert on cost anomalies.** A misconfigured service can run up thousands in hours. Set budgets and alerts. Investigate anomalies immediately.

| Cost Category | Optimization Levers |
|---------------|---------------------|
| Compute | Right-sizing, reserved capacity, spot instances |
| Storage | Tiered storage (hot/warm/cold), lifecycle policies |
| Data transfer | Keep data close, CDN for distribution |
| Databases | Right-size, reserved capacity, connection pooling |
| Non-production | Schedule shutdown, smaller instances |

**Make cost visible to teams.** When teams see their costs, they optimize. Allocate costs to teams, include in dashboards, and discuss in planning. Hidden costs stay high.

**Review regularly.** Cloud pricing changes. New instance types offer better price-performance. Reserved instances expire. Monthly or quarterly cost reviews catch optimization opportunities.

> **Common mistake:** Optimizing prematurely. If you're early-stage, engineer time is more expensive than cloud costs. Optimize when cloud spend is significant, not when it's $500/month.

### Going Deeper

- [AWS Cost Optimization Pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html) — AWS best practices
- [GCP Cost Optimization](https://cloud.google.com/architecture/framework/cost-optimization) — GCP guidance
- [FinOps Foundation](https://www.finops.org/) — Cloud financial management practices
- [The Frugal Architect](https://thefrugalarchitect.com/) — Werner Vogels on cost-conscious architecture

### TL;DR

- Tag resources and establish cost visibility first
- Right-size before committing to reserved capacity
- Use spot instances for fault-tolerant workloads (60-90% savings)
- Shut down non-production environments during off-hours
- Make costs visible to teams; hidden costs stay high
- Review regularly—pricing and usage change

---

## Zero-Trust Architecture

### Core Concept

Zero trust assumes the network is compromised. Instead of trusting traffic inside a perimeter, every request is authenticated, every connection is encrypted, and every access is logged. "Never trust, always verify."

Traditional security relies on perimeter defense: a firewall protects the inside from the outside. Once you're inside the network, you're trusted. This fails when attackers breach the perimeter or when insiders go rogue.

Zero trust inverts this. Network location grants no trust. A request from inside the data center is treated the same as a request from the public internet. Identity and context—not IP address—determine access.

### Practical Guidance

**Start with mTLS for service-to-service communication.** Mutual TLS authenticates both ends of a connection and encrypts traffic. Service meshes (Linkerd, Istio) provide this automatically. Without a mesh, use certificate management tools (cert-manager, Vault).

**mTLS operational requirements:**

| Aspect | Guidance |
|--------|----------|
| Certificate rotation | Automate with short-lived certs (hours, not months). cert-manager or mesh handles this. |
| Private CA | Your CA is a critical secret. Compromise means attackers mint valid certs. Use HSMs or managed CA (AWS Private CA, Google CAS). |
| Workload identity | Consider [SPIFFE/SPIRE](https://spiffe.io/) for identity without static secrets. |
| Monitoring | Alert on cert expiry, failed handshakes, and CA operations. |

> **Common mistake:** Setting up mTLS but never rotating certificates. Treat cert rotation like password rotation—automate it from day one.

**Use identity-aware proxies for internal tools.** Don't expose admin panels to the internet behind a VPN. Use identity-aware proxies (Google IAP, Cloudflare Access, Pomerium) that authenticate users before allowing access. The tool never sees unauthenticated traffic.

**Treat VPNs as legacy.** VPNs provide network access, not application access. Once on the VPN, users can reach everything. Zero trust provides application-specific access: authenticate to the application, not the network.

| Traditional VPN | Zero Trust |
|-----------------|------------|
| Network-level access | Application-level access |
| Trust after connection | Verify every request |
| All-or-nothing access | Least-privilege per app |
| Single point of entry | Identity everywhere |

**Implement least privilege everywhere.** Services should have the minimum permissions needed. Database credentials should be scoped to specific tables. API keys should be scoped to specific operations. Broad permissions are attack vectors.

**Log everything access-related.** Authentication attempts, authorization decisions, and resource access should all be logged. When (not if) a breach occurs, logs determine the scope of damage.

**When zero trust is overkill:** If you have fewer than 10 services, no compliance requirements, and a small team, start simpler. Use strong authentication (SSO, MFA) and encrypted connections. Graduate to full zero trust as you scale.

> **Common mistake:** Treating VPN as security. VPNs protect network traffic in transit but grant broad access once connected. An attacker with VPN credentials can reach everything. Zero trust limits blast radius.

### Going Deeper

- [BeyondCorp: A New Approach to Enterprise Security](https://research.google/pubs/pub43231/) — Google's zero-trust paper
- [NIST Zero Trust Architecture](https://www.nist.gov/publications/zero-trust-architecture) — US government framework
- [Cloudflare Access](https://www.cloudflare.com/products/zero-trust/access/) — Identity-aware proxy
- [Pomerium](https://www.pomerium.com/) — Open-source identity-aware proxy

### TL;DR

- Zero trust assumes the network is compromised
- mTLS for service-to-service is highest ROI
- Identity-aware proxies replace VPNs for internal tools
- VPNs grant network access; zero trust grants application access
- Implement least privilege; log all access decisions
- Start simpler if small; graduate to zero trust as you scale

---

## Disaster Recovery

### Core Concept

Disaster recovery (DR) plans for catastrophic failures: region outages, data center fires, ransomware attacks. The goal isn't preventing disasters—it's recovering from them fast enough that the business survives.

Two metrics define DR requirements:

**Recovery Time Objective (RTO)**: How long can you be down? An e-commerce site during Black Friday has different RTO than an internal HR tool.

**Recovery Point Objective (RPO)**: How much data can you lose? Financial systems may need zero data loss. Analytics pipelines may tolerate losing a day.

### Practical Guidance

**Know your tier explicitly.** Different applications need different DR:

| Tier | RTO | RPO | Cost | Approach |
|------|-----|-----|------|----------|
| 1 | Minutes | Zero | $$$ | Multi-region active-active |
| 2 | Hours | Minutes | $$ | Warm standby in second region |
| 3 | Days | Hours | $ | Backups + runbook |

Most startups are Tier 3, and that's fine. Know your tier explicitly rather than assuming you're Tier 1 and not funding it.

**Tier 3 (most applications)**: Regular backups, documented recovery procedures, tested annually. Acceptable for internal tools, non-critical applications, early-stage startups.

**Tier 2 (important but not critical)**: Warm standby in second region. Database replicas, pre-provisioned infrastructure that can be activated. Recovery in hours. Acceptable for most SaaS applications.

**Tier 1 (business-critical)**: Active-active across regions. Traffic automatically fails over. Zero downtime during regional outages. Required for payment processing, healthcare, critical infrastructure.

**Test recovery or you don't have DR.** A backup you've never restored isn't a backup—it's hope. An untested runbook is fiction. Annual DR drills minimum:

```
DR Drill checklist:
[ ] Restore database from backup to isolated environment
[ ] Verify data integrity after restore
[ ] Bring up application stack from scratch
[ ] Measure actual recovery time
[ ] Document what broke and fix it
```

**Automate what you can.** Manual runbooks fail under pressure. Automate database restoration, infrastructure provisioning, and DNS failover. The fewer manual steps, the faster and more reliable recovery.

**Consider blast radius.** A single-region deployment has a large blast radius: region outage = complete outage. Multi-region reduces blast radius but adds complexity. Match investment to risk tolerance.

**Separate backups from primary infrastructure.** If ransomware encrypts your backups along with your data, you have no backups. Store backups in a different account, different region, with different credentials. Air-gapped backups for the most critical data.

> **Common mistake:** Assuming cloud provider handles DR. Cloud providers handle their infrastructure failures, not yours. If you delete your database, they don't have a backup unless you configured one. DR is your responsibility.

### Going Deeper

- [AWS Disaster Recovery](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html) — DR patterns on AWS
- [Google Cloud DR Planning](https://cloud.google.com/architecture/dr-scenarios-planning-guide) — GCP guidance
- [Chaos Engineering](https://principlesofchaos.org/) — Testing failure modes
- [Backblaze Backup Statistics](https://www.backblaze.com/blog/backblaze-drive-stats-for-2024/) — Why backups matter

### TL;DR

- RTO = how long down; RPO = how much data lost
- Know your tier: most startups are Tier 3 (backups + runbook)
- If you haven't tested recovery, you don't have DR
- Automate restoration; manual runbooks fail under pressure
- Separate backups from primary infrastructure (different account, credentials)
- Cloud providers handle their failures, not yours—DR is your job

---

# Appendix

---

## Latency Numbers Every Engineer Should Know

Understanding latency orders of magnitude helps you make informed architecture decisions. These numbers are approximate and vary by hardware, but the relative magnitudes remain stable.

### The Numbers (2026)

| Operation | Latency | Notes |
|-----------|---------|-------|
| L1 cache reference | 1 ns | On-CPU cache |
| L2 cache reference | 4 ns | On-CPU cache |
| L3 cache reference | 12 ns | Shared CPU cache |
| Main memory reference | 100 ns | RAM access |
| SSD random read | 16 μs | NVMe SSD |
| SSD sequential read (1 MB) | 50 μs | NVMe SSD |
| HDD random read | 2 ms | Spinning disk (if you still have them) |
| Round trip same datacenter | 500 μs | Network within AZ |
| Round trip cross-region | 50-150 ms | US East to US West: ~60ms |
| Round trip cross-continent | 100-300 ms | US to Europe: ~80ms; US to Asia: ~150ms |

### Derived Numbers

| Operation | Approximate Latency |
|-----------|---------------------|
| Read 1 MB from memory | 3 μs |
| Read 1 MB from SSD | 50 μs |
| Read 1 MB from network (1 Gbps) | 10 ms |
| Database query (indexed, same region) | 1-5 ms |
| Database query (complex, same region) | 10-100 ms |
| Redis GET (same region) | 0.5-1 ms |
| HTTP request (same region) | 1-10 ms |
| HTTP request (cross-region) | 50-200 ms |
| LLM API call (streaming start) | 200-500 ms |
| LLM API call (full response) | 1-30 s |

### Visual Scale

```
1 ns   █
10 ns  ██████████
100 ns ████████████████████████████████████████████████████████████████████████████████████████████████████
1 μs   [████████████████████████████████████████████████████████████████████████████████████████████████████] x 10
1 ms   [above] x 1,000
1 s    [above] x 1,000,000
```

A 100ms network round trip is 100 million times slower than an L1 cache hit.

### What This Means

**Memory vs. network is 1,000x-1,000,000x.** Accessing data over the network is vastly slower than accessing it locally. This is why caching works so well.

**Same-datacenter vs. cross-region is 100x-1,000x.** Keep data and computation close together. A service calling a database across the ocean adds 100ms+ to every query.

**SSD vs. memory is 100x-1,000x.** SSDs are fast but memory is faster. Keep hot data in memory; accept disk latency for cold data.

**Sequential vs. random is 10x-100x.** Sequential access patterns (streaming, batch processing) are dramatically faster than random access. Design for sequential access when possible.

### Practical Implications

1. **Cache aggressively.** Moving a database query (5ms) to Redis (1ms) to in-memory (0.01ms) provides orders-of-magnitude improvement.

2. **Colocate data and compute.** Cross-region calls add 50-150ms. If your frontend is in us-east-1, your backend and database should be too.

3. **Batch network calls.** 10 sequential 5ms calls = 50ms. 1 batched call = 5ms. Prefer batch APIs.

4. **Budget your latency.** If your SLA is 200ms, and network round-trip is 100ms, you have 100ms for everything else. Know your budget.

5. **Measure, don't assume.** These numbers are approximations. Your actual latency depends on hardware, load, and configuration. Measure your system.

### Sources

These numbers are updated from Jeff Dean's original "Latency Numbers Every Programmer Should Know" (2012), adjusted for modern hardware (NVMe SSDs, faster networks, 2026 data).

- [Original Google latency numbers](https://static.googleusercontent.com/media/research.google.com/en//people/jeff/stanford-295-talk.pdf)
- [Colin Scott's interactive version](https://colin-scott.github.io/personal_website/research/interactive_latency.html)
- [Simon Eskildsen's updated numbers](https://github.com/sirupsen/napkin-math)

---

## Tools Reference

Opinionated tool recommendations by category. These aren't exhaustive—they're starting points that work for most teams.

### Databases

| Category | Recommended | Alternatives | Notes |
|----------|-------------|--------------|-------|
| **Relational (default)** | PostgreSQL | MySQL, MariaDB | Start here. Handles more than you expect. |
| **Document** | MongoDB, DynamoDB | Firestore, Couchbase | When flexible schemas matter more than joins. |
| **Wide-column** | ScyllaDB | Cassandra, HBase | Write-heavy, predictable query patterns. |
| **Graph** | Neo4j | Amazon Neptune, TigerGraph | When relationships are first-class. |
| **Time-series** | TimescaleDB | InfluxDB, QuestDB | TimescaleDB is PostgreSQL under the hood. |
| **Key-value/Cache** | Redis | Memcached, DragonflyDB | Redis does more than caching. |
| **Search** | Elasticsearch | OpenSearch, Meilisearch | Full-text search, log aggregation. |
| **Vector** | pgvector | Pinecone, Weaviate, Qdrant | Start with pgvector if using Postgres. |

### Message Queues & Streaming

| Category | Recommended | Alternatives | Notes |
|----------|-------------|--------------|-------|
| **Streaming platform** | Kafka | Redpanda, Pulsar | Redpanda is Kafka-compatible, simpler ops. |
| **Task queue** | Redis + worker | Celery, Sidekiq, BullMQ | Language-specific; choose by ecosystem. |
| **Managed queue** | SQS | Cloud Pub/Sub, Azure Queue | Simple, managed, scales automatically. |
| **Event bus** | SNS+SQS | EventBridge, Cloud Pub/Sub | Fan-out patterns. |

### Observability

| Category | Recommended | Alternatives | Notes |
|----------|-------------|--------------|-------|
| **Metrics** | Prometheus + Grafana | Datadog, New Relic | Prometheus is the standard; Grafana visualizes. |
| **Logging** | Loki | Elasticsearch, CloudWatch | Loki uses same labels as Prometheus. |
| **Tracing** | Jaeger | Zipkin, Tempo | Integrates with OpenTelemetry. |
| **All-in-one** | Datadog | New Relic, Honeycomb | Higher cost, lower ops burden. |
| **Instrumentation** | OpenTelemetry | - | Vendor-neutral; instrument once. |

### Infrastructure & Deployment

| Category | Recommended | Alternatives | Notes |
|----------|-------------|--------------|-------|
| **Container orchestration** | Kubernetes | ECS, Nomad | Use managed (EKS, GKE, AKS). |
| **CI/CD** | GitHub Actions | GitLab CI, CircleCI | Where your code lives matters. |
| **Infrastructure as Code** | Terraform | Pulumi, CloudFormation | Terraform is multi-cloud. |
| **GitOps** | ArgoCD | Flux | Sync cluster state from Git. |
| **Service mesh** | Linkerd | Istio | Linkerd is simpler; Istio is more powerful. |
| **Secrets** | HashiCorp Vault | AWS Secrets Manager | Vault if multi-cloud; managed if single. |

### API & Communication

| Category | Recommended | Alternatives | Notes |
|----------|-------------|--------------|-------|
| **API Gateway** | Kong | AWS API Gateway, Envoy | Kong for self-managed; AWS for serverless. |
| **Load Balancer** | NGINX | HAProxy, Envoy, Traefik | NGINX for simplicity; Envoy for features. |
| **CDN** | Cloudflare | Fastly, CloudFront | Cloudflare's free tier is generous. |
| **DNS** | Cloudflare DNS | Route 53, Cloud DNS | Fast, DDoS protection included. |

### Development & Testing

| Category | Recommended | Alternatives | Notes |
|----------|-------------|--------------|-------|
| **Local dev** | Docker Compose | Tilt, Skaffold | Compose for simple; Tilt/Skaffold for K8s. |
| **API testing** | Bruno | Postman, Insomnia | Bruno stores tests in Git-friendly format. |
| **Load testing** | k6 | Locust, Gatling | k6 uses JavaScript, good DX. |
| **Chaos engineering** | Chaos Monkey | Litmus, Gremlin | Start with Chaos Monkey, graduate to Gremlin. |

### AI/ML

| Category | Recommended | Alternatives | Notes |
|----------|-------------|--------------|-------|
| **Experiment tracking** | MLflow | Weights & Biases, Neptune | MLflow is open-source. |
| **Feature store** | Feast | Tecton, AWS Feature Store | Feast is open-source. |
| **Model serving** | Triton | TensorFlow Serving, Seldon | Triton is multi-framework. |
| **LLM APIs** | OpenAI, Anthropic | Cohere, Google Vertex | Choose by capability and compliance needs. |
| **LLM orchestration** | LangChain | LlamaIndex, Haystack | Evolving rapidly; choose cautiously. |

### How to Choose

**Start with the "Recommended" column.** These tools have large communities, good documentation, and proven track records.

**Consider managed vs. self-hosted.** Managed services cost more but reduce operational burden. If you have a small team, managed is usually worth it.

**Prefer boring technology.** New tools are exciting but risky. PostgreSQL, Redis, and NGINX have decades of production use. Boring is good.

**Match tools to team expertise.** The best tool your team doesn't know is worse than a good tool they know well.

**Avoid premature optimization.** Don't adopt Kafka for 100 messages/second. Don't shard PostgreSQL at 10GB. Simple solutions often suffice longer than expected.

### Updates

This reference reflects the landscape as of early 2026. The ecosystem evolves rapidly—verify current recommendations before major decisions.

---

## Case Studies

Real-world system design decisions from companies operating at scale. Each study links to the concepts covered in the main sections.

---

### Discord

Discord handles trillions of messages across millions of concurrent users. Their engineering decisions illustrate how theoretical tradeoffs play out in production.

#### Context

Discord launched in 2015 as a voice chat app for gamers. By 2020, it had grown to 140 million monthly users sending billions of messages daily. The original MongoDB-based architecture couldn't keep up—message queries became the bottleneck.

The core challenge: messages are read far more than written, users expect real-time delivery, and chat history must be searchable. A single popular server (Discord's term for a community) might have millions of messages and thousands of concurrent readers.

#### Problem

MongoDB struggled with Discord's read patterns. Messages cluster by time—users read recent messages far more than old ones. But MongoDB stored messages by ID, scattering recent messages across disk. Read amplification killed performance.

Hot partitions emerged on popular servers. A viral moment in a large community would spike traffic to one partition while others sat idle. The database couldn't redistribute load fast enough.

Message fanout created another challenge. A message sent to a server with 100,000 online members must reach all of them within seconds. The write path needed to handle spikes of millions of messages per minute.

#### Decisions

**Migrated from MongoDB to Cassandra for message storage.** Cassandra's write-optimized LSM trees handle Discord's write-heavy workload. Data is partitioned by (channel_id, bucket) where bucket is a time window, keeping recent messages together on disk. (→ [Partitioning](#partitioning))

**Chose Cassandra over DynamoDB.** DynamoDB seemed like the obvious AWS choice, but Discord needed predictable latency at scale. DynamoDB's hot partition throttling and per-partition throughput limits didn't match Discord's access patterns—one popular channel could easily exceed partition limits. Cassandra gave them more control over data distribution and tuning. (→ [Database Selection](#database-selection))

**Accepted eventual consistency for message delivery.** Messages might appear out of order briefly during high load. Discord decided users tolerate occasional ordering glitches better than delayed delivery. (→ [Consistency Models](#consistency-models))

**Surprising decision: Relaxed ordering guarantees.** Most chat systems guarantee message ordering within a channel. Discord doesn't—during load spikes, messages might briefly appear out of order. This seems wrong for a chat app, but Discord found that perceived latency matters more than perfect ordering. Users notice delayed messages; they rarely notice that two messages sent 50ms apart appeared in the wrong order.

**Used a write-through cache for hot channels.** Recent messages from active channels stay in memory. Reads hit the cache first, falling through to Cassandra only for cache misses or historical queries. (→ [Caching Strategies](#caching-strategies))

**Implemented custom fanout with lazy evaluation.** Instead of pushing messages to every client immediately, Discord's gateway maintains subscription state. Clients fetch messages on reconnect rather than receiving a flood of queued updates. (→ [Event-Driven Architecture](#event-driven-architecture))

**Migrated to ScyllaDB for performance.** In 2022, Discord moved from Cassandra to ScyllaDB, a C++ rewrite with the same API. P99 latency dropped from 40-125ms to 15ms. Same data model, different implementation.

#### Results

The Cassandra architecture scaled Discord through 4x growth. Message query latency stabilized even as volume increased. The time-bucketed partition strategy eliminated hot partition issues for normal usage.

The ScyllaDB migration demonstrated that implementation matters as much as architecture. Same data model, same consistency guarantees, dramatically different performance. Discord's team measured a 5x improvement in P99 latency.

Discord's lesson: partition keys should match access patterns. Users query messages by channel and time, so (channel_id, time_bucket) as the partition key keeps related data together. MongoDB's document ID partitioning ignored this, causing read amplification.

**Lesson learned: Design partitioning from day one.** Discord's MongoDB architecture used default partitioning—by document ID. When they migrated to Cassandra, redesigning partitioning was the hardest part. They now advise: choose your partition key based on your most important query pattern before writing a single line of code. Retrofitting is expensive.

**Sources:**
- [How Discord Stores Trillions of Messages](https://discord.com/blog/how-discord-stores-trillions-of-messages)
- [How Discord Stores Billions of Messages (2017)](https://discord.com/blog/how-discord-stores-billions-of-messages)

---

### Figma

Figma enables real-time collaborative design with thousands of simultaneous editors. Their architecture demonstrates how to build multiplayer experiences at scale.

#### Context

Figma launched in 2016 as a browser-based design tool. Unlike desktop tools, Figma needed collaboration from day one—multiple designers editing the same file simultaneously, seeing each other's changes in real-time.

The challenge: design files are complex (thousands of objects with properties and relationships), multiple users edit concurrently, and changes must appear instantly. Traditional client-server architectures can't handle this—round trips to the server would feel laggy.

#### Problem

Naive approaches fail for concurrent editing. Locking (only one user edits at a time) kills collaboration. Last-write-wins causes lost work. Operational transformation (used by Google Docs) is complex and hard to extend.

Design files have structure that text doesn't: layers, groups, components, constraints, and references between objects. Changes to one object might affect others. The synchronization algorithm must understand this structure.

Latency compounds for distributed users. A designer in Tokyo editing with someone in New York experiences 150ms+ round trips. Changes must feel local while eventually converging globally.

#### Decisions

**Built on CRDTs for conflict resolution.** Conflict-free Replicated Data Types (CRDTs) allow concurrent edits without coordination. Operations are designed to always merge correctly, regardless of order. (→ [Consistency Models](#consistency-models))

**Local-first execution.** Changes apply immediately to the local state. The UI reflects edits instantly without waiting for server acknowledgment. The server eventually broadcasts changes to other clients.

**Multiplayer server in Rust.** Figma's multiplayer server coordinates sessions, broadcasts changes, and handles presence (showing who's editing what). Rust provided the performance needed to handle thousands of concurrent connections.

**Document structure optimized for CRDTs.** Figma's internal representation maps naturally to CRDT operations. Moving a layer, changing a color, and adding a shape all become CRDT operations that merge cleanly.

**Built custom CRDTs instead of using off-the-shelf libraries.** Figma evaluated Yjs, Automerge, and other CRDT libraries. None fit their specific data model—design files have unique semantics (layers, groups, constraints) that generic CRDTs don't handle well. Custom CRDTs let them optimize for their exact operations. (→ [Data Modeling](#data-modeling))

**Surprising decision: Accepted CRDT operational complexity.** CRDTs sound elegant—operations merge automatically! In practice, Figma's CRDT implementation required solving subtle problems: handling deleted objects that other operations reference, managing garbage collection of tombstones, and ensuring deterministic merge order across all clients. This complexity lives in their code forever. Most applications don't need this—Figma accepted it because real-time collaboration is their core product, not a feature.

**Eventual consistency with operational awareness.** Users see others' cursors and selections in real-time. This awareness helps collaborators coordinate naturally, reducing conflicts that would need programmatic resolution.

#### Results

Figma handles millions of concurrent editing sessions. Latency stays low even for users across continents. Conflicts resolve correctly—users don't lose work or see corrupted documents.

The CRDT approach scaled to their needs, but required deep investment. Figma's CRDTs are custom-built for their specific data model. Off-the-shelf CRDTs (like Yjs or Automerge) might work for simpler applications.

**Lesson learned: Start with off-the-shelf, go custom only when necessary.** Figma built custom CRDTs because they had to—their scale and data model demanded it. But they advise most teams to start with libraries like Yjs. If 90% of the library fits your needs, the 10% gap is cheaper than building from scratch. Figma would use Yjs today for a new collaborative text editor; they only needed custom CRDTs for their specific design file semantics.

**Sources:**
- [How Figma's multiplayer technology works](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/)
- [Realtime Editing of Ordered Sequences](https://www.figma.com/blog/realtime-editing-of-ordered-sequences/)

---

### Stripe

Stripe processes billions of dollars in transactions annually with extreme reliability requirements. Their architecture demonstrates how to build financial systems where consistency and durability are non-negotiable.

#### Context

Stripe handles payments—charging cards, moving money, preventing fraud. Every transaction must process exactly once. Double-charging a customer is unacceptable; losing a payment is unacceptable. The system must work even when components fail.

Stripe also exposes APIs to millions of developers. API stability matters as much as internal reliability. Breaking changes or inconsistent behavior damage trust.

#### Problem

Distributed transactions are hard. A single payment might involve: validating the card, checking fraud signals, creating a charge record, updating balances, sending webhooks, and triggering payouts. Any step can fail.

Financial data has strict consistency requirements. Account balances must be correct—not eventually consistent, but actually consistent. Audit trails must be complete. Regulations require durability.

Scale compounds complexity. Stripe processes millions of API requests with varying latency requirements: card validation needs to be fast (affecting checkout conversion); batch payouts can be slower.

#### Decisions

**Idempotency keys for exactly-once semantics.** Clients include unique keys with requests. Stripe stores results keyed by idempotency key. Retried requests return the stored result rather than reprocessing. This makes exactly-once semantics the client's responsibility but provides the building blocks. (→ [Streaming vs Batch](#streaming-vs-batch))

**ACID transactions for financial state.** Core financial operations use PostgreSQL with strong consistency. Stripe chose operational complexity (managing PostgreSQL at scale) over relaxed consistency. Money requires correctness.

**Event sourcing for audit trails.** State changes are recorded as immutable events. Account balances can be reconstructed from events. Audit trails are complete by construction.

**API versioning at the request level.** Each API request specifies a version. Stripe maintains backwards compatibility across versions. Internal services must handle multiple versions simultaneously.

**Surprising decision: Made idempotency the client's responsibility.** Stripe could have handled idempotency server-side, tracking all requests automatically. Instead, they require clients to send idempotency keys. This seems like pushing complexity onto users, but it gives clients control: they choose which operations should be idempotent, and they own the deduplication logic. The alternative—automatic server-side deduplication—would require Stripe to guess client intent.

**Custom distributed tracing for payment debugging.** Payment failures are high-stakes debugging scenarios. Stripe built custom tracing that captures the full request lifecycle: API gateway, fraud checks, card network calls, and webhook delivery. A single trace ID follows a payment through every system. This goes beyond standard observability—it's forensic infrastructure for answering "why did this specific payment fail?" (→ [Observability](#observability))

**Incremental migration, not big bang.** Stripe rewrites systems gradually, running old and new in parallel, shifting traffic incrementally. This slows velocity but prevents catastrophic failures. (→ [Deployment Strategies](#deployment-strategies))

#### Results

Stripe maintains extremely high uptime for a financial system. Exactly-once semantics (via idempotency) prevent double-charging. Event sourcing provides complete audit trails.

The approach requires significant engineering investment. Stripe has dedicated teams for database infrastructure, API platform, and reliability engineering. Smaller companies might accept relaxed consistency for lower complexity.

**Lesson learned: Request-level tracing is non-negotiable for payments.** Early in Stripe's history, debugging payment failures meant correlating logs across systems manually. This was slow and error-prone. Stripe now considers end-to-end request tracing essential infrastructure, not nice-to-have observability. If you're building payment systems, instrument request tracing before you have scale problems—retrofitting is painful and failures are expensive.

**Sources:**
- [Idempotency Keys: How Stripe Performs Exactly-Once Delivery](https://stripe.com/blog/idempotency)
- [Online Migrations at Scale](https://stripe.com/blog/online-migrations)

---

### Netflix

Netflix streams to 250+ million subscribers globally. Their architecture pioneered microservices at scale, chaos engineering, and adaptive streaming.

#### Context

Netflix moved from DVDs to streaming in 2007. By 2008, a database corruption incident caused three days of downtime. They decided to migrate to AWS and rebuild for resilience. The goal: no single point of failure should cause user-visible impact.

Netflix's scale is massive: thousands of services, millions of concurrent streams, and traffic spikes for popular releases. The system must handle failures gracefully—a failing recommendation service shouldn't prevent users from watching.

#### Problem

Monolithic architectures have coupled failure modes. One component failing brings down everything. Netflix needed independent services that degrade gracefully—if recommendations fail, users can still browse and watch.

Global scale requires global distribution. Content must be close to users (video files) while personalization requires centralized state (user preferences, watch history). The architecture must bridge this gap.

Streaming quality must adapt to network conditions. Users on poor connections should see lower quality rather than buffering. The player must detect conditions and request appropriate bitrates.

#### Decisions

**Microservices for failure isolation.** Each capability (recommendations, search, playback) runs independently. Failures are contained; the whole system degrades gracefully rather than failing catastrophically. (→ [Partitioning](#partitioning))

**Chaos engineering for resilience validation.** Netflix built Chaos Monkey to randomly terminate instances in production. If the system can't handle instance failures gracefully, it fails during controlled chaos, not during real outages. (→ [Resilience Patterns](#resilience-patterns))

**Surprising decision: Running chaos in production, not staging.** Most companies test failure handling in staging. Netflix runs Chaos Monkey in production during business hours. This seems reckless, but Netflix argues that staging environments never match production's complexity. If Chaos Monkey causes problems, it reveals real resilience gaps—better to find them during controlled experiments than during actual outages. The cultural shift matters: engineers design for failure because they know failures will happen.

**Chaos engineering as culture, not tooling.** Chaos Monkey is famous, but Netflix's real innovation is cultural. Every engineer understands that their service will experience failures—instances will terminate, dependencies will become slow, network will partition. This assumption changes how they design systems. Chaos tools validate the assumption; the culture makes it an engineering requirement. (→ [Incident Management](#incident-management))

**Open Connect CDN for content delivery.** Netflix built their own CDN, placing servers inside ISPs worldwide. Video files are delivered from servers meters away from users, not across the internet. This provides better quality and reduces internet congestion.

**Adaptive bitrate streaming.** Netflix encodes content at multiple quality levels. The player monitors bandwidth and adjusts quality dynamically. Users on poor connections see lower quality rather than buffering.

**Data infrastructure for personalization.** Everything is personalized: thumbnails, recommendations, search results. This requires massive data infrastructure (Kafka, Spark, Flink) processing billions of events daily.

**Observability at scale.** Netflix built and open-sourced multiple observability tools (Atlas for metrics, Zuul for edge routing). When you have thousands of services, observability is existential. (→ [Observability](#observability))

#### Results

Netflix achieved their resilience goals. Major AWS outages cause minimal user impact. Chaos engineering became an industry practice.

The microservices approach introduced complexity: distributed tracing, service meshes, and organizational coordination. Netflix has 2,000+ engineers to manage this complexity. Smaller organizations should evaluate whether they need this architecture.

**Lesson learned: Service ownership includes data ownership.** Early Netflix microservices shared databases—multiple services reading and writing to the same tables. This created hidden coupling: changing a table schema required coordinating across teams. Netflix now enforces strict data ownership: each service owns its data, and other services access it through APIs only. This rule seemed bureaucratic initially but proved essential for independent deployability. (→ [Data Modeling](#data-modeling))

**Sources:**
- [Netflix Tech Blog](https://netflixtechblog.com/) — The canonical source for Netflix architecture
- [Completing the Netflix Cloud Migration](https://about.netflix.com/en/news/completing-the-netflix-cloud-migration)
- [Chaos Engineering at Netflix](https://netflixtechblog.com/tagged/chaos-engineering)

---

## Contributing

Found an error? Have a suggestion? This guide is maintained on GitHub. Issues and pull requests welcome.

## License

This work is licensed under a Creative Commons Attribution 4.0 International License.
