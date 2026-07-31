# `.cbmignore` — Excluding Files from Indexing

`.cbmignore` is a project-specific ignore file that controls which files the
indexer sees. It uses gitignore-style syntax and is read from the **root of
the indexed directory** (`<repo>/.cbmignore`). Nested `.cbmignore` files in
subdirectories are not read.

It applies at **file discovery time** — the directory walk that selects files
for parsing. Every indexing path uses the same discovery: the initial
`index_repository`, manual re-indexing, and background auto-sync. A path
matched by `.cbmignore` never enters the graph. Changes to `.cbmignore` take
effect on the next (re-)index.

Unlike `.gitignore`, it has no effect on git itself — it only shapes what the
indexer sees. Commit it to share indexing excludes with your team, or list it
in `.gitignore` to keep personal excludes untracked.

To verify it works: directory subtrees skipped during discovery are reported
in the `index_repository` response under `excluded`
(`{"dirs": [up to 25 paths], "count": <total>, "truncated": <bool>}`).

## Syntax

One pattern per line. Blank lines are ignored, lines starting with `#` are
comments, and trailing whitespace is trimmed.

| Feature | Meaning |
|---|---|
| `*` | matches any run of characters, except `/` |
| `?` | matches exactly one character, except `/` |
| `**` | matches across directory boundaries (`**/name`, `dir/**`, `a/**/b`) |
| `[abc]`, `[a-z]` | character classes; `[!a-z]` / `[^a-z]` negate the class |
| trailing `/` | pattern matches **directories only** |
| `/` anywhere else | anchors the pattern to the repo root |
| no `/` in pattern | matches the file/directory name at **any depth** |
| leading `!` | negation — re-includes a previously matched path; the **last matching pattern wins** |

Examples:

```gitignore
# Generated protobuf output, anywhere in the tree
*.pb.go

# A specific top-level directory (leading / anchors to the repo root)
/third_party/

# Any directory named "snapshots", at any depth (trailing / = directories only)
snapshots/

# Everything under any fixtures directory
**/fixtures/**

# Anchored glob: generated clients for any single-character API version
/api/v?/generated/

# Character class: yearly log folders 2020-2029
/logs/202[0-9]/

# Ignore all YAML, but keep CI configs (negation — last match wins)
*.yaml
!ci.yaml
```

## Precedence

Discovery applies its filters in a fixed order. A `.cbmignore` negation can
explicitly re-include a path rejected by repository, nested, or global Git
ignore rules. For directories:

1. **Built-in skip list** — `.git`, `node_modules`, `dist`, `target`,
   `vendor`, tool caches, etc. (60+ names; the fast/moderate index modes add
   more, e.g. `docs`, `examples`, `testdata`). A `.cbmignore` negation can
   re-include ordinary skip-list directories, but never the safety core:
   `.git`, `node_modules`, `.worktrees`, or `.claude-worktrees`.
2. **Repo `.gitignore`** — `<repo>/.gitignore` merged with
   `<git-common-dir>/info/exclude` (worktree-aware); later patterns win on
   conflict. Honored even when the indexed directory is not a git repo root.
3. **Nested `.gitignore` files** — picked up during the walk and matched
   relative to their own directory.
4. **`.cbmignore`** — a positive match skips the path; a negated match
   re-includes paths rejected by layers 2, 3, or 5.
5. **Git global excludes** — `core.excludesFile` from `~/.gitconfig` or the
   XDG git config (default `$XDG_CONFIG_HOME/git/ignore`); consulted only
   when the project is a git repo with a config.

For files, built-in suffix filters (`.png`, `.o`, `.db`, …; fast modes add
archives, media, lockfiles, `.min.js`, …) and fast-mode filename/substring
filters run **before** the ignore files, and a maximum-file-size cap runs
after them; none of these are overridable from `.cbmignore`.

## Negation (`!`)

- **Within `.cbmignore`**: standard gitignore semantics. Patterns are
  evaluated top to bottom and the last matching pattern wins, so
  `!pattern` re-includes something an earlier line excluded.
- **Full include overrides**: a negated glob opens every ignored ancestor
  needed to reach files it can match. You only need the file glob itself; do
  not add separate negations for each parent directory. Ancestors are opened
  for traversal only, not indexed by that fact.
- **Across layers**: a `.cbmignore` negation overrides repository
  `.gitignore`/`info/exclude`, nested `.gitignore`, and Git global excludes.
  It also re-includes ordinary built-in skip-list directories such as `obj/`,
  `dist/`, and `target/`. It cannot override the safety core (`.git`,
  `node_modules`, `.worktrees`, `.claude-worktrees`), built-in suffix or
  filename filters, fast-mode filters, or the size cap.
- **Symlinks**: on POSIX, a symlink is followed only when a negated rule
  directly matches it or could match a descendant through it. Its resolved
  target must remain inside the indexed repository. Other symlinks remain
  skipped.

For example, to index a toolchain subtree while leaving it ignored by Git:

```gitignore
!/.toolchain/x86_64-linux/packages/**/*.h
!/.toolchain/x86_64-linux/packages/**/*.hh
!/.toolchain/x86_64-linux/packages/**/*.hpp
```

This follows the active `packages` symlink without traversing other toolchain
cache directories. The re-included files still pass the normal source-file
filters, so object files, libraries, archives, and other unsupported files
remain excluded.
