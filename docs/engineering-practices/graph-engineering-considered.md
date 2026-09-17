# Engineering Practice: Graph Engineering (Considered, Not Adopted)

**Status:** Evaluated and set aside. Recorded here because a rejected option is worth as much as a chosen one.

## What it is

Graph engineering models data as nodes and edges rather than rows or documents, and queries it by traversal: relationships are first-class, multi-hop questions are cheap, and a schema can grow without a migration. In an AI platform it usually shows up as a knowledge graph behind retrieval (GraphRAG), where an answer is assembled by walking entities and their relationships instead of ranking text chunks.

## Where we looked at it

| Candidate | Why it looked promising | Why it did not fit |
|---|---|---|
| **Husbandry corpus as a knowledge graph** | Species, diets, symptoms, protocols and care sheets are genuinely a connected domain, and provenance is naturally an edge | [009](../adrs/animal_monitoring/009-adr-retrieval-index-design.md) already chose structure-aware chunking with hybrid keyword and dense search over a small self-hosted store. The corpus is closed and curated, so the retrieval problem is precision over a known set, not discovery across an unknown one. A graph adds an ontology to build and maintain, and vet time is the scarcest resource we have |
| **Estate topology** | Zones, rides, enclosures, sensors and gateways form an obvious hierarchy | It is small, stable and answered perfectly well by the device registry. Traversal depth never exceeds what a join handles |
| **Crowd flow between zones** | The strongest of the three. Zones as nodes and visitor transitions as edges is a real graph, and it is what a digital twin would reason over | It lives inside [footfall/002](../adrs/footfall_and_popularity/002-adr-phased-rollout-crowd-and-popularity-analytics.md) Step 2, which is **Proposed** and gated on visitor volume. Step 2 describes it as a learned transition matrix plus a deterministic queueing calculation, which needs no graph database. This is a modelling technique inside one phase, not an estate-wide practice |

## Why we set it aside

Three reasons, in order of weight:

1. **It solves a problem we do not have yet.** Our retrieval is precision over a closed, curated corpus. Graph retrieval earns its complexity on open, sprawling, multi-hop corpora.
2. **The operating cost lands on the wrong people.** An ontology needs curation, and the curators would be the vet and the keepers, whose time is already the largest recurring line in [ADR-007](../adrs/platform/ADR-007-cost-model-and-investment-strategy.md).
3. **It would contradict decisions we already made well.** [008](../adrs/animal_monitoring/008-adr-rag-over-a-curated-corpus.md) rejected unbounded autonomous retrieval because it makes the citation check meaningless. Multi-hop traversal reopens exactly that question, and citation integrity is what makes retrieval safe to put in front of a keeper.

## What would change our mind

Any one of these would justify reopening it as a proper ADR:

- The husbandry corpus stops being closed and curated, for instance by ingesting external veterinary literature at volume
- Keepers routinely need multi-hop answers ("which species in this zone share a symptom with a diet changed in the last month") that chunk retrieval measurably fails
- Footfall Step 2 ships, proves out, and the transition model outgrows a matrix into something with entity semantics
- A second estate is onboarded and topology stops being small and stable

## What we are doing instead

Retrieval quality is pursued through [009](../adrs/animal_monitoring/009-adr-retrieval-index-design.md)'s hybrid search and metadata filtering, and corpus health through curation and provenance tracking under [MLOps](mlops.md). If those stop delivering, this page is where the alternative is already written down.

<p align="right"><a href="../../README.md">↑ Back to README</a></p>
