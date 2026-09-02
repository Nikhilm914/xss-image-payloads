# XSS Image Payloads

> Ready-to-use, image-based **Cross-Site Scripting (XSS)** proof-of-concept files for **authorized** security testing, bug-bounty research, and defensive validation.

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Payloads](https://img.shields.io/badge/payloads-4-brightgreen.svg)
![PRs](https://img.shields.io/badge/PRs-welcome-orange.svg)
![Use](https://img.shields.io/badge/use-authorized%20testing%20only-red.svg)

Three battle-ready techniques for slipping JavaScript past image-upload and
file-handling features: a scriptable **SVG**, an **EXIF-metadata** payload, and a
valid **JPEG/HTML polyglot**. Each file is a *real* image that renders normally —
the payload rides along quietly until the target mishandles it.

---

## ⚠️ Authorized testing only

These are **intentional attack payloads**. Use them **only** against systems you own
or are **explicitly authorized** to test (a signed engagement, or an in-scope
bug-bounty program). Unauthorized use is illegal. See **[SECURITY.md](SECURITY.md)**
for the full responsible-use policy. You are responsible for what you do with them.

---

## 📦 What's inside

| File | Technique | Fires when… |
|------|-----------|-------------|
| [`svg_xss_poc.svg`](svg_xss_poc.svg) | `onload` handler on the root `<svg>` (full image) | the SVG is served/opened as a document or embedded same-origin |
| [`svg_xss_poc_minimal.svg`](svg_xss_poc_minimal.svg) | Same technique, 3-line readable version | — same as above; use this to *read* the payload |
| [`exif_xss_poc.jpg`](exif_xss_poc.jpg) | HTML injected into **EXIF** text fields | an app reflects image metadata into a page without encoding |
| [`jpeg_html_polyglot_poc.jpg`](jpeg_html_polyglot_poc.jpg) | Valid JPEG **+** valid HTML/JS (polyglot) | the file is served or sniffed as `text/html` |

All payloads use the harmless, non-destructive probe **`alert(document.domain)`** —
it proves script execution *and* prints the origin it ran in (so you know it fired on
the target, not on the file's own domain).

---

## 🧬 How each one works

### 1 — SVG `onload` XSS  ·  `svg_xss_poc.svg`

SVG is XML, and XML can carry script. An event handler on the root element runs as
soon as the document loads:

```xml
<svg xmlns="http://www.w3.org/2000/svg" onload="alert(document.domain)"> ... </svg>
```

**Triggers when** the SVG is the top-level document (opened directly / in a new tab)
or embedded via `<object>`, `<iframe>`, or `<embed>` **from the target's own origin** —
e.g. uploaded as an avatar and then served from the app's domain. If that origin is the
target, the alert shows the *target's* domain → **stored XSS**.

> **Note:** an SVG loaded through an `<img>` tag does **not** run scripts. The win is
> when the app serves the SVG as a document (wrong `Content-Type`, inline rendering,
> or a "view file" endpoint). `svg_xss_poc_minimal.svg` is the same payload trimmed to
> a few readable lines.

### 2 — EXIF metadata XSS  ·  `exif_xss_poc.jpg`

The image is clean; the **metadata** carries the payload. The same HTML sits in four
EXIF text fields (`ImageDescription`, `Software`, `Artist`, `UserComment`):

```
<img src=x onerror=alert(document.domain)>
```

**Triggers when** an application reads those fields and prints them into a page
**without HTML-encoding** — photo galleries, "image details/EXIF viewer" panels,
CMS media libraries, camera-info widgets. Inspect it yourself:

```bash
exiftool exif_xss_poc.jpg
```

### 3 — JPEG / HTML polyglot  ·  `jpeg_html_polyglot_poc.jpg`

One file, two valid formats. It's a fully valid JPEG that also carries HTML/JS inside
a JPEG **comment segment**:

```
<script>alert(document.domain)</script>
```

**Triggers when** the file is delivered or content-sniffed as `text/html` — a
misconfigured upload handler, a "download/preview" endpoint that echoes file bytes with
the wrong `Content-Type`, or legacy MIME-sniffing. Browsers scan the whole response for
HTML and execute the `<script>`, while image viewers still see a normal picture.

---

## 🧪 Quick reproduction

Spin up a local server and interact with the files the way a target app would:

```bash
git clone https://github.com/0xyoozy/xss-image-payloads.git
cd xss-image-payloads
python3 -m http.server 8000
```

- **SVG** — browse to `http://localhost:8000/svg_xss_poc.svg` (served as a document → alert).
- **EXIF** — feed `exif_xss_poc.jpg` to any feature that displays image metadata, or run `exiftool` on it.
- **Polyglot** — request it with an HTML content type, e.g.:

  ```bash
  # serve the polyglot as text/html to demonstrate the HTML side
  python3 -c "import http.server,socketserver as s; \
h=http.server.SimpleHTTPRequestHandler; h.extensions_map['.jpg']='text/html'; \
s.TCPServer(('',8001),h).serve_forever()"
  # then open http://localhost:8001/jpeg_html_polyglot_poc.jpg
  ```

**Where to try them in the wild (in-scope only):** avatar/profile-picture uploads,
file/attachment uploads, image-proxy and thumbnail services, markdown/rich-text image
embeds, EXIF/metadata viewers, and any "preview/download" endpoint.

---

## 🛡️ Remediation (for defenders)

If one of these fires against your app, here's how to shut it down:

- **Serve uploads from a sandboxed origin** (a separate domain/cookieless host) so script
  execution can't touch the main app's session.
- **Force the right `Content-Type`** and send **`X-Content-Type-Options: nosniff`** on all
  user-supplied files.
- **For SVG:** either **sanitize** it (strip `<script>` and all `on*` handlers — e.g.
  DOMPurify with the SVG profile) or **rasterize** it to PNG; and serve with
  `Content-Disposition: attachment` where inline rendering isn't needed.
- **Strip metadata** on upload, and **HTML-encode** any metadata you *do* display.
- **Deploy a strict Content-Security-Policy** to block inline script as defense-in-depth.

---

## ✅ Provenance

These files carry **no tracking, no beacons, and no third-party metadata** — just the
image data, the documented payloads, and (on the JPEG) a `YooZy` copyright tag. Verify:

```bash
grep -aic -E 'c2pa|jumb' *.jpg *.svg   # -> 0
```

---

## 📚 References

- OWASP — [Cross Site Scripting (XSS)](https://owasp.org/www-community/attacks/xss/)
- OWASP — [Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
- PortSwigger Web Security Academy — [Cross-site scripting](https://portswigger.net/web-security/cross-site-scripting)
- Ange Albertini — [corkami: file-format polyglots](https://github.com/corkami/pocs)

---

## 🤝 Contributing

Got another image-XSS trick, a filter bypass, or a cleaner PoC? Open an issue or a pull
request. Keep payloads **non-destructive** (`alert(document.domain)`-style probes only).

## 📄 License

Released under the **[MIT License](LICENSE)** — © 2026 YooZy.

## 👤 Author

**YooZy** · GitHub [@0xyoozy](https://github.com/0xyoozy)
