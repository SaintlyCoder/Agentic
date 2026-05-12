Breaking Down Plans to Tracer Bullets
I mentioned earlier that I do not really read the full PLAN or SPEC anymore. There is, however, one area where I still very much want to be involved: the breakdown into tasks or issues.
This is where the shape of the work gets decided. If the work is broken down badly, the AI can generate a huge amount of code while still delaying the moment where anyone can tell whether the system actually works.
The classic mistake is to break work down horizontally.
A horizontal plan follows the layers of the system:
1. Build the database schema.
2. Build the API layer.
3. Build the backend services.
4. Build the frontend screens.
This looks tidy. It feels logical. It mirrors the architecture diagram.
But it has a serious problem: you can do a lot of work without proving that anything works end-to-end.
Until the database, API, services, frontend, tests, and deployment path are connected, there is no working user-visible behavior. There is just a collection of partially completed layers, each making assumptions about the others.
That creates several risks.
Why Horizontal Implementation Is Weak
1. You Delay End-to-End Validation
You cannot tell whether the feature actually works until very late in the process.
The schema might look correct. The API might look correct. The frontend might look correct. But the integration between them may still fail.
By the time you discover that, a lot of code has already been written.
2. Bugs Are Discovered Too Late
Horizontal implementation hides integration problems.
Maybe the frontend needs data in a different shape than the API provides. Maybe the API cannot efficiently query the schema. Maybe the backend service made an assumption that does not hold once the real UI flow exists.
You only discover these issues once the layers are finally wired together.
That is late feedback, and late feedback is expensive.
3. It Creates Fake Progress
Horizontal work can produce a lot of visible output: many files, many commits, many generated components, many endpoints.
But none of it may be usable yet.
You get volume, not verified value.
This is especially dangerous with AI because AI is very good at producing plausible-looking code. It can create the feeling of progress long before there is anything integrated, testable, or reviewable as a real product behavior.
4. It Increases Integration Risk
Each layer is built based on assumptions about the layers above and below it.
The longer those assumptions go untested, the more risk accumulates. When integration finally happens, you may discover that the pieces do not fit together cleanly.
At that point, the work is harder to unwind because the assumptions have spread across the whole implementation.
5. It Is Hard to Review
Reviewing a whole database layer, then a whole API layer, then a whole frontend layer is cognitively expensive.
The reviewer cannot easily ask: “What user behavior does this support?”
Instead, they have to infer how dozens of disconnected changes might eventually combine into a working system.
That is a bad review surface for humans, and it is an even worse control surface for AI.
The Better Approach: Vertical Slices
The better approach is to break the work into vertical slices.
A vertical slice implements one small piece of functionality all the way through the system.
Instead of this:
> Phase 1: build all database models
Phase 2: build all APIs
Phase 3: build all frontend pages
You do this:
> Slice 1: user can create a quiz
Slice 2: user can answer a quiz question
Slice 3: user can see their score
Slice 4: user can redeem a coupon
Each slice crosses the relevant layers:
Frontend → API → service → database → tests → deployment
The point is not that every slice must touch every possible part of the system. The point is that each slice should go far enough through the stack to produce something integrated, testable, and reviewable.
A good vertical slice gives you a complete thread of behavior.
You can point to it and say:
> This is the user action.
This is the UI that triggers it.
This is the API it calls.
This is the service logic.
This is the data it reads or writes.
This is how we test it.
This is how we know it works.
That makes the work traceable.
Traceable Issues
This is why, inside the PRD-to-issues skill, I use the rule:
> Create independently grabbable issues using vertical slices.
The process is roughly:
1. Locate the PRD.
2. Explore the codebase, especially if this is a fresh session.
3. Draft vertical slices.
4. Break the PRD into traceable issues.
A traceable issue is one that can be understood, implemented, tested, and reviewed as a coherent unit of user-visible behavior.
It should not be:
> Add all database tables.
Or:
> Build all API endpoints.
Or:
> Implement the frontend.
Those are horizontal chunks. They are too broad, too disconnected, and too slow to produce feedback.
A better issue is:
> Allow a user to create a quiz from the dashboard.
That issue might require a small schema change, one API endpoint, one service method, one frontend flow, and one test path. But the work is unified by a behavior the user can actually perform.
That is much easier to assign. It is much easier to review. It is much easier to test. And it gives the AI a much clearer target.
Why I Think of Them as Tracer Bullets
The metaphor I like here is tracer bullets.
Imagine firing into the night sky. If you only fire ordinary bullets, you cannot see where they are going. You are shooting blind.
Tracer rounds solve that problem. Every few rounds, one glows as it travels, drawing a visible line through the sky. It gives you feedback. It shows you whether your aim is correct.
That is what a vertical slice does in software.
It gives you a visible line through the system.
It shows whether the work is landing where intended.
Without vertical slices, the AI is often coding blind. It may be producing code, but you do not know whether that code is moving the product toward a working, integrated system.
With vertical slices, you increase the rate of feedback. You can test earlier. You can review earlier. You can correct direction earlier.
That is the core rule:
> Do not break plans down by architectural layer. Break them down into tracer bullets: small, traceable, vertical slices of user-visible functionality.
