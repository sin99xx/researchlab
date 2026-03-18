# BFCache for Bug Hunters

## The browser feature that breaks the oldest assumption in web exploitation: navigation does not always kill the page

This note stands on top of other people’s work. If you care about BFCache seriously, read **Philip Walton** and **Barry Pollard** on web.dev, read the **W3C TAG BFCache guidance**, and read **Jorian Woltjer’s** writeup on the CSP nonce bypass chain that uses BFCache plus disk cache behavior; his writeup also credits **@arkark** for an important technique in that space. I am not trying to replace them. I am trying to translate the useful part into a bug-hunter’s mental model. ([web.dev][1])

Most developers hear “back/forward cache” and think “nice, the back button is faster.” That framing is good for product teams and bad for security work. BFCache is not the HTTP cache. It is not just cached HTML. Modern browsers can keep the entire page in memory — including DOM state and the JavaScript heap — pause execution when the user navigates away, and resume that exact page when they come back. MDN is explicit about that distinction: BFCache stores the full in-memory page state, whereas the HTTP cache stores prior responses. ([MDN Web Docs][2])

That one fact is enough to change how you think about a lot of bugs.

If your exploitation model assumes “user leaves page” means “page dies,” BFCache is already breaking your assumptions. If a product team assumes “user comes back” means “fresh load,” BFCache is already breaking theirs. The feature is really about document liveness, not performance. The W3C TAG guide says the core standards problem is support for **non-fully-active documents**: documents that are no longer the active page, but are not yet gone either. ([W3C TAG][3])

That is the right starting point for bug hunting:

**BFCache creates attack surface wherever a system relied on navigation as a destruction boundary.**

## What BFCache means in attacker language

The attacker’s translation is simple.

A page can survive navigation.
Its heap can survive navigation.
Its UI can survive navigation.
Its assumptions can survive navigation.
And the application may never have written explicit rules for what “restored state” is allowed to mean.

web.dev’s guidance revolves around `pageshow` and `pagehide`, with `event.persisted` telling you whether the page was restored from BFCache. The fact those events exist is itself the giveaway: browser vendors are telling you, plainly, that this is not a normal reload path. It is a suspend/resume path. ([web.dev][1])

So do not ask “can BFCache be exploited” in the abstract. Ask a narrower and better question:

**What security property in this application quietly depended on the page being dead after navigation?**

That is where the bugs are.

## The first class of bugs: stale auth state that still looks alive

This is the obvious one, but it is obvious because it is real.

A user logs out.
A user switches accounts.
A user loses privilege.
A user’s role changes server-side.
A session expires.
A token is revoked.

But the page they were on is restored from BFCache, not reloaded, and the UI comes back looking authenticated, privileged, or current. Walton and Pollard use exactly this class of example in their guidance, warning that pages may need to refresh or revalidate on `pageshow` when restored from BFCache. ([web.dev][1])

The novice mistake is to stop at “sensitive information flashes briefly.” Sometimes that is the issue. The better bug-hunter question is whether the restored page can still **do** anything before the application realizes it is stale.

Look for:

* privileged buttons wired to stale client-side state,
* forms whose CSRF token or privilege assumptions are not revalidated on restore,
* account-switch flows where the page’s heap still points at the old identity,
* dashboards that visually switch accounts but keep actions bound to restored objects,
* and restore paths where the frontend trusts its own suspended memory more than the current session reality.

The strongest findings here are not “old data was visible.” They are “restored state still drove side effects.”

## The second class: logout is treated as a network event, not a page-state event

A lot of logout logic is written as though one server response is enough to clean reality. Clear cookie, redirect, done. That logic was always slightly optimistic. BFCache makes the optimism easier to see.

Chrome’s current BFCache behavior also matters here. Chrome rolled out support for using BFCache on some pages even when they send `Cache-Control: no-store`, specifically when Chrome believes it is safe to do so, with extra mitigation such as eviction on cookie changes and shorter retention. That means “we set `no-store`, so back navigation will definitely refetch” is no longer a safe assumption in Chrome. ([Chrome for Developers][4])

That is a very good 2026 bug-hunting update because a lot of security teams still carry the older instinct that “sensitive pages with `no-store` won’t come back from BFCache.” You do not get to lean on that anymore.

So if you are assessing logout or privilege-drop flows, test them like this:

* logout in one tab, restore in another,
* logout and hit back immediately,
* switch accounts and then back/forward aggressively,
* revoke permission server-side, then navigate history instead of reloading.

If the product’s security story depends on “this page would normally have been reloaded,” it is probably worth staying there longer.

## The third class: BFCache turns lifecycle bugs into security bugs

BFCache is brutal on old lifecycle assumptions. web.dev’s guidance is blunt: do not use `unload`; use `pagehide`, and expect that the page might be cached instead of destroyed. On desktop Chrome and Firefox, `unload` listeners can prevent BFCache eligibility, while Safari behaves differently enough that relying on `unload` is unreliable anyway. ([web.dev][1])

That sounds like a performance note. It is also a security note.

If a product assumes cleanup happens on `unload`, ask what exactly was supposed to be cleaned up:

* open privileged channels,
* locks,
* temporary secrets in memory,
* UI disable flags,
* ephemeral anti-replay state,
* cross-window coordination,
* postMessage trust setup,
* background polling assumptions,
* or “one time only” client-side transitions.

Once you frame BFCache as a suspend/resume model, any security property implemented as “clean this up when we leave the page” becomes suspicious unless the code is explicitly written against `pagehide`/`pageshow` or equivalent lifecycle semantics.

The general bug idea is:

**the page was never actually reset, but the application thought it was.**

## The fourth class: state resurrection chains

This is where BFCache gets genuinely interesting.

The most instructive public example is Jorian Woltjer’s nonce-based CSP bypass using disk cache behavior, where BFCache is part of a larger browser-state chain. His one-line summary is that you can get a nonce reused with BFCache falling back to disk cache after leaking it, then force the HTML injection to be refetched under conditions that let the known nonce become useful. ([jorianwoltjer.com][5])

The point is not merely “here is a clever CSP bypass.” The point is methodological:

**BFCache interacts with other browser storage and restore layers.**

So when you are attacking something sensitive to “freshness,” do not stop at the page itself. Think about chains involving:

* BFCache restore first, network later,
* BFCache eviction followed by disk-cache reuse,
* page-specific secrets or nonces whose lifetime assumptions were tied to reload,
* and flows where the application mixes “this came from memory” with “this came from a fresh response” without realizing it.

This is elite browser-bug territory because it forces you to stop thinking in request/response terms and start thinking in history/lifecycle/state-machine terms.

A lot of appsec work is still one layer too high for that.

## The fifth class: UI truth and security truth diverge on restore

One underrated BFCache bug pattern is not direct auth bypass but **restore-time UI desynchronization**.

A page comes back instantly.
The UI looks complete.
The JavaScript heap is intact.
The user can click before revalidation finishes.
The frontend still has old flags, old objects, old route guards, old “canEdit” booleans, old cached membership, or old workflow state.

If that page was written under the assumption that initial render always follows a fresh bootstrap, BFCache can make the UI *feel* authoritative when it is merely historical.

That yields several practical bug classes:

* restored admin controls that should be gone,
* restored step-up-authenticated screens after step-up state has ended,
* restored purchase/approval screens after the backing object has changed,
* restored onboarding/verification/one-time setup screens that can still submit stale transitions,
* restored “impersonation mode” or “support mode” pages that look active after the mode is gone.

These are often dismissed as product bugs because the page eventually corrects itself. That is the wrong instinct. A security review should care about the **window before correction** and about any side effect reachable within it.

If the exploit is “user hits Back, clicks before revalidation, and action succeeds,” that is not cosmetic.

## The sixth class: shared-resource bugs that were hidden by page death

The TAG guide’s focus on non-fully-active documents exists for a reason: APIs and state models often behave badly if they were designed around the assumption that only one “live enough” document exists at a time. ([W3C TAG][3])

For bug hunters, this suggests a broader test strategy:

Look for features that assume navigation releases something.

Examples:

* edit locks,
* document ownership sessions,
* active-draft markers,
* one-user-at-a-time admin consoles,
* “this page is open elsewhere” coordination,
* session-bound WebSocket or channel logic,
* and anti-duplication flows around approvals, payments, or destructive actions.

web.dev explicitly advises developers to close connections or shared-resource handles on lifecycle events and reopen them on restore. The existence of that advice tells you the platform itself expects state-sharing bugs here. ([web.dev][1])

Your exploit question becomes:

**What if the old page did not actually relinquish what the application assumed it relinquished?**

That can turn into stale writers, duplicate actions, cross-tab confusion, or raceable restore windows.

## The seventh class: BFCache eligibility itself is recon

This is the most underused practical trick.

Modern tooling is much better here than it used to be. Chrome DevTools has a BFCache panel, and the `PerformanceNavigationTiming.notRestoredReasons` surface tells developers why a page was blocked from BFCache or not restored. Chrome says it shipped from Chrome 123 and is rolling out gradually; MDN documents the API and the higher-level monitoring guidance. ([Chrome for Developers][6])

Most people read that as developer ergonomics. You should also read it as recon.

Why?

Because BFCache blockers often tell you a lot about what the page is doing:

* presence of unload-like lifecycle assumptions,
* sensitive APIs or connections,
* document relationships,
* embedded frame behavior,
* and other stateful features the application might not realize are security-relevant.

A page that stubbornly refuses BFCache is often a page with interesting lifecycle semantics. A page that **does** BFCache when the team assumed it would not is often even more interesting.

In other words: BFCache eligibility is not just a performance metric. It is a map of where the application still believes in old navigation semantics.

## The eighth class: “no-store means safe” is dead as a lazy design assumption

This deserves its own section because people are still behind on it.

Chrome’s current position is that some `Cache-Control: no-store` pages can still use BFCache when Chrome judges it safe, with mitigations such as eviction on cookie changes and shorter timeouts. Other browsers may differ, and not every `no-store` page becomes eligible, but the old security instinct — “we used `no-store`, therefore this won’t be restored from BFCache” — is no longer reliable enough to build assumptions on. ([Chrome for Developers][4])

That changes the testing posture on anything “sensitive”:

* account settings,
* billing,
* support consoles,
* admin pages,
* medical/legal/financial dashboards,
* privileged internal tools,
* and pages teams explicitly marked `no-store` because they thought that ended the discussion.

Now the real question is not whether they set the header. It is whether they wrote restore semantics.

If not, there may be a bug.

## What I would actually test first

Not “does BFCache exist.” Assume it does.

Test whether coming back to the page gives you anything the product team thought leaving the page would have removed:

* stale privilege,
* stale identity,
* stale capability,
* stale UI authority,
* stale token usage,
* stale workflow transitions,
* stale locks,
* or stale trust between windows, frames, or tabs.

Then test whether the correction path is synchronous, asynchronous, or absent.

Then test whether a side effect is reachable before correction.

Then test whether navigation plus restore plus another browser state layer — disk cache, service worker behavior, frame relationships, account switching, or embedded flows — yields something that no single request/response view would reveal.

That is how BFCache becomes interesting.

## Final thesis

BFCache is not a performance feature for bug hunters. It is a state-preservation feature that destroys a very old assumption: **the page is not necessarily dead just because the user left it.**

Once you accept that, the vulnerability ideas stop looking exotic.

Logout bugs.
Account-switch desync.
Restored privileged UI.
Suspend/resume lifecycle bugs.
Shared-resource confusion.
Nonce or freshness assumptions broken by state resurrection chains.
And teams relying on `no-store` or navigation itself as if either one were a destruction primitive.

That is the real lesson.

The browser learned how to keep pages alive.
A lot of applications never learned what that means.

[1]: https://web.dev/articles/bfcache "Back/forward cache | Articles | web.dev"
[2]: https://developer.mozilla.org/en-US/docs/Glossary/bfcache "bfcache - Glossary - MDN"
[3]: https://w3ctag.github.io/bfcache-guide/ "Supporting BFCached Documents - GitHub Pages"
[4]: https://developer.chrome.com/docs/web-platform/bfcache-ccns "Enabling bfcache for Cache-Control: no-store - Chrome Developers"
[5]: https://jorianwoltjer.com/blog/p/research/nonce-csp-bypass-using-disk-cache "Nonce CSP bypass using Disk Cache - Jorian Woltjer"
[6]: https://developer.chrome.com/docs/web-platform/bfcache-notrestoredreasons/ "Back/forward cache notRestoredReasons API | Web Platform | Chrome for ..."
