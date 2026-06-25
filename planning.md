

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
Audit Log Writer (SQLite)
  │    Writes: content_id, creator_id, timestamp, llm_score,
  │            stylometric_score, confidence, verdict, status
  ▼


Turn your architecture narrative into a diagram. Draw the two main flows: (1) submission flow — POST /submit → signal 1 → signal 2 → confidence scoring → transparency label → audit log → response; (2) appeal flow — POST /appeal → status update → audit log → response. Label each arrow with what passes between components (raw text, signal score, combined score, label text). You'll include this diagram in your planning.md and use it as context when prompting AI tools to generate code.
## I think thats above



