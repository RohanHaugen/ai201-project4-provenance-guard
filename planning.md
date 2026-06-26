

Read the required features list in full. Then write, in plain English, the path a single piece of text takes from submission to the label a user sees. Name every system component it touches and what each one does. This is your architecture narrative — you'll use it in your planning.md and README.
## Text Path:
Content Submission Endpoint -> Rate Limiting -> Multi-Signal Detection -> Confidence Scoring with Uncertainty -> Transparency Label 
The message is submitted through content submission, which accepts a text based piece. It then passes through rate limiting, which limits how fast information can be gathered in order to not overwhelm the system. It then passes throughmulti-signal detection, which will classify the text as AI or not through semantic and structural means. It will then pass through confidence scoring, which should give the chances that it is AI or not. After that it should produce the transparency label which will declare the text as either human, AI, or uncertain.

Decide on your two detection signals before you write any code. For each signal, write: what property of the text it measures, why that property differs between human and AI writing, and what it can't capture (every signal has blind spots). If you can't describe the blind spot, you don't understand the signal yet.
## Detection Signals:
LLM-based classification (Groq): Measures cadence of text, AI writing often has certain word choices or symbol preferences that should be able to be obtained with an LLM classifier. This cannot capture or will fail with human emotionally flat writing such as some scientific papers or text written by non-native English speakers, or sufficiently odd AI-generated text.

Stylometric heuristics: Measures sentence length variance, amount of unique words, and unique punctuation density. Human text tends to have high variation due to people having differently sized thoughts, AI tends to hover around the same length. The same follows for unique words and punctuation, humans preferring more unique words and fewer unique puncutation, as people know a lot of words which AI doesn't tend to use due to them not being used often, and people not using unique punctuation because they forget they exist or aren't sure if it should be used there. The blindspots are short texts, since the total score will just be lower, so its harder to get statistical data from that, poetry since it has a weird sentence structure, and deliberate attempts to seem more human or AI.


Think through the false positive problem: what happens when your system misclassifies a human writer's work? Trace that scenario through your system — how does the confidence score reflect the uncertainty, what does the label say, and how does the creator appeal? This scenario should inform your decisions in Milestone 2.
## False Positive Problem
The confidence score should reflect the uncertainty, with a bias for false negatives where an AI work is classified as human, since people get hurt when they are classified as AI. Therefore, the area of humans should be wider than that of AI. The confidence score should be primarily the llm score since AI word choice shows up more than the structure of the text due to variable sizes, but the structural analysis should be a backup for things like scientific texts. The label should just say either AI, human, or uncertain. The creator should be able to reappeal with the post that got incorrectly flagged and their reasoning. If they've already appealed it shouldn't create a new ticket, it should just say that its already under review. 



Sketch (on paper or in a text file) your API surface: what endpoints do you need? What does each one accept and return? You're not writing code yet — you're defining the contract that all other code will implement.
# API surface
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

Turn your architecture narrative into a diagram. Draw the two main flows: (1) submission flow — POST /submit → signal 1 → signal 2 → confidence scoring → transparency label → audit log → response; (2) appeal flow — POST /appeal → status update → audit log → response. Label each arrow with what passes between components (raw text, signal score, combined score, label text). You'll include this diagram in your planning.md and use it as context when prompting AI tools to generate code.

Appeal
  │
  ▼
Input Validation (content_id, creator_reasoning)
  │
  ▼
Database Lookup by content_id
  ├────────► error if not found
  ▼
Status Update: "classified" → "under_review"
  │
  ▼
Audit Log Writer
  │    Appends: appeal_timestamp, appeal_reasoning to record
  ▼
Response
  {content_id, status: "under_review", message}



## Detection Signals
LLM-based Classification: measures semantics of whether a work reads like AI, should capture things like perfect grammar without a specific voice or other things that read like AI. 
Output: An object containing the score from 0.0 to 1.0, with 1.0 being 100% thinking to be AI and 0.0 to be entirely human and the rationale for the decision. 
Stylometric Heuristics: measures structural properties of text. Should capture things like AI invariance in sentence length, AI preferred word choices, and AI use of non-standard punctuation. 
Output: a float from 0.0 to 1.0, 0 not ai and 1 ai.
These should be combined by multiplying each by a ratio and adding them together.
## Uncertainty Representatoin
A confidence score of 0.6 means that it has a higher chance of being AI than not, but should be uncertain. The llm signal should be multiplied by 0.6 since it performs better across multiple text lengths, while the heuristis signal should be multiplied by 0.4 since it fails at smaller text lengths, but is still important. They should then be added together. Likely AI should encompass from 1.0 down to 0.70, since we want to err on the side of human. Uncertain should span from 0.7 down to 0.6, since we should be more lenient for humans and at around 50% we can't really tell for sure, then human should span from 0.6 down to 0.0. To favor humans, 0.6 should be human and 0.7 should be uncertain. 
## Transparency Label Design
Likely AI-Generated: Our analysis of this text has indicated there is a high probability of this text being AI-generated. Confidence: {insert uncertainty here in*10}%. If this assessment is incorrect, you can appeal this classification.

Likely Human: Our analysis of this text has found no significant indicators of being AI-generated. Confidence in human authorship: {1-uncertainty all times 10}%.

Uncertain:Our analysis of this text has returned inconclusive results on whether this text was written with AI assistance. Confidence in AI-generatedness: {uncertainty*10}%. If this assessment is incorrect, you can appeal this classification.

## Appeals Workflow
The author can submit an appeal. They should provide the content id of the post in question and rationale for why their post isn't what it was classified as. When the appeal is recieved, the post gets flagged as under_review. The post and timestamp would get flagged as needing review. A human reviewer would see the post in question, see the rationale the author provided, and the timestamps relavent to the situation(maybe).
## Anticipated Edge Cases
Academic papers by non-native English speakers, the combination of very strict writing standards that tend to deemphasize personal voice combined with specific technical terms seems like it would confuse both signals.

Short poems, poems have specific sentence structure and their length means that the heuristics will get confused, and the specific linguistic traits of poems mean that word choice can be somewhat forced, thereby making it harder to tell if some word choice is AI-preferred, in addition to minimal use of puncutation which is another major way AI can be detected.

## Architecture
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
The submission should take the text submitted by the user, and the content id and timestamp should be assigned later. The appeal should take the rationale and the content id, create a ticket for an appeal with the original verdict and desired verdict(maybe swap to just human) to be reviewed by a human later.

## AI Tool Plan
M3: I will provide the groq signal section and the architecture diagram to describe the submission start. I will ask Claude to generate the Flask app skeleton and first signal function using the specifications of the documents provided. I will verify this by calling the function with AI-generated text and human generated text, and observe the output.
M4: I will provide the heuristics signal section, the architecture diagram, and the uncertainty section. I will ask Claude to generate the heuristics function according to the specs provided and the uncertainty combination. I will also ask it how different configurations of the heuristics function would vary the results. I will verify this by giving it obvious ai-generated examples and obvious human examples, then an edited ai-generated piece of text.
M5: I will provide the label section, the appeal section, and the architecture diagram. I will ask Claude to generate the label generation function that takes the scores and returns a label with the confidence percentages (i could probably do this myself), and I will ask it to produce the /appeal endpoint. I will test this by giving it ai-generated, human, and ai-generated but edited text. 