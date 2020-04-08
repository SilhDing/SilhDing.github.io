---
title: "Project: A Text-based Multi-model Search Engine"
date: 2018-12-05 23:56:39
tags:
    - "language technology"
    - "Java"
categories:
    - "technique"
    - "course"
    - "project"
---

Below is a brief summary of this project:

- Implemented a text-based large-scale search engine with Lucene API on corpus of 500,000+ documents.
- Supported multiple retrieval models (unranked/ranked Boolean, Okapi BM25, Indri) and multiple operators.
- Realized pseudo-relevance feedback that smartly expands original queries to improve retrieval performance.
- Exploited machine learning ideas to implement Learning-to-Rank by calling SVMrank to train retrieval model.
- Enhanced diversity of search results by implicitly/explicitly considering multiple intents in the retrieval results.
