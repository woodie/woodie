# Cowork: cross-project notes

Context for any Cowork session on any repo in this account. Project-specific
history and design conventions live in each repo's own `docs/COWORK.md` where
one exists; this file covers habits that apply everywhere.

## Git lock files

If `git add`/`git commit` fails with `Unable to create '.git/index.lock'`
(or `HEAD.lock`) `: File exists`, it's almost never a real stale lock from a
crashed git process. The working copy is mounted from the user's Mac, and the
sandbox needs explicit one-time permission to delete files there — `rm -f` on
the lock file fails with `Operation not permitted` until that permission is
granted. Fix: call the `allow_cowork_file_delete` tool with the lock file's
path; once granted, `rm -f .git/*.lock` succeeds immediately and the commit
can be retried. The permission covers the whole mounted folder for the session,
so subsequent lock files can be removed without asking again.

Note: older `docs/COWORK.md` files in this account (zouk, lambada) document a
stricter workaround — "never run git from the sandbox" — established before
`allow_cowork_file_delete` was available. That rule is superseded by the above.

## Working on unfamiliar stacks

Some repos here are in languages or frameworks where the Cowork sandbox has no
local toolchain (Go for lambada, Kotlin for next-caltrain-kotlin, Swift for
zouk, etc.). The working pattern that's proven reliable across all of them:

1. Claude reads the source and makes edits by inspection.
2. Claude describes what each change does and why.
3. The user runs the relevant build/test command on their own Mac and confirms.

This means Claude can work effectively on any stack without a per-repo
`docs/COWORK.md` just to document the toolchain gap — the gap is universal and
handled the same way everywhere. A project only needs its own `docs/COWORK.md`
for things that are actually specific to it: design conventions, past decisions,
known gotchas in the code.

When picking up a project cold, reading the repo's own `docs/COWORK.md` (if
present), then `README.md`, then the key source files is enough to get oriented
before making any changes.

## Typos and unclear input

The user often hits enter before catching typos. Read messages charitably and
proceed on best-guess intent for small or obvious errors. For cases where a
typo could meaningfully change direction — "not" vs "now", "remove" vs "move",
anything where the wrong interpretation would send a session down the wrong path
— stop and ask before diving in.

## Test structure

Tests should be structured hierarchically so the output reads like
specification documentation in RSpec `-fd` / Quick-Nimble style. The structure
should make it easy to add new cases inside an existing context rather than
requiring new top-level blocks. BDD frameworks that don't expose this hierarchy
in their output are missing the point.

Each `context` establishes one concrete scenario -- via `beforeEach` (or a
`let`/`var` it sets up) -- so that every `it` nested under it can just state
what's true once that scenario holds. An `it` should never compute or derive
the scenario itself (e.g. `let fifteenSecondsLater =
downloadedAt.addingTimeInterval(15)` inside the `it` body): if a value only
exists to set up the situation the test is about, it belongs in the
enclosing `context`'s `beforeEach`, not buried in one `it`. This is what
actually makes "add new cases inside an existing context" possible -- a
second `it` can assert something else true under that same already-built
scenario without re-deriving it. When an `it` is doing its own setup work,
that's a sign the setup wants to become its own named `context` first, with
the assertion(s) following as `it`s underneath it.

Code should be organized in modules or classes so that units can be tested in
isolation, without going through web, mail, or other external interfaces. If
a piece of code can't be tested without hitting an external interface, that's
a signal to refactor, not to write an integration test.

## Comments

Do not write "novels" above methods or inline. Long comments become noise
that obscures the basic flow, and they go stale. Default to zero comments
per method/fixture/test case, not one. A well-named method, variable, or
`it`/`context` description often makes a comment redundant on its own -- if
the comment would just restate what the name already says, skip it. This
applies per-instance, not just in aggregate: one line above every method in
a file is still "a ton of comments" even though each individual line looks
small next to the "novel" it's not writing.

Reach for a comment only when something is genuinely non-obvious from the
name and surrounding code alone -- and when you do, keep it to one
self-contained line at the exact spot it's needed (a real gotcha: a subtle
ordering requirement, a workaround for a library bug, a non-obvious
invariant). The line doesn't need to cite a doc by name -- if a companion
doc exists, treat it as expected reading alongside the code, the same way
each repo's own `docs/COWORK.md` already is, rather than pointing to it
from every single comment.

Decision history, rejected alternatives, and longer rationale belong in
docs, not in code comments. When that material adds up enough that
one-liners aren't enough on their own (or when retrofitting a file that
already has long-comment debt), pull it into a `docs/COMMENTS.md`,
organized by file and then by the type/property/function each note is
about -- see zouk's `docs/COWORK.md` ("Comment convention") for the pattern
this was first established with. Retrofitting existing long comments is a
comment-only edit (no behavior change): catalog every comment in the file,
extract it into `docs/COMMENTS.md`, then strip the source down to at most
one line per spot. A stale comment is worse than no comment, so keep code
comments and docs in sync when either changes.

## Code style

Match the spirit and best practices of the existing code — don't impose new
patterns. If an optimization or refactor could impact maintainability, ask
explicitly rather than just doing it. The goal is code the user can read and
maintain, not code that showcases what Claude finds elegant.
