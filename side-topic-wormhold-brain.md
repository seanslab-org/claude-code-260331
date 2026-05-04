
The wormhole intuition

A flat 2D universe forces you to traverse intermediate space — to connect Jack and "delivery on 9/9/2019" you walk a path through indexes, predicates, joins. The SQL

facts

table was exactly that: edges drawn on a flat plane, every connection requiring a traversable route. A wormhole is fundamentally different:

it's a topological shortcut that makes distance irrelevant.

Two points that are far apart in the embedding space (or in narrative time, or in your meeting calendar) become

adjacent

through a single direct link. You don't traverse — you arrive.

Why this maps to engrams

In the brain, when you think "Jack," a cascade fires across distributed cells — the "Jack" concept neuron, the "delivery" concept neuron, the temporal context, the emotional weight, all activate together

simultaneously

. There's no path-walking. The synaptic links are wormholes: a single connection collapses arbitrary semantic and temporal distance. This is why concept cells (Quian Quiroga's Jennifer Aniston neuron) work — one neuron's firing pulls in the entire associated assembly through wormhole-like direct connections, not through a 2D graph traversal.

What this means for our architecture

The mistake of the flat

facts

table was treating every connection as an

edge in a 2D graph

— discrete, traversable, requiring schema. Real memory connections aren't edges; they're

wormholes through high-dimensional space

. What replaces "edges as wormholes": -

The sentence-vector galaxy.

Every utterance is a point in 768-dimensional space. Semantically related things are close

regardless

of when they were said, who said them, or how the meeting calendar is structured. A query collapses to its nearest neighbors instantly — that's a wormhole. -

RAPTOR's hierarchical clusters.

Constellations are wormholes-of-wormholes — high-altitude summary nodes connect to dozens of leaves at once, without traversal. -

The LLM at recall.

When a synthesis prompt reads K retrieved chunks, it forms

new

wormholes at query time — Jack mentioning pricing in March connects to Jack's tone in September, not because we precomputed an edge but because the LLM activates both regions of the galaxy together. So our v9 architecture is

wormhole-shaped, not edge-shaped

. The galaxy is the manifold; nearest-neighbor is the wormhole; LLM recall opens new wormholes on demand.

The deeper point

The brain's superpower is that

distance in the network has nothing to do with semantic distance

. Two ideas can be a single synapse apart even if they were learned years apart and look nothing alike. SQL graphs preserve calendar time and schema topology; wormhole networks preserve

semantic relevance

. Our design now bets on the latter — embeddings as the substrate, structure imposed only where it earns its keep (RAPTOR). You named the thing. The flat

facts

table was Euclidean memory; the galaxy + RAPTOR + LLM-at-recall is closer to the brain's wormhole topology — and that's why dropping the facts table felt like a relief, not a loss.