<p align="center">
  <img src="./banner.svg" alt="sin99xx — security researcher" width="100%">
</p>

<p align="center">
  <a href="https://hackerone.com/sin99xx"><code>hackerone.com/sin99xx</code></a>
  &nbsp;·&nbsp; web / api &nbsp;·&nbsp; llm app sec &nbsp;·&nbsp; virtualization internals &nbsp;·&nbsp; logic bugs
</p>

<br>

Self-taught. I write the tooling, then I read the code. The handle is `sinner` — no competition, I only see me.

<br>

## ▌ CVEs

> Four assigned. Each one is a real primitive against shipping software — no dupes, no triage wins.

<table>
<tr>
<td width="160" valign="top">

**`CVE-2026-48004`**
QEMU
<sub>heap UAF</sub>

</td>
<td valign="top">

**Use-after-free in device emulation.** The bug class behind guest-to-host escape in virtualized and multi-tenant cloud — a freed object reachable from guest-controlled state is the front half of a hypervisor break.
<br><sub>fixed upstream · <a href="https://github.com/qemu/qemu/commit/5a8da7e979f1f56b1cab82c2354833f309f1a78f"><code>5a8da7e</code></a></sub>

</td>
</tr>
<tr>
<td width="160" valign="top">

**`CVE-2026-48840`**
Exim
<sub>pre-auth disclosure</sub>

</td>
<td valign="top">

**Uninitialised-stack leak in the PROXY-protocol parser**, reachable before authentication. Usable as an ASLR-defeat primitive — it weakens exploit mitigations on internet-facing mail servers before anyone logs in.
<br><sub><a href="https://exim.org/static/doc/security/EXIM-Security-2026-05-19.1.txt">EXIM-Security-2026-05-19.1</a></sub>

</td>
</tr>
<tr>
<td width="160" valign="top">

**`CVE-2025-5009`**
Google Gemini · iOS
<sub>info disclosure</sub>

</td>
<td valign="top">

**Snippet-share leaked the entire conversation history** instead of the selected snippet. Private user data exposed in a consumer AI product at scale.

</td>
</tr>
<tr>
<td width="160" valign="top">

**`CVE-2026-4XXXX`**
Joomla
<sub>reserved</sub>

</td>
<td valign="top">

Disclosure pending.

</td>
</tr>
</table>

<br>

## ▌ Automated vulnerability research

I build AI-agent frameworks that drive zero-day discovery through large codebases — orchestration that triages reachable surface and surfaces candidate bugs for a human to confirm. The CVEs above came out of this loop.

```
agents narrow the search.  the exploitation is mine.
```

<br>

## ▌ Bug bounty

Accepted findings across private and public programs on HackerOne.

<table>
<tr><td><b>KOHO</b></td><td><code>rank #2</code></td></tr>
</table>

`KOHO` · `1win` · `DoorDash` · `Monero` · `Whoop` · `Eightfold` · and private programs.
Also reported a vulnerability in the Bugcrowd platform itself.

<br>

<p align="center"><sub>still mid-journey · the work speaks</sub></p>
