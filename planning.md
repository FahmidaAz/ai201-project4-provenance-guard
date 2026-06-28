Provenance Guard — Planning Document

AI201 Project 4 | Fahmida Azmin


Architecture

Submission Flow

POST /submit  (accepts: text, creator_id)
      │
      ├──► llm_signal(text)
      │         └── Calls Groq API (llama-3.3-70b-versatile)
      │             Returns score 0.0–1.0
      │
      ├──► stylometric_signal(text)
      │         └── Pure Python math (sentence variance + TTR)
      │             Returns score 0.0–1.0
      │
      ├──► confidence = (llm_score × 0.6) + (stylo_score × 0.4)
      │
      ├──► get_attribution(confidence)
      │         └── "likely_ai" | "uncertain" | "likely_human"
      │
      ├──► get_label(confidence)
      │         └── Transparency label text for the user
      │
      ├──► append entry to audit_log
      │
      └──► return JSON { content_id, attribution, confidence, label }

Appeal Flow

POST /appeal  (accepts: content_id, creator_reasoning)
      │
      ├──► find entry in audit_log by content_id
      ├──► update status: "classified" → "under_review"
      ├──► store creator_reasoning in log entry
      └──► return confirmation JSON


Detection Signals

Signal 1: LLM Classifier (Groq)

PropertyDetailMeasuresWhether text reads as AI-generated based on semantic patterns, tone uniformity, and hedging languageOutputFloat 0.0–1.0 (1.0 = clearly AI-generated)Why chosenLLMs detect subtle semantic patterns — overly balanced phrasing, lack of personal voice, generic transitions like "Furthermore" and "It is important to note"Blind spotWell-written formal human prose (academic papers, legal writing) may score high; casual AI output with slang may score low

Signal 2: Stylometric Heuristics (Pure Python)

PropertyDetailMeasuresSentence length variance + type-token ratio (vocabulary diversity)OutputFloat 0.0–1.0 (1.0 = AI-like uniformity)Why chosenAI writing is statistically more uniform — similar sentence lengths, repeated vocabulary. Human writing is messier and more variableBlind spotNon-native English speakers write more uniformly, which may inflate their AI score. Short texts (< 2 sentences) cannot be reliably scored

Why These Two Signals Together

The signals are genuinely independent — one is semantic (what the text means), one is structural (how the text is built). A piece of writing could fool one signal but not both. That independence makes the combined score more reliable than either signal alone.


Confidence Scoring

Formula: confidence = (llm_score × 0.6) + (stylometric_score × 0.4)

Signal 1 is weighted higher (0.6) because semantic analysis is more reliable than surface-level statistics for distinguishing AI from human writing.

Score RangeAttributionMeaning> 0.75likely_aiStrong indicators of AI generation across both signals0.45 – 0.75uncertainMixed signals — system cannot determine with confidence< 0.45likely_humanStrong indicators of human authorship

What 0.5 means: The system is genuinely split — one signal says AI, the other says human, or both are near the midpoint. The user sees the uncertain label and is offered an appeal path.

Design note: The threshold for likely_ai is intentionally high (0.75) because a false positive — labeling a human writer's work as AI — is worse than a false negative on a creative writing platform. When in doubt, the system returns uncertain rather than accusing.


Transparency Labels

High-Confidence AI (score > 0.75)


"This content was likely AI-generated. Our system detected patterns consistent with AI writing tools. If this is your original work, you can submit an appeal."



Uncertain (score 0.45 – 0.75)


"We were unable to determine with confidence whether this content is AI-generated or human-written. Confidence score: [score]. You may appeal if you believe this is incorrect."



High-Confidence Human (score < 0.45)


"This content appears to be human-written. No action required."




Appeals Workflow

Endpoint: POST /appeal

Accepts:


content_id — the ID returned by /submit
creator_reasoning — the creator's explanation in plain text


What happens on appeal:


System locates the original audit log entry by content_id
Status updates from "classified" to "under_review"
creator_reasoning is stored alongside the original decision
Confirmation returned to the creator


What a human reviewer sees in the log:


Original attribution and confidence score
Both individual signal scores (llm_score, stylometric_score)
The creator's reasoning
Timestamp of original submission and appeal


Automated re-classification: Not implemented. Appeals are flagged for human review only.


Rate Limiting

Limits: 10 requests per minute, 100 requests per day (per IP address)

Reasoning:


A real writer submits work infrequently — 10/minute is generous for legitimate use
100/day prevents automated scripts from flooding the detection pipeline
These limits stop abuse while not affecting any real creative workflow



Edge Cases


Formal academic writing by humans — high sentence uniformity and formal vocabulary may inflate the stylometric score, pushing borderline cases toward uncertain or likely_ai. The appeal path exists for this reason.
Casual or slang-heavy AI output — high vocabulary variance from informal language may lower the stylometric score, causing some AI content to appear uncertain. The LLM signal should partially compensate for this.
Very short text (1–2 sentences) — stylometric analysis requires at least 2 sentences to compute variance. Short submissions default to a neutral stylometric score of 0.5 and rely more heavily on the LLM signal.
Non-native English writers — more uniform sentence structure may inflate AI scores. The uncertain tier and appeals workflow are the mitigation here, not the detection signals themselves.



AI Tool Plan

M3 — Submission endpoint + Signal 1


Provide to AI: Detection signals section + architecture diagram
Ask for: Flask app skeleton with POST /submit stub + llm_signal() function
Verify: Call function directly with 3 test inputs before wiring into endpoint; check score varies meaningfully


M4 — Signal 2 + confidence scoring


Provide to AI: Detection signals section + confidence scoring section + architecture diagram
Ask for: stylometric_signal() function + scoring combination logic
Verify: Clearly AI text scores above 0.75; clearly human text scores below 0.45; borderline cases land in uncertain range


M5 — Production layer


Provide to AI: Transparency labels section + appeals workflow section + architecture diagram
Ask for: get_label() function + POST /appeal endpoint
Verify: All three label variants are reachable by submitting inputs at different confidence levels; appeal correctly updates status to under_review and appears in /log