# `encodeURIComponent` polyfill

The Fuuz V8 sandbox does NOT provide `encodeURIComponent`. It also doesn't provide `btoa` (so base64 isn't a fallback). The only working approach for building `data:text/html` URIs is to implement percent-encoding manually.

## Why this matters

A `data:text/html` URI must be percent-encoded for any character that isn't in the unreserved set (`A-Z`, `a-z`, `0-9`, `-`, `_`, `.`, `~`). HTML contains plenty of those characters — `<`, `>`, `"`, `'`, spaces, equals signs, etc. — plus any non-ASCII content (curly quotes, accented characters, emoji).

A naive replacement like `html.replace(/</g, '%3C')` covers a handful but misses Unicode, surrogate pairs, and dozens of edge-case characters. Browsers will render anyway by best-effort decoding, but the result is unreliable — characters break, JS in the inline HTML fails to parse, the page silently breaks.

## Why not base64

You can't. `btoa` doesn't exist in the V8 sandbox. Even if it did:
- Base64 inflates the URI by ~33%, hurting load time.
- `data:text/html;base64,...` has the same browser size limits and no advantages.

Percent-encoding is the only option, and you write your own.

## The polyfill

This is RFC 3986 compliant: ASCII unreserved characters pass through, all others are emitted as UTF-8 byte sequences in `%XX` form. Surrogate pairs (high+low pairs for codepoints above U+FFFF, like emoji) combine into 4-byte UTF-8.

```javascript
function encodeURIComponentPolyfill(str) {
  // RFC 3986 unreserved: A-Z a-z 0-9 - _ . ~
  // Everything else → UTF-8 bytes → %XX
  const unreserved = {};
  const safe = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_.~";
  for (let i = 0; i < safe.length; i++) unreserved[safe.charCodeAt(i)] = true;

  const out = [];
  const hex = function(n) {
    const h = n.toString(16).toUpperCase();
    return h.length === 1 ? '0' + h : h;
  };

  for (let i = 0; i < str.length; i++) {
    const c = str.charCodeAt(i);

    if (c < 0x80) {
      // ASCII range
      if (unreserved[c]) {
        out.push(str.charAt(i));
      } else {
        out.push('%' + hex(c));
      }
    } else if (c < 0x800) {
      // 2-byte UTF-8
      out.push('%' + hex(0xC0 | (c >> 6))
             + '%' + hex(0x80 | (c & 0x3F)));
    } else if (c >= 0xD800 && c <= 0xDBFF) {
      // High surrogate — combine with the following low surrogate for a 4-byte UTF-8 sequence
      const next = str.charCodeAt(i + 1);
      const cp = 0x10000 + ((c & 0x3FF) << 10) + (next & 0x3FF);
      i++;  // consume the low surrogate
      out.push('%' + hex(0xF0 | (cp >> 18))
             + '%' + hex(0x80 | ((cp >> 12) & 0x3F))
             + '%' + hex(0x80 | ((cp >> 6) & 0x3F))
             + '%' + hex(0x80 | (cp & 0x3F)));
    } else {
      // 3-byte UTF-8 (Basic Multilingual Plane non-ASCII)
      out.push('%' + hex(0xE0 | (c >> 12))
             + '%' + hex(0x80 | ((c >> 6) & 0x3F))
             + '%' + hex(0x80 | (c & 0x3F)));
    }
  }
  return out.join('');
}
```

## Usage

```javascript
const html = TPL_HEAD + JSON.stringify(state) + TPL_TAIL;
const dataUri = 'data:text/html;charset=utf-8,' + encodeURIComponentPolyfill(html);
return { dashboardUri: dataUri };
```

The `charset=utf-8` declaration is important — it tells the browser to interpret the percent-decoded bytes as UTF-8, which is what the polyfill emits.

## Verifying it works

If the dashboard renders garbled text, broken layout, or fails JS parsing:

1. Copy the data URI from the flow output.
2. Paste it into a browser address bar. The dashboard should render exactly as expected.
3. If it renders correctly in the browser but not inside Fuuz, the problem is with the screen's webpage element config (permissions, dynamic fields), not the encoding.
4. If it doesn't render correctly in the browser, the encoding broke something. Check for non-ASCII characters in your HTML — comments, emoji, curly quotes in strings — and confirm the polyfill is being used (not a partial replacement).

## Don't try shortcuts

- ❌ `html.replace(/[<>"']/g, ...)` — misses everything except a handful of common ASCII characters
- ❌ JSON.stringify the HTML — different escaping rules, breaks the URI
- ❌ `btoa(html)` — doesn't exist
- ❌ Loading a base64 library — adds complexity and still uses the wrong encoding
- ✅ The polyfill above — copy it verbatim
