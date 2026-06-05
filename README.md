<h1 align="center">sin99xx</h1>
<p align="center"><em>Warisjeet Singh · aka sinner</em> · self-taught security researcher · bug bounty hunter · CVE author</p>
<p align="center">
  <a href="https://hackerone.com/sin99xx"><img src="https://img.shields.io/badge/HackerOne-sin99xx-494649?style=flat-square&logo=hackerone&logoColor=white"></a>
  <img src="https://img.shields.io/badge/CVEs-4-c1121f?style=flat-square">
</p>

```
$ whoami

sin99xx — no competition, i only see me.

$ cat focus.txt

> Web / API security
> AI / LLM application security
> Virtualization internals
> Business-logic & state-machine bugs
```
### Automated Vulnerability Research & AI Orchestration

Designed and deployed custom AI-agent frameworks to automate zero-day discovery in complex codebases.

### CVEs

- **`CVE-2026-48004` — QEMU (heap use-after-free).** Reported a use-after-free in QEMU's device emulation, the class of memory-safety bug that can underpin guest-to-host escapes in virtualized and multi-tenant cloud environments. Fixed upstream ([commit 5a8da7e](https://github.com/qemu/qemu/commit/5a8da7e979f1f56b1cab82c2354833f309f1a78f)).
  
- **`CVE-2026-48840` — Exim (pre-auth info disclosure).** Found an uninitialised-stack leak in the PROXY-protocol parser, reachable pre-authentication and usable as an ASLR-defeat primitive — meaningful because it weakens exploit mitigations on internet-facing mail servers before any login. ([advisory](https://exim.org/static/doc/security/EXIM-Security-2026-05-19.1.txt))
  
- **`CVE-2025-5009` — Google Gemini (iOS, information disclosure).** Snippet-sharing flaw that leaked full conversation history rather than the selected snippet, exposing private user data in a shipping consumer AI product.
  
- **`CVE-2026-4XXXX` — Joomla.** Reserved — disclosure pending.

### Bug bounty

Active across private and public programs on HackerOne — **KOHO · 1win · DoorDash · Monero · Whoop · Eightfold · and other private programs** — with accepted findings.

> Also reported a vulnerability in the **Bugcrowd platform itself**. 🙃

### Stats

```
KOHO           rank #2
```

### Where to find me

- HackerOne — [@sin99xx](https://hackerone.com/sin99xx)

---

<p align="center"><sub>Still mid-journey. The work speaks.</sub></p>
