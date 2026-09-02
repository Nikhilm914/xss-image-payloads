# XSS Image Payloads

> Ready-to-use, image-based **Cross-Site Scripting (XSS)** proof-of-concept files for authorized security testing, bug-bounty research, and defensive validation.

![License](https://img.shields.io/badge/license-MIT-blue)
![Payloads](https://img.shields.io/badge/payloads-4-brightgreen)
![PRs](https://img.shields.io/badge/PRs-welcome-orange)
![Use](https://img.shields.io/badge/use-authorized%20testing%20only-critical)

Three distinct techniques for smuggling JavaScript through image-upload and
file-handling features: a scriptable **SVG**, an **EXIF-metadata** payload, and a
valid **JPEG/HTML polyglot**. Every file is a real, viewable image — the payload
stays dormant until the target application mishandles it.

> [!WARNING]
> These are intentional attack payloads. Use them **only** against systems you own or
> are explicitly authorized to test (a signed engagement or an in-scope bug-bounty
> program). Unauthorized use is illegal. See [SECURITY.md](SECURITY.md) for the full
> responsible-use policy.

## Contents

| File | Technique | Executes when… |
|------|-----------|----------------|
| [`svg_xss_poc.svg`](svg_xss_poc.svg) | `onload` handler on the root `<svg>` (full image) | the SVG is served or opened as a document, or embedded same-origin |
| [`svg_xss_poc_minimal.svg`](svg_xss_poc_minimal.svg) | Same technique, minimal readable version | same as above — use this one to read the payload |
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

### 2. EXIF metadata XSS — `exif_xss_poc.jpg`

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

### 3. JPEG/HTML polyglot — `jpeg_html_polyglot_poc.jpg`

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
- **EXIF** — run `exiftool exif_xss_poc.jpg`, then feed the file to any feature that displays image metadata.
- **Polyglot** — force it to be parsed as HTML, then open it:

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
  user-supplied files.
- **For SVG,** either sanitize it (strip `<script>` and every `on*` handler, e.g. with
  DOMPurify's SVG profile) or rasterize it to PNG; serve with
  `Content-Disposition: attachment` where inline rendering is not needed.
- **Strip metadata on upload,** and HTML-encode any metadata you do display.
- **Apply a strict Content-Security-Policy** to block inline script as defense in depth.

## Verifying the files

These files carry no tracking, no beacons, and no third-party or generator metadata —
only image data, the documented payloads, and a `YooZy` copyright tag on the JPEGs.
Confirm it yourself:

```bash
grep -aic -E 'c2pa|jumb' *.jpg *.svg   # expected: 0
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
