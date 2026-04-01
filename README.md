# QueryCore

QueryCore is a compact, educational document search engine implemented in Python. It builds an inverted index from plain text files in the `data/` folder, computes TF–IDF scores, and supports ranked queries via a small command-line interface.

This README documents how the project is organized, how the main algorithms work, how persistent storage works (pickle), and how to run and extend the code.

## Project structure

- `main.py` — CLI entrypoint. Loads an existing index or builds it from files in `data/`, then runs an interactive query loop.
- `data/` — plain text documents (one `.txt` file per document) used as the corpus.
- `src/parser.py` — parsing, tokenization, stopword removal and utilities for reading documents.
- `src/indexer.py` — builds the inverted index and records per-document token counts/lengths.
- `src/ranker.py` — computes IDF values used by TF–IDF scoring.
- `src/search.py` — query processing, AND/OR boolean logic, and TF–IDF scoring to rank documents.
- `src/snippet.py` — extracts short text snippets around the first matching query token.
- `src/storage.py` — persistent read/write of the built index using Python `pickle`.
- `utils/stopwords.py` — small stopword set used by the tokenizer.

## Key concepts and implementation details

Below are the important internals and the exact data shapes used across modules.

### Document representation

When parsed, each document is represented as a dictionary:

```
documents = {
	"filename.txt": {
		"tokens": ["word1", "word2", ...],
		"text": "original file text as a single string"
	},
	...
}
```

The parser in `src/parser.py` performs the following steps:
- lowercasing
- removal of non-alphanumeric characters
- whitespace tokenization (split on spaces)
- removal of stopwords from `utils/stopwords.py`

### Inverted index and doc lengths

The inverted index is a mapping from token -> postings (document -> raw term frequency):

```
index = {
	"token": {"doc1.txt": freq_in_doc1, "doc2.txt": freq_in_doc2, ...},
	...
}

doc_lengths = {"doc1.txt": total_token_count_in_doc1, ...}
```

The index and `doc_lengths` are created by `src/indexer.py` by counting tokens per document.

### TF–IDF scoring (ranking)

TF–IDF scoring used to rank documents follows this implementation:

- Term frequency (TF) for token t in document d: tf = freq(t, d) / doc_length(d)
- Inverse document frequency (IDF) for token t: idf(t) = log(N / df(t))
	- N is the total number of documents
	- df(t) is the number of documents that contain token t
- Final score for a document is the sum over query tokens of tf * idf for that document.

IDF is computed in `src/ranker.py` using Python's `math.log` and the full index.

### Persistent storage (pickle)

To avoid rebuilding the index every run, the index, `doc_lengths`, and `idf` are saved to disk using Python's `pickle` module.

- Storage path: `index/index.pkl`
- Stored object (a single pickle dict):

```
{
	"index": index,
	"doc_lengths": doc_lengths,
	"idf": idf
}
```

- `src/storage.py` exposes `save_index(index, doc_lengths, idf)` and `load_index()` which return `None` when no saved index exists.

To force a rebuild of the index, remove the `index/` directory (or `index/index.pkl`) and restart `main.py`.

### Query processing and AND/OR logic

`src/search.py` implements a simple query flow:

1. Query text is normalized and tokenized using the same `clean_text`/`tokenize`/stopword pipeline as documents.
2. For each query token, the set of documents containing that token is collected.
3. If the user chooses `AND` mode, the intersection of all per-token doc sets is used. If `OR`, the union is used.
4. For each document in the selected set, a TF–IDF score is computed as described above.
5. Results are returned as a mapping document -> score and are displayed in descending score order.

Edge cases:
- Empty queries or queries that tokenize to an empty list return no results.
- If a query token is not in the index it contributes nothing to the score; in `AND` mode it may reduce the candidate set to empty.

### Snippet generation

`src/snippet.py` extracts a short snippet by finding the first word in the raw document text that matches a lowercased query token and returning a window of words around it. If nothing matches, it returns the first ~120 characters of the text.

## How to run

Requirements

- Python 3.8+ (tested with CPython). No external libraries required.

Commands

```bash
python main.py
```

Basic workflow

1. On first run, the program will build the index from files in `data/` and persist it to `index/index.pkl`.
2. On subsequent runs it will load the persisted index for faster startup.
3. At the prompt, enter a query string and choose search mode `AND` or `OR`.
4. Type `exit` to quit.

Example

- Query: "mars exploration"
- Mode: OR
- Output: A ranked list of filenames and snippets showing where the query tokens appear along with computed TF–IDF scores.

## Data and typing (contract)

Function contracts (short):

- `parse_all_documents(folder_path) -> dict` — returns the `documents` mapping described above.
- `build_inverted_index(documents) -> (index, doc_lengths)` — builds the inverted index and doc length map.
- `compute_idf(index, total_docs) -> dict` — returns per-token IDF values.
- `save_index(index, doc_lengths, idf)` / `load_index()` — persist/load index state. `load_index()` returns `None` if no saved index is found.
- `search(index, doc_lengths, idf, query, mode) -> dict` — returns a mapping doc -> score.

Error modes

- Missing `data/` files: parser will skip non-`.txt` files but will raise if files cannot be read (permissions). This is a deliberate minimal implementation.
- Missing `index/index.pkl`: `load_index()` returns `None` and `main.py` triggers an index rebuild.

## Extending and improvement ideas

- Add stemming/lemmatization (NLTK, spaCy) to improve token matching.
- Replace simple tokenization with regex tokenizers that preserve contractions if desired.
- Store the index in a more robust format (SQLite, LMDB) for faster random access and concurrent reads.
- Add phrase search and positional indexes so phrase queries return more precise snippets.
- Add unit tests for parser/indexer/search/ranker and a small CI workflow.
- Add a web UI or API wrapper for remote queries.

## Troubleshooting

- If results are empty and you expect matches, check that your query tokens are not all stopwords (see `utils/stopwords.py`).
- To force rebuilding the index, remove the `index/` directory and run `python main.py` again.

If you want, I can also add a short CONTRIBUTING section, a test harness, or convert the storage to JSON/SQLite — tell me which one to implement next.
