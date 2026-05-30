# Chunked `tree-sitter-highlight` Planning Brief

## Objective

Add chunked/non-contiguous source support to the `tree-sitter-highlight` crate so clients can highlight rope-backed buffers without first materializing the entire source as a contiguous byte slice.

This fork lives at:

- Repository: `https://github.com/mkxl/tree-sitter`
- Local checkout: `/tmp/opencode/tree-sitter-highlight-fork`
- Branch: `michael/chunked-tree-sitter-highlight`
- Target crate: `crates/highlight`

## Motivation

`mkutils` has a rope-backed Ratatui highlighter. It currently reimplements large parts of `tree-sitter-highlight` to avoid the upstream `Highlighter::highlight(..., source: &[u8], ...)` contiguous-source requirement.

Long term, the preferred design is:

- Keep `tree-sitter-highlight` responsible for highlight semantics.
- Add a source-provider API to `tree-sitter-highlight` for chunked text.
- Let `mkutils` provide a `Rope` source provider.
- Let `mkutils` keep only language registration/theme/Ratatui conversion code.

## Current Baseline

The current upstream-ish API is:

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

Internally, the crate already uses APIs that can support chunked text:

- `Parser::parse_with_options` accepts a callback that can return chunks.
- `QueryCursor::matches` and `QueryCursor::captures` accept a `TextProvider`.

The contiguous-source dependency remains because `HighlightIter` and `HighlightIterLayer` store `source: &'a [u8]` and use it for:

- parser input callback: `&source[i..]`
- query text provider: `source`
- injection language capture: `capture.node.utf8_text(source)`
- locals names: `str::from_utf8(&source[range])`
- source length: `source.len()`
- HTML rendering: `&source[start..end]`

## Desired Public API Shape

Preserve the existing byte-slice API.

Add a new API similar to:

```rust
pub trait ChunkedSource<'a>: Clone {
    type Chunk: AsRef<[u8]> + 'a;
    type Chunks: Iterator<Item = Self::Chunk> + 'a;

    fn len(&self) -> usize;
    fn chunk_at(&mut self, byte_offset: usize, position: Point) -> Self::Chunk;
    fn chunks_for_node(&mut self, node: Node) -> Self::Chunks;
    fn text_for_range(&self, range: Range<usize>) -> Vec<u8>;
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

The existing `highlight(..., source: &[u8], ...)` should become a thin wrapper around a byte-slice provider.

## Key Compatibility Requirements

- Existing `Highlighter::highlight` behavior must not regress.
- Existing `HighlightEvent` shape should remain unchanged.
- Existing `HtmlRenderer` can remain byte-slice based initially, but a chunked rendering path may be useful later.
- Existing C API can remain unchanged initially.
- Cancellation behavior must be preserved.
- Highlight semantics for injections, combined injections, locals, and layer precedence should remain the same.

## Known Hard Part

The naive generic implementation hit compiler issues.

Attempted approach:

- Make `HighlightIter<'a, F, S>` generic over a chunked source provider.
- Make `HighlightIterLayer<'a, S>` store:

```rust
iter::Peekable<_QueryCaptures<'a, 'a, ChunkedTextProvider<S>, S::Chunk>>
```

Problems:

- Upstream stores live `QueryCaptures` iterators inside `HighlightIterLayer`.
- It relies on transmuting between `QueryCaptures` and a mirror `_QueryCaptures` type.
- Once `QueryCaptures` is parameterized by arbitrary source/provider types, the existing transmute pattern gets much harder to satisfy safely.
- Borrow lifetimes around `tree`, `cursor`, and `matches/captures` become entangled with the provider lifetime.

The last local WIP did not compile. If you want a clean start, reset the fork branch before implementing:

```bash
git restore crates/highlight/src/highlight.rs
```

Do not push the current WIP unless it compiles.

## Recommended Design Direction

Avoid storing a generic `QueryCaptures` iterator directly in `HighlightIterLayer`.

Instead, first introduce an owned capture stream representation. For example:

```rust
struct CaptureEvent<'tree> {
    match_id: u32,
    pattern_index: usize,
    capture_index: usize,
    capture: QueryCapture<'tree>,
}
```

Potentially collect captures for a layer into a `Vec<CaptureEvent>` or a small custom cursor before storing the layer.

Tradeoff:

- This may allocate more than upstream’s streaming iterator.
- It drastically simplifies source-provider generics/lifetimes.
- It makes chunked support much easier to implement correctly first.

A possible staged plan:

1. Refactor existing byte-slice implementation to collect a layer’s query captures into an owned `Vec` while preserving behavior.
2. Keep tests passing with byte-slice input.
3. Add `ChunkedSource` and `ByteSliceSource`.
4. Replace parser/query source access with provider calls.
5. Add a chunked test provider that deliberately splits text into small pieces.
6. Add tests proving output matches byte-slice `highlight` for the same source.
7. Only after correctness, consider optimizing away allocations.

## Suggested Tests

Add tests in `crates/highlight` that compare old byte-slice highlighting with new chunked-source highlighting.

Suggested fixture languages:

- Rust grammar, for normal highlights and locals.
- Markdown with fenced code block, for injections.
- Markdown inline link injection, for inline injections.

Suggested source fixtures:

```rust
fn f(x: i32) { x; }
```

```markdown
```rust
fn f(x: i32) { x; }
```
```

```markdown
[label](https://example.com)
```

Suggested assertions:

- Byte-slice `highlight` event stream equals chunked-source event stream.
- Repeat with chunk boundaries in awkward locations, e.g. every 1 byte, every 3 bytes, and around newlines.
- Cancellation still returns `Error::Cancelled`.

## Questions For The Planning Agent

1. Should the first implementation prioritize correctness over preserving streaming/no-allocation behavior?
2. Is it acceptable to collect each layer’s captures into a `Vec` internally?
3. Should `ChunkedSource::text_for_range` return `Vec<u8>`, `Cow<[u8]>`, or an iterator over chunks?
4. Should UTF-16 support be chunk-aware now, or should chunked source support initially be UTF-8 only?
5. Should `HtmlRenderer` get chunked rendering support in the same PR, or remain byte-slice only?
6. Should the C API remain unchanged for the first branch?
7. Should the source provider trait live in `tree-sitter-highlight`, or should it reuse/extend `tree_sitter::TextProvider` somehow?
8. How should injection language names and local variable names be represented internally for non-contiguous text: owned `String`, `Vec<u8>`, or borrowed when possible?
9. What tests are required before `mkutils` should depend on this fork?
10. Is this intended as a long-term fork only, or should the design be upstreamable?

## mkutils Integration Target

Once the fork works, `mkutils` should depend on it via git branch, likely:

```toml
tree-sitter-highlight = { git = "https://github.com/mkxl/tree-sitter.git", branch = "michael/chunked-tree-sitter-highlight", package = "tree-sitter-highlight" }
```

Then `mkutils` can remove most custom highlight semantics and keep:

- built-in language registration
- aliases
- theme mapping from capture names to Ratatui `Style`
- conversion of `HighlightEvent` source ranges into `ratatui::Line`s via `Rope`

## Current mkutils Reference

The current working `mkutils` branch has a custom implementation that can serve as behavioral reference:

- Repo: `/tmp/opencode/mkutils`
- Branch: `michael/tree-sitter-ratatui-highlighter`
- Important file: `mkutils/src/tree_sitter_highlighter.rs`

It currently supports:

- rope-backed parser input
- rope-backed query `TextProvider`
- markdown/markdown-inline/Lean/Rust registration
- injections
- combined injections
- locals
- cancellation
- public language registration and aliases
- Ratatui output

Use it as a consumer/reference, not necessarily as the implementation template.
