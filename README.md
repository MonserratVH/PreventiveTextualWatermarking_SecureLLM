# Preventive Textual Watermarking for Secure LLM Request Pipelines

## Abstract

Large Language Models (LLMs) are increasingly deployed as general-purpose reasoning engines across diverse domains. However, current interaction paradigms implicitly assume that all incoming requests are trustworthy, well-formed, and authorized. This assumption exposes LLM-based systems to a wide range of risks, including prompt tampering, unauthorized access, replay attacks, and uncontrolled prompt manipulation.

Existing defenses in adversarial NLP largely focus on **reactive mechanisms**, such as output filtering or adversarial detection after model execution. In contrast, **preventive mechanisms for ensuring input integrity and provenance before model invocation remain underexplored**.

In this project, we propose a **preventive textual watermarking mechanism** designed as a *security belt* for LLM request pipelines. The mechanism embeds an **invisible textual watermark** into user requests using deterministic encoding rules. Before a request reaches the LLM, a verifier reconstructs the expected watermark and validates its integrity. Only requests that pass this verification step are allowed to access the model.

The proposed approach is **model-agnostic**, requires no access to gradients or internal model parameters, and operates independently of the downstream LLM architecture.

---

## Preventive Watermarking Mechanism

We assume that **not all prompts reaching an LLM should be trusted by default**. Instead, requests must be generated through an authorized mechanism that embeds a verifiable structure into the text itself.

Our hypothesis is that **lightweight, invisible textual watermarks**, when combined with deterministic reconstruction, can serve as a preventive gate that enforces request integrity *before* model execution. Unlike cryptographic authentication or API-level checks, this mechanism operates directly at the **textual level**, making it compatible with any LLM interface.

The proposed mechanism relies on three core principles:

1. **Imperceptibility**  
   The watermark does not alter the visible content or semantics of the request.

2. **Determinism**  
   The watermark can be reconstructed exactly from the visible text using shared parameters.

3. **Preventive gating**  
   Requests that fail verification are blocked before reaching the LLM.

---

## Watermark Encoding Process

The watermark insertion process is performed by the `Encoder` module. Given a clean user request, the encoder:

1. **Derives a secret key** from a reference corpus using a cryptographic hash function.  
   This corpus-based key derivation is **used exclusively for experimental reproducibility** and controlled evaluation.  
   In practical deployments, **any standard cryptographic key management scheme** (e.g., pre-shared keys, HSM-backed keys, or KMS-managed secrets) can be used without modifying the watermarking logic.

2. **Determines instance-level and term-level hash table sizes** based on the secret key and input statistics.

3. **Distributes characters across hash tables**, inserting invisible Unicode whitespace markers into unused slots.

4. **Produces a watermarked request**, which is visually identical to the original text but contains a hidden structural signature.

The output of this process is wrapped into a `MarkedRequest` object, which includes:
- the watermarked payload,
- a nonce,
- a timestamp,
- a session identifier,
- encoding latency.

---

## Belt Verification (Preventive Gate)

The `BeltVerifier` acts as a **preventive security belt** placed immediately before the LLM.

Given a `MarkedRequest`, the verifier:

1. Removes invisible markers to recover the visible text.
2. Re-encodes the visible text using the same deterministic mechanism.
3. Reconstructs the expected watermark using the shared secret key.
4. Compares the reconstructed watermark with the observed one.
5. Emits a verification status: `VALID` or `INVALID`.

Only requests with an exact match are allowed to proceed. This design ensures that **any modification to the request—intentional or accidental—breaks watermark consistency and triggers rejection**.

---

## End-to-End Pipeline

The full pipeline is implemented in `WatermarkedLLMPipeline` and follows this sequence:

1. User submits a clean text request.
2. The `Encoder` inserts an invisible watermark.
3. The `BeltVerifier` validates watermark integrity.
4. If the request is `VALID`:
   - invisible markers are removed,
   - clean text is forwarded to the LLM.
5. If the request is `INVALID`:
   - the request is blocked,
   - the LLM is never invoked.

This pipeline enforces **prompt-level authorization**, independent of the LLM provider or deployment environment.

---

## Repository Structure

```text
.
├── watermarked_belt.py   # Encoder, verifier, pipeline, and mock LLM
├── README.md             # Project documentation
