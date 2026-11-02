"""Day 2: two context experiments.

Experiment 1: context is not memory.
    A separate API call knows nothing about earlier calls unless you re-send history.

Experiment 2: the buried fact (Lost in the Middle).
    Hide a fact at the start, middle or end of filler text of increasing length
    and check whether the model finds it.

Run:
    python code/day02_context_experiments.py

Note: experiment 2 sends several long inputs, so it uses more tokens than the other
scripts. Lower LENGTHS_IN_WORDS if you want it cheaper.
"""

import anthropic

MODEL = "claude-haiku-5-5"

client = anthropic.Anthropic()


def ask(messages, max_tokens=100):
    response = client.messages.create(
        model=MODEL,
        max_tokens=max_tokens,
        messages=messages,
    )
    return response.content[0].text.strip()


# ---------------------------------------------------------------------------
# Experiment 1: context is not memory
# ---------------------------------------------------------------------------
def experiment_no_memory():
    print("=== Experiment 1: context is not memory ===")

    first = ask([{"role": "user", "content": "My name is Atharv."}])
    print(f"Call 1 reply: {first}\n")

    without_history = ask([{"role": "user", "content": "What is my name?"}])
    print(f"Call 2 (new call, NO history): {without_history}\n")

    with_history = ask(
        [
            {"role": "user", "content": "My name is Atharv."},
            {"role": "assistant", "content": first},
            {"role": "user", "content": "What is my name?"},
        ]
    )
    print(f"Call 3 (history re-sent): {with_history}\n")


# ---------------------------------------------------------------------------
# Experiment 2: buried fact
# ---------------------------------------------------------------------------
SECRET = "4417"
FACT = f"The access code for the archive room is {SECRET}."
QUESTION = "What is the access code for the archive room? Answer with just the number."

# Neutral filler sentences, with distractor numbers so the answer isn't trivial.
FILLER_SENTENCES = [
    "The committee reviewed the quarterly report and noted steady progress.",
    "Rainfall in the northern region was slightly above the seasonal average.",
    "The locker number assigned to the visiting team was 8830.",
    "A new bus route was proposed to reduce travel time across the city.",
    "The library extended its opening hours during the examination period.",
    "The warehouse inventory showed 2951 units of the older model.",
    "Volunteers organised a cleanup drive along the riverbank on Sunday.",
    "The training session covered safety procedures and basic first aid.",
    "The room booking reference for the conference was 6072.",
    "Local farmers reported a good harvest despite the late monsoon.",
]

LENGTHS_IN_WORDS = [500, 5_000, 20_000]


def build_filler(n_words):
    words = []
    i = 0
    while len(words) < n_words:
        words.extend(FILLER_SENTENCES[i % len(FILLER_SENTENCES)].split())
        i += 1
    return words[:n_words]


def insert_fact(words, position):
    if position == "start":
        index = 0
    elif position == "middle":
        index = len(words) // 2
    else:
        index = len(words)
    return " ".join(words[:index] + [FACT] + words[index:])


def experiment_buried_fact():
    print("=== Experiment 2: buried fact ===")
    print(f"{'words':>7} {'position':>9}  result   answer")

    for n_words in LENGTHS_IN_WORDS:
        filler = build_filler(n_words)
        for position in ("start", "middle", "end"):
            text = insert_fact(filler, position)
            prompt = f"{text}\n\n{QUESTION}"
            answer = ask([{"role": "user", "content": prompt}], max_tokens=30)
            result = "PASS" if SECRET in answer else "FAIL"
            print(f"{n_words:>7} {position:>9}  {result:6}   {answer[:60]}")

    print(
        "\nIf everything passes, that is a real finding too: this model handled "
        "these lengths fine. Try longer texts, subtler facts, or more distractors."
    )


if __name__ == "__main__":
    experiment_no_memory()
    experiment_buried_fact()
