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
