---
title: "Making a Legacy SaaS Platform AI-Ready"
description: "How we added semantic search to FullCircle to make compliance data conversational, agent-traversable, and truly AI-ready."
publishDate: "17 Apr 2026"
tags: ["ai", "semantic-search", "embeddings", "saas", "postgres"]
draft: false
pinned: true
---

When fullCircle was built in 2019, data had one job: respond to the right REST endpoint. That worked for years.

When AI arrived, we assumed our existing data interface would translate. We built an MCP server mirroring our REST endpoints — if they worked for our app, surely they'd be enough to get AI going. We found their limitations fast.

REST endpoints are designed to answer specific questions, but AI needs to ask questions we haven't thought of yet. After launching our Chat Beta, we monitored the logs eagerly. Almost immediately we saw _"Does the uploaded evidence satisfy this request?"_ — and our MCP server fell flat on its face.

What our users actually needed was compliance data that was conversational, traversable by agents, and searchable without knowing the right keywords. Semantic search is how we gave them that.

## Why Embeddings Won

Keyword search was never going to work when a user asks, _"Which of my risks are related to this control?"_ Instead of doing complex and expensive query manipulation to run different permutations of keyword search, we could leverage semantic search to generate a semantic understanding of the control and match it to the semantic representations of that user's risks. It becomes a simple geometric calculation with deterministic similarity scores — which our data-focused users love.

The approach is well-established: take your query, risk statement, or control description, pass it through an embedding model, and get back a vector representing the semantic meaning of that text. To find matches, you find which other vectors are geometrically closest.

### Cost: API vs. Self-Hosted

Embedding does have a cost, but it's surprisingly cheap. One of our earliest decisions was whether to self-host the embedding service or go API. After running the numbers:

- **Self-hosted** (cheapest GPU instance on AWS on-demand): ~$4,600/year
- **API** (embed entire backlog): ~$48, with ongoing costs <$10/year at current usage

It was very difficult to justify paying for an EC2 instance when the API calls were that cheap. These embedding models are essentially slimmed-down LLMs with billions of parameters — the infrastructure cost to run them yourself is real.

### Privacy Considerations

Using an API meant our data would be processed by a third party. As part of our broader AI buildout, we implemented consent controls to restrict which clients' data could be shared with LLMs. We hooked into those same permissions to skip embedding data for clients who hadn't opted in, respecting any privacy agreements without building a separate system.

### Storage

The choice was obvious: use the `pgvector` extension to extend our existing Postgres instance. Embedding tables live alongside their source entities, keeping direct relationships intact. No new infrastructure, no data synchronization problem.

## The Implementation

We were already using OpenAI for our base LLM, so we went with one of their embedding models. Given the low cost, we chose the largest model available at the time: `text-embedding-3-large`.

### Dimensions and Indexing

We planned to use cosine similarity search indexed with HNSW (Hierarchical Navigable Small World) for speed. HNSW only supports up to 2,000 dimensions, but `text-embedding-3-large` natively outputs 3,072. Fortunately, OpenAI provides an API parameter to request a smaller vector — and the performance tradeoff is negligible.

As OpenAI notes: _"a text-embedding-3-large embedding can be shortened to a size of 256 while still outperforming an unshortened text-embedding-ada-002 embedding with a size of 1536."_ We settled on 2,000 dimensions — the HNSW maximum — and got the best of both worlds.

### Database Design

We created two tables:

- **`embeddings`** — references the original entity as a foreign key
- **`embeddings_chunks`** — stores the actual vectors, with a foreign key back to `embeddings`

This one-to-many relationship supports chunking large entities for optimal retrieval and keeps everything easily traversable.

### Triggering Embeddings

We added a `fieldsToEmbed` field to our model interface — an array of strings mapping to column names on the entity table. On save, our Postgres client wrapper kicks off a non-blocking workflow to queue the embedding.

The workflow hashes the values of the specified columns. If an embedding request comes in and the stored hash matches, re-embedding is skipped entirely. This let us specify not just _which_ entities to embed, but _which fields_ to actually match on.

### The Queue Problem

Our initial plan: queue an embedding job per request for workers to pick up. This fell apart quickly. The volume of embedding requests dwarfed our typical worker queue size, making queue length-based autoscaling metrics useless. Worse, we had no reliable way to stay under OpenAI's API rate limits when many workers were making requests concurrently.

We transitioned to a **database queue** — a record per embedding request, processed on a controlled cron schedule. Rate limiting became trivial: make X requests every Y minutes, and you'll never get throttled.

## What This Unlocked

With data that's semantically searchable, the use cases exploded.

**OmniSearch**: users (or an LLM) type a query and get semantic matches across all their data. _"Which of my risks are related to this control?"_ now works — grab the embedded control, search for closest-matching embedded risks, return them.

**Evidence evaluation**: _"Does the uploaded evidence satisfy this request?"_ added another layer — actually parsing uploaded files. But once parsed, the output text fits perfectly into this embedding pipeline. Between having the parsed text and its semantic representation, our agents can reliably answer this question.

We also generate summaries and hypothetical _compliance questions_ for each file at parse time (inspired by [HyDE](https://arxiv.org/abs/2212.10496)), which dramatically improves retrieval quality. More on that in the next post.

---

Designing and deploying the file parser came with its own challenges — including a major architectural change (the same push vs. pull issue we hit with the embeddings pipeline). Stay tuned for Part 2.
