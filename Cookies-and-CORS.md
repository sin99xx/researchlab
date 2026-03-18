# Cookies and CORS for People Looking for Bugs

## Not how they work. Where they fail.

Most teams do not have a cookie model or a CORS model. They have folklore.

That is why these bugs survive. Someone knows that `HttpOnly` is good. Someone else knows that `Access-Control-Allow-Origin` has to be “set correctly.” But very few systems are built from the actual browser trust boundary outward. They are built from framework defaults inward. That is exactly why cookie and CORS bugs keep showing up in mature applications: not because the primitives are mysterious, but because the implementation assumptions are usually lazier than the browser. Cookies are still fundamentally specified by RFC 6265, with `rfc6265bis` actively updating the model, while modern CORS behavior is defined by the Fetch Standard rather than some standalone “CORS RFC.” ([IETF Datatracker][1])

So the right question is not “what is a cookie?” or “what is CORS?” The right question is: **where do engineers assume the browser is giving them a guarantee that the specs never actually gave them?**

That is where the findings are.

## 1. Cookie scope collapse: the quiet sibling-subdomain bug

One of the best cookie findings is still the least glamorous: a system treats multiple subdomains as though they are cleanly separated, while the cookie model does not. RFC 6265 allows a server to widen a cookie with the `Domain` attribute, and that means a weaker or less trusted sibling subdomain can sometimes interfere with cookies used by a stronger one. The spec is quite candid about cookie security defects: cookies have “security and privacy infelicities,” and the model does not magically give you integrity across subdomains. ([IETF Datatracker][1])

The bug-hunter version of this is simple: if a company has `app.example.com`, `admin.example.com`, `help.example.com`, `old.example.com`, and some half-abandoned marketing surface on a sibling host, ask whether any of them can set cookies scoped broadly enough to collide with the main session space. If the session cookie is not host-bound and the estate is large, there is often something to do there. Current browser guidance strongly pushes `__Host-` for sensitive host-only cookies precisely because it prevents `Domain` scoping and requires `Path=/` plus `Secure`. If the crown-jewel session cookie is still just a generic broad-scope cookie, that is not modern defensive posture. ([MDN Web Docs][2])

The attack idea is not “steal the cookie.” It is often **confuse the server about which cookie value it sees first**, or overwrite a weaker cookie that later participates in auth, CSRF, feature-flag, or tenant selection logic.

## 2. Path confusion: developers treat `Path` like isolation, but the RFC never did

This one is still criminally underappreciated. Teams mentally partition cookies by URL space: `/admin` cookies belong to admin, `/app` cookies belong to app, `/api` cookies belong to API. RFC 6265 does not bless that model. It says the `Path` attribute does **not** provide integrity protection. Browsers will accept arbitrary path attributes in `Set-Cookie`, which means `Path` is a delivery filter, not a security boundary. ([IETF Datatracker][1])

That creates a very specific review target: any application that assumes path scoping is enough to prevent one part of the site from interfering with another. If there is path-based multitenancy, path-based admin separation, or path-based auth state partitioning, look harder. The vulnerability is usually not “cookie theft.” It is cookie shadowing, unexpected precedence, or the server reading the wrong value under ambiguous state.

If a team says “it’s safe because that cookie is only valid under `/admin`,” that is not reassuring. That is an invitation.

## 3. `Secure` is not integrity, and the spec says so out loud

A lot of people talk about `Secure` as though it means “safe cookie.” The RFC is less sentimental. It explicitly says the `Secure` attribute protects confidentiality of transmission but does **not** provide integrity in the presence of an active network attacker. In other words, `Secure` is necessary, but it was never enough to carry the whole design. ([IETF Datatracker][1])

Why does that matter in practice? Because teams often stop their analysis there. “It’s `Secure`, so we’re done.” No. The real question is whether the session design assumes the cookie cannot be replaced, confused, downgraded, or raced by other state mechanisms. If the broader architecture treats `Secure` as the only hardening measure, you may have a design that looks modern in headers and old in threat model.

This is one reason the browser guidance around `__Host-` and `__Secure-` prefixes matters more than many teams realize. The prefix is not cosmetic naming. It is a browser-checkable constraint on how the cookie is allowed to be set. ([MDN Web Docs][2])

## 4. SameSite is where auth flows and security reality start diverging

The modern cookie story is not just host, path, and lifetime anymore. It is deeply tied to **same-site versus cross-site** request context. The active `rfc6265bis` work bakes that modern behavior into the standard, and browser documentation now treats `SameSite` as fundamental, not optional polish. If a cookie is intended to participate in cross-site contexts, the modern rule is `SameSite=None; Secure`. If it is not, the browser may suppress it in exactly the places where legacy systems still expect it to appear. ([IETF Datatracker][3])

This creates two very good classes of bugs.

The first is the straightforward one: a team bolted on `SameSite=None` to “make auth work” and quietly reopened cross-site request surface they no longer understand. That is worth testing for CSRF, login CSRF, federated flow confusion, and endpoint assumptions that only held while the cookie was implicitly harder to send.

The second is subtler and often better: the cookie no longer rides in the intended cross-site flow, so the application invents a fallback. It passes tokens in query strings, fragments, JSON bodies, local storage, or custom headers because “cookies were unreliable.” Those fallback paths are often weaker than the cookie model they replaced. That is a real 2026 bug-hunting instinct: do not just test the cookie policy. Test the **bad compensating architecture** that got built after the cookie policy surprised the team.

## 5. Partitioned cookies are new enough that people are still reasoning about them badly

Partitioned cookies—CHIPS—are now part of the modern browser story. MDN currently describes them as baseline-available in 2025, with a separate cookie jar per top-level site for opted-in cookies. The point is to preserve legitimate embedded state without reviving the old universal third-party cookie model. ([MDN Web Docs][4])

That means there is a fresh class of implementation mistakes: teams who believe partitioned cookies behave like old third-party cookies, and teams who do not realize their embedded auth assumptions are now top-level-site-dependent. Good places to look are SSO widgets, payment embeds, support widgets, account switchers, and SaaS control panels embedded cross-site.

The finding is rarely “the browser is wrong.” The finding is usually that the application’s state machine still assumes globally shared third-party state, while the browser has moved to partitioned state. That mismatch can produce auth confusion, session fixation-like behavior inside embeds, broken logout, or tenant mix-ups.

## 6. CORS is not auth. It is response-read permission for browser JavaScript

If you want one sentence that kills half the nonsense around CORS, it is this:

**CORS does not decide whether the browser can send the request. It decides whether browser JavaScript can read the response.**

That is the modern model in the Fetch Standard and current MDN guidance. It is why so many teams misdiagnose CORS bugs. They think “request blocked” means “request never happened.” In reality, the network interaction may have happened; the browser may simply refuse to expose the result to the calling origin because the response did not satisfy the CORS protocol. ([Fetch Standard][5])

That distinction produces two different bug classes.

One is the obvious data-exposure class: a response becomes readable cross-origin when it should not.

The other is the state-change class: teams assume CORS protects side-effecting endpoints because “other sites can’t call them,” when in reality CORS is not a CSRF defense and does not stop the browser from issuing many kinds of cross-origin traffic. The bug is the mistaken belief that a read-control protocol is a send-control protocol.

If you see a team leaning on CORS as though it were auth, that is not a small misunderstanding. That is usually an architectural opening.

## 7. Reflected Origin with credentials: still good, but only if the allowlist is actually broken

Credentialed CORS remains one of the most productive review areas. Modern guidance is clear: if the request carries credentials, the server must return a specific allowed origin, not `*`, and `Access-Control-Allow-Credentials: true` must be present for the browser to expose the response to frontend code. MDN also notes that `Set-Cookie` will not take effect on a credentialed cross-origin response if ACAO is `*` instead of an actual origin. ([MDN Web Docs][6])

So the real bug is not “I saw `Access-Control-Allow-Credentials: true`.” The real bug is **how the origin gets approved**.

Look for:

* suffix matching instead of exact origin matching,
* substring checks,
* regex mistakes,
* protocol confusion,
* acceptance of `null`,
* and trust in attacker-controlled subdomains.

That is where the serious CORS findings come from. The header reflection is just the visible symptom. The underlying bug is origin parser weakness.

## 8. `null` origin and weird-origin trust is a better test than most scanners do

A lot of origin validation code is written by people who think an origin is “basically a hostname.” It is not. Scheme and port are part of origin. And there are edge contexts that produce `Origin: null`, which is why treating `null` as harmless or equivalent to “no origin” is still dangerous. PortSwigger’s current CORS material still calls out `null` as a meaningful ACAO value with real exploitation implications when trusted carelessly. ([PortSwigger][7])

This is the kind of issue that separates actual review from checkbox scanning. Do not just test your normal attacker origin. Test weird-but-valid origins, sandboxed/null-like contexts where applicable, scheme changes, localhost variants in dev-heavy systems, and mixed-port setups. Most broken allowlists are not broken for the clean demo case. They are broken at the parsing margins.

## 9. Missing `Vary: Origin` is a cache bug masquerading as a CORS bug

This one is genuinely underrated. If a server dynamically returns `Access-Control-Allow-Origin` based on the incoming `Origin` but forgets `Vary: Origin`, then an HTTP cache can serve a response tailored for one origin to a different origin. MDN’s CORS guidance explicitly shows `Vary: Origin` alongside explicit ACAO behavior because without it, caches do not know that the response depends on Origin. ([MDN Web Docs][8])

People often describe this as “just a reliability issue.” It can be, but that framing is too soft. Any time authorization-like response exposure is origin-dependent and caching does not know it, you should at least think in terms of cache confusion and policy cross-talk. Sometimes the result is only broken frontend behavior. Sometimes the result is cross-tenant or cross-origin exposure in front of shared infrastructure.

The important point is that CORS policy is not just headers. It is headers plus caches plus how responses vary.

## 10. `Set-Cookie` being invisible to JS creates a very specific class of bad workarounds

Modern browsers do not expose `Set-Cookie` to frontend JavaScript. The Fetch model treats it as a forbidden response-header name, and MDN is explicit that browsers filter it out from responses exposed to frontend code. ([MDN Web Docs][9])

On its face, that sounds defensive. It is. But it also creates a recurring engineering anti-pattern: teams want the frontend to “know the session was set,” discover they cannot read `Set-Cookie`, and then add another signal. They return session tokens in JSON, echo back auth state in custom exposed headers, or create a “login successful with token” response purely so the SPA can observe something it was never supposed to need directly.

That is a beautiful place to hunt. The spec-compliant cookie path is often fine. The **extra observability path** added for frontend convenience is often where the leak is.

## 11. Preflight mismatches are often boring until they are not

CORS preflight is usually treated as framework furniture. The browser sends an OPTIONS request describing the method and headers it wants to use; the server answers with what it will allow. That basic model is in modern CORS guidance and the Fetch standard. ([MDN Web Docs][10])

The good bugs show up when the application’s actual enforcement and the preflight policy drift apart.

If the preflight says a method or header is fine, but the actual route logic enforces something weaker, stronger, or just different, you get inconsistent state. Sometimes that only creates operational weirdness. Sometimes it reveals an internal API shape or enables request forms the team never meant to be callable cross-origin.

The review mindset here is not “does OPTIONS return 200.” It is “does the browser-facing permission model match the application-facing enforcement model on every path that matters.”

## 12. The best cookie/CORS bugs are usually trust-boundary bugs, not header bugs

That is the larger lesson tying all of this together.

The highest-signal findings are rarely “missing Secure” or “forgot ACAO.” Those are surface indicators. The real bugs are:

* trusting sibling subdomains too much,
* treating `Path` as isolation,
* treating `Secure` as integrity,
* using `SameSite=None` without understanding the resulting cross-site state model,
* reasoning about partitioned cookies with an outdated third-party cookie mindset,
* treating CORS like auth,
* validating origins loosely,
* forgetting that caches participate in CORS,
* and building unsafe fallback mechanisms because the real browser model was inconvenient.

That is where the elite stuff lives. Not in memorizing headers, but in asking one unfriendly question:

**What guarantee does this team think the browser is giving them that the spec never actually promised?**

That is the place to start.

[1]: https://datatracker.ietf.org/doc/html/rfc6265 "RFC 6265 - HTTP State Management Mechanism - IETF Datatracker"
[2]: https://developer.mozilla.org/en-US/docs/Web/Security/Practical_implementation_guides/Cookies "Secure cookie configuration - Security - MDN"
[3]: https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-rfc6265bis "draft-ietf-httpbis-rfc6265bis-22"
[4]: https://developer.mozilla.org/en-US/docs/Web/Privacy/Guides/Privacy_sandbox/Partitioned_cookies "Cookies Having Independent Partitioned State (CHIPS) - MDN Web Docs"
[5]: https://fetch.spec.whatwg.org/ "Fetch Standard"
[6]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Allow-Credentials "Access-Control-Allow-Credentials header - HTTP | MDN"
[7]: https://portswigger.net/web-security/cors/access-control-allow-origin "CORS and the Access-Control-Allow-Origin response header"
[8]: https://developer.mozilla.org/en-US/docs/Web/Security/Practical_implementation_guides/CORS "Cross-Origin Resource Sharing (CORS) configuration - MDN"
[9]: https://developer.mozilla.org/en-US/docs/Web/API/Headers/getSetCookie "Headers: getSetCookie () method - Web APIs | MDN"
[10]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS "Cross-Origin Resource Sharing (CORS) - HTTP | MDN"
