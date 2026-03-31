# Relationship Scoring Algorithm — Code Pattern

A non-AI feature that demonstrates algorithmic thinking. Every contact gets a "relationship health" score (0-100) based on interaction history.

## The Insight

Not all interactions are equal, and not all recency is equal. A coffee meeting last week is worth more than 10 emails three months ago. The scoring system captures this with two dimensions: **type weight** and **time decay**.

## Type Weights

```typescript
const INTERACTION_WEIGHTS: Record<InteractionType, number> = {
  meeting:  15,  // Highest signal — you made time to be there
  call:     10,  // Synchronous but less commitment
  social:    8,  // Social events, casual encounters
  email:     6,  // Async, but requires thought
  message:   4,  // Quick, low-effort
  other:     3,  // Catch-all
};
```

**Why these numbers?** Meetings require the most intentional effort — scheduling, showing up, dedicating time. Messages are the lowest effort. The weights reflect relationship investment, not communication frequency.

## Time Decay

```typescript
const HALF_LIFE_DAYS = 30;
const DECAY_CONSTANT = Math.LN2 / HALF_LIFE_DAYS;

// An interaction's contribution halves every 30 days
function decayedScore(weight: number, daysSinceInteraction: number): number {
  return weight * Math.exp(-DECAY_CONSTANT * daysSinceInteraction);
}
```

**Why 30-day half-life?** Most professional relationships need monthly contact to stay warm. After 30 days without interaction, the "warmth" of the last touchpoint has halved. After 60 days, it's at 25%. After 90 days, ~12%. This matches intuition about relationship fade.

## Aggregate Score

```typescript
function calculateRelationshipScore(interactions: Interaction[]): number {
  const now = new Date();

  const rawScore = interactions.reduce((sum, interaction) => {
    const daysSince = differenceInDays(now, interaction.occurredAt);
    const weight = INTERACTION_WEIGHTS[interaction.type];
    return sum + decayedScore(weight, daysSince);
  }, 0);

  // Cap at 100 for UI normalization
  return Math.min(100, Math.round(rawScore));
}
```

## Recalculation Strategy

Scores are recalculated daily via a cron job (`/api/cron/scoring`), not on every interaction. This is a deliberate trade-off:

- **Pro:** No latency on interaction creation. Batch processing is more efficient.
- **Con:** Scores can be up to 24 hours stale.
- **Why it's fine:** Relationship health changes slowly. A 24-hour delay is invisible to the user.

## What This Enables

The score powers two features:
1. **"Reach out to" suggestions** — contacts with rapidly declining scores surface as prompts
2. **Circle visualization** — contacts are placed in concentric rings based on relationship strength, not manual categorization
