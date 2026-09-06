# When the page is too big for a data URI

## The real limit, measured

**Chrome refuses to navigate a frame to a `data:` URI of 2,048 KB or more.** Not "around 2 MB" —
2,048 KB, and the failure is silent: the frame simply does not navigate.

Percent-encoding inflates. A page of **1,844 KB of source became a 2,516 KB URI** and was
blocked. Budget on the *encoded* size, never the source size.

## The fix: gzip, and inflate in the page

The browser already has `DecompressionStream`. Compress the HTML in the flow, embed the
compressed bytes, and inflate on load. Measured on the Manifold dashboard:

| | |
|---|--:|
| Source HTML | 1,458 KB |
| Gzipped blob | **528 KB** |
| Resulting URI | **~1.2 MB** — with headroom to grow |

That is a page which is impossible to ship raw, shipping comfortably.

The inflate step is a few lines: decode the embedded string to a `Uint8Array`, pipe it through
`new DecompressionStream('gzip')`, and write the result into the document. No library.

## Order of attack when you are over budget

1. **Compress** — gzip the page as above. Biggest win by far, and it costs nothing at runtime.
2. **Compress the models** — a raw GLB is 150–180× larger than its Draco equivalent. See
   `references/3d-and-webgl.md`.
3. **Minify** — strip whitespace. Backticked template literals preserve every space and newline,
   which matters when a library is hundreds of lines.
4. **Trim the data** — send the fields the page renders, not the whole record.

## Still never use base64

The V8 sandbox has no `btoa`, and base64 inflates ~33% against percent-encoding's smaller
average. Use the percent-encoding polyfill — see `references/encode-uri-polyfill.md`.
