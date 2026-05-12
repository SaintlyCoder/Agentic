One of the things we are often trying to do in this modern age of AI-assisted development is verbalise good software design practice into plain English.
That sounds new, but many of the ideas are not new at all. The old classic software books already contain a lot of the language we need.
For example:
The Pragmatic Programmer by Andrew Hunt and David Thomas gives us ideas like tracer bullets, prototypes, orthogonality, and programming by coincidence.
Refactoring by Martin Fowler gives us a vocabulary for improving the design of existing code without changing its behaviour.
Working Effectively with Legacy Code by Michael Feathers teaches us how to create seams, add tests, and make unsafe systems safer to change.
Clean Code by Robert C. Martin gives us a language for readability, naming, functions, and maintainability.
Domain-Driven Design by Eric Evans gives us concepts like bounded contexts, ubiquitous language, aggregates, and modelling around the domain.
Patterns of Enterprise Application Architecture by Martin Fowler gives us a vocabulary for common architectural shapes in business software.
What has changed is not that these principles have suddenly become true. They were already true.
What has changed is that we now need to express them in a way that AI agents can follow.
That means turning tacit engineering judgement into explicit instructions. Instead of vaguely telling an AI to “implement the feature,” we need to say things like:
> Break this PRD into independently grabbable vertical slices.
Each issue should be traceable to a user-visible behaviour.
Avoid horizontal tasks like “build the schema” or “implement the API layer.”
Each slice should include the smallest necessary frontend, backend, data, test, and deployment changes to validate the behaviour end-to-end.
This is where the classic literature becomes extremely useful. It gives us a tested vocabulary for steering work.
“Tracer bullets” is a perfect example. It is not just a metaphor. It is a software delivery strategy. It says: do not disappear into a cave for weeks building layers in isolation. Fire something through the whole system early so you can see whether you are aiming in the right direction.
That principle becomes even more important with AI.
AI is extremely capable of producing code, but it is also very capable of producing the wrong code confidently and at scale. So the goal is not merely to generate more implementation. The goal is to increase the frequency and quality of feedback.
Vertical slices do that.
They make the work:
easier to assign,
easier to test,
easier to review,
easier to integrate,
easier to abandon or redirect if the approach is wrong.
So when I say I want to be involved in the task or issue breakdown, this is why. The breakdown determines whether the AI is going to work in a feedback-rich way or whether it is going to code blind.
A good issue breakdown should not just describe components to build. It should create tracer bullets through the product.
Each issue should answer:
> What user-visible behaviour are we proving?
What is the smallest end-to-end path that validates it?
What layers need to change for this slice to work?
How will we know it is done?
How can a reviewer trace the behaviour through the implementation?
That is the shape of good AI work.
Not giant horizontal phases.
Not layer-by-layer scaffolding.
Not impressive-looking piles of unintegrated code.
Small, traceable, vertical slices.
Tracer bullets.
