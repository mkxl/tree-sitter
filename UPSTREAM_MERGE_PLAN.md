# Upstream Merge Plan For Chunked `tree-sitter-highlight`

## Goal

Get chunked/non-contiguous source support accepted upstream in `tree-sitter-highlight`, while minimizing maintainer review burden and making the change look like an additive, compatibility-preserving library improvement rather than a fork-specific patch.

The central value proposition is:

- `tree-sitter-highlight` already owns highlight semantics for injections, locals, precedence, and cancellation.
- Tree-sitter parser/query APIs already support chunked text.
- The highlighter currently forces clients with rope-backed buffers to materialize a full contiguous `&[u8]`.
- A small source-provider API removes that copying requirement while preserving the existing byte-slice API.

## Current Fork Context

- Fork remote: `git@github.com:mkxl/tree-sitter.git`
- Implementation branch: `michael/chunked-tree-sitter-highlight`
- Planning branch: `michael/chunked-highlighting-upstream-plan`
- Main crate touched: `crates/highlight`

The implementation branch currently contains more than a minimal upstream PR branch should contain. Before opening an upstream PR, create a clean branch from upstream `master`/`main` and cherry-pick only the code and tests that should be reviewed.

## Recommended Action Items

### 1. Open An Upstream Issue First

Open an issue before submitting the PR. The goal is to get maintainer buy-in on the problem and broad API shape, especially because this adds public API and changes internal capture iteration.

Use this title:

```text
tree-sitter-highlight: support chunked/non-contiguous source providers
```

Use this issue body:

```markdown
## Problem

`tree-sitter-highlight` currently requires callers to pass the full source document as one contiguous `&[u8]`:

```rust
Highlighter::highlight(
    &mut self,
    config: &HighlightConfiguration,
    source: &[u8],
    encoding: Option<u32>,
    cancellation_flag: Option<&AtomicUsize>,
    injection_callback: impl FnMut(&str) -> Option<&HighlightConfiguration>,
)
```

That works well for callers that already have a byte slice, but it is costly for editor-like clients backed by ropes or other non-contiguous text structures. Those clients have to copy/materialize the whole document before highlighting, even though Tree-sitter's parser and query APIs can already consume chunked input.

## Existing Tree-sitter Support

The lower-level APIs already support this kind of input:

- `Parser::parse_with_options` accepts a callback that returns input chunks.
- `QueryCursor::matches` and `QueryCursor::captures` accept a `TextProvider`.

The remaining limitation is inside `tree-sitter-highlight`, where `HighlightIter` and `HighlightIterLayer` store and slice the original `&[u8]` for parser input, query text, injections, locals, source length, and rendering ranges.

## Proposed Direction

Add a source-provider API to `tree-sitter-highlight`, while preserving the existing byte-slice API as a compatibility wrapper.

Sketch:

```rust
pub trait ChunkedSource<'a>: Clone {
    type Chunk: AsRef<[u8]> + 'a;
    type Chunks: Iterator<Item = Self::Chunk> + 'a;

    fn len(&self) -> usize;
    fn is_empty(&self) -> bool;
    fn chunk_at(&mut self, byte_offset: usize, position: Point) -> Self::Chunk;
    fn chunks_for_node(&mut self, node: Node) -> Self::Chunks;
    fn text_for_range(&self, range: Range<usize>) -> Cow<'a, [u8]>;
}

impl Highlighter {
    pub fn highlight_with_source<'a, S>(
        &'a mut self,
        config: &'a HighlightConfiguration,
        source: S,
        encoding: Option<u32>,
        cancellation_flag: Option<&'a AtomicUsize>,
        injection_callback: impl FnMut(&str) -> Option<&'a HighlightConfiguration> + 'a,
    ) -> Result<impl Iterator<Item = Result<HighlightEvent, Error>> + 'a, Error>
    where
        S: ChunkedSource<'a> + 'a;
}
```

The existing `highlight(..., source: &[u8], ...)` method would remain unchanged and delegate internally to the new source-provider path. `ChunkedSource` can be implemented directly for `&[u8]`, `&str`, and `&String`.

## Internal Implementation Tradeoff

The main internal challenge is that `HighlightIterLayer` currently stores a live query-capture iterator. Making that iterator generic over arbitrary source providers leads to difficult lifetimes around the tree, cursor, query iterator, and text provider.

One pragmatic approach is to collect query capture metadata for each layer into an owned internal stream:

- Store pattern index.
- Store captured `Node`s and capture indices.
- Store capture event order.
- Track internal match indices to preserve `QueryMatch::remove()` behavior.

This does **not** store source text. It only stores query/capture metadata. Source text is still read from the provider only when needed for injection language names, local variable names, and query text predicates.

This may allocate more than the current streaming implementation, but it significantly simplifies the source-provider implementation and keeps behavior easier to reason about. If this tradeoff is concerning, I can split the work so the capture-stream refactor is reviewed separately from the public API addition.

## Compatibility

- Existing `Highlighter::highlight(..., &[u8], ...)` API remains unchanged.
- `HighlightEvent` remains unchanged.
- `HtmlRenderer` can remain byte-slice based.
- The C API can remain unchanged.
- Cancellation behavior should be preserved.
- Highlight semantics for injections, combined injections, locals, and layer precedence should remain the same.

## Questions

1. Is a `ChunkedSource` trait in `tree-sitter-highlight` an acceptable API direction?
2. Should the trait try to reuse or mirror `tree_sitter::TextProvider`, or is a highlight-specific source provider preferable because parser input, query text, source length, and range text are all needed?
3. Is an owned capture metadata stream acceptable as a correctness-first implementation, with possible future optimization if allocation overhead matters?
4. Should UTF-16 chunked parsing be included in the first PR, or would UTF-8 chunked input be a better first step?
5. Would maintainers prefer this as one PR or split into smaller PRs?

If this sounds reasonable, I can put together a PR with parity tests showing that byte-slice and chunked-source highlighting produce identical `HighlightEvent` streams across normal highlighting, injections, locals, UTF-16, and cancellation.
```

### 2. Wait For API Feedback Before Opening The PR

Do not immediately open the full PR unless maintainers have already responded positively elsewhere. Wait for feedback on:

- Trait name: `ChunkedSource`, `SourceProvider`, `HighlightSource`, etc.
- Method name: `highlight_with_source`, `highlight_with_provider`, or making `highlight` generic.
- Whether `text_for_range` should return `Cow<'a, [u8]>`, `Vec<u8>`, or chunks.
- Whether `&mut self` on provider methods is acceptable.
- Whether the capture-stream allocation tradeoff is acceptable.

If there is no response after about a week, open a focused PR anyway and reference the issue.

### 3. Prepare A Clean Upstream PR Branch

Create a fresh branch from upstream, not from the planning branch.

Suggested commands:

```sh
git remote add upstream git@github.com:tree-sitter/tree-sitter.git
git fetch upstream
git switch -c chunked-tree-sitter-highlight upstream/master
```

If upstream uses `main` instead of `master`, use `upstream/main`.

Cherry-pick only implementation commits that belong in the upstream PR.

Do **not** include:

- `CHUNKED_HIGHLIGHTING_PR.md`
- `UPSTREAM_MERGE_PLAN.md`
- version bump/reset commits
- fork-only planning commits

Before opening the PR, verify that the branch contains only code/tests/docs that should be accepted upstream.

### 4. Decide Whether To Split The PR

Preferred path if maintainers are receptive: one PR.

Preferred path if maintainers are cautious: split into smaller PRs.

#### Option A: Single PR

Use this if maintainers agree with the issue proposal or explicitly ask to review the full implementation.

Expected contents:

- `ChunkedSource` trait.
- `ChunkedSource` impls for `&[u8]`, `&str`, and `&String`.
- `Highlighter::highlight_with_source`.
- Existing `Highlighter::highlight` preserved as a byte-slice wrapper.
- Owned capture stream refactor.
- UTF-8 and UTF-16 chunked parser input.
- Parity tests.

#### Option B: Split PRs

Use this if maintainers want smaller review units.

PR 1: Capture Stream Refactor

- No public chunked-source API.
- Keep byte-slice API only.
- Replace stored live `QueryCaptures` iterator with owned capture metadata stream.
- Prove existing tests still pass.
- This isolates the most sensitive internal refactor.

PR 2: Chunked Source API

- Add `ChunkedSource`.
- Add `highlight_with_source`.
- Implement `ChunkedSource` for `&[u8]`, `&str`, and `&String`.
- Wire UTF-8 parser and query source access through the provider.
- Add UTF-8 parity tests.

PR 3: UTF-16 And Ergonomics

- Add chunk-aware UTF-16 parser input.
- Add UTF-16 LE/BE parity tests.
- Add optional utility API such as `highlight_capture_names()` if maintainers want it.

### 5. Keep `highlight_capture_names()` Separate Unless Needed

`HighlightConfiguration::highlight_capture_names()` is useful, but it is not essential to chunked highlighting. It may distract reviewers because it is a separate public API addition.

Recommended upstream strategy:

- If using a single PR and maintainers are relaxed, include it with a short explanation.
- If trying to maximize merge odds, remove it from the chunked-source PR and submit it separately as a tiny follow-up.

Separate PR title:

```text
feat(highlight): expose highlight-only capture names
```

Separate PR body:

```markdown
## Summary

Add `HighlightConfiguration::highlight_capture_names()` to expose capture names from only the highlights query, excluding injection and locals captures.

## Motivation

`HighlightConfiguration::names()` returns capture names from the combined injections + locals + highlights query. Consumers that need to build UI/theme metadata from only the actual highlights query currently need to parse the highlights query themselves.

This stores the highlight-only capture names during configuration construction using `Query::new(&language, highlights_query)?.capture_names()` and exposes them as immutable metadata.

## Compatibility

This is additive. Existing `names()` behavior is unchanged.
```

### 6. Add Or Run Benchmarks If Possible

The owned capture stream can allocate more than the current streaming implementation. The most likely reviewer concern will be performance.

Recommended benchmark story:

- Run existing benchmark suite if one exists for highlighting.
- If no highlight benchmark exists, add a small benchmark comparing byte-slice highlighting before and after the refactor on representative JavaScript/Rust/HTML inputs.
- If adding benchmarks is too much for the PR, at least mention the allocation tradeoff and invite follow-up optimization.

Suggested benchmark note for the PR:

```markdown
This PR intentionally prioritizes correctness and API support over eliminating all allocation overhead. The owned capture stream stores only query/capture metadata, not source text. If this shows measurable overhead in real workloads, the capture stream can be optimized in a follow-up without changing the public source-provider API.
```

### 7. Run Verification Before Opening PR

Run these commands from the upstream-clean PR branch:

```sh
cargo +stable fmt --all --check
cargo +stable check -p tree-sitter-highlight
cargo +stable clippy -p tree-sitter-highlight
cargo +stable test -p tree-sitter-highlight
cargo +stable check -p tree-sitter-cli --tests --no-default-features
cargo +stable test -p tree-sitter-cli --no-default-features test_chunked
cargo +stable test -p tree-sitter-cli --no-default-features test_utf16_chunked_source_matches_byte_slice_highlighting
cargo +stable check -p tree-sitter-cli --tests
```

If the CLI highlight fixtures are missing locally, populate them through the Nix fixture bundle or run in the project’s Nix environment.

### 8. Open The Upstream PR

Use this title for a single PR:

```text
feat(highlight): support chunked source providers
```

Use this PR body:

```markdown
## Summary

- Add `Highlighter::highlight_with_source` for highlighting source text provided in chunks.
- Keep the existing `Highlighter::highlight(..., source: &[u8], ...)` API unchanged as a compatibility wrapper.
- Add a `ChunkedSource` trait with implementations for `&[u8]`, `&str`, and `&String`.
- Refactor highlight layers to store owned query capture metadata instead of a live `QueryCaptures` iterator.
- Add parity tests showing chunked-source highlighting matches byte-slice highlighting.

## Motivation

`tree-sitter-highlight` currently requires callers to provide a contiguous `&[u8]`. Rope-backed editors and terminal UI clients need to materialize the whole document before highlighting, even though Tree-sitter's parser and query APIs already support chunked input.

This PR lets those clients keep using `tree-sitter-highlight` for highlight semantics while providing source text from non-contiguous storage.

## API

The existing API remains unchanged:

```rust
Highlighter::highlight(config, source, encoding, cancellation_flag, injection_callback)
```

The new API is opt-in:

```rust
Highlighter::highlight_with_source(config, source, encoding, cancellation_flag, injection_callback)
```

where `source` implements:

```rust
pub trait ChunkedSource<'a>: Clone {
    type Chunk: AsRef<[u8]> + 'a;
    type Chunks: Iterator<Item = Self::Chunk> + 'a;

    fn len(&self) -> usize;
    fn is_empty(&self) -> bool;
    fn chunk_at(&mut self, byte_offset: usize, position: Point) -> Self::Chunk;
    fn chunks_for_node(&mut self, node: Node) -> Self::Chunks;
    fn text_for_range(&self, range: Range<usize>) -> Cow<'a, [u8]>;
}
```

`ChunkedSource` is implemented directly for `&[u8]`, `&str`, and `&String`.

## Internal Refactor

The previous highlighter stored a live `QueryCaptures` iterator inside each layer. Making that generic over arbitrary source providers creates difficult lifetimes around the tree, query cursor, query iterator, and text provider.

This PR instead collects each layer's query capture metadata into an owned internal stream. The stream stores capture metadata and event order, but not source text. Source text is still read from the provider only when needed for injection language names, local names, parser input, or query text predicates.

This trades some allocation for a simpler and safer implementation. The allocation is limited to query/capture metadata and can be optimized later without changing the public API.

## Compatibility

- Existing byte-slice callers continue to use `Highlighter::highlight` unchanged.
- `HighlightEvent` is unchanged.
- `HtmlRenderer` remains byte-slice based.
- The C API is unchanged.
- Cancellation remains supported during parsing, capture collection, and event iteration.

## Tests

Added tests for:

- UTF-8 chunked-source parity against byte-slice highlighting.
- UTF-16 LE/BE chunked-source parity.
- Injection and nested layer behavior.
- Local variable tracking.
- Cancellation.
- `&str` and `&String` source providers.

## Verification

```sh
cargo +stable fmt --all --check
cargo +stable check -p tree-sitter-highlight
cargo +stable clippy -p tree-sitter-highlight
cargo +stable test -p tree-sitter-highlight
cargo +stable check -p tree-sitter-cli --tests --no-default-features
cargo +stable test -p tree-sitter-cli --no-default-features test_chunked
cargo +stable test -p tree-sitter-cli --no-default-features test_utf16_chunked_source_matches_byte_slice_highlighting
cargo +stable check -p tree-sitter-cli --tests
git diff --check
```
```

### 9. Prepare Review Response Templates

Use these if maintainers ask common questions.

#### Why not make `highlight` generic instead of adding `highlight_with_source`?

```markdown
Keeping `highlight` concrete preserves source compatibility for existing callers. If `highlight` becomes generic over `ChunkedSource`, Rust stops applying some existing coercions. For example, callers passing `&Vec<u8>` or byte-string literals may infer `S = &Vec<u8>` or `S = &[u8; N]` rather than coercing to `&[u8]`.

Keeping `highlight(..., &[u8], ...)` unchanged avoids that compatibility risk, while `highlight_with_source` provides the new generic path.
```

#### Why does `ChunkedSource` need `text_for_range` in addition to `chunks_for_node`?

```markdown
Tree-sitter query predicates can use `TextProvider`, which maps naturally to `chunks_for_node`. The highlighter also needs text for byte ranges that are not always represented as query text-provider calls, such as injection language captures and local variable names. `text_for_range` lets providers return borrowed bytes for contiguous sources and owned bytes for non-contiguous sources.
```

#### Why do provider methods take `&mut self`?

```markdown
The parser input callback and Tree-sitter `TextProvider` are already mutable/stateful. `&mut self` lets rope-backed providers keep reusable cursors or small caches without requiring interior mutability. Immutable providers like `&[u8]`, `&str`, and `&String` can still implement the trait without mutating anything.
```

#### Why collect captures into a vector?

```markdown
The existing layer stores a live `QueryCaptures` iterator. Storing that iterator while making the source provider generic creates difficult lifetimes between the tree, query cursor, query iterator, and text provider. The owned stream avoids that lifetime coupling.

The stream stores only metadata: pattern index, captured nodes, capture indices, and capture order. It does not store source text. This is a correctness-first tradeoff that keeps the public API simple and can be optimized later if needed.
```

#### Does this regress byte-slice users?

```markdown
The existing byte-slice API is unchanged and delegates through the same source-provider implementation via `ChunkedSource for &[u8]`. The tests compare byte-slice and chunked-source event streams to ensure behavior stays equivalent.
```

#### Why not reuse `tree_sitter::TextProvider` directly as the public trait?

```markdown
`TextProvider` covers query text for nodes, but the highlighter needs more than that: parser input chunks, source length, and range text for injection names and locals. A highlight-specific source trait can bridge parser input, query input, and range text while still using `TextProvider` internally.
```

## Final Checklist Before Upstream PR

- [ ] Upstream issue opened and linked.
- [ ] Maintainer feedback incorporated or acknowledged.
- [ ] Clean PR branch created from upstream `master`/`main`.
- [ ] Fork-only markdown files removed from PR branch.
- [ ] Version bump/reset commits excluded from PR branch unless requested.
- [ ] Decide whether `highlight_capture_names()` belongs in this PR or a separate PR.
- [ ] Verification commands pass.
- [ ] PR body includes allocation tradeoff and compatibility notes.
- [ ] PR title uses repository style.
- [ ] Be ready to split into capture-stream and chunked-source PRs if requested.
