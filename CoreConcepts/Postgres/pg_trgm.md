# PostgreSQL pg_trgm Extension Guide

## Table of Contents
1. [Introduction](#introduction)
2. [What is a Trigram?](#what-is-a-trigram)
3. [How pg_trgm Works](#how-pg_trgm-works)
4. [Installation and Setup](#installation-and-setup)
5. [Key Functions and Operators](#key-functions-and-operators)
    - [Similarity Operator and Function](#similarity-operator-and-function)
    - [Word Similarity](#word-similarity)
    - [Distance Operator](#distance-operator)
6. [Indexing for Performance (GIN and GiST)](#indexing-for-performance-gin-and-gist)
7. [Real-World Use Cases](#real-world-use-cases)
8. [Summary](#summary)

---

## Introduction

The `pg_trgm` extension is a powerful module built into PostgreSQL that provides functions and operators for determining the similarity of text based on trigram matching, along with support for fast fuzzy searching. 

Traditional database text search (like `LIKE '%query%'`) requires sequential scans that scale poorly as your tables grow. `pg_trgm` bridges this gap by enabling high-performance, indexed wild-card and fuzzy searches.

---

## What is a Trigram?

A trigram is a group of three consecutive characters taken from a string. To understand how PostgreSQL analyzes strings with `pg_trgm`, spaces are automatically added to the edges of the word to pad them out.

For example, the word `"cat"` is considered with space padding as `"  c"`, `" cat"`, `"cat "`, and `"at "`. 
Specifically, the trigrams generated for `"cat"` are:
- `  c`
- ` ca`
- `cat`
- `at `

PostgreSQL breaks down any input string into these overlapping three-character chunks to compare overlap ratios between two different strings.

---

## How pg_trgm Works

When you compare two strings using `pg_trgm`, the extension calculates a **similarity score** ranging from `0.0` (completely different) to `1.0` (identical strings). 

The formula for trigram similarity is:
$$\text{Similarity} = \frac{\text{Number of shared trigrams}}{\text{Total unique trigrams in both strings}}$$

By default, PostgreSQL considers two strings similar if their similarity score exceeds `0.3`, though this threshold is fully customizable.

---

## Installation and Setup

Because `pg_trgm` is a standard contributed module, it comes pre-packaged with PostgreSQL. You only need to enable it inside your specific database by running:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

To verify that it installed correctly, you can run:

```sql
SELECT * FROM pg_available_extensions WHERE name = 'pg_trgm';
```

---

## Key Functions and Operators

`pg_trgm` provides several operators and functions to help you query text flexibly.

### Similarity Operator and Function

- **Operator (`%`)**: Returns `true` if the similarity between two strings exceeds the current similarity threshold.
- **Function (`similarity(text, text)`)**: Returns a floating-point number representing the exact similarity score between two strings.

```sql
-- Returns the similarity score between 'postgresql' and 'postgre'
SELECT similarity('postgresql', 'postgre');

-- Uses the % operator to find matching names based on default threshold (0.3)
SELECT * FROM users WHERE username % 'jonathan';
```

### Word Similarity

Sometimes you want to check if a string contains a smaller word or phrase with high similarity. `pg_trgm` offers word similarity functions:

- **Operator (`<%`)**: Returns true if an ordered subset of the right string has high similarity to the left string.
- **Function (`word_similarity(text, text)`)**: Calculates the degree of similarity of the most similar word in the target string.

### Distance Operator

- **Operator (`<->`)**: Returns the trigram distance between two strings, calculated as `1 - similarity(text, text)`. This is especially useful for nearest-neighbor sorting and KNN (K-Nearest Neighbors) queries.

```sql
-- Sorts results by how close they match the search term
SELECT name, name <-> 'alexander' AS distance
FROM contacts
ORDER BY distance
LIMIT 5;
```

---

## Indexing for Performance (GIN and GiST)

The true power of `pg_trgm` shines when combined with indexes. Without an index, fuzzy text matching still requires scanning every row. With a **GIN** (Generalized Inverted Index) or **GiST** (Generalized Search Tree) index, queries using `%`, `ILIKE`, or `<->` can execute in milliseconds.

### Creating a GIN Index (Recommended for search speed)
GIN indexes are typically faster for lookup queries using `LIKE`, `ILIKE`, and `%`:

```sql
CREATE INDEX idx_users_username_gin ON users USING gin (username gin_trgm_ops);
```

### Creating a GiST Index (Recommended for sorting/KNN)
GiST indexes are slower for general lookups than GIN, but they support ordering via distance operators (`<->`) much more natively:

```sql
CREATE INDEX idx_users_username_gist ON users USING gist (username gist_trgm_ops);
```

---

## Real-World Use Cases

1. **Typo-tolerant Search**: Helping users find products or accounts even when they misspell search terms (e.g., searching for "iPhon" finds "iPhone").
2. **Autocomplete and Suggestions**: Ranking nearest matching terms as a user types into an input box.
3. **Deduplication**: Finding duplicate customer records or entries with slight variations in spelling or formatting (e.g., "Acme Corp" vs "Acme Corporation").

---

## Summary

The `pg_trgm` extension is an essential tool in any PostgreSQL developer's toolkit. By understanding trigrams, leveraging similarity operators, and applying GIN or GiST indexes, you can build fast, robust, and forgiving search experiences directly inside your database without needing external search engines like Elasticsearch for simpler use cases.