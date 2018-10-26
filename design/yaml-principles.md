# YAML language principles

Here are the principles we're using to evaluate YAML suggestions, especially when they affect the base syntax.
These are *very* rough in priority order.

1. Make simple things simple, complex things possible.
Consider the new user and how they grow-up / level-up.
Don't forget the deep expert who has to read and write these things day in and day out.
2. Be opinionated.
Try to do the right thing automatically.
Give a way to bail out if we guess wrong.
3. Follow the principle of least surprise.
If you have to make a decision with no clear "right" answer, consider which way will be less surprising once learned.
That'll be easier to *remember*.
4. Understand what others chose to do and why.
Our system is different in large and small ways from other CI/CD systems.
This principle *doesn't* mean "just do what <X> does".
5. Our language is terse, moreso for common things.
It's "job", not "jobToRun".
A well-placed adjective can clear up a lot of confusion, though: it's "displayName", not "name".
