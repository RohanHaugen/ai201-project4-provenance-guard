

## Architecture overview: the path a submission takes from input to transparency label
Text Input: Takes text
  │
  ▼
Rate Limiter: prevent information if too many requests have ben recieved
  │
  ▼
Input Validation (text, creator_id)
  │
  ├──► Signal 1: Groq LLM Classifier
  │         Input: text
  │         Output: llm_score (0.0–1.0), rationale string
  │
  ├──► Signal 2: Stylometric Heuristics
  │         Input: text
  │         Metrics: sentence_length_variance, unique_words, punctuation_ratio
  │         Output: stylometric_score (0.0–1.0)
  │
  ▼
Confidence
  │    input: llm_score and stylometric_score
  │    combined = 0.6 × llm_score + 0.4 × stylometric_score
  │    output: map(combined → "likely_ai" | "uncertain" | "likely_human")
  ▼
Label Generator
  │    Input: combined score  verdict
  │    Output: label text string for display
  ▼
Audit Log with SQLite 
  │    Writes: content_id, creator_id, timestamp, llm_score,
  │            stylometric_score, confidence, verdict, status
  ▼
Output

Appeal
  │
  ▼
Input Validation (content_id, creator_reasoning)
  │
  ▼
Database Lookup by content_id
  │    ──► error if not found
  ▼
Status Update: "classified" → "under_review"
  │
  ▼
Audit Log Writer
  │    Appends: appeal_timestamp, appeal_reasoning to record
  ▼
Response
  {content_id, status: "under_review", message}

## Detection signals: what each signal measures, why you chose it, and what it misses
LLM-based Classification: measures semantics of whether a work reads like AI, should capture things like perfect grammar without a specific voice or other things that read like AI. 
Output: An object containing the score from 0.0 to 1.0, with 1.0 being 100% thinking to be AI and 0.0 to be entirely human and the rationale for the decision. 
Stylometric Heuristics: measures structural properties of text. Should capture things like AI invariance in sentence length, AI preferred word choices, and AI use of non-standard punctuation. 
Output: a float from 0.0 to 1.0, 0 not ai and 1 ai.
The reason I chose these is because AI detection can be fooled by word choice, so stylometric heuristics is there to ensure that even if the words are replaced, if the underlying structure is AI, it will be detected.
The LLM-based classification can be tricked with personal voice and word choice, since that is what it is looking for. The heuristics can be fooled by short pieces of text, since heuristics relies upon scale.
## Confidence scoring: how you combined signals into a score, how you validated it's meaningful, and two example submissions with noticeably different confidence scores (one high-confidence, one lower-confidence) showing the actual scores
I used 60% llm and 40% heuristics, my breakpoints are 0.72 for AI and 0.4 for human, with the middle being uncertain.
Example 1: ok so i finally tried that new ramen place downtown and honestly? 
underwhelming. the broth was fine but they put WAY too much sodium in it and 
i was thirsty for like three hours after. my friend got the spicy version and 
said it was better. probably won't go back unless someone drags me there
"confidence": 0.2101
"llm_score": 0.1
"stylometric_score": 0.3753
Example 2:
"The relationship between monetary policy and asset price inflation has been 
extensively studied in the literature. Central banks face a fundamental tension 
between their mandate for price stability and the unintended consequences of 
prolonged low interest rates on equity and real estate valuations."
"confidence": 0.6613
"llm_score": 0.8
"stylometric_score": 0.4533
## Transparency label: typed description of all three variants (high-confidence AI, human, uncertain) showing the exact text each one displays; screenshot or mockup optional
Likely AI-generated
Our analysis found strong indicators that this content may have been created with 
AI assistance. Confidence: [XX]%. The author can contest this classification 
if they believe it's incorrect.

Authorship uncertain
Our tools couldn't confidently determine whether this content was written by a 
person or with AI assistance. Confidence in AI authorship: [XX]%. The author 
can provide more context if they'd like.

Likely human-written
Our analysis found no significant indicators of AI generation in this content. 
Confidence in human authorship: [XX]%.
## Rate limiting: the limits you chose and your reasoning for those specific values
I chose to use 10 per minute and 50 per day, since once every 6 seconds is far too many, people cannot write that fast. Similarly, 50 per day is the maximum, since people could submit 10 or 20 drafts in a day, but over 50 means that they are submitting one every 30 minutes without sleeping.
$ for i in $(seq 1 12); do   curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/submit     -H "Content-Type: application/json"     -d '{"text": "This is a test submission for rate limit testing purposes only.", "creator_id": "ratelimit-test"}'; done
200
200
200
200
200
200
200
200
200
200
429
429

## Known limitations: at least one specific type of content your system would likely misclassify and why
Short AI-created human edited writing. The LLM looks for personal voice and word choice, meaning that it will think texts containing such elements are human-created. The intended means of stopping this, the heuristics, don't work at short text amounts. Additionally, proper grammar cannot be properly input due to the limitations of bash, making errors that seem human, while also preventing the use of the em-dash, a common sign of artificial generation.
## Spec reflection: one way the spec helped you, one way implementation diverged from it and why
One way the spec helped me was when implementing the labels, since they were intended to be clear statements, and although not quite the same, the message remains.. One way the implementation diverged from the spec was with the signals, I had written down puncutation density, however I realized that that wasn't the best way since its more so about the specific punctuations used rather than an abundance or lack of them, more so that humans will basically never use em-dashes while AI loves them, alongside proper hyphenation and parentheses use, while humans tend to overuse commas instead, see here. I changed it to focus on weird punctuation instead of punctuation amount. 
## AI usage section: at least 2 specific instances describing what you directed the AI to do and what you revised or overrode
I provided the architecture diagram and the signal 1 spec to ask Claude to implement the Flask app skeleton and Signal 1. I changed the Flask app skeleton when I started looking through the Flask guide. 

I provided the Signal 2 spec and the architecture diagram and asked Claude to implement Signal 2 and the score combination. The main change I made was altering Signal 2 to focus on weird punctuation instead of punctuation amounts, alongside changing the placeholder label to not use the old Flask app structure. 