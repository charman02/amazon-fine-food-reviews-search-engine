# Amazon Fine Food Reviews Search Engine

## Introduction & Problem Statement
• Problem – Finding relevant product reviews is difficult due to the large number of reviews available online

• Solution – A search engine which allows the user to enter a query and get a ranked list of documents as an output (most relevant reviews first)

• Dataset – Amazon Fine Food Reviews dataset (500,000 reviews from Kaggle)
  - https://www.kaggle.com/datasets/snap/amazon-fine-food-reviews
  - Includes product, user information, ratings and a plain text review

## Search Engine Design & Architecture
• Data Collection (Kaggle – Amazon Fine Food Reviews)

• Preprocessing (Lowercase, Remove punctuation, Tokenization, Stop words removal, Lemmatization)

• Indexing (BM25-f)

• Query Processing (CLI)

• Retrieval & Ranking (BM25-f)

• User Interface (CLI)

## Preprocessing
• Kept Product ID, Score, Summary, and Text from dataset

• Dropped N/A values

• Removed text duplicates

• Lowercase, Remove punctuation, Tokenization, Stop words removal, Lemmatization

## Indexing Model – BM25-f
• Indexing Engine:
  - Implemented BM25 using Elasticsearch
  - Approximated BM25-f by indexing multiple fields (summary, text) and assigning weights via relevance tuning
• Configuration:
  - Custom parameters: k1 = 1.2, b = 0.75
  - 1 primary shard and 1 replica
• Structure:
  - Fields: productid, summary, text, rating
  - Created a structured mapping:
    - Product_id: Keyword for exact matching
    - Summary and text: For full text matching
    - Rating: For numeric ratings
  - Stored in an inverted index, mapping terms -> document IDs -> term frequency
• Optimization:
  - Disabled automatic refresh during indexing
  - Used bulk ingestion to process ~500,000 reviews efficiently

## Evaluation
• Performs exceptionally well (high Precision@k, Recall@k, and nDCG@k) when queries refer to broader concepts and use keywords or phrasing that directly match how relevant reviews are written

• Struggles with query specificity
