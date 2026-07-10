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

## Pushing

Claude commits freely once changes are ready, but doesn't run `git push`
(or anything else that touches a remote — opening PRs, pushing tags) without
being asked. Committing is local and easy to undo; pushing isn't, so it stays
a deliberate, separate step the user takes (or explicitly requests).

This isn't just a policy choice — the sandbox has no actual network route to
GitHub, RubyGems, or similar (confirmed: `ssh -T git@github.com` fails DNS
resolution, `git ls-remote` fails, `curl rubygems.org` times out). `gh` and
gem-push credentials aren't present either. So `git push`, `git push --tags`,
`gh release create`, `gem push`, and any `go get`/`swift package update`
against a not-yet-fetchable new version all have to happen on the user's own
Mac regardless of how the request is phrased. The reliable pattern: commit
and tag locally in the sandbox, hand the user the exact push/publish commands
(as a copy-pasteable block), and let them run and confirm.

## Tagging releases

Verify before tagging, every time — this account has a documented history of
a stale-`main`/lock-file mixup landing a tag on the wrong commit (see zouk's
`docs/COWORK.md`, the `v1.6.0`/`v1.7.0` incidents, and `docs/DELIVERY.md`'s
pre-flight checklist there). After committing, run `git log -1` and actually
read it before tagging; after tagging, confirm the tag really points at that
commit.

For an **annotated** tag, `git rev-parse <tag>` returns the tag *object's*
own hash, not the commit's — that will look like a mismatch against `git
rev-parse HEAD` even when the tag is completely correct. Use `git rev-parse
<tag>^{commit}` (or `git log -1 <tag>`) to dereference to the actual commit
before comparing. Don't mistake this for the lock-file bug above; check
`^{commit}` first.

## Shared libraries across sibling repos

`humane`/`humane-ruby`/`humane-swift` (consumed by `lambada`, `scandalous`,
`zouk`) is the recurring shape in this account: one piece of logic, ported to
several languages, pulled into several apps. When one of those libraries gets
a real behavior change, the sequence that's worked cleanly:

1. Implement + spec the change in the library itself, in every language,
   confirmed on real hardware.
2. Publish the library first (tag, push, `gh release create`, `gem push` as
   applicable) — the consumers' version pins need something real to resolve
   against.
3. For each consumer: bump its pin, then regenerate the *real* lockfile with
   the actual toolchain (`bundle install`, `go mod tidy`, `swift package
   update`) — bumping just the version string in `Gemfile`/`go.mod`/
   `Package.swift` isn't enough, the lockfile (`Gemfile.lock`, `go.sum`,
   `Package.resolved`) needs a real fetch to pick up the right checksum/
   revision. None of this is possible from the sandbox (see "Pushing"), so
   it's the user's own toolchain doing the resolving.
4. Run that consumer's real test suite for confirmation, then decide whether
   the consumer itself needs its own version bump/tag (it does if the
   behavior change is user-visible, e.g. wording a person would notice).
5. Once every consumer's confirmed, boundary-value tests that only re-verify
   the *library's* own behavior (not the consumer's wiring or repo-specific
   logic) are safe to prune from each consumer's spec suite — see any of the
   three consumer repos' `docs/COWORK.md` for what "wiring vs. duplicate
   coverage" looks like in practice.

## Remote/embedded shells

`touch -d "N minutes ago"` (and similar natural-language relative dates) is
not reliable everywhere — hit a case on a Raspberry Pi where it silently
resolved to a *future* timestamp instead of a past one, magnitude roughly
right but sign flipped. Root cause wasn't tracked down; the fix was to stop
relying on relative-phrase parsing and compute the epoch offset directly
instead, which no shell/locale/`date`-implementation quirk can misread:

```
touch -d "@$(($(date +%s) - 2670))" filename   # 2670 seconds ago, unambiguous
```

Reach for this form on the first sign of a suspicious result (future instead
of past, wrong magnitude) rather than trying more relative-phrase variants.

## Handing off long text

When the user needs to paste something substantial into another surface (a
GitHub issue/PR comment, a form, etc.), write it to a file and let them copy
from that rather than constructing a shell one-liner or heredoc for them to
paste into their terminal — a multi-line `gh issue close --comment "$(cat
<<'EOF' ... EOF)"` block turned out to be "too much" to paste cleanly in
practice. A `present_files`'d markdown file plus the plain command (no
inline heredoc) is the more robust handoff; offer the heredoc version only
if asked for something scriptable.

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
