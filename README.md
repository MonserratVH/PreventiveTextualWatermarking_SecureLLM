# Preventive Textual Watermarking for Secure LLM Request Pipelines

#### Abstract

Large Language Models (LLMs) are increasingly deployed as general-purpose reasoning engines across diverse domains. However, current interaction paradigms implicitly assume that all incoming requests are trustworthy, well-formed, and authorized. This assumption exposes LLM-based systems to a wide range of risks, including prompt tampering, unauthorized access, replay attacks, and uncontrolled prompt manipulation.

Existing defenses in adversarial NLP largely focus on **reactive mechanisms**, such as output filtering or adversarial detection after model execution. In contrast, **preventive mechanisms for ensuring input integrity and provenance before model invocation remain underexplored**.

In this project, we propose a **preventive textual watermarking mechanism** designed as a *security belt* for LLM request pipelines. The mechanism embeds an **invisible textual watermark** into user requests using deterministic encoding rules. Before a request reaches the LLM, a verifier reconstructs the expected watermark and validates its integrity. Only requests that pass this verification step are allowed to access the model.

The proposed approach is **model-agnostic**, requires no access to gradients or internal model parameters, and operates independently of the downstream LLM architecture.



## Preventive Watermarking Mechanism

We assume that **not all prompts reaching an LLM should be trusted by default**. Instead, requests must be generated through an authorized mechanism that embeds a verifiable structure into the text itself.

Our hypothesis is that **invisible textual watermarks**, when combined with deterministic reconstruction, can serve as a preventive gate that enforces request integrity *before* model execution. Unlike cryptographic authentication or API-level checks, this mechanism operates directly at the **textual level**, making it compatible with any LLM interface.

The proposed mechanism relies on three core principles:

1. **Imperceptibility**  
   The watermark does not alter the visible content or semantics of the request.

2. **Determinism**  
   The watermark can be reconstructed exactly from the visible text using shared parameters.

3. **Preventive gating**  
   Requests that fail verification are blocked before reaching the LLM.

---

#### Watermark Encoding Process
![Preventive Textual Watermarking for Secure LLM](https://github.com/MonserratVH/PreventiveTextualWatermarking_SecureLLM/blob/main/Figures/LLM_SafetyBelt.jpg)

The watermark insertion process is performed by the `Encoder` module. Given a clean user request, the encoder:

1. **Derives a secret key** from a reference corpus using a cryptographic hash function.  
   This corpus-based key derivation is **used exclusively for experimental reproducibility** and controlled evaluation.  
   In practical deployments, **any standard cryptographic key management scheme** (e.g., pre-shared keys, HSM-backed keys, or KMS-managed secrets) can be used without modifying the watermarking logic.

2. **Determines instance-level and term-level hash table sizes** based on the secret key using hash tables and hash functions.

3. **Distributes characters across hash tables**, inserting invisible Unicode markers into unused slots.

4. **Produces a watermarked request**, which is visually identical to the original text but contains a hidden structural signature.

The output of this process is wrapped into a `MarkedRequest` object, which includes:
- the watermarked payload,
- a nonce,
- a timestamp,
- a session identifier,
- encoding latency.

---

#### Belt Verification (Preventive Gate)

The `BeltVerifier` acts as a **preventive security belt** placed immediately before the LLM.

Given a `MarkedRequest`, the verifier:

1. Removes invisible markers to recover the visible text.
2. Re-encodes the visible text using the same deterministic mechanism.
3. Reconstructs the expected watermark using the shared secret key.
4. Compares the reconstructed watermark with the observed one.
5. Emits a verification status: `VALID` or `INVALID`.

Only requests with an exact match are allowed to proceed. This design ensures that **any modification to the request—intentional or accidental—breaks watermark consistency and triggers rejection**.

---

#### End-to-End Pipeline

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

#### Repository Structure

```text
.
├── Figures
├── watermarked_belt.ipynb   # Encoder, verifier, pipeline, and mock LLM
├── README.md             # Project documentation

```

---

#### Usage Example
```python
# Clean user request
raw_text = "Explain why gradient masking does not guarantee robustness."

#### Encode the request (insert watermark)
encoder = Encoder(raw_text, session_id=42)
meta_request = encoder.meta_request

#### Verify the watermark
belt = BeltVerifier(meta_request)
verification = belt.as_result()

print("Verification status:", verification.status)

#### If valid, extract clean text for the LLM
if verification.status == "VALID":
    clean_text = belt.decode_visible_text()
    print("Text passed to LLM:", clean_text)
else:
    print("Request blocked.")
```

---

#### Design Characteristics
- Preventive rather than reactive
- LLM-agnostic
- Independent of model internals
- Lightweight and modular
- Compatible with adversarial NLP threat models

--- 

#### Limitations and Assumptions 
- This mechanism does not provide cryptographic authentication.
- It assumes a trusted execution environment for the encoder and verifier.
- Compromise of the secret key would invalidate the verification process.

These limitations are consistent with the explicit threat model considered in adversarial NLP research and are intentionally scoped. 


## Research Context

This project contributes to research on:

- Adversarial NLP
- Preventive defense mechanisms
- Textual watermarking
- Integrity and provenance in LLM request pipelines

It can be used as:
- a research prototype,
- a teaching artifact,
- or a baseline for further defensive designs.


## Researchers

- _Monserrat Vázquez-Hernández_  
    mvazquez@inaoe.mx  
    https://orcid.org/0000-0001-9206-5706

- _Luis Alberto Morales-Rosales_  
    lamorales@conacyt.mx  
    [https://orcid.org/0000-0002-4753-9375](https://orcid.org/0000-0002-4753-9375)
  

- _Ignacio Algredo-Badillo_  
    algredobadillo@inaoep.mx  
    https://orcid.org/0000-0002-4748-3500

- _Luis Villaseñor-Pineda_  
    villasen@inaoep.mx  
    https://orcid.org/0000-0003-1294-9128

## Acknowledgements
- This work is supported by CONAHCYT/México scholarship 814461. Besides, it was founded by Catedras-CONAHCYT projects 882 and 613
