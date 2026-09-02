# XSS Image Payloads

> Ready-to-use, image-based **Cross-Site Scripting (XSS)** proof-of-concept files for authorized security testing, bug-bounty research, and defensive validation.

![License](https://img.shields.io/badge/license-MIT-blue)
![Payloads](https://img.shields.io/badge/payloads-5-brightgreen)
![PRs](https://img.shields.io/badge/PRs-welcome-orange)
![Use](https://img.shields.io/badge/use-authorized%20testing%20only-critical)

Four distinct techniques for smuggling JavaScript through image-upload and
file-handling features: a scriptable **SVG**, a **GIF/JavaScript polyglot** for CSP
bypass, an **EXIF-metadata** payload, and a **JPEG/HTML polyglot**. Every file is a
real, valid image — the payload stays dormant until the target application mishandles it.

> [!WARNING]
> These are intentional attack payloads. Use them **only** against systems you own or
> are explicitly authorized to test (a signed engagement or an in-scope bug-bounty
> program). Unauthorized use is illegal. See [SECURITY.md](SECURITY.md) for the full
> responsible-use policy.

## Repository layout

```
xss-image-payloads/
├── svg_xss_poc.svg              # 1. SVG onload XSS (full image)
├── svg_xss_poc_minimal.svg      #    same payload, minimal readable version
├── gif_js_polyglot_poc.gif      # 2. GIF + JavaScript polyglot (CSP bypass)
├── exif_xss_poc.jpg             # 3. XSS via EXIF metadata fields
├── jpeg_html_polyglot_poc.jpg   # 4. JPEG + HTML/JS polyglot
├── README.md
├── SECURITY.md
└── LICENSE
```

## Contents

| File | Technique | Executes when… |
|------|-----------|----------------|
| [`svg_xss_poc.svg`](svg_xss_poc.svg) | `onload` handler on the root `<svg>` (full image) | the SVG is served or opened as a document, or embedded same-origin |
| [`svg_xss_poc_minimal.svg`](svg_xss_poc_minimal.svg) | Same technique, minimal readable version | same as above — use this one to read the payload |
| [`gif_js_polyglot_poc.gif`](gif_js_polyglot_poc.gif) | Valid GIF **and** valid JavaScript (polyglot) | the uploaded `.gif` is later loaded via `<script src>` |
| [`exif_xss_poc.jpg`](exif_xss_poc.jpg) | HTML injected into EXIF text fields | an app reflects image metadata into a page without encoding |
| [`jpeg_html_polyglot_poc.jpg`](jpeg_html_polyglot_poc.jpg) | Valid JPEG **and** valid HTML/JS (polyglot) | the file is served or sniffed as `text/html` |

Every payload uses the harmless probe `alert(document.domain)` — it confirms script
execution and prints the origin it ran in, so you can tell it fired on the target
rather than on the file's own domain.

## Techniques

### 1. SVG `onload` XSS — `svg_xss_poc.svg`

SVG is XML, and XML can carry script. An event handler on the root element runs the
moment the document loads:

```xml
<svg xmlns="http://www.w3.org/2000/svg" onload="alert(document.domain)"> ... </svg>
```

It fires when the SVG is the top-level document (opened directly or in a new tab) or is
embedded via `<object>`, `<iframe>`, or `<embed>` **from the target's own origin** — for
example, uploaded as an avatar and then served from the application's domain. If that
origin is the target, the alert shows the target's domain, i.e. **stored XSS**.

> [!NOTE]
> An SVG loaded through an `<img>` tag does not run scripts. The win is when the app
> serves the SVG as a document (wrong `Content-Type`, inline rendering, or a
> "view file" endpoint). `svg_xss_poc_minimal.svg` is the same payload trimmed to a few
> readable lines.

### 2. GIF/JavaScript polyglot (CSP bypass) — `gif_js_polyglot_poc.gif`

One file that is **both a valid GIF image and valid JavaScript**. The GIF signature
`GIF89a` doubles as a JavaScript variable; the two logical-screen width bytes are set to
`/*`, which opens a JS comment that swallows the binary image data, and the file ends by
closing that comment and running the payload:

```
GIF89a/* …binary GIF data… */=alert(document.domain)//
```

To a browser reading it as JavaScript, that is simply `GIF89a = alert(document.domain)`.

It fires when an app accepts a `.gif` upload — it passes image validation because it
genuinely is a valid GIF — and later references it as a script,
`<script src="/uploads/evil.gif">`. Because the script is then served from the site's own
origin, it **bypasses a `script-src 'self'` Content-Security-Policy**, one of the most
useful upload-based bypasses there is.

> [!NOTE]
> Here the picture is cosmetic — the value is a file that is a *valid* GIF (so it clears
> upload filters) **and** *executes as JavaScript*. Verify both:
> `identify gif_js_polyglot_poc.gif` sees an image, and `node --check` sees valid JS.

### 3. EXIF metadata XSS — `exif_xss_poc.jpg`

The image data is clean; the metadata carries the payload. The same HTML sits in four
EXIF text fields (`ImageDescription`, `Software`, `Artist`, `UserComment`):

```html
<img src=x onerror=alert(document.domain)>
```

It fires when an application reads those fields and writes them into a page **without
HTML-encoding** — photo galleries, "image details" panels, CMS media libraries, and
camera-info widgets are common culprits. Inspect the fields yourself:

```bash
exiftool exif_xss_poc.jpg
```

### 4. JPEG/HTML polyglot — `jpeg_html_polyglot_poc.jpg`

One file, two valid formats. It is a fully valid JPEG that also carries HTML/JS inside a
JPEG comment segment:

```html
<script>alert(document.domain)</script>
```

It fires when the file is delivered or content-sniffed as `text/html` — a misconfigured
upload handler, a preview/download endpoint that echoes file bytes with the wrong
`Content-Type`, or legacy MIME sniffing. Browsers scan the whole response for HTML and
run the `<script>`, while image viewers still see a normal picture.

## Reproduction

Clone the repo and serve it locally, then interact with each file the way a target app
would:

```bash
git clone https://github.com/0xyoozy/xss-image-payloads.git
cd xss-image-payloads
python3 -m http.server 8000
```

- **SVG** — open `http://localhost:8000/svg_xss_poc.svg` (served as a document, so `onload` fires).
- **GIF/JS polyglot** — confirm it is both a valid image and valid JavaScript, then load it as a script:

  ```bash
  identify gif_js_polyglot_poc.gif        # valid GIF image
  cp gif_js_polyglot_poc.gif poc.js && node --check poc.js && echo "valid JavaScript"
  # exploit shape on a target that serves uploads back:
  #   <script src="https://target/uploads/gif_js_polyglot_poc.gif"></script>
  ```

- **EXIF** — run `exiftool exif_xss_poc.jpg`, then feed the file to any feature that displays image metadata.
- **JPEG/HTML polyglot** — force it to be parsed as HTML, then open it:

  ```bash
  cp jpeg_html_polyglot_poc.jpg poc.html
  # open http://localhost:8000/poc.html  ->  the <script> executes
  ```

**Where to try them (in scope only):** avatar and profile-picture uploads, file and
attachment uploads, image-proxy and thumbnail services, markdown or rich-text image
embeds, EXIF/metadata viewers, and any preview or download endpoint.

## Remediation

For defenders — how to shut each of these down:

- **Serve uploads from a sandboxed origin** (a separate, cookieless host) so any script
  execution cannot reach the main application's session.
- **Set the correct `Content-Type`** and send `X-Content-Type-Options: nosniff` on all
  user-supplied files — this alone defeats the JPEG/HTML and GIF/JS tricks.
- **For SVG,** either sanitize it (strip `<script>` and every `on*` handler, e.g. with
  DOMPurify's SVG profile) or rasterize it to PNG; serve with
  `Content-Disposition: attachment` where inline rendering is not needed.
- **Never load user uploads as scripts,** and pin a strict `script-src` in your CSP.
- **Strip metadata on upload,** and HTML-encode any metadata you do display.

## Verifying the files

These files carry no tracking, no beacons, and no third-party or generator metadata —
only image data, the documented payloads, and a `YooZy` copyright tag on the JPEGs.
Confirm it yourself:

```bash
grep -aic -E 'c2pa|jumb' *.jpg *.svg *.gif   # expected: 0
```

## References

- OWASP — [Cross-Site Scripting (XSS)](https://owasp.org/www-community/attacks/xss/)
- OWASP — [Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
- PortSwigger Web Security Academy — [Cross-site scripting](https://portswigger.net/web-security/cross-site-scripting)
- corkami — [file-format polyglots](https://github.com/corkami/pocs)

## Contributing

Have another image-XSS technique, a filter bypass, or a cleaner PoC? Open an issue or a
pull request. Keep payloads non-destructive (`alert(document.domain)`-style probes only).

## License

Released under the [MIT License](LICENSE) — © 2026 YooZy.

## Author

**YooZy** — GitHub [@0xyoozy](https://github.com/0xyoozy)
