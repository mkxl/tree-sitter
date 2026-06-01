# Chunked Highlighting PR Description

## Summary

- Add `Highlighter::highlight_with_source` for highlighting non-contiguous source buffers without requiring callers to materialize a single `&[u8]`.
- Preserve the existing `Highlighter::highlight` byte-slice API as a wrapper around the new source-provider path.
- Refactor highlight iteration to collect query capture metadata into an owned internal stream, which avoids storing source-provider-parametrized `QueryCaptures` iterators across layer lifetimes.
- Add chunked-source parity tests for UTF-8 and UTF-16 input, injection handling, locals, cancellation, and highlight-only capture-name reporting.

## Motivation

`tree-sitter-highlight` previously required the entire source document as one contiguous byte slice. That is inconvenient for editor and terminal UI clients backed by ropes or other chunked text structures, because they must copy the entire buffer before highlighting.

Tree-sitter's parser and query APIs already support chunked input:

- `Parser::parse_with_options` accepts a callback that returns source chunks.
- `QueryCursor::matches` and `QueryCursor::captures` accept a `TextProvider`.

This change lets `tree-sitter-highlight` use those capabilities directly, so clients can keep relying on the crate's existing highlight semantics while providing source text from non-contiguous storage.

## Public API Changes

The existing byte-slice API remains available:

```rust
Highlighter::highlight(config, source, encoding, cancellation_flag, injection_callback)
```

A new source-provider API is added:

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

```rust
Highlighter::highlight_with_source(config, source, encoding, cancellation_flag, injection_callback)
```

`ChunkedSource` is implemented directly for `&[u8]`, `&str`, and `&String`, and `highlight` delegates to `highlight_with_source` through the byte-slice implementation.

`HighlightConfiguration::highlight_capture_names()` is also added to expose capture names from only the highlights query, excluding locals and injections captures.

## Implementation Details

The main internal change is replacing the stored live `QueryCaptures` iterator with an owned capture stream. Each layer now stores query capture metadata instead of a query cursor iterator borrowed from the tree, cursor, and source provider.

The owned stream stores:

- Pattern index.
- Captured `Node`s and capture indices.
- Capture event order.
- Internal match indices used to preserve `QueryMatch::remove()` behavior.

It does not store source substrings. Source text is still read from the provider only where semantics require it, such as:

- Injection language capture text.
- Local definition and reference names.
- Query text predicates through Tree-sitter's `TextProvider` API.

This keeps the first implementation straightforward and avoids the lifetime complexity of storing a generic `QueryCaptures` iterator inside each highlight layer.

## UTF-16 Support

The chunked source path supports UTF-8 parsing through `ChunkedSource::chunk_at`.

UTF-16 LE and UTF-16 BE parsing now read bounded chunks from the source provider and decode those chunks to `Vec<u16>` for Tree-sitter's UTF-16 parser callbacks. This avoids materializing the whole source document before parsing while preserving the existing `encoding` behavior.

## Compatibility

- Existing `Highlighter::highlight` behavior is preserved.
- `HighlightEvent` shape is unchanged.
- `HtmlRenderer` remains byte-slice based.
- The C API remains unchanged.
- Cancellation is checked during parsing, eager capture collection, and event iteration.
- Injection handling, combined injections, locals, and layer precedence continue to use the existing semantics.

## Testing

Added tests compare byte-slice highlighting against chunked-source highlighting using deliberately awkward chunk sizes.

Coverage includes:

- UTF-8 chunked-source parity.
- UTF-16 LE and UTF-16 BE chunked-source parity.
- Injections and nested layers.
- Local variable tracking.
- Cancellation.
- Highlight-only capture names excluding injections and locals.

Verification run locally:

```sh
cargo +stable fmt --all --check
cargo +stable check -p tree-sitter-highlight
cargo +stable clippy -p tree-sitter-highlight
cargo +stable test -p tree-sitter-highlight
cargo +stable check -p tree-sitter-cli --tests --no-default-features
cargo +stable test -p tree-sitter-cli --no-default-features test_chunked
cargo +stable test -p tree-sitter-cli --no-default-features test_utf16_chunked_source_matches_byte_slice_highlighting
cargo +stable test -p tree-sitter-cli --no-default-features test_highlight_capture_names
cargo +stable check -p tree-sitter-cli --tests
git diff --check
```

## Tradeoffs

Collecting captures into an owned stream can allocate more than the previous streaming approach. The benefit is a much simpler and safer lifetime model for arbitrary source providers. The implementation keeps source text out of the capture stream, so the additional storage is limited to query/capture metadata.

Future optimizations can revisit streaming capture iteration if allocation overhead becomes a practical issue.
