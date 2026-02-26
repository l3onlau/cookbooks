### Recipe Name: Edge-Optimized Agentic RAG

### Description

A simple, complete chain for building an AI chatbot operating strictly on a 4.8GB usable VRAM / 12GB RAM budget. The pipeline is designed to minimize hallucinations and maximize faithfulness through strict routing and evaluation.

### Ingredients (Swappable)

* **Primary LLM (gpu):** `qwen3:4b-instruct-2507-q4_K_M` (via Ollama). Use KV cache quantization (8-bit is highly recommended to save memory). Justification: This model performs pretty solid for its size. The thinking variant is too slow with DSPy's Chain of Thought and lacks flexibility, while the older hybrid model performs worse than both newer counterparts with respective mode.
* **Cross-Encoder (cpu):** `tomaarsen/Qwen3-Reranker-0.6B-seq-cls`, `Zerank 1 Small` or `BAAI/bge-reranker-v2-m3` (Great for high-accuracy reranking on low resources).
* **Bi-Encoder (gpu):** `qwen3-embedding:0.6b-q8_0` (Provides solid baseline retrieval and potential semantic synergy with the Qwen3 LLM).
* **NLI (Natural Language Inference) Model (cpu):** Choose a lightweight model (like `deberta-v3-xsmall-mnli`, `tasksource/ModernBERT-base-nli` or `MoritzLaurer/DeBERTa-v3-large-mnli-fever-anli-ling-wanli`) to act as a judge for hallucination detection.
* **Nano LLMs (cpu/gpu):** Optional, specialized models (like `SmolLM2`) for specific micro-tasks.
* **Evaluation Dataset:** Optional file containing `question`, `reference_answer`, and `expected_result`.
* **Dataset Uses:** Essential for DSPy optimization, dynamic few-shot injection (a massive zero-VRAM accuracy boost!), local RAG evaluation during hyperparameter tweaking, generated answers for debugging,and cloud-to-edge LoRA fine-tuning.
* **Data Generation (Synthetic/Distillation):** Use techniques like Inverted RAG (context-first generation), Adversarial Generation (testing if the AI can say "I don't know"), and Complexity Scaling (Evol-Instruct) to simulate noisy, real-world inputs.



**Example Synthetic Data Prompt:**

> System: You are an expert QA engineer building a SOTA evaluation dataset for a 4B parameter RAG pipeline.
> Task: Read the following [SOURCE_CHUNK]. Generate 3 JSON objects mapping to this exact schema: `{"question": string, "guidance": string, "expected_state": string, "reference_answer": string}`.
> * One straightforward question.
> * One complex, multi-step reasoning question.
> * One adversarial question where the answer is NOT in the text.
> 
> 
> Source Chunk: [Insert your FAISS chunk here]

---

### Instructions

**1. Data Ingestion Pipelines (Note: The *Docling* library is highly recommended for these pipelines)**

* **The Context Orphan Trap (Metadata Prepending):** Fix this using Metadata Injection so chunks don't lose their source context.
* **Structural/Semantic Chunking (Ditching Token Limits):** Fix this using Markdown-Aware Splitting to keep logical sections intact.
* **Boilerplate Poisoning (The Silent Killer):** Fix this by using Trafilatura and Regex stripping to clean raw web data.

**2. Core RAG Skills**

* **Hybrid Search:** Combine dense (vector) and sparse (BM25/keyword) retrieval.
* **Zero-VRAM Reranking:** Offload reranking to the CPU or use a lightweight cross-encoder.
* **Small-to-Big Retrieval:** Also known as sentence-window retrieval; fetch small chunks for matching, but feed the surrounding context to the LLM.
* **HyDE (Hypothetical Document Embeddings):** *(Note: this often fails with 4B models—usually, smaller models lack the world knowledge to generate accurate hypothetical answers)*.
* **Step-Back Prompting:** A highly effective alternative to HyDE for smaller models to extract higher-level concepts.
* **Prompt Compression:** Use tools like LLMLingua to aggressively shrink context window usage.
* **Self-Querying:** Perform pre-retrieval metadata extraction to filter vector databases accurately.
* **Re-Reading:** Interleave context with instructions to keep the LLM focused.
* **Fallback Routing:** CRAG (Corrective RAG) might be too heavy, but you can use threshold-based routing to trigger retries, execute error handling, fallback to alternative retrieval routes, or take other actions if retrieval confidence is low.

**3. AI Chaining & Multi-Agent Architecture**

* **Dynamic Offloading:** To pull off multi-agent workflows on limited RAM, use a Finite State Machine (FSM) to load and unload specific models dynamically. 
* **Graph Chaining:** Construct the workflow using appropriate nodes (actions) and edges (conditional logic).
* **Recommended Agent Roles:**
* **Router:** Directs the user's query to the right tool or sub-agent.
* **Tool Caller:** Executes web searches, database queries, etc.
* **Synthesizer:** Dedicated purely to generating the final, polished response.
* **Prompt Engineer:** `SmolLM2` is excellent for optimizing prompts on the fly; research other nano-models for specific routing tasks.



**4. Hallucination Mitigation & Output Judging**

* **NLI as a Cross-Encoder Judge:** Because a 4B parameter model will struggle to reliably self-correct, use a dedicated NLI model to judge the final output. Have it check for entailment (ensuring the answer matches the retrieved facts) and contradiction (flagging hallucinations). If it fails the check, route the chain back to retry or fail gracefully.

**5. Strict Format Enforcement**

* **GBNF & Structured Outputs:** State-of-the-art pipelines use GBNF (Grammar-Based Network Format). This intercepts the LLM's token generation at the C++ level (via `llama.cpp`) to guarantee the output perfectly matches a Pydantic/JSON schema. It prevents the model from generating formatting errors, which is critical for agentic chaining.
* **Alternative Frameworks (Pydantic AI / DSPy / Instructor):** You can manage structured outputs at the Python level instead of the C++ level using these frameworks.
* **Pros:** Easier to integrate into complex Python logic, seamless validation, and built-in "retry" loops if the model gets the schema wrong. DSPy even auto-optimizes the prompts to improve formatting accuracy over time.
* **Cons:** They rely on the LLM's inherent ability to follow JSON instructions. For a 4B parameter model, this can lead to frequent validation failures and costly retry loops that eat up processing time, whereas GBNF guarantees 100% compliance on the first pass.





### Notes

#### Hardware Note: CPU Deployment

While this guide targets a strict VRAM budget, a purely CPU-based setup with sufficient system RAM (e.g., 16GB+) can also execute this pipeline effectively. Although inference speeds will be slower compared to GPU acceleration, utilizing efficient backends like llama.cpp on a modern multi-core processor can yield perfectly reasonable and usable speeds depending on your specific hardware specifications.

#### Baseline Hyperparameters & Settings

Rather than relying on static, hardcoded baseline settings for chunking, retrieval, and generation, you should dynamically determine the best configuration for your specific data:

* **LLM Evaluation:** Use an LLM-as-a-judge or your designated NLI model to empirically evaluate generation quality and faithfullness across different settings.
* **Active Research:** Consult current documentation and community benchmarks for your chosen models (like Qwen3:4b and your cross-encoder) to find their natively optimized context limits and parameters.
* **Iterative Experimentation:** Actively tweak and test variables—such as chunk sizes, overlap thresholds, Top-K retrieval limits, and sampling temperatures—against your Evaluation Dataset to find the exact sweet spot between speed, RAM usage, and response accuracy.

#### Model Scaling via Partial Offloading

You can also utilize an 8B parameter model, as Ollama will partially offload the model to the CPU. In my situation, the LLM will slow down by around 100%, but the overall speed penalty will be lower as it is only part of the pipeline and quite manageable depending on your pipeline's needs.

#### Reusability and Tooling

Keep in mind that some parts of this pipeline can be separated into distinct tools or implemented using OpenMCP (Model Context Protocol). This approach significantly enhances reusability, simplifies maintenance, and allows for much easier scaling of individual components later on.

#### Reference Implementation

This repository provides the implementation of the modular agentic architecture described in the recipe above:
**[Modular Agentic Chat](https://github.com/l3onlau/modular_agentic_chat)**