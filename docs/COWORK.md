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

Code should be organized in modules or classes so that units can be tested in
isolation, without going through web, mail, or other external interfaces. If
a piece of code can't be tested without hitting an external interface, that's
a signal to refactor, not to write an integration test.

## Comments

Do not write "novels" above methods. Long inline comments become noise that
obscures the basic flow, and they go stale. The rule: one line referencing a
relevant design doc (`docs/COWORK.md`, `docs/DELIVERY.md`, etc.) if context is
genuinely needed; otherwise nothing. Decision history and rationale belong in
docs, not in code comments. A stale comment is worse than no comment.

## Code style

Match the spirit and best practices of the existing code — don't impose new
patterns. If an optimization or refactor could impact maintainability, ask
explicitly rather than just doing it. The goal is code the user can read and
maintain, not code that showcases what Claude finds elegant.
