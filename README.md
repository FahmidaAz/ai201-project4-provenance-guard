Provenance Guard

AI201 Project 4 | Fahmida Azmin

A backend API that classifies text-based creative content as AI-generated or human-written, returns a confidence score, surfaces a transparency label, and handles creator appeals. Built with Flask + Groq.


Architecture Overview

A submitted piece of text takes the following path through the system:

POST /submit  (text + creator_id)
      │
      ├──► Signal 1: llm_signal()
      │         Calls Groq API (llama-3.3-70b-versatile)
      │         Returns score 0.0–1.0
      │
      ├──► Signal 2: stylometric_signal()
      │         Pure Python — sentence length variance + type-token ratio
      │         Returns score 0.0–1.0
      │
      ├──► confidence = (llm_score × 0.6) + (stylo_score × 0.4)
      │
      ├──► get_attribution() → "likely_ai" | "uncertain" | "likely_human"
      │
      ├──► get_label() → transparency label text
      │
      ├──► append to in-memory audit_log
      │
      └──► return JSON response

POST /appeal  (content_id + creator_reasoning)
      │
      ├──► find entry by content_id
      ├──► update status → "under_review"
      ├──► store creator_reasoning
      └──► return confirmation

GET /log → returns all audit entries as JSON


API Endpoints

MethodEndpointDescriptionPOST/submitSubmit text for attribution analysisPOST/appealContest a classification by content_idGET/logView structured audit log entries

POST /submit

Request:

json{
  "text": "Your content here...",
  "creator_id": "user-123"
}

Response:

json{
  "content_id": "3f7a2b1e-...",
  "attribution": "likely_ai",
  "confidence": 0.78,
  "label": "This content was likely AI-generated..."
}

POST /appeal

Request:

json{
  "content_id": "3f7a2b1e-...",
  "creator_reasoning": "I wrote this myself. I am a non-native English speaker."
}

Response:

json{
  "content_id": "3f7a2b1e-...",
  "message": "Appeal received. Your content is now under review.",
  "status": "under_review"
}


Detection Signals

Signal 1: LLM Classifier (Groq)

Uses llama-3.3-70b-versatile to assess whether text reads as AI-generated or human-written based on semantic patterns — uniform tone, hedging language ("It is important to note"), overly balanced phrasing, and lack of personal voice.

Output: Float 0.0–1.0 (1.0 = clearly AI-generated)

Why chosen: LLMs detect subtle semantic coherence patterns that statistical methods miss. A model that generates text also recognizes the signatures of generated text.

What it misses: Casual AI output with slang may score low. Well-written formal human prose may score high.

Signal 2: Stylometric Heuristics (Pure Python)

Computes two statistical properties of the text:


Sentence length variance — AI text tends to have similar sentence lengths; human writing varies more
Type-token ratio (TTR) — vocabulary diversity (unique words ÷ total words); AI text reuses vocabulary more uniformly


Output: Float 0.0–1.0 (1.0 = AI-like uniformity)

Why chosen: Genuinely independent from Signal 1 — one is semantic, one is structural. That independence makes the combined score more reliable than either alone. No API calls required.

What it misses: Non-native English speakers write more uniformly, which may inflate their AI score. Texts shorter than 2 sentences cannot be reliably scored.


Confidence Scoring

Formula: confidence = (llm_score × 0.6) + (stylometric_score × 0.4)

Signal 1 is weighted higher (0.6) because semantic analysis is more reliable than surface statistics for this task.

Score RangeAttributionLabel Shown> 0.75likely_aiHigh-confidence AI label0.45 – 0.75uncertainUncertain label with score< 0.45likely_humanHuman-written label

The threshold for likely_ai is intentionally high (0.75) because a false positive — labeling a human's work as AI — is worse than a false negative on a creative platform.

Example Submissions with Different Confidence Scores

High-confidence human (confidence: 0.203)


Input: "ok so i finally tried that new ramen place downtown and honestly? underwhelming. the broth was fine but they put WAY too much sodium in it..."
Attribution: likely_human — casual language, high sentence variance, diverse vocabulary



Uncertain — formal human writing (confidence: 0.634)


Input: "The relationship between monetary policy and asset price inflation has been extensively studied in the literature..."
Attribution: uncertain — formal structure triggers stylometric signal but LLM detects human reasoning patterns




Transparency Labels

High-Confidence AI (score > 0.75)


"This content was likely AI-generated. Our system detected patterns consistent with AI writing tools. If this is your original work, you can submit an appeal."



Uncertain (score 0.45 – 0.75)


"We were unable to determine with confidence whether this content is AI-generated or human-written. Confidence score: [score]. You may appeal if you believe this is incorrect."



High-Confidence Human (score < 0.45)


"This content appears to be human-written. No action required."




Rate Limiting

Limits: 10 requests per minute, 100 requests per day (per IP address)

Reasoning:


A real writer submits work infrequently — 10/minute is generous for legitimate use
100/day prevents automated scripts from flooding the detection pipeline
These limits reflect realistic creative platform usage while blocking abuse


When the limit is exceeded, the API returns HTTP 429 Too Many Requests.


Audit Log

Every attribution decision is logged with the following structure:

json{
  "content_id": "3f7a2b1e-...",
  "creator_id": "test-user-1",
  "timestamp": "2026-06-29T03:18:34.209982Z",
  "attribution": "uncertain",
  "confidence": 0.584,
  "llm_score": 0.7,
  "stylometric_score": 0.41,
  "status": "classified",
  "appeal_reasoning": null
}

When an appeal is filed, status updates to "under_review" and appeal_reasoning is populated. View the log at GET /log.


Known Limitations

Formal academic writing by humans is the most likely false positive. High sentence uniformity and formal vocabulary patterns (consistent sentence length, low TTR for domain-specific jargon) cause the stylometric signal to score formal human writing similarly to AI output. A monetary policy analysis written by an economist will consistently land in the uncertain range regardless of authorship. The appeal path exists for this reason — the system acknowledges its uncertainty rather than making a confident wrong call.


Spec Reflection

Where the spec helped: Defining the three transparency label variants in planning.md before writing any code made the get_label() function straightforward to implement — the exact text was already decided, so implementation was just translating the spec into a conditional.

Where implementation diverged: The spec anticipated likely_ai scores above 0.75 for clearly AI-generated text. In practice, the AI text test (paradigm shift paragraph) scored 0.584 — landing in uncertain rather than likely_ai. The LLM signal returned 0.7 but the stylometric signal returned lower, pulling the combined score down. This revealed that the stylometric signal is less sensitive to polished AI prose than expected. In production, I would increase the LLM weight to 0.7 or add a third signal.


AI Usage

Instance 1 — Flask app skeleton and detection functions:
I provided the architecture diagram from planning.md and the detection signals section, and asked the AI to generate the Flask app skeleton with POST /submit, llm_signal(), and stylometric_signal(). The generated stylometric_signal() normalized variance incorrectly (divided by 1000 instead of 100), causing all stylometric scores to cluster near 1.0. I corrected the normalization formula to min(variance / 100, 1.0) based on testing with known inputs.

Instance 2 — Refuse prompt for RepairSafe (Lab 4 reference):
In the prior lab, I asked the AI to pressure-test a refuse-tier system prompt by identifying ways an LLM might still provide dangerous instructions. It surfaced the "here's how professionals do it" loophole. I revised the prompt to explicitly prohibit describing the process in any framing, which fixed the behavior.


Setup

bashgit clone https://github.com/FahmidaAz/ai201-project4-provenance-guard
cd ai201-project4-provenance-guard
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt

Create a .env file:

GROQ_API_KEY=your_key_here

Run:

bashpython app.py

Test:

bash# Windows PowerShell
$body = '{"text": "Your text here", "creator_id": "test-user"}'
Invoke-WebRequest -Uri http://localhost:5000/submit -Method POST -ContentType "application/json" -Body $body | Select-Object -ExpandProperty Content


Stack


API framework: Flask
Rate limiting: Flask-Limiter
LLM: Groq (llama-3.3-70b-versatile)
Stylometrics: Pure Python (no external libraries)
Audit log: In-memory (runtime) — persisted via /log endpoint                                                                       




**Test Case Result**
<img width="1280" height="800" alt="Screenshot (381)" src="https://github.com/user-attachments/assets/fc637ce3-2fff-4748-871b-a5b07253dd51" />
<img width="1280" height="800" alt="Screenshot (384)" src="https://github.com/user-attachments/assets/a2ba496e-1f17-469e-babf-6f9b4c3ddf73" />
<img width="1280" height="800" alt="Screenshot (383)" src="https://github.com/user-attachments/assets/85140289-3644-492f-9c05-df8523c84d7c" />
<img width="1280" height="800" alt="Screenshot (382)" src="https://github.com/user-attachments/assets/109bffc2-7c7b-499e-86a3-72d0505e71e6" />


