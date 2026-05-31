# TODO

## Chunked Highlighting Follow-Ups

- Add true chunk-aware UTF-16 support for `Highlighter::highlight_with_source`. The initial chunked source path supports UTF-8 without materializing the full source, but UTF-16 LE/BE still materializes internally before parsing.
- Add UTF-16 LE/BE chunked-source parity tests once UTF-16 can read directly from a chunked provider.
- Add `HighlightConfiguration::highlight_capture_names()` to expose capture names from only the highlights query, excluding locals and injections captures. This likely requires either storing the original highlights query string or storing the derived highlight-only capture names when constructing the configuration.
