# Healthy Eating Chatbot

A local retrieval-augmented chatbot that answers nutrition and recipe questions. An
embedding-based intent router sends each question down one of three retrieval paths — a
SQLite nutrition database, a ranked nutrient lookup, or live recipe scraping — and a small
local model (Llama 3.2 1B on Ollama) writes the answer from what was retrieved.

Graduate coursework project, San Jose State University, Nov–Dec 2025.

## How it works

```mermaid
flowchart LR
    Q[User question] --> R{Intent router<br/>all-MiniLM-L6-v2}
    R -->|"How much protein in chicken?"| N[Food matcher<br/>fuzzy + n-gram + embedding]
    N --> DB[(SQLite<br/>2,395 foods)]
    R -->|"What foods are low in fat?"| H[Nutrient + direction<br/>detector]
    H --> DB
    R -->|"How do I make meatballs?"| W[spaCy recipe-name<br/>extraction]
    W --> AR[allrecipes.com<br/>search + scrape]
    DB --> P[Prompt with<br/>retrieved context]
    AR --> P
    P --> L[Llama 3.2 1B<br/>via Ollama]
    L --> A[Answer]
```

**Intent routing.** Each intent has a handful of example questions. The router embeds the
user's question, takes the best cosine similarity against each intent's examples, and picks
the highest. No LLM call is spent on classification.

**Finding the food in the question** is the hard part of the nutrition path. Users misspell
("avacado"), use multi-word names ("chicken breast"), and name foods the table doesn't hold
verbatim. The matcher runs in stages:

1. Normalize and tokenize the question.
2. Spell-correct tokens against the food list with RapidFuzz (score ≥ 80).
3. Match the longest n-grams first, so "chicken breast" wins over "chicken".
4. Fall back to embedding similarity (cosine ≥ 0.70) when nothing matches exactly.

**Ranked lookups** ("foods high in vitamin C") detect the nutrient and the direction, then
return the top or bottom five foods for that column.

**Recipes** come from allrecipes.com: spaCy pulls the dish name out of the question, the
scraper runs a search, then parses the top result's ingredients, directions and nutrition.

## Prompt-strategy comparison

[`batch.py`](src/scripts/batch.py) runs 16 questions through the full pipeline — nutrition
lookups, ranked lookups, recipes, and one prompt-injection attempt. I ran it under four
prompt styles; each run's answers and retrieved context are in the `.xlsx` files at the repo
root.

| Prompt style | Results file |
|---|---|
| Baseline system prompt | [`intent_detection_results.xlsx`](intent_detection_results.xlsx) |
| One-shot answer templates | [`intent_detection_resultsOneShot.xlsx`](intent_detection_resultsOneShot.xlsx) |
| Role prompting | [`intent_detection_resultsRolePrompting.xlsx`](intent_detection_resultsRolePrompting.xlsx) |
| Chain of thought | [`intent_detection_results_ChainOfThought.xlsx`](intent_detection_results_ChainOfThought.xlsx) |

The answers were reviewed by hand, not scored, so these are observations rather than
measurements:

- **Chain of thought made a 1B model worse.** It produced long derivations with invented
  arithmetic instead of reading the number off the retrieved row.
- **Role prompting was the most fluent and the least grounded.** It drifted into general
  knowledge that was not in the retrieved context.
- **One-shot templates were sometimes applied to the wrong intent** — the banana-carbohydrates
  question came back as a smoothie recipe.
- **Retrieval passed through data the model could not sanity-check.** Ranked lists put
  three wines among the top five sources of vitamin A and listed cooked pork at 560.3 g of
  protein. The
  dataset's units needed validating before they reached the prompt.
- **No prompt resisted the injection attempt.** Asked to "ignore all previous instructions"
  and list its systems, every variant described systems it does not have; chain of thought
  invented a "Global Network Information Center". The model has no tools, so nothing real was
  exposed — but no prompt held the boundary either.

## What I would do next

- Score answers against a rubric (or an LLM judge) instead of reading them.
- Validate nutrient units at load time, before they can reach a prompt.
- Add an input guard for off-topic and injection attempts rather than relying on the prompt.
- Replace scraping with a recipe dataset or API; the scraper breaks whenever the site's
  markup changes.
- Try a 3–8B model; the 1B model was the main source of reasoning errors.

## Running it

Requires Python 3.10+ and [Ollama](https://ollama.com).

```bash
ollama pull llama3.2:1b
pip install -r requirements.txt
python -m spacy download en_core_web_sm

# run from the repository root, where Nutrition.db lives
python src/scripts/main.py      # answers one sample question
python src/scripts/batch.py     # runs the 16-question batch and writes an .xlsx
```

Run from the repository root: the scripts open `Nutrition.db` by relative path, and SQLite
silently creates an empty database if it is not found.

## Layout

```
Nutrition.db                  SQLite nutrition table (2,395 foods, 34 nutrient columns)
intent_detection_results*.xlsx  batch outputs, one per prompt style
src/scripts/
  main.py                     intent router, prompt, Ollama call
  nutritiondb.py              food matching and nutrition queries
  webscraping.py              recipe search and parsing
  batch.py                    16-question evaluation run
src/notebooks/                exploration notebooks for each component
```

## AI assistance

Claude wrote two pieces of this code, both marked in comments: the n-gram matching function
in `nutritiondb.py` and the output-writing part of `batch.py`.
