# Cowork: cross-project notes

Context for any Cowork session on any repo in this account. Project-specific
history and design conventions live in each repo's own `docs/COWORK.md` where
one exists; this file covers habits that apply everywhere.

## Goal: RSpec's BDD joy, in every language, kept human-maintainable

The throughline across `spec`/`expect`/`gorderly` and every language this
account writes tests in (Go, Ruby, Swift, whatever's next) is bringing
RSpec's actual experience -- real nested `describe`/`context`/`it` output,
tests that read like documentation, a `subject`/`let` discipline that keeps
scenarios DRY -- to languages that don't have it natively, without giving up
the standard toolchain to get it. "Test structure" below is the concrete
rulebook this produces.

Equally important: the result has to be code Woodie can pick up himself, not
just code Claude can maintain in the moment. That means boring, readable
structure over clever abstraction, comments limited to what's genuinely
non-obvious (see "Comments" below), and matching each language's own idiom
rather than importing a pattern verbatim from wherever it was invented --
Go's `before` hook standing in for Ruby's real `let` is the clearest example
of this so far, not a compromise but the correct per-language translation.
If a change only makes sense with a whole session's context behind it, it
isn't done yet -- it needs to read cold, the same way "Working on unfamiliar
stacks" below already assumes Claude itself has to read a repo cold to work
on it.

## `docs/COWORK.md` is never what a README points to

Every repo's `docs/COWORK.md` (this file included) is for Claude and Woodie
to come up to speed on a project's own history, decisions, and gotchas --
not for an outside reader. Every repo also needs something written on top
of that for humans: `README.md` at minimum, `docs/DEVELOPMENT.md` too for
anything with a real dev loop (build/test/deploy commands, config, ops
runbooks -- see `lambada`'s or `xctidy`'s for the shape). A `README.md`/
`DEVELOPMENT.md` must never send a reader to `docs/COWORK.md` for "the
reasoning" or "why" behind something -- if a fact is worth a human reader
knowing, it gets written where they'll actually see it (inline, or in
whichever human doc covers that topic), not deferred to the Claude/Woodie
file. Found and fixed across `spec`, `expect`, `huck`, and `humane-kotlin`'s
READMEs in one pass -- each had a "see `docs/COWORK.md` for why" pointer
that had to become a real inline sentence or two instead. `lambada`'s
`docs/DEVELOPMENT.md` has one narrow, deliberately-marked exception (a
file-descriptor leak theory, explicitly flagged "unconfirmed, not a
documented fact") -- that's the only shape of COWORK.md reference that's
ever acceptable in a human doc: pointing at something explicitly unvetted,
never at settled reasoning a reader needs.

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

Run the repo's `check` target (or `lint` + `test` if there's no combined
`check`) *before* pushing and *before* tagging — not after. The sandbox has
no toolchain for most repos here (see "Working on unfamiliar stacks"), so
a commit gets made on inspection-only confidence; that's fine, but it
means the sandbox itself can never confirm a change is clean, only the
user's own Mac can. Hand off the verification command and wait for a real
clean result *before* creating the tag or telling the user to push, not
after. Both `gorderly` and `xctidy` got re-tagged mid-session (`gorderly`
v0.3.0 → v0.3.1; `xctidy` v0.3.0 → v0.3.1 → v0.3.2) because tags and push
instructions went out first and `make check`/CI caught real failures
(`errcheck`, two `line_length` violations, one `identifier_name`
violation) only afterward — each one forced a retag, and one of them
(`gorderly`) collided with a push that had already landed on the remote
mid-session, requiring a `git reset --hard origin/main` and a fresh
forward commit to recover (see `gorderly`'s own `docs/COWORK.md` for that
incident). None of this was harmful, but all of it was avoidable: run
`check`, confirm clean, *then* tag, *then* tell the user to push.

Don't amend a commit once there's any chance it's already been pushed.
Amending rewrites the commit's hash; if the old hash already reached the
remote (even if you weren't told so explicitly, or the push happened
between your own turns), the local and remote histories now genuinely
diverge — not a lock file, not a stale ref, a real fork — and the next
push gets rejected as non-fast-forward. If a fix is needed after a commit
might already be public, make it a new commit on top instead of an amend,
and re-verify the fast-forward relationship (`git merge-base --is-ancestor
origin/main HEAD`) before tagging or pushing again.

For an **annotated** tag, `git rev-parse <tag>` returns the tag *object's*
own hash, not the commit's — that will look like a mismatch against `git
rev-parse HEAD` even when the tag is completely correct. Use `git rev-parse
<tag>^{commit}` (or `git log -1 <tag>`) to dereference to the actual commit
before comparing. Don't mistake this for the lock-file bug above; check
`^{commit}` first.

A `git push --tags` that runs right after a plain `git push` failed
silently (a shell typo turning it into a no-op, in one real case — `it
push` instead of `git push`) can still look completely successful: the tag
push transfers whatever commit it points at along with it, even if that
commit isn't the tip of any pushed branch yet. A tag-triggered release
workflow builds correctly regardless, since it checks out the tag directly
— but the branch itself (`main`) is left silently stale on the remote,
missing whatever commit the tag was on. Don't treat a clean tag-push output
as proof the preceding branch push also succeeded; check `git status -sb`
for `ahead`/`behind` against the remote-tracking branch explicitly.

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

## Releasing across multiple repos

When a release spans more than one repo -- tagging/pushing/`gh release
create` across `humane`/`humane-ruby`/`humane-swift`, or a library plus its
consumers (see "Shared libraries across sibling repos" above) -- hand off
one copy-pasteable code block per repo/step, not a single block chaining
several `cd`/`git`/`gh` commands together. A combined block that partly
fails (a hung prompt, a slow network call, an interactive `gh` login, a
lock file specific to one repo) can silently skip or block everything
chained after it, and interleaved output from several repos makes it hard
to tell which step actually ran where -- exactly the kind of thing "Pushing"
above already flags for a single `git push`/`git push --tags` pair, just
worse with three repos instead of one. Separate blocks let the user run and
confirm each repo independently, re-paste just the one that failed, and
only move to the next block once the previous one's output looks right.
Slower to hand off than one big paste, but nothing to untangle when a step
in the middle goes sideways.

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

## Command hand-offs: always cd to the repo first

Woodie may have several terminals open across different repos at once, so a
handed-off command should never assume "whichever directory the terminal
happens to be in" — start every hand-off with an explicit `cd
~/workspace/<repo>` rather than a bare `git push`/`make check`/etc. Applies
to every command hand-off, not just git: the same ambiguity exists for
`make`, `gh`, or any other repo-scoped command.

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

## Go version policy: target current Go, don't preserve legacy compatibility

When a `go.mod`'s `go` directive needs bumping to compile a real feature
(generics, `t.Context()`, whatever's next), bump it to what that feature
actually needs and move on -- don't add build tags, reflection fallbacks,
or any other compatibility shim to keep working on an older toolchain.
Upgrading Go itself is low-friction for anyone consuming these projects;
contorting the code to dodge that upgrade is the "odd thing" worth
avoiding, not the version bump. `spec`'s fork bumping from `go 1.13` to
`go 1.24` for `Var[T]`/`it.Context()`, and `expect`'s own `go.mod` getting
auto-bumped to `go 1.24` by `go mod tidy` to satisfy that dependency, are
both fine as-is -- no need to chase a lower floor for either.

## Verification commands: prefer Makefile targets

When a repo has a `Makefile` with `lint`/`test`/`check` targets (the
`humane`/`humane-ruby`/`humane-swift` family all do, one Makefile per
language wrapping that language's own toolchain), hand off `make
build`/`make test`/`make lint`/`make check` in verification command blocks
rather than the raw underlying commands (`go test`/`ginkgo-fd`, `bundle
exec rspec`, `swift test`) -- `test`/`lint` are always verbose (the
per-example breakdown), `check` is terse (silent on success, full log on
any failure), matching across all three languages even though the literal
commands underneath differ. Only fall back to the raw command directly if
a repo has no Makefile, or the specific raw form itself is what's being
discussed (e.g. diagnosing why one particular `swift test` flag matters).

## Command-line first, GUI steps spelled out in full

Woodie's default is command-line tools over GUI apps wherever a CLI path
exists -- the next-caltrain-kotlin/next-caltrain-swift `sim.sh`/`Makefile`
cleanup (merging scripts, adding `lint`/`test`/`check` targets, making the
default simulator/emulator configurable) is a good example of the shape he
likes: a small number of scriptable entry points instead of remembering IDE
menus. He's also explicit that he doesn't have Xcode's GUI memorized ("no
clue what happens in Xcode").

Some steps genuinely have no CLI path, though -- `xcrun altool` is
deprecated, and archiving/uploading an iOS build only works through Xcode's
Organizer and the Distribute App wizard (see next-caltrain-swift's
`docs/RELEASE.md`). For steps like this:

- Don't assume familiarity with the app's menus/wizards -- spell out the
  exact menu names and button labels, not a high-level "then archive and
  upload it."
- Offer a choice between an interactive walkthrough (computer-use teach
  mode) and a plain-text explanation, per Cowork's teach-mode convention --
  but expect plain-text to be the pick; that's what he's chosen every time
  so far.
- Confirm each step via a screenshot before describing the next one, rather
  than dumping the whole sequence up front. App Store Connect's UI shifts
  around occasionally, and a screenshot confirms which exact screen he
  actually landed on before the next instruction is given.
- Once a walkthrough proves out for a given task, fold the concrete
  click-by-click steps into that project's own runbook (e.g.
  `docs/RELEASE.md`) so the next release doesn't need the whole
  conversation re-derived from scratch -- see the Archive/Distribute
  section there for the version this produced.

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

In Go specifically, never put a literal `/` inside a `describe`/`context`/
`it` name when using `spec` (or plain `t.Run`). `spec`'s default flat mode
joins nested names with `/`, `go test -v` uses `/` as the real subtest
hierarchy separator, and `gorderly` rebuilds its tree by splitting on `/` --
a literal slash in a name (e.g. naming a block after an HTTP route, `"GET
/download/{filename}"`) is indistinguishable from those real separators and
silently produces empty or misnested tree nodes. Found and fixed for real
in `lambada`'s `cmd/lambada-web/main_test.go` -- see its `docs/COWORK.md`.
Write route-shaped names without the slash instead (`"GET download by
filename"`, not `"GET /download/{filename}"`).

## Comments

No repo in this account should carry multi-line comment blocks in source --
not `///` doc-comment paragraphs, not stacked `//`/`#` notes. That's a hard
rule, not a style preference: if a comment needs more than one line, it
belongs in that repo's `docs/COMMENTS.md`, not in the file.

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

**Exception: mocking and stubbing.** A test double (a struct embedding a
real interface, then overriding specific methods to intercept them -- the
`spyT` pattern in `expect`) is worth breaking the one-line rule for. Go has
no built-in mocking, the embed-then-override mechanism isn't obvious on
sight, and getting it subtly wrong (calling a method that isn't overridden,
assuming the embedded value is non-nil) is an easy mistake. Write a real
comment at the point of use, wrapped at ~80 columns, explaining what the
double does and how the mechanism works for someone seeing it cold. This
is still about the current mechanism, not history -- why the technique was
chosen over an alternative, or how the code evolved to look this way,
still belongs in `docs/COWORK.md`, not the comment.

Simple substitutions get the same treatment, just in one line instead of a
paragraph: swapping a package-level var for `t.TempDir()` (standing in for
a real configured directory) is a stub too, even though the mechanism is
trivial. Name it as such right at the call site --
`attachmentDir = t.TempDir() // stub implementation` (`lambada`'s
`attachments_test.go`, `main_test.go`, `scanfiles_test.go`) -- so a reader
doesn't mistake it for arbitrary fixture setup. The rule is the same either
way: if a value is standing in for something real, say so; the only thing
that changes with complexity is one line versus several.

## Prose: don't lead a sentence with a lowercase project name

Repo/library names stay lowercase in backticks (`expect`, `spec`, `gorderly`)
to match their real package names, but that collides with English
capitalization at the start of a sentence -- "`expect` is a matcher
library..." reads like a typo. Rephrase so the name isn't the first word,
rather than capitalizing it or dropping the backticks: "With `expect`,
you get..." or "Using `gorderly`, ..." instead of "`expect` lets you...".
Applies to README/COWORK prose; headers are fine either way (e.g. "###
`expect`'s matcher list").

## Code style

Match the spirit and best practices of the existing code — don't impose new
patterns. If an optimization or refactor could impact maintainability, ask
explicitly rather than just doing it. The goal is code the user can read and
maintain, not code that showcases what Claude finds elegant.
