# Bazel Codebase Guide for Gemini

Welcome to the Bazel repository. This document provides essential context and
workflows for working with the Bazel codebase.

## 🏗️ Building & Running Bazel

To build Bazel from source, use a pre-installed `bazel` or `bazelisk` binary.

*   **Fast Iteration Build (Recommended):** `bazel build //src:bazel-dev`
*   **Standard Build:** `bazel build //src:bazel`

### ⚡ Performance Tips

*   **Remote Execution:** If you have access to a Remote Build Execution (RBE)
cluster, use `--config=remote` to significantly speed up your builds:
    ```bash
    bazel build --config=remote //src:bazel-dev
    ```

### 🔄 Iterative Development Workflow

To test changes safely without interfering with your primary workspace,
use a separate binary and output base:

1.  **Build and Copy:**
    ```bash
    bazel build //src:bazel-dev && cp bazel-bin/src/bazel-dev /tmp/bazel
    ```
2.  **Run commands:**
    ```bash
    /tmp/bazel --output_base=/tmp/ob-dev <command>
    ```

**Note:** Using a custom `--output_base` (e.g., `/tmp/ob-dev`) is crucial to
avoid locking your main workspace server and ensures your development
environment remains isolated.

## 🧪 Testing

Tests are located primarily in `src/test`.

### Finding Relevant Tests
If you modify a file, find tests that transitively depend on it:
```bash
bazel query "rdeps(//src/test/..., path/to/file.java)"
```

### Running Tests

*   **Unit Tests:** Typically `java_test` targets.
*   **Integration Tests:**
    *   Java-based: Subclasses of `BuildIntegrationTestCase`.
    *   Shell-based: Located in `src/test/shell`, using a bash test framework.

## 📜 Architecture & Codebase

Bazel uses a **Client/Server** architecture. The client (C++) is a lightweight
wrapper that starts and communicates with a long-lived Java server.

*   **Skyframe:** The incremental evaluation framework. Most core logic is
implemented as `SkyFunction`s evaluating `SkyKey`s into `SkyValue`s.
*   **Starlark:** The configuration language used for `BUILD` and `.bzl` files.
*   **Loading/Analysis/Execution:** The three phases of a Bazel command.

For architectural details or introspection capabilities,
see: `docs/contribute/codebase.mdx`

## 🧹 Linting & Formatting

*   **Java:** Follows Google Java Style.
*   **Starlark:** Use `buildifier` for `BUILD` and `.bzl` files.



## 🛡️ Strict Embedding Separation, Zero-Fallback Law & 24/7 Dual-GPU Invariant
- **Reference**: `/home/m1st/.agents/rules/RULE_Strict_Embedding_Separation_And_Dual_Pipeline.md`

### 1. Das Absolute Fallback-Verbot (Zero-Fallback Law)
Unter keinen Umständen, zu keinem Zeitpunkt und aus keinem Grund darf ein Fallback zwischen verschiedenen Embedding-Modellen stattfinden.
* **Geltende Aktion:** Fällt ein Embedding-Modell aus oder ist überlastet, MUSS die Operation sofort hart fehlschlagen (`Fail-Fast`) oder die Payload transaktional in einer Queue (NATS/SQLite) verharren, bis das exakte Modell bereit ist.
* **Verboten:** Kein stiller oder dynamischer Modellwechsel (weder Jina -> Gemma noch umgekehrt).

### 2. Warum ein Embedding-Fallback mathematisch & informationstheoretisch unmöglich ist
* **Topologische Inkompatibilität heterogener Vektorräume (Non-Isomorphism):**
  Jedes Modell $f_\theta: \mathcal{X} \to \mathbb{R}^D$ projiziert Text in eine spezifische, gelernte Riemannsche Mannigfaltigkeit. Jina v5 ($D=256$) und EmbeddingGemma ($D=768$) spannen zwei völlig inkompatible geometrische Räume auf. Die Basisvektoren der semantischen Achsen sind ohne explizite Procrustes-Transformation nicht ausgerichtet.
* **Kollaps der Kosinus-Ähnlichkeit ($	ext{sim} \approx 0$):**
  Wird eine Suchanfrage mit Modell $B$ berechnet ($v_q = f_B(q)$), während der Dokumentenkorpus mit Modell $A$ indiziert wurde ($v_d = f_A(d)$), verhält sich das Skalarprodukt mathematisch wie das zweier rein zufälliger Vektoren auf einer hochdimensionalen Einheitssphäre:
  $$\mathbb{E}[\text{sim}(u, v)] = 0 \quad \text{mit Varianz} \quad \sigma^2 = \frac{1}{D}$$
  Der Nearest-Neighbor-Algorithmus (HNSW/k-NN) liefert stochastisches Rauschen. Das RAG-System erhält völlig falsche oder irrelevante Kontexte.
* **Irreversible Index-Vergiftung (Index Poisoning):**
  Wird auch nur ein einziger Vektor von Modell $B$ als "Fallback" in den Index von Modell $A$ geschrieben, verunreinigt er die Distanzgraphen und Clusterzentren dauerhaft.
* **Das Gesetz des Fail-Fast:**
  Ein Ausfall muss hart abbrechen (`HTTP 503 Service Unavailable / IngestionQueueBlocked`).

### 3. Duale 24/7 Erfassungspflicht (GPU-Only)
* **GPU-Only Mandat:** Es läuft absolut nichts auf der CPU — GPU ONLY (NVIDIA GB10 CUDA) für ausnahmslos jedes Embedding-Modell.
* **24/7 Parallelität:** Sowohl `jina-embeddings-v5-omni-nano-classification` (256D, ~4,1 GB VRAM) als auch `google/embeddinggemma-300m` (768D, ~1,2 GB VRAM) laufen dauerhaft 24/7 im VRAM (Summe ~5,3 GB VRAM).
* **Duale Erfassung:** Jeder zu indizierende Text/Chunk wird immer von beiden Modellen parallel eingebettet und getrennt persistiert.

### 4. Idioten- & Failsafe-Sicherung auf Datenbankebene
* **SQLite Schema CHECK-Constraints:**
  `model_signature TEXT NOT NULL CHECK(model_signature = '...')` und `dimension INTEGER NOT NULL CHECK(dimension = ...)` erzwingen atomare Abbrüche auf Engine-Ebene bei Modell-Mismatches.
* **Qdrant Collection Constraints:**
  Strikte Trennung in separate Collections (`dgx_text_embeddings_jina_256` vs `dgx_text_embeddings_gemma_768`) mit fixierter Vektordimension.


## ⚡ High-Quality Systems Programming Languages Priority (No-Python Policy)
- **Reference**: `/home/m1st/.agents/rules/RULE_High_Quality_Systems_Programming_Languages.md`
- **Rule**:
  1. **Bevorzugte Sprachen:** High Quality **Golang (Go), Rust, C++, Zig, PowerShell, C** sind IMMER und AUSNAHMSLOS die bevorzugten Programmiersprachen.
  2. **Kein Python:** Python ist für neue Daemons, Watcher, Automatisierungen, APIs, CLI-Tools und Dienste strikt untersagt (GIL-Bottlenecks, Dependency-Drift, Speicherineffizienz).
  3. **Natives Systems-Engineering:** Alle Hintergrunddienste, Caching-Ebenen und Task-Runner müssen als native, speichersichere und nebenläufige Binaries kompiliert werden.
