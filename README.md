# Amazon Fine Food Reviews Search Engine

A BM25 full-text search engine over the [Amazon Fine Food Reviews](https://www.kaggle.com/datasets/snap/amazon-fine-food-reviews)
dataset. Enter a free-text query and get back the most relevant reviews, ranked. The engine handles text
preprocessing (tokenization, stopword removal, lemmatization), indexes the corpus in Elasticsearch with
BM25 scoring, and includes a small information-retrieval evaluation harness (Precision@k, Recall@k,
NDCG@k) implemented from scratch.

Built as an information-retrieval project to explore how classic BM25 ranking behaves on a large,
real-world review corpus.

**At a glance** — measured in the committed notebook run:

| | |
|---|---|
| Raw corpus | ~568K reviews (242 MB Kaggle download) |
| After dedup + null-filtering | **393,576** reviews indexed |
| Preprocessing throughput | 393,576 docs in **6 m 20 s** (~1,035 docs/sec) |
| Avg field length | 35.78 tokens (`text`) · 4.09 (`summary`) |
| Ranking | Elasticsearch **7.9.2** BM25, `k1 = 1.2`, `b = 0.75` |
| Metrics | Precision@k · Recall@k · DCG@k · NDCG@k, hand-implemented |

Ranking is Elasticsearch's built-in Lucene BM25 — not a reimplementation. The work here is the
preprocessing pipeline, the index and field mapping, the query design (field-boosted `multi_match` as a
BM25F approximation), and the from-scratch metrics. See
[what the committed run actually measured](#what-the-committed-run-actually-measured--and-what-it-didnt)
for an honest read of the evaluation numbers.

## Tech Stack

- **Python** (Jupyter / Google Colab notebook)
- **Elasticsearch 7.9** — inverted index and BM25 ranking
- **NLTK** — tokenization (`word_tokenize`) and lemmatization (`WordNetLemmatizer`)
- **scikit-learn** — English stopword list (`CountVectorizer`)
- **pandas** — data loading and cleaning
- **kagglehub** — dataset download

## How It Works

The pipeline runs in five stages, each a section of the notebook:

1. **Data collection** — downloads the Amazon Fine Food Reviews CSV via `kagglehub` and loads it with
   pandas. Only four fields are kept: `ProductId`, `Score`, `Summary`, and review `Text`.
2. **Preprocessing** — drops rows with missing summaries and duplicate review text, then for each review:
   lowercases, strips punctuation (regex), tokenizes, removes English stopwords, and lemmatizes. After
   deduplication the corpus is ~393K reviews.
3. **Indexing** — creates a `food-reviews` Elasticsearch index configured with the BM25 similarity
   (`k1 = 1.2`, `b = 0.75`). Documents are bulk-ingested with automatic refresh disabled during the load
   for throughput. The field mapping is:

   | Field       | Type      | Purpose                                    |
   |-------------|-----------|--------------------------------------------|
   | `productid` | keyword   | exact-match product identifier             |
   | `summary`   | text      | analyzed for full-text search              |
   | `text`      | text      | analyzed for full-text search              |
   | `rating`    | integer   | numeric filter                             |

4. **Query processing** — a query is matched against `summary` and `text` using a `multi_match`
   (`most_fields`) query with `fuzziness: AUTO` for typo tolerance. The `text` field is boosted (`text^3`)
   so hits in the review body outweigh hits in the short summary — an approximation of field-weighted
   (BM25F-style) ranking. A `rating >= 3` range filter restricts results to positively-rated reviews.
5. **Evaluation** — returns the top 5 hits with Elasticsearch's `explain` output, then supports manual
   relevance judgments and computes Precision@k, Recall@k, and NDCG@k.

## Design Notes

- **Why field boosting instead of true BM25F:** Elasticsearch's default similarity scores each field
  independently. Rather than implement full BM25F, the engine approximates it by boosting the `text`
  field over `summary` at query time — simple, and effective for a corpus where the body carries most
  of the signal.
- **Bulk-indexing throughput:** `refresh_interval` is set to `-1` during ingestion so Elasticsearch
  doesn't rebuild searchable segments on every write; a manual refresh is issued once the bulk load
  finishes.
- **Fuzziness for real queries:** review text and search queries are noisy, so `fuzziness: AUTO` lets
  near-miss spellings still match.

## Running It

The project is a self-contained notebook (`amazon_fine_food_reviews_search_engine.ipynb`) designed to
run top-to-bottom in **Google Colab**, which has the Linux environment the setup cells assume.

1. Open the notebook in [Google Colab](https://colab.research.google.com/).
2. Run the cells in order. The early cells install dependencies and download / launch a local
   Elasticsearch 7.9.2 server inside the runtime:
   ```bash
   pip install kagglehub elasticsearch==7.9.1 numpy==1.24.3
   wget -q https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-7.9.2-linux-x86_64.tar.gz
   ```
3. Dataset download uses `kagglehub`, which requires Kaggle credentials. Follow the
   [kagglehub authentication guide](https://github.com/Kaggle/kagglehub#authenticate) if prompted.
4. When the query cell runs, enter a search phrase at the prompt.

> **Note:** the notebook stands up a fresh Elasticsearch instance in the runtime and re-indexes the full
> corpus, so a clean run takes a few minutes (preprocessing ~393K reviews and bulk indexing dominate).

## Example

```
Enter your search query: best hot chocolate
```

Returns the top 5 matching reviews, each with its product ID, summary, review text, rating, and the
BM25 score / explanation. For this query the top hits are all high-rated hot-chocolate reviews, e.g.:

```
Summary: Best hot chocolate
Text:    tried hot chocolate best rich chocolaty apparently chocolate
Rating:  5

Summary: best hot chocolate!
Text:    hot chocolate rich chocolaty best definitely better commercial mix grocery
Rating:  5
```

## Evaluation

The notebook includes standard IR metrics implemented from scratch:

- `precision_at_k` — fraction of the top-k results that are relevant
- `recall_at_k` — fraction of all relevant documents found in the top-k
- `dcg_at_k` / `ndcg_at_k` — discounted cumulative gain, normalized against the ideal ranking

Relevance is judged manually: after a search, you mark each returned hit as relevant (`1`) or not (`0`),
and the metrics are computed over those judgments. This is a lightweight, human-in-the-loop evaluation
rather than an automated benchmark over a fixed relevance-judged query set.

### What the committed run actually measured — and what it didn't

The notebook's saved output shows `Precision@5 = Recall@5 = NDCG@5 = 1.00` for the query
`"best chocolate with rich flavor"`. **Those numbers are not a result, and shouldn't be read as one.**
Three reasons, all visible in the evaluation cell:

1. `retrieved_docs` and `relevant_docs` are hardcoded to the **same five IDs**, so all three metrics
   are 1.00 by construction regardless of what the engine returned.
2. The `manual_rels` list actually collected from the relevance prompts is never passed to the metric
   functions — it's built and then unused.
3. The IDs used are `productid`, which is a **product** identifier, not a document identifier. Two
   distinct reviews of the same product collide (`B000OP5G1E` appears twice in the top 5), so
   set-membership scoring is measuring the wrong key.

What the run *does* legitimately demonstrate is the retrieval path and the scoring transparency: the
`explain` output confirms Lucene applying textbook BM25 term-by-term — `idf = log(1 + (N - n + 0.5) /
(n + 0.5))` over `N = 393,576` documents, `tf` saturating with `k1 = 1.2` and length-normalizing with
`b = 0.75` against an average field length of `35.78` tokens (`text`) and `4.09` (`summary`).

The intuition that motivated the boost — that IDF dominates on low-frequency terms, so rare or highly
specific query vocabulary makes ranking brittle — is visible in those per-term explanations, but it is
**an observation from reading score breakdowns, not a measured finding.** Establishing it would take a
fixed query set with graded judgments, which is the first item below.

## Limitations & Possible Extensions

- **The evaluation harness needs to be finished before any quality claim is made.** Concretely: judge
  on `_id` rather than `productid`, pass the collected `manual_rels` into the metric calls instead of
  hardcoded lists, and score a fixed set of queries (broad, rare-term, and negation cases) at several
  values of *k* rather than one query at k=5.
- The interface is a notebook prompt — a CLI or small web frontend would make it usable as a standalone tool.
- Ranking is out-of-the-box BM25 with field boosting; learned re-ranking or semantic (embedding) retrieval
  would be natural next steps.
- `fuzziness: AUTO` is applied to every term, which also lets rare, deliberately-specific query words
  match near-neighbours — worth measuring against exact matching once the harness above exists.

## License

MIT — see [LICENSE](LICENSE).