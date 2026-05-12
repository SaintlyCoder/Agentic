Here’s the clean version of the argument:
Core thesis
Complexity is your biggest enemy. It is also your AI agent’s biggest enemy.
An agent is only as good as the context it can see. If the relevant code, architecture, constraints, side effects, and historical decisions do not fit into its usable context, it will start making plausible but wrong changes.
And that is where a lot of agent-generated garbage comes from.
Not because the model is “stupid,” but because it is operating with partial visibility.
Important correction
A 200,000-token context window does not mean 200,000 lines of code.
Code is token-dense. Depending on language and style, 200k tokens might represent something like tens of thousands of lines, not hundreds of thousands. A 600k-line codebase is far beyond what the agent can fully ingest at once.
So the real bottleneck is not only the model. It is:
> Can the system retrieve exactly the relevant code for the task?
That is a hard, largely unsolved information-retrieval problem.
Polished tweet version
Complexity is your biggest enemy. It is also your AI agent’s biggest enemy.
Agents do not understand your whole codebase. They understand the slice of it that fits into context.
So when your codebase becomes too large, too interconnected, or too messy, the agent starts making changes without seeing everything it needs to see.
Even with a 200k-token context window, a large repo cannot simply be “understood.” The agent depends on retrieval: did it find the right files, the right abstractions, the hidden coupling, the old edge cases?
That retrieval problem is not solved.
And now the agent adds more code. More files. More indirection. More coupling. More complexity.
Eventually, the agent creates a codebase that future agents cannot fit into context either.
It digs its own context-window grave.
AI did not invent garbage code. It learned a lot of it from us. The internet contains some pearls, yes, but also decades of overengineering, cargo-cult patterns, and broken abstractions.
So the danger is not just that agents write bad code.
The danger is that agents accelerate complexity faster than anyone can understand it.
Stronger aphorism
> The agent’s biggest enemy is not hallucination. It is complexity. Hallucination is often what happens after complexity exceeds context.
More provocative version
AI agents are very good at producing code.
That is also the problem.
Because every line they add increases the amount of context needed for the next task. Every abstraction, helper, wrapper, service, and dependency becomes another thing the next agent must retrieve and understand.
At some point, the codebase becomes larger than the agent’s effective context.
Then the agent is no longer programming against the system.
It is programming against a lossy summary of the system.
That is where the garbage begins.
Complexity was already our biggest enemy. Agents just make the feedback loop faster.
