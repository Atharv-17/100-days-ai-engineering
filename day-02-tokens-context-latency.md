# Day 2: Tokens, Context and Latency

## Key ideas (rewrite in your own words)

### Tokens
- Models read and charge in **tokens**, not words. A token can be a whole word, part of a word, punctuation, or a space plus a word.
- Different tokenizers split the same text differently, so counts vary by model.
- English averages about 4-5 characters per token (roughly 0.75 words).
- Common words are usually one token. Rare or new words get split into pieces.
- A leading space belongs to the token: `" hello"` and `"hello"` are different tokens.
- Long numbers are split into chunks (often about 3 digits). This is one reason models slip on arithmetic; the bigger reason is that they predict likely tokens rather than calculating.
- Code splits into keywords, symbols, and merged whitespace.

### Why some text costs more
- Tokenizers are trained on a mix of languages that is usually English-heavy. Text the tokenizer saw little of gets broken into tiny pieces (characters or even bytes).
- Same meaning, different cost: Hinglish is about 2x English, Hindi (Devanagari) is about 4-5x. Code and long numbers are also expensive.
- Consequence: for a Hindi product, token cost, latency, and context usage are all worse. Budget for it, or test a model with a more efficient tokenizer.

### Input vs output tokens
- You pay for both. Output is usually priced higher.
- Cost per request = (input tokens x input price) + (output tokens x output price). Multiply by request volume for the real bill.
- Output is generated one token at a time, because each token depends on the previous ones.

### Latency
- Two numbers decide how fast a response feels:
  - **Time to first token (TTFT):** the wait before anything appears. Grows with input length and server load.
  - **Generation speed (tokens/s):** how fast the rest streams out.
- Rough formula: `total time ~ TTFT + (output tokens / tokens per second)`
- Long answers take longer than long questions, because output is generated token by token.
- **Streaming** doesn't reduce total time, but text starts appearing after the TTFT, so it feels faster.
- Ways to reduce latency: cap `max_tokens`, ask for concise answers, stream, use a smaller model for simple tasks, send only the context needed.

### What fills the context window
One shared token budget per request, drawn on by:
1. System instructions
2. Conversation history
3. Documents pasted in
4. Examples (few-shot)
5. Tool results
6. The model's own answer

Input and output compete for the same limit. Chat history is re-sent every turn, so message 20 costs far more than message 1.

### Context is not knowledge or memory

| | What it is | Lifetime | Who controls it |
|---|---|---|---|
| Context | Text in the window for this request | One request | You, on every call |
| Training knowledge | What the model learned in training | Fixed, with a cutoff date | Nobody, until a new model |
| Memory | Information saved across sessions | Persistent | You have to build it |

- The model doesn't remember the last request. Chat apps feel like they remember because the app re-sends the history.
- New or private information has to be put into context (documents, tool results). This is the reason RAG exists.
- Even "memory" ends up as context: store summaries or facts, retrieve the relevant ones, put them in the next request.

### Bigger windows aren't better windows
- A bigger window raises the limit, not the quality.
- **Lost in the Middle:** models use information at the start and end of a long input better than information in the middle (a U-shaped accuracy curve). Linked to how attention and position work and to patterns in training data; exact causes are still studied. Newer models have improved, so test on your own task.
- **Dilution:** irrelevant text competes with relevant text.
- **Cost and latency:** you pay for every input token on every request, and long inputs raise TTFT.
- Fixes: send less (only the relevant parts), put key information at the start or end, repeat the question after the document, split into steps, use clear structure, test on your own data.
- **Key judgment:** does this task need *more* context, or *better-chosen* context?

---

## Experiment 1: Tokenizer playground

Tokenizer used: _______________ (e.g. gpt-4 / gpt-3.5-turbo)

| Sample | Characters | Tokens | Tokens per char | x English (total tokens) |
|---|---|---|---|---|
| English | 260 | 49 | 0.19 | 1.0 |
| Hinglish | 299 | 97 | 0.32 | ~2.0 |
| Hindi (Devanagari) | | | ~0.9 | |
| Code | | | | |
| Numbers | | | | |

Observations:
- English words were mostly whole tokens with the leading space attached.
- Hinglish: the English words stayed whole, but the Roman-script Hindi words were chopped into small fragments.
- Hindi: almost every character was its own token.
- Numbers split into chunks; code split into keywords, symbols, and whitespace.

## Experiment 2: API token counts and cost

Model: _______________ | Input price: $____ per 1M | Output price: $____ per 1M | 10,000 requests, 100-token reply

| Sample | API tokens | Playground tokens | Cost (10,000 requests) |
|---|---|---|---|
| English | | | |
| Hindi | | | |
| Hinglish | | | |
| Code | | | |
| Numbers | | | |

- Hindi cost as a multiple of English: ______
- Did API counts differ from the playground? By how much? ______

## Experiment 3: Latency

| Prompt | Output tokens | TTFT | Total time | Tokens/s |
|---|---|---|---|---|
| Short ("Name one planet.") | | | | |
| Long (400-word essay) | | | | |

- Did TTFT stay similar while total time jumped? ______
- Did the formula hold? ______

## Experiment 4: Context

- **No-memory test:** a separate call without history ______ (did it know my name?). With history included ______.
- **Buried fact (start / middle / end):** ______
- Did I reproduce Lost in the Middle? If not, at what length did I try? ______

---

## Self-test answers (write from memory, then check)
1. Why does the same sentence cost more in Hindi? ______
2. 2,000 input and 500 output tokens: which half costs more? ______
3. Why does a long answer take longer than a long question? ______
4. Five things that share the context window: ______
5. Why does a chat app seem to remember? Where does memory come from? ______
6. A model misses a fact in a 100-page document. What do I try first? ______
7. Context vs training knowledge vs memory: ______

## Outcome answer
*How do you estimate what a request will cost and roughly how long it will take, and when does a task need better-chosen context rather than more context?*

> Cost: count input and output tokens, multiply by the per-token prices, then multiply by request volume. Remember that Hindi and Hinglish, code, and numbers use more tokens for the same meaning. Time: roughly TTFT plus output tokens divided by tokens per second, so output length is the main driver. If a model lacks a fact, add it (more context). If it has the fact but it's buried among a lot of text, or the input is large and costly, send only the relevant parts (better-chosen context), because bigger windows raise the limit but not necessarily the quality.

(Rewrite this in your own words.)

## Interview questions and how to answer them

### How to deliver any answer
1. **Lead with the one-line answer**, then explain. ("Short version: ... Here's why.")
2. **Use a concrete number or example** from your own experiments. "In my test, the same paragraph was 49 tokens in English and about 5x that in Hindi" beats a generic claim.
3. **Finish with the practical consequence** (cost, latency, design decision). Interviewers want to know you'd build differently because of it.
4. **Keep each answer to 30-45 seconds.** Stop, and let them ask follow-ups.
5. **If you're unsure, say what you know and what you'd test.** "It depends on the model, so I'd measure it on my data" is a strong answer, not a weak one.

### Q1. What is a token, and why does it matter?
**Say:** "A token is the chunk of text a model reads and charges in. It can be a word, part of a word, or punctuation. It matters because cost, context limits, and latency are all counted in tokens, not words. English is roughly 4-5 characters per token, but other languages and code can be much worse."

### Q2. Why does the same sentence cost more in Hindi than in English?
**Say:** "Tokenizers are trained on a mix of text that's usually English-heavy, so they've learned few Hindi chunks. They fall back to tiny pieces, sometimes single characters or bytes. In my test, Hindi came out at roughly 0.9 tokens per character against 0.19 for English, so around 4-5x the tokens for the same meaning. For a Hindi product I'd budget for that, or test a model with a more efficient tokenizer."

### Q3. How do you estimate the cost of an LLM feature?
**Say:** "Cost per request is input tokens times the input price plus output tokens times the output price. Output is usually priced higher, so I estimate both. Then I multiply by expected request volume. I'd also check the language mix, because Hindi or code inflates token counts, and the chat history, because it's re-sent every turn so cost grows with conversation length."

### Q4. What's the difference between input and output tokens?
**Say:** "You pay for both, and output is typically priced higher. Output is also generated one token at a time, so it drives latency much more than input does. To control cost and time I cap max tokens and ask for concise answers."

### Q5. Why is a long answer slower than a long question?
**Say:** "The model processes the whole input in one pass, which affects time to first token. But the output is generated one token at a time, because each token depends on the previous ones. So total time is roughly time to first token plus output tokens divided by tokens per second. Output length is the main lever."

### Q6. What is time to first token, and how do you make an app feel faster?
**Say:** "Time to first token is the wait before anything appears. Streaming doesn't reduce total time, but text starts showing after the first token, so it feels much faster. Beyond that I'd cap output length, use a smaller model for simple tasks, and send only the context that's needed."

### Q7. What is a context window, and what fills it?
**Say:** "It's the token budget for one request. System instructions, conversation history, documents, examples, tool results, and the model's own answer all share it, so input and output compete for the same limit. In a chat, the history is re-sent every turn, which is why long conversations get expensive."

### Q8. Does an LLM remember previous conversations?
**Say:** "No. The model itself is stateless. Chat apps feel like they remember because the app re-sends the history each time, and 'memory' features work by saving summaries or facts and retrieving the relevant ones into the next request. So memory is something you engineer, and it ends up as context."

### Q9. What's the difference between context, training knowledge, and memory?
**Say:** "Training knowledge is what the model learned, fixed with a cutoff date. Context is what's in the window for this request, temporary and controlled by me. Memory is information saved across sessions, which I have to build. If the model needs something newer or private, I put it into context through documents or tools, which is the idea behind RAG."

### Q10. What is the Lost in the Middle problem?
**Say:** "Models tend to use information at the start and end of a long input more reliably than information in the middle, giving a U-shaped accuracy curve. It's linked to how attention and position work and to training-data patterns, and the exact causes are still studied. Newer models have improved, so I'd test it on my own task. To reduce it, the strongest fix is sending less, only the relevant parts. I'd also put key information at the start or end and repeat the question after the document."

### Q11. If a model has a 1M token window, why not just paste everything in?
**Say:** "A bigger window raises the limit, not the quality. You pay for every input token on every request, latency goes up, irrelevant text dilutes focus, and the middle can be missed. So I ask whether the task needs more context or better-chosen context. If the model has the facts but they're buried, I retrieve and send only the relevant pieces. It's cheaper, faster, and usually more accurate."

### Q12. A model keeps missing a fact in a long document. What do you do?
**Say:** "First I check whether the fact is actually in the context. Then I'd shorten the input to only the relevant sections, move the key information to the start or end, put the question after the document, and maybe split it into two steps: extract the relevant passages, then answer from them. And I'd test the change on a set of my own examples rather than trusting one run."

### Q13. Why do models make mistakes on arithmetic or counting letters?
**Say:** "Two reasons. Tokenization splits numbers and words into chunks that don't line up with digits or letters, and more importantly the model predicts likely tokens rather than executing a calculation. For anything that must be exact, I'd have the model call code or a calculator."

### Q14. Do all models tokenize the same way?
**Say:** "No. Each model family has its own tokenizer, so the same text gives different counts and splits. That's why I use the provider's token-counting tool for real cost estimates, not a rule of thumb or another model's tokenizer."

### Likely follow-ups to prepare
- "How would you reduce the cost of a chatbot that's getting expensive?" Trim or summarize history, shorten the system prompt, cap output length, use a smaller model for simple turns, and retrieve only relevant context.
- "How would you support Hindi users cost-effectively?" Measure token counts on real Hindi text, compare models' tokenizers, and budget for roughly 2-5x the tokens.
- "How would you test whether long context hurts your task?" Place a known fact at the start, middle, and end of inputs of increasing length and measure accuracy across many runs.

### Practice routine
1. Cover the "Say" text and answer each question out loud in under 45 seconds.
2. Record yourself on your phone for two or three of them and listen back.
3. Replace my wording with your own, and add your real numbers from the experiments.

## Open questions
-
