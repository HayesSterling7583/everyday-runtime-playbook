# Plain Keyword Search or RAG for a Game Product Catalog

Short answer: ship keyword retrieval first for a game catalog, then add a replaceable reranker only if a labeled query set shows that lexical matching misses important intent. Do not put generation in the request path just to reorder products. Retrieval-augmented generation (RAG) combines retrieved non-parametric memory with a parametric generator; that is useful when the output must synthesize an answer, but catalog search usually needs stable product IDs, filters, and a ranked list. For an indie team, the least complex useful system is keyword candidate retrieval followed by optional reranking of a small candidate set under a hard latency budget.

The decision is narrow. A player searching for `co-op space survival` should see available games, not prose about them. Keep exact filters such as platform, age rating, availability, and supported player count outside any model score. Retrieve perhaps 40 eligible candidates with text matching, rerank those candidates when time permits, and return the keyword order when it does not. That preserves a complete, testable path while leaving room to improve relevance.

Ship that first.

## Build the fallback before the reranker

The data flow is small enough to reason about: normalize the query, apply hard catalog filters, score searchable fields, take a bounded candidate set, and optionally ask a reranker for a new order. The response contains catalog records already stored by the application. There is no generated description and no model-created product fact.

This TypeScript example uses a transparent token-overlap score so the boundary is visible. It is intentionally not a production text-search engine. The useful part is the contract: the candidate stage owns eligibility, the reranker may change order but may not invent IDs, and a deadline restores the original ranking.

```ts
type Game = {
  id: string;
  title: string;
  description: string;
  tags: string[];
  platforms: string[];
  available: boolean;
};

type RankedGame = Game & { score: number };

interface Reranker {
  rank(query: string, candidates: RankedGame[]): Promise<string[]>;
}

const games: Game[] = [
  {
    id: "orbital-camp",
    title: "Orbital Camp",
    description: "Cooperative survival and base building in deep space",
    tags: ["co-op", "survival", "crafting"],
    platforms: ["pc"],
    available: true,
  },
  {
    id: "red-drift",
    title: "Red Drift",
    description: "Solo racing across a desert planet",
    tags: ["racing", "single-player"],
    platforms: ["pc", "console"],
    available: true,
  },
  {
    id: "station-seven",
    title: "Station Seven",
    description: "Four-player exploration with resource management",
    tags: ["co-op", "space", "strategy"],
    platforms: ["console"],
    available: false,
  },
];

function tokens(value: string): string[] {
  return value.toLowerCase().match(/[a-z0-9]+/g) ?? [];
}

function keywordCandidates(
  query: string,
  platform: string,
  limit = 40,
): RankedGame[] {
  const queryTerms = new Set(tokens(query));

  return games
    .filter((game) => game.available && game.platforms.includes(platform))
    .map((game) => {
      const titleTerms = tokens(game.title);
      const bodyTerms = tokens(`${game.description} ${game.tags.join(" ")}`);
      const titleHits = titleTerms.filter((term) => queryTerms.has(term)).length;
      const bodyHits = bodyTerms.filter((term) => queryTerms.has(term)).length;
      return { ...game, score: titleHits * 3 + bodyHits };
    })
    .filter((game) => game.score > 0)
    .sort((a, b) => b.score - a.score || a.id.localeCompare(b.id))
    .slice(0, limit);
}

async function withDeadline<T>(work: Promise<T>, milliseconds: number): Promise<T> {
  let timer: ReturnType<typeof setTimeout> | undefined;
  const timeout = new Promise<never>((_, reject) => {
    timer = setTimeout(() => reject(new Error("rerank deadline exceeded")), milliseconds);
  });

  try {
    return await Promise.race([work, timeout]);
  } finally {
    if (timer) clearTimeout(timer);
  }
}

async function searchGames(
  query: string,
  platform: string,
  reranker?: Reranker,
): Promise<RankedGame[]> {
  const candidates = keywordCandidates(query, platform);
  if (!reranker || candidates.length < 2) return candidates;

  try {
    const ids = await withDeadline(reranker.rank(query, candidates), 80);
    const allowed = new Map(candidates.map((game) => [game.id, game]));
    const reordered = ids.flatMap((id) => {
      const game = allowed.get(id);
      if (!game) return [];
      allowed.delete(id);
      return [game];
    });
    return [...reordered, ...allowed.values()];
  } catch {
    return candidates;
  }
}
```

The `80` millisecond deadline and `40` candidate limit are example policy values, not universal recommendations or benchmark results. Set both from the service's end-to-end budget and measured traffic. Their real purpose is to make two potentially unbounded inputs explicit: how much work a reranker receives and how long search waits for it.

There is also a less obvious correctness rule in the example. The reranker returns IDs, and the application intersects them with the candidate map. Unknown IDs disappear; omitted candidates remain in their original order. A model or remote service therefore cannot smuggle an unavailable item past filtering. Keep that property when replacing the toy scorer with an inverted index or the reranker with a different implementation.

## Should a Game Product Catalog Use Plain Keyword Search or RAG?

Start with misses, not architecture fashion. Keyword retrieval will struggle when the catalog and players use different language, when intent depends on several attributes expressed together, or when descriptions are sparse. A reranker can be tested against those cases. RAG is a larger move: the original RAG paper defines a system whose generation is conditioned on retrieved passages, with a parametric model and a non-parametric memory. If the interface only returns catalog items, generation adds a component without supplying the required output shape.

The dividing line is the job. Use ranking for `find games like this`; consider retrieval plus generation for `explain which of these games fits a four-person group and cite the catalog evidence`. Even then, retrieve and filter the eligible items first. Generated text should describe the selected records, not decide whether a discontinued or incompatible record is eligible.

This is the trade-off.

Plain keyword search has a real limitation: it depends on lexical overlap, so a relevant record can be missed when the player and catalog use different terms. The optional reranker has the opposite operational weakness. It may recover intent from richer context, but it consumes part of the request budget and creates another timeout boundary. RAG carries both retrieval and generation responsibilities; it is unsuitable when the product surface needs only IDs and ranks, yet it becomes a reasonable candidate when the surface must compose a sourced answer. None is universally better. The output contract decides which complexity has a job.

**Retrieval quality must buy its latency.** Compare the optional reranker with the keyword baseline on the same queries and the same frozen catalog snapshot. A richer method has earned deployment only when its gains occur on queries that matter and the additional latency fits the product budget. Otherwise the simpler path is already doing the job.

## Measure the ranking players actually receive

Build a small judgment set from the game vocabulary before changing the stack. Each row needs a query, any hard filters, and graded relevance for catalog IDs. Include exact-title queries, broad genre queries, attribute combinations such as `local co-op puzzle`, and zero-result cases. Split near-duplicates deliberately: `space co-op` and `couch co-op` should not collapse into one test merely because both contain `co-op`.

Then replay every candidate implementation against that fixed set. Record a ranking metric at the visible result depth, no-result rate, end-to-end latency percentiles, timeout count, and the share of requests that used fallback. The metric name matters less than preserving the judgments and comparing like with like. Do not evaluate a reranker on a candidate set that already excluded the relevant item; log candidate recall separately so a retrieval miss is not blamed on ordering.

One short table catches many expensive category errors:

| Failure | What the trace should reveal | Response |
| --- | --- | --- |
| Relevant game never became a candidate | Expected ID absent before reranking | Improve fields, aliases, or candidate depth |
| Eligible candidates were reordered poorly | Expected ID present, final rank worse | Retrain, retune, or disable reranking for that query class |
| Filtered game appeared | Final ID absent from eligible candidate IDs | Reject the reranker output and fix the boundary |
| Reranker missed its deadline | Fallback used with candidate order intact | Tighten the limit or remove the reranker from the path |
| Catalog edit is not searchable | Indexed version trails the catalog version | Repair ingestion before changing ranking |

Use slices as well as an aggregate score. Multiplayer mode, platform, mature-content filtering, new releases, and long-tail tags can fail in different ways. An overall improvement can hide a damaging regression in one of them.

One number will not settle it.

## Operate one search path, with an optional stage

Deployment should make the reranker disposable. Log a request identifier, normalized-query fingerprint, catalog snapshot version, candidate IDs, final IDs, elapsed time for each stage, and whether fallback ran. Avoid storing raw queries by default if they may contain personal data; define retention from the application's actual privacy needs. Search diagnostics do not justify collecting data without limits.

Roll out with shadow evaluation first: compute the alternate order without showing it, then compare it with the current result and watch resource use. A later exposure test can answer whether offline relevance gains change player behavior, but it should not replace relevance judgments. Clicks reflect position, presentation, availability, and prior popularity as well as intent.

Catalog updates deserve equal attention. Index immutable IDs, publish a catalog version with each index build, and make the serving process expose which version answered a query. If a price, platform, or availability field changes, eligibility must follow the authoritative catalog state even when descriptive text is reindexed later. Ranking stale inventory beautifully is still wrong.

Freshness wins first.

Before release, walk the path in prose: an eligible record enters the candidate set; every returned ID is validated; the deadline falls back to the stored candidate order; traces distinguish retrieval from reranking; evaluation uses a fixed snapshot; and the feature can be disabled without taking search down. That is the operational checklist. If any sentence is false, the extra stage is not ready.

For most small game catalogs, stop here. Plain retrieval remains the dependable product, reranking is a measured enhancement, and generation belongs only in an interface that actually needs generated language. This keeps the quality-versus-latency decision reversible instead of turning it into a platform commitment.

## References

- Patrick Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks": https://arxiv.org/abs/2005.11401

## Further reading

- The original RAG paper and its retrieval-plus-generation formulation: https://arxiv.org/abs/2005.11401
