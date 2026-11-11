# System Design — From Zero to Staff Engineer

> **The definitive, open-source system design handbook.** Structured for self-study, interview prep, and real-world architecture. No fluff — just rigorous, production-grade knowledge.

<p align="center">

[![Stars](https://img.shields.io/github/stars/karanpratapsingh/system-design?style=social)](https://github.com/karanpratapsingh/system-design)
![Content](https://img.shields.io/badge/content-9%20chapters%20%7C%20300%2B%20topics-blue)
![Status](https://img.shields.io/badge/status-actively%20maintained-success)
![License](https://img.shields.io/badge/license-MIT-green)
![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)

</p>

---

## 📋 Prerequisites

### Required

| Area | What You Need | Why |
|------|---------------|-----|
| **Data Structures & Algorithms** | Arrays, hash maps, trees, graphs; Big-O | Underpins every estimation & data structure choice |
| **Basic Networking** | HTTP, DNS, TCP/IP at a conceptual level | Otherwise Ch 02 will feel alien |
| **One Backend Language** | Any (Python, Java, Go, …) — enough to read pseudocode | Examples are language-agnostic but code-adjacent |
| **Databases 101** | What a table, index, and transaction are | Ch 04 builds from there |

### Helpful but Not Required

| Area | Helps With |
|------|------------|
| Operating Systems (processes, threads, I/O) | Concurrency, replication, storage engines |
| Concurrency & threading | Locks, race conditions, async patterns |
| Cloud basics (AWS/GCP/Azure) | Mapping abstract components to managed services |
| Prior internship / production experience | Grounding trade-offs in real pain |

---

## 🏆 How System Design Interviews Are Actually Scored

> Most candidates think the interviewer is checking if your diagram "looks right."
> They're not. Here's the rubric most FAANG interviewers use internally.

### The 4 Pillars (Weighted)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Pillar              Weight    What the interviewer is listening for    │
│  ─────────────────────────────────────────────────────────────────      │
│  1. Requirements     15%       Did you clarify scope & constraints?     │
│     & Estimation               Functional vs non-functional, QPS math   │
│  2. High-Level       30%       Components, data flow, API contracts     │
│     Design                                                              │
│  3. Deep Dives       30%       2-3 deep dives with trade-off reasoning  │
│  4. Trade-offs &     25%       Failure modes, scaling limits, costs     │
│     Wrap-up                    Follow-up handling, communication        │
└─────────────────────────────────────────────────────────────────────────┘
```

| Signal | Strong Hire | Weak / No Hire |
|--------|-------------|----------------|
| **Scope** | Narrows deliberately, states assumptions | Tries to design everything, never clarifies |
| **Estimation** | Quantitative (QPS, storage, bandwidth) | Hand-wavy ("it should scale") |
| **Component choice** | Justified with trade-off ("we pick Cassandra *because* write-heavy + tunable consistency") | Buzzword list ("use Kafka, Redis, Cassandra") |
| **Deep dive** | 2–3 areas with depth (e.g., feed ranking, cache invalidation) | Stays shallow everywhere |
| **Failure mode** | Proactively discusses SPOF, retry storm, hot shard | Only when prompted |
| **Communication** | Structured, checks in, handles pushback | Rambles or goes silent drawing |

---

## 💰 The Cost of Scale — Why Design Matters Financially

```
Requests/day  ──►  Infra cost  ──►  Design decisions that move the needle
─────────────────────────────────────────────────────────────────────────
10K / day     ~$50 / mo        Anything works; optimize for velocity
1M / day      ~$500 / mo       Add CDN + cache; DB still single node
100M / day    ~$5K–20K / mo    Sharding, read replicas, async workers
1B / day      ~$100K+ / mo     Multi-region, custom storage, tail-latency work
10B / day     ~$1M+ / mo       Every byte stored & every ms matters
```

| Optimization | Typical Saving | When It Matters |
|--------------|---------------|-----------------|
| CDN for static + edge cache | 60–80% origin offload | > 10M req/day |
| Compression (gzip → zstd/brotli) | 30–50% bandwidth | Media / API heavy |
| Columnar storage (Parquet/ORC) | 5–10× storage vs row store | Analytics / OLAP |
| Right-sizing instances (not over-provisioning) | 20–40% compute | Always — use autoscaling + spot |
| Batching writes (WAL + async flush) | 3–5× write throughput | Ingestion pipelines |

> 📐 **Estimation drill:** A 1 KB profile image × 100M users × 3 replicas = ~300 GB.
> The same image at 1 MB (uncompressed) = 300 TB — a 1000× cost difference from one decision.

---

## 📖 Glossary (Quick Reference)

> Full glossary lives in [`GLOSSARY.md`](./GLOSSARY.md). 

---

## 🔗 Resource Compilation — Full Attribution

This repo synthesizes and rewrites material from the following. We cite inline where specific
numbers, diagrams, or arguments are drawn from a source.

### Primary

| Source | Author(s) | What We Drew On |
|--------|-----------|-----------------|
| [karanpratapsingh/system-design](https://github.com/karanpratapsingh/system-design) | Karan Pratap Singh | Chapter scaffolding, progressive disclosure, open-source ethos |
| *Grokking the System Design Interview* (Educative) | Design Gurus | RESHADED template, 15+ case study structures, interview patterns |
| *Designing Data-Intensive Applications* | Martin Kleppmann | Storage engines, replication, partitioning, transactions, consensus, stream processing |
| *System Design Interview Vol. 1 & 2* | Alex Xu | URL Shortener, News Feed, Chat, Notification, Search case studies |
| *Understanding Distributed Systems* | Roberto Vitillo | Gossip, failure detection, load balancing depth |

### Supplementary

| Source | Why It Matters |
|--------|----------------|
| ByteByteGo (Alex Xu) — Newsletter, YouTube, Blog | Visual explainers, trade-off tables, weekly patterns |
| High Scalability Blog | Real architecture histories (YouTube, Pinterest, Stack Overflow scale) |
| AWS / GCP / Azure Architecture Blogs | Managed-service mappings, well-architected frameworks |
| InfoQ & ACM Queue | Practitioner essays (e.g., "The Tail at Scale" — Dean & Barroso) |
| *Site Reliability Engineering* & *SRE Workbook* (Google) | SLOs, error budgets, incident response, toil |
| *Designing Distributed Systems* (Brendan Burns) | Sidecar, ambassador, adapter patterns on Kubernetes |
| *Database Internals* (Alex Petrov) | B-Tree, LSM, WAL deep dives |
| *Release It!* (Michael Nygard) | Stability patterns — circuit breaker, bulkhead |
| Netflix Tech Blog, Uber Eng Blog, Discord Eng Blog | Case studies with load numbers |

### How Compilation Works (Example)

> **Topic: Rate Limiting**
> - Grokking gives the interview framing (token bucket vs sliding window).
> - DDIA explains the concurrency & clock-skew subtleties.
> - Stripe / Cloudflare blogs give production numbers (burst, refill, Redis vs in-memory).
> - This repo merges them into one chapter: intuition → algorithms → distributed design → interview drill.

---

## 💬 Community & Support

| Channel | Purpose |
|---------|---------|
| **GitHub Issues** | Bugs, requests, chapter proposals |
| **GitHub Discussions** | Q&A, design reviews, mock partners |
| **Pull Requests** | Contributions — see style guide above |

> **Mock interview partners:** Open a Discussion with tag `mock-request` — include your timezone and target level.

---

## 📄 License

MIT — use it, fork it, teach from it. If you build something cool, let us know.

---

## 🔍 Search & Navigation Tips

- **GitHub search:** Press `t` on the repo page to fuzzy-find any file by name.
- **VS Code:** `Ctrl+P` → type chapter name; `Ctrl+Shift+F` to search content.
- **Cheatsheet drills:** Print `10-cheatsheets/*.md` double-sided — one page each.
- **Mermaid rendering:** GitHub renders Mermaid natively; for offline viewing use the
  [Mermaid Live Editor](https://mermaid.live/) or VS Code extension `bierner.markdown-mermaid`.
- **Link hygiene:** All internal links are relative — they work on GitHub, docsify, MkDocs, and local preview.

---

<p align="center">
  <em>“You can't scale what you don't understand. Understand first — then scale.”</em><br/>
  — Start with <a href="./00-preface/00-what-is-system-design.md">00 — What Is System Design?</a> →
</p>
