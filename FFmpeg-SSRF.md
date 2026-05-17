# Inner-Parser SSRF

## The bug class your URL validator was never going to catch

If you've been doing SSRF long enough, you've stopped writing exploits and started writing checklists. Block IP literals in every base. Block internal hostnames. Re-validate after redirects. Check decimal-encoded. Check IPv6 dual-stack. Check the `0177.0.0.1` thing. Check `[::ffff:127.0.0.1]`. Check DNS rebinding windows.

That checklist works on the SSRF where there's **one** fetcher.

This is about the SSRF where there's two — where every item on that checklist gets implemented correctly, the team is genuinely proud of their validator, and the bug ships anyway because **a second piece of code, sitting one layer deeper, also knows how to make outbound HTTP requests, and nobody told it about the rules.**

**Two parts of a system disagreeing about whose job it was to enforce the boundary.**

Inner-parser SSRF is that family showing up in outbound network requests. Once you see the shape, you find it everywhere.

---

## The shape

A service takes a URL from you. The validator on that URL is good. Excellent, even. Scheme allowlist, IP-literal block, hostname block, redirect re-validation. Everything you'd want.

The validator's job ends the moment it hands the bytes off to whatever processes the file.

After that point, **some other piece of code is parsing what's inside those bytes.** And that other piece of code has its own opinions about what URLs mean. Almost every rich file format embeds references — to other files, to remote resources, to includes, to external entities, to fragments, to fonts, to thumbnails. The library that parses the format is, by design, willing to follow those references.

The bug is that **the inner library was written assuming the bytes it receives are trusted.** Because they came through the validator. Because someone else's code put them there. Because nobody on that team was thinking of the media decoder, or the image library, or the office-doc parser, as a network client.

But it is one. It always was. The validator just couldn't see it.

---

## The proof

Here's the version of this I just shipped on a target whose name doesn't matter. The endpoint took an audio file for processing. Three schemes accepted on the URL. IP literals blocked in every encoding I tried. Internal hostnames blocked. Redirects re-validated.

I stopped attacking the URL.

The platform let me upload files to its own storage and then reference them by an internal URI. So I uploaded a file. The file extension said `.mp3`. The MIME type said `audio/mpeg`. The validator was happy on every checkable property.

The actual bytes were this:

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:1
#EXTINF:1.0,
https://my-inspector.example/probe-A.ts
#EXTINF:1.0,
https://my-inspector.example/probe-B.ts
#EXT-X-ENDLIST
```

That's a streaming playlist — the format used for HLS video. When a media library sees `#EXTM3U` as the first line of a file, it stops thinking of the file as data and starts thinking of it as **a list of things to download**. That's what playlists are for.

The validator never read the body. The decoder did. Within a few seconds my inspector started lighting up — two GET requests, from a cloud IP that was visibly not the same fleet as the platform's normal outbound traffic. Different cloud provider entirely. Different `User-Agent`: the unmistakable `Lavf/...` signature of FFmpeg's internal HTTP fetcher.

Two fetchers. One trust boundary. Zero coordination.

| What was protected | By what | Held? |
|---|---|---|
| The submitted URL | Outer validator (scheme, IP, hostname, redirects) | ✅ |
| The URLs *inside* the submitted file | (nothing) | ❌ |

That's the whole bug. There was exactly one layer of defense, and it was looking at the wrong thing.

---

## Where to hunt

This is the part where you go run.

Anywhere a service does `fetch(user_url) → hand bytes to library`, the library is a candidate. The question is whether the library follows URLs in the content. Many do. Most teams don't realize it.

**Media:**
- FFmpeg / libavformat (HLS playlists, DASH manifests, concat protocol, subfile protocol, RTMP/RTSP, anything you let through `-protocol_whitelist`)
- GStreamer (playlist elements, manifest demuxers)
- video thumbnail generators that wrap one of the above
- audio fingerprinters and transcoders — same underlying libraries

**Images and graphics:**
- ImageMagick / GraphicsMagick (`MVG`, `SVG`, `MSL`, the ghosts of Imagetragick — still a class, not just a CVE)
- librsvg and SVG-supporting image libs (`<image href="...">`, `<use href="...">`)
- icon and favicon generators
- AI image services that pre-process or transcode uploads

**Documents:**
- Office parsers that follow external relationships, linked images, OLE references (`.docx`, `.xlsx`, `.pptx` are zip files full of XML that can reference outside resources)
- PDF processors that resolve embedded URIs, remote fonts, XFA references, GoToR actions
- HTML-to-PDF converters (`wkhtmltopdf`, headless Chrome wrappers, Puppeteer/Playwright endpoints) — these are *literally browsers* sitting behind your URL validator
- Markdown renderers that fetch images server-side for preview/OG cards

**Data and structured formats:**
- XML parsers with external entities enabled (XXE is inner-parser SSRF wearing a different hat)
- XSLT processors with `document()` enabled
- some YAML libraries with custom tag resolvers
- LDAP referral chasing

**Web stack:**
- link-preview and OG-card services
- webhook receivers that fetch linked assets out of payloads
- AI/LLM "browse this URL" tools wrapped behind a "safe" frontend validator
- import-from-URL features on every CMS, project manager, design tool, and dashboard tool you've ever signed up for

In every one of these, the team's mental model is that the *outer* URL is the attack surface. The inner library is "just processing the file." It is not just processing the file. It is, in a real and load-bearing sense, a second HTTP client running on your infrastructure with no allowlist.

---

## How to test

Three moves, in order.

**One: confirm the inner fetcher exists.** Upload a file in the expected format whose body contains a reference to an inspector you control — a remote `<image>` in an SVG, a remote font in a PDF, an `EXT-X-STREAM-INF` line in an m3u8, a `<v:imagedata r:href>` in a DOCX, an XML external entity (try parameter entities + OOB DTD if direct doesn't work), an `<img>` in HTML being rendered to PDF. Submit the file through the normal endpoint. Watch the inspector. If anything fires, you have an inner fetcher and you now know its egress identity.

**Two: confirm the inner fetcher does not share the outer fetcher's policy.** Re-aim the inner reference at the things the outer validator was blocking. Cloud metadata endpoints (`169.254.169.254`, `metadata.google.internal`, IMDSv2 if you can talk to it, the Azure `169.254.169.254/metadata/instance` path with the required header). Internal-looking hostnames. Localhost in every encoding. If the outer validator wouldn't let you talk to these from the URL field and the inner one will, you have crossed a boundary the team didn't know existed.

**Three: confirm scope.** What protocols does the inner library understand? FFmpeg, in particular, is a protocol monster — it speaks `file://`, `concat:`, `subfile:`, `crypto:`, `rtp://`, `srtp://`, `pipe:`, `unix://`, and more, unless an explicit `-protocol_whitelist` has been set. Image libraries vary. Office parsers vary. Each enabled protocol is a separate question about what the bug actually gets you — read local files, exfil via DNS, hit non-HTTP services, pivot into something stranger.

If you get to step three and the answers are interesting, you've got a real report.

---

## What the fix actually has to do

Most teams' first instinct is to add the same outer-validator logic to the inner fetcher. That's correct, but it's not enough.

The deeper fix is to **stop treating the inner library as a passive consumer of trusted bytes.** Once you've named it a network client, you can give it a network client's restrictions: an explicit protocol allowlist; an egress proxy with the same IP/hostname policy as the outer fetcher; a refusal to process content whose magic bytes don't match the declared format; in some cases, disabling the relevant feature in the library at build time (FFmpeg supports `--disable-demuxer=hls`, ImageMagick has policy.xml, libxml2 has `XML_PARSE_NONET`).

The architectural fix is to align egress policy across **every** subsystem on the path. If the outer fetcher runs in one cloud with one policy and the decoder fleet runs in a different cloud with a different policy, you didn't harden a system. You hardened a fence.

---

## Final thesis

SSRF is taught as a URL-validation problem. That framing holds for the simplest version. It stops holding the moment more than one component in your system is willing to make outbound requests.

The question that does hold up, the one I keep coming back to in every writeup in this repo, is some variant of this:

**Which components in this system are doing the security-critical thing, and which of them are quietly trusting somebody else to have done it for them?**

In [BFCache](./BFCache-for-bug-hunters.md), it was navigation. In [Cookies and CORS](./Cookies-and-CORS.md), it was browser primitives. In [JWTs](./JWTs-deep-dive.md), it was verification policy. In [SMTPUTF8](<./Rfc-6531(SMTPUTF8).md>), it was identity equivalence. Here, it's outbound network requests.

If a system's answer is "the front layer checks, and the rest of them assume," the bug is already there. You just have to find the inner parser that's willing to dial out for you.

In my case it was a media library reading a playlist.

It usually is something like that.
