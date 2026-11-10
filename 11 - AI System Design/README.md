# Chapter 11: System Design for AI 🤖

## 11.1 Why AI Systems Are Different

```
┌─────────────────────────────────────────────────────────────────┐
│  AI SYSTEM DESIGN ≠ TRADITIONAL SYSTEM DESIGN                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TRADITIONAL:                                                   │
│  • Deterministic computation (same input → same output)         │
│  • Stateless or simple state (sessions, cache)                  │
│  • CPU-bound, horizontal scaling straightforward                │
│  • Cost ∝ requests × compute                                    │
│                                                                 │
│  AI SYSTEMS:                                                    │
│  • PROBABILISTIC output (same input → distribution of outputs)  │
│  • MASSIVE STATE: Model weights (GB–TB) + KV Cache (GB/request) │
│  • GPU-BOUND: Compute ∝ tokens² (attention), memory-bound       │
│  • Cost ∝ tokens × GPU-hours (expensive, scarce hardware)       │
│  • LATENCY SENSITIVE: Time-to-first-token (TTFT) critical       │
│  • QUALITY MATTERS: Hallucination, drift, evaluation needed     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### The Core Tension: Latency vs Throughput vs Cost

```
                    ╭──────────────────────────────────────────╮
                    │        AI SERVING TRIANGLE               │
                    │                                          │
                    │        LOW LATENCY                       │
                    │        (TTFT < 200ms)                    │
                    │            ▲                             │
                    │           / \                            │
                    │          /   \                           │
                    │         /     \                          │
                    │        /       \                         │
                    │       /         \                        │
                    │      /           \                       │
                    │     /             \                      │
                    │    /               \                     │
                    │   /                 \                    │
                    │  /                   \                   │
                    │ /                     \                  │
                    │─────────────────────────\────────────────│
                    │ HIGH THROUGHPUT           \   LOW COST   │
                    │ (tokens/sec/$)            \  ($/M tokens)│
                    │                            \             │
                    │                             \            │
                    │                              \           │
                    │                               \          │
                    │                                \         │
                    │                                 \        │
                    │                                  \       │
                    │                                   \      │
                    │                                    \     │
                    │                                     \    │
                    │                                      \   │
                    │                                       \  │
                    │                                        \/|
                    │                                         ▼
                    │                              (Pick 2, optimize for 1)
                    ╰──────────────────────────────────────────╯

OPTIMIZATION TARGETS BY USE CASE:
  • Chat/Interactive:     LOW LATENCY (TTFT)  → Small batches, powerful GPUs
  • Batch/Async:          HIGH THROUGHPUT     → Large batches, continuous batching
  • Cost-sensitive:       LOW COST            → Quantization, smaller models, spot GPUs
```

---

## 11.2 LLM Inference Fundamentals

### Autoregressive Decoding

```
PREFILL PHASE (Parallel)          DECODE PHASE (Sequential)
─────────────────────             ──────────────────────
Input: "The quick brown"          Generated so far: "The quick brown fox"
                                      │
        ┌───────────────────────────┼───────────────────────┐
        ▼                           ▼                       ▼
    ┌─────────┐                 ┌─────────┐             ┌─────────┐
    │ Layer 1 │                 │ Layer 1 │             │ Layer 1 │
    │  Q K V  │──KV Cache──────►│  Q K V  │──KV Cache──►│  Q K V  │
    └─────────┘                 └─────────┘             └─────────┘
        │                           │                       │
        ▼                           ▼                       ▼
    ┌─────────┐                 ┌─────────┐             ┌─────────┐
    │ Layer 2 │                 │ Layer 2 │             │ Layer 2 │
    └─────────┘                 └─────────┘             └─────────┘
        │                           │                       │
        ▼                           ▼                       ▼
        ...                         ...                     ...
        │                           │                       │
        ▼                           ▼                       ▼
    ┌─────────┐                 ┌─────────┐             ┌─────────┐
    │ Layer N │                 │ Layer N │             │ Layer N │
    └─────────┘                 └─────────┘             └─────────┘
        │                           │                       │
        ▼                           ▼                       ▼
    Next token                    Next token              Next token
    probs                         probs                   probs
                                  (use cached K,V)        (use cached K,V)
```

### KV Cache Memory Math

```
For a 70B parameter model (Llama-3-70B):
  • Layers: 80
  • Heads: 64 (GQA: 8 KV heads)
  • Head dim: 128
  • Precision: FP16 (2 bytes)

KV Cache per token:
  = 2 (K,V) × 80 layers × 8 KV heads × 128 dim × 2 bytes
  = 327,680 bytes ≈ 0.33 MB per token

Context window 4096 tokens:
  = 0.33 MB × 4096 ≈ 1.3 GB per request

Context window 128K tokens:
  = 0.33 MB × 131,072 ≈ 43 GB per request  ⚠️

Model weights (FP16):
  = 70B × 2 bytes = 140 GB

TOTAL per request (4096 ctx): 140 GB + 1.3 GB = 141.3 GB
→ Needs 2× H100 80GB just for ONE request!
```

---

## 11.3 Serving Architecture Patterns

### Synchronous vs Asynchronous Inference

```python
# SYNCHRONOUS: Request blocks until complete
@app.post("/v1/chat/completions")
async def chat_sync(request: ChatRequest):
    result = await model.generate(request)
    return result

# ASYNCHRONOUS: Return task_id, poll for result
@app.post("/v1/chat/completions/async")
async def chat_async(request: ChatRequest):
    task_id = str(uuid.uuid4())
    queue.enqueue(task_id, request)
    return {"task_id": task_id}

@app.get("/v1/tasks/{task_id}")
async def get_task(task_id: str):
    return task_store.get(task_id)

# STREAMING: Return tokens as they generate (SSE)
@app.post("/v1/chat/completions")
async def chat_stream(request: ChatRequest):
    async def token_generator():
        async for token in model.stream_generate(request):
            yield f"data: {json.dumps({'token': token})}\n\n"
        yield "data: [DONE]\n\n"
    return StreamingResponse(token_generator(), media_type="text/event-stream")
```

| Pattern | Latency | Throughput | Use Case |
|---------|---------|------------|----------|
| **Sync** | Low (immediate) | Low | Short responses, chat |
| **Async** | High (polling) | High | Long generation, documents |
| **Streaming (SSE)** | Low TTFT | Medium | Interactive chat, real-time feel |

---

## 11.4 Inference Optimization Techniques

### Continuous Batching (vLLM / PagedAttention)

```
STATIC BATCHING (v1):              CONTINUOUS BATCHING (v2/vLLM):
─────────────────────────          ──────────────────────────────
Batch: [Req1, Req2, Req3]          Step 1: [Req1, Req2, Req3]
  │                                    │
  ▼                                    ▼
Step 1: All process                 Step 1: Req1 done → evict
  │                                    │
  ▼                                    ▼
Step 2: All process                 Step 2: [Req2, Req3, Req4(new)]
  │                                    │
  ▼                                    ▼
Step 3: Req1 done, wait             Step 3: Req2 done → evict
  │                                    │
  ▼                                    ▼
Step 4: Req2 done...                Step 4: [Req3, Req4, Req5(new)]
  │
  ▼
Wasted slots while others finish   → NO WASTED SLOTS!
                                     → 2-4× throughput improvement
```

### PagedAttention (vLLM) — Eliminates KV Fragmentation

```
TRADITIONAL (Contiguous):          PAGEDATTENTION (vLLM):
─────────────────────────          ────────────────────

Req A: [KV KV KV KV KV KV KV]      Physical KV Pages (16 tokens each):
Req B: [KV KV KV]  ───────────►    ┌────┬────┬────┬────┐
Req C: [KV KV KV KV KV]            │ P1 │ P2 │ P3 │ P4 │ ...
                                   └────┴────┴────┴────┘
Fragmented,                        Page Table per Request:
can't reuse gaps                   Req A: [P1, P2, P3, P4]
                                    Req B: [P5]
                                    Req C: [P6, P7, P8]
                                    → Near-zero fragmentation
                                    → Dynamic allocation
                                    → Prefix sharing (same system prompt)
```

### Quantization

| Precision | Model Size | Speedup | Quality Loss | Use Case |
|-----------|------------|---------|--------------|----------|
| **FP16/BF16** | 2× params | Baseline | None | Reference |
| **INT8** | 2× smaller | 1.5–2× | Minimal | Production default |
| **INT4 (GPTQ/AWQ)** | 4× smaller | 2–3× | Small (1–2%) | Cost-sensitive |
| **FP8** | 2× smaller | 2× (H100) | Minimal | H100 native |
| **Binary/Ternary** | 8–16× smaller | 4–8× | Significant | Edge, extreme |

### Speculative Decoding

```
DRAFT MODEL (Small, Fast)          TARGET MODEL (Large, Slow)
─────────────────────────          ─────────────────────────

Generate K tokens fast             Verify in parallel
  │                                    │
  ▼                                    ▼
[tok1, tok2, tok3] ──► Accept? ──► [tok1✓, tok2✓, tok3✗]
                                     │
                          Accepted: 2 tokens
                          Rejected: Recompute tok3
                          
Result: 2–3× throughput for predictable outputs (code, extraction)
        Skip for high-temp creative (low acceptance rate)
```

---

## 11.5 Model Parallelism (When Model > GPU Memory)

```
TENSOR PARALLELISM (TP)              PIPELINE PARALLELISM (PP)
─────────────────────────            ─────────────────────────

Split WEIGHTS across GPUs            Split LAYERS across GPUs
within each layer                    (Layer 1-20 on GPU 0,
                                     Layer 21-40 on GPU 1)

GPU 0: W[:half]    GPU 1: W[half:]   GPU 0: Layers 1-20
       │                  │                 │
       ▼                  ▼                 ▼
   All-Reduce          All-Reduce       Activation
   (per layer)         (per layer)      Transfer
       │                  │                 │
       └────────┬─────────┘                 │
                ▼                           ▼
           Output                    GPU 1: Layers 21-40
                                     (wait for activation)

Latency: LOW (single layer)          Latency: HIGH (bubble)
Comm: HIGH (NVLink required)         Comm: LOW (Ethernet OK)
Best: Within node (8 GPUs)           Best: Across nodes
                                    
EXPERT PARALLELISM (MoE):            DATA PARALLELISM:
─────────────────────────            ─────────────────────
Route tokens to experts              Full model replica
on different GPUs                    per GPU, shard data
                                    
Mixtral 8×7B: 2 experts/token        Standard for < 1 GPU models
```

---

## 11.6 RAG (Retrieval-Augmented Generation) Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│  RAG ARCHITECTURE                                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USER QUERY                                                     │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │  QUERY      │───►│  VECTOR     │───►│  TOP-K      │          │
│  │  EMBEDDING  │    │  DATABASE   │    │  RETRIEVAL  │          │
│  │  (e.g.,     │    │  (Pinecone, │    │  (cosine    │          │
│  │   text-     │    │   Weaviate, │    │  similarity)│          │
│  │   embedding │    │   pgvector) │    └──────┬──────┘          │
│  │   -3-small) │    └─────────────┘           │                 │
│  └─────────────┘                              ▼                 │
│       │                              ┌─────────────────┐        │
│       │                              │  CONTEXT        │        │
│       │                              │  ASSEMBLY       │        │
│       │                              │  (rerank,       │        │
│       │                              │   truncate,     │        │
│       │                              │   cite)         │        │
│       │                              └────────┬────────┘        │
│       │                                       │                 │
│       ▼                                       ▼                 │
│  ┌─────────────────────────────────────────────────┐            │
│  │              LLM GENERATION                     │            │
│  │  Prompt: [System] + [Context] + [User Query]    │            │
│  └─────────────────────────────────────────────────┘            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### RAG Design Decisions

| Component | Options | Trade-offs |
|-----------|---------|------------|
| **Embedding Model** | text-embedding-3-small/large, BGE, E5 | Cost vs quality vs latency |
| **Chunking** | Fixed size, semantic, recursive | Retrieval precision vs context |
| **Vector DB** | Pinecone, Weaviate, pgvector, Milvus, Qdrant | Managed vs self-hosted, scale |
| **Retrieval** | Dense, sparse (BM25), hybrid | Recall vs precision |
| **Reranking** | Cross-encoder, Cohere, LLM | Quality vs latency (+50-200ms) |
| **Citation** | Post-hoc, in-prompt, constrained | Accuracy vs token cost |

---

## 11.7 AI Infrastructure Stack

### Model Serving Layer

| Framework | Best For | Streaming | Continuous Batching | Multi-LoRA |
|-----------|----------|-----------|---------------------|------------|
| **vLLM** | LLM throughput | ✅ | ✅ (PagedAttention) | ✅ |
| **SGLang** | Structured output, agents | ✅ | ✅ | ✅ |
| **TensorRT-LLM** | NVIDIA max perf | ✅ | ✅ | Limited |
| **TGI (HuggingFace)** | Easy deployment | ✅ | ✅ | ✅ |
| **Trition** | Multi-framework, ensembles | ✅ | Manual | Via backend |
| **BentoML** | Python-first, any model | ✅ | Via vLLM/TGI | ✅ |

### Orchestration Layer

```
Kubernetes + KServe / Knative
        │
        ├── Autoscaling: KEDA (Kafka lag, custom metrics)
        ├── GPU Scheduling: NVIDIA GPU Operator, MIG
        ├── Model Registry: MLflow, W&B, custom
        ├── Feature Store: Feast, Tecton, Hopsworks
        └── Traffic Split: Istio, Knative, Flagger
```

### GPU Autoscaling Metrics

```yaml
# KEDA ScaledObject for LLM
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: llm-scaler
spec:
  scaleTargetRef:
    name: vllm-deployment
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: vllm:num_requests_waiting
        query: sum(vllm:num_requests_waiting)
        threshold: "10"  # Scale when >10 waiting
    - type: prometheus
      metadata:
        metricName: vllm:gpu_cache_usage
        query: avg(vllm:gpu_cache_usage_percent)
        threshold: "80"  # Scale at 80% KV cache
```

---

## 11.8 Semantic Caching (AI-Specific)

```
TRADITIONAL CACHE:                 SEMANTIC CACHE:
─────────────────────              ─────────────────

Key: "What is 2+2?"                Key: Embedding("What is 2+2?")
Value: "4"                         Value: "4"

Key: "What is two plus two?"       Query: Embedding("What is two plus two?")
Value: "4"                         Search: Vector DB (similarity > 0.95)
                                    Hit! Return cached "4"

Hit rate: Exact match only         Hit rate: 30–70% for similar queries
                                    (paraphrases, typos, rephrasing)
```

### Implementation

```python
class SemanticCache:
    def __init__(self, vector_db, embed_model, threshold=0.95):
        self.vector_db = vector_db
        self.embed_model = embed_model
        self.threshold = threshold
    
    async def get(self, prompt: str) -> Optional[str]:
        embedding = await self.embed_model.encode(prompt)
        results = await self.vector_db.search(
            vector=embedding, 
            top_k=1, 
            score_threshold=self.threshold
        )
        if results:
            return results[0].metadata["response"]
        return None
    
    async def set(self, prompt: str, response: str):
        embedding = await self.embed_model.encode(prompt)
        await self.vector_db.upsert(
            id=hash(prompt),
            vector=embedding,
            metadata={"prompt": prompt, "response": response}
        )
```

---

## 11.9 AI Gateway (GenAI Gateway)

```
┌─────────────────────────────────────────────────────────────────┐
│  AI GATEWAY RESPONSIBILITIES                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ROUTING & LOAD BALANCING                                       │
│  • Route to model by complexity (Haiku → Sonnet → Opus)         │
│  • Route by tenant tier (free → small, pro → large)             │
│  • Cache-aware routing (same prefix → same GPU for KV reuse)    │
│                                                                 │
│  COST CONTROL                                                   │
│  • Per-tenant token budgets (daily/monthly)                     │
│  • Model fallback (Opus down → Sonnet)                          │
│  • Request rewriting (shorten context, lower max_tokens)        │
│                                                                 │
│  SAFETY & GOVERNANCE                                            │
│  • Prompt injection detection                                   │
│  • PII redaction (input/output)                                 │
│  • Content moderation (input/output)                            │
│  • Audit logging (all prompts + responses)                      │
│                                                                 │
│  OBSERVABILITY                                                  │
│  • Token usage (input/output/cached) per tenant/model           │
│  • Latency: TTFT, ITL (inter-token latency), E2E                │
│  • Quality: LLM-as-judge scores, user feedback                  │
│  • Cost: $/request, $/token, $/tenant                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11.10 MLOps for LLMs (LLMOps)

### Model Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│  LLM LIFECYCLE                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EXPERIMENTATION                                                │
│  ├── Prompt engineering                                         │
│  ├── Few-shot optimization                                      │
│  ├── RAG pipeline tuning                                        │
│  ├── Evaluation: LLM-as-judge, human eval, benchmarks           │
│  └── Tracking: MLflow, W&B, LangSmith                           │
│                                                                 │
│  DEVELOPMENT                                                    │
│  ├── Prompt versioning (git for prompts)                        │
│  ├── CI/CD: Test prompts against golden sets                    │
│  ├── Regression testing (semantic similarity)                   │
│  ├── Red-teaming (prompt injection, jailbreak)                  │
│  └── Cost estimation per prompt                                 │
│                                                                 │
│  STAGING                                                        │
│  ├── Canary: 1% traffic to new prompt/model                     │
│  ├── Shadow: Mirror traffic, compare outputs offline            │
│  ├── A/B: Interleave, measure business metrics                  │
│  └── Automated rollback on quality regression                   │
│                                                                 │
│  PRODUCTION                                                     │
│  ├── Monitoring: Drift (embedding shift), hallucination rate    │
│  ├── Feedback loop: User thumbs up/down → fine-tune data        │
│  ├── Cost tracking: Per-tenant, per-model, per-feature          │
│  └── Scheduled re-evaluation (weekly/monthly)                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Evaluation Metrics

| Category | Metrics |
|----------|---------|
| **Quality** | LLM-as-judge (1-5), BLEU/ROUGE (legacy), semantic similarity, factuality |
| **Safety** | Refusal rate, harmful content %, PII leakage, injection success |
| **Performance** | TTFT (p50/p95/p99), ITL, throughput (tok/s), GPU utilization |
| **Cost** | $/M tokens, $/request, cost per feature/tenant |
| **Reliability** | Error rate, timeout rate, fallback rate, cache hit rate |

---

## 11.11 AI System Design Interview Problems

### Tier 1: LLM Serving

| Problem | Key Concepts |
|---------|--------------|
| **Design ChatGPT** | Streaming, KV cache, continuous batching, multi-model |
| **Design LLM API Gateway** | Routing, rate limiting, cost control, safety |
| **Design Code Assistant** | Speculative decoding, prefix caching, context window |

### Tier 2: RAG & Knowledge

| Problem | Key Concepts |
|---------|--------------|
| **Design Enterprise RAG** | Chunking, hybrid search, reranking, citations, multi-tenant |
| **Design Document Q&A** | Long context, hierarchical retrieval, table extraction |
| **Design Knowledge Graph + LLM** | GraphRAG, entity extraction, Cypher generation |

### Tier 3: MLOps & Platform

| Problem | Key Concepts |
|---------|--------------|
| **Design Model Serving Platform** | Multi-model, autoscaling, GPU scheduling, canary |
| **Design Feature Store for LLMs** | Embedding storage, online/offline, freshness |
| **Design Fine-tuning Pipeline** | LoRA, distributed training, evaluation, deployment |

### Tier 4: Advanced/Staff

| Problem | Key Concepts |
|---------|--------------|
| **Design Multi-Agent System** | Orchestration, tool use, state management, human-in-loop |
| **Design Real-time Voice AI** | Streaming STT→LLM→TTS, low latency, barge-in |
| **Design AI at Edge** | Quantization, distillation, ONNX/TensorRT, offline |

---

## 11.12 Capacity Estimation for AI Systems

### Example: Design ChatGPT at Scale

```
ASSUMPTIONS:
  • 100M DAU
  • 10 messages/user/day
  • 500 input tokens, 500 output tokens avg
  • Read:Write = 1 (chat is interactive)

CALCULATIONS:
  1. Requests/day = 100M × 10 = 1B
  2. Avg QPS = 1B / 86,400 ≈ 11,500
  3. Peak QPS = 11,500 × 3 = 34,500
  
  4. Tokens/day = 1B req × 1000 tokens = 1T tokens
  5. Token throughput needed = 1T / 86,400 ≈ 11.5M tok/s
  
  6. vLLM on H100: ~3,000 tok/s per GPU (continuous batching)
  7. GPUs needed = 11.5M / 3,000 ≈ 3,800 H100s
  
  8. KV Cache: 34.5K concurrent × 0.33 MB/token × 1000 tokens
     = 11.3 TB KV cache (distributed across GPUs)
  
  9. Cost: 3,800 × $2.50/hr = $9,500/hr = ~$7M/month
  
  OPTIMIZATIONS:
  • INT8 quantization → 2× throughput → 1,900 GPUs
  • Speculative decoding → 2× → 950 GPUs  
  • Smaller model for simple queries (Haiku) → 50% traffic
  • Semantic caching → 30% hit rate → 30% less compute
  
  FINAL: ~500 GPUs with full optimization stack
```

---

## 11.13 GPU Reliability Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│  GPU-SPECIFIC FAILURE MODES                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. GPU OOM (Out of Memory)                                     │
│     → Batch size auto-reduction                                 │
│     → Request queuing with backpressure                         │
│     → Graceful degradation: shorter context, smaller model      │
│                                                                 │
│  2. GPU Hang / Kernel Timeout                                   │
│     → Watchdog timer (NVIDIA: `__cudaTimeout`)                  │
│     → Process isolation (1 model per process)                   │
│     → Auto-restart on failure                                   │
│                                                                 │
│  3. NVLink / Fabric Failure                                     │
│     → Health checks (ncclTest, p2p bandwidth)                   │
│     → Fallback to Ethernet (slower but works)                   │
│     → Drain node, reschedule                                    │
│                                                                 │
│  4. Thermal Throttling                                          │
│     → DCGM monitoring (temp, power, clock)                      │
│     → Reduce batch size, increase fan                           │
│     → Migrate workloads                                         │
│                                                                 │
│  5. Driver / CUDA Version Mismatch                              │
│     → Containerized with pinned versions                        │
│     → Compatibility matrix in CI                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11.14 Security for AI Systems

| Threat | Mitigation |
|--------|------------|
| **Prompt Injection** | Input classification, instruction hierarchy, sandboxed execution |
| **Data Leakage** | PII redaction, output filtering, no training on user data |
| **Model Extraction** | Rate limiting, watermarking, API-only access |
| **Adversarial Examples** | Input validation, ensemble detection, robust training |
| **Supply Chain** | Model signing (sigstore), SBOM, provenance verification |
| **Jailbreak** | Constitutional AI, red-teaming, continuous evaluation |

---

## ✅ Exercises

1. **Capacity**: Estimate GPUs for 1M DAU chat app with 20 msg/day, 1K tokens/msg.
2. **RAG Design**: Design RAG for 10M legal documents. Chunking, vector DB, reranking strategy.
3. **Optimization**: Your p99 TTFT is 2s (SLO: 500ms). Diagnose and fix with 3 techniques.
4. **Gateway**: Design AI Gateway for multi-tenant with: free/pro/enterprise tiers, cost caps, model routing.

---

## 🔁 Review Prompts

- Why is KV cache the binding constraint for LLM serving?
- What's the difference between tensor parallelism and pipeline parallelism?
- How does continuous batching improve GPU utilization?
- What metrics would you monitor for an LLM serving platform?
- How does semantic caching differ from traditional caching?

---

## 📚 Further Reading

- [vLLM Documentation](https://docs.vllm.ai/)
- [SGLang Documentation](https://github.com/sgl-project/sglang)
- *LLM Serving at Scale* — vLLM / TensorRT-LLM papers
- [MLflow LLMOps Guide](https://mlflow.org/docs/latest/llms/index.html)
- *Designing ML Systems* — Chip Huyen (Ch on Serving)
- [Google's LLM Serving Paper](https://arxiv.org/abs/2303.08774)
- [PagedAttention Paper](https://arxiv.org/abs/2309.06180)
