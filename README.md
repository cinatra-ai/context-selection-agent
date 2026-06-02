# Context Selection Agent

Let people choose which workspace artifacts feed into another agent's context before it runs. When a larger agent needs supporting material — an ICP, a brand voice, a product portfolio, a strategy document — this helper surfaces the eligible artifacts in a quick picker, locks the user's choice to a specific revision, and returns those pinned references so the calling agent's run stays reproducible later.

## Capabilities

- Surface the artifacts eligible for a given context slot in an interactive picker
- Confirm the user's selection before the calling agent continues
- Pin each chosen artifact to a specific revision so the run is reproducible
- Record an audit trail of which artifacts were chosen, for which slot, and by whom
- Return the pinned references for the calling agent to consume
