```
                  ptr ─┐                           ptr ─┐
    ┌───────────┐     │             ┌───────────┐     │
    │  object   │◄────┘             │  object   │◄ ─ ─┘  dangling
    │  (live)   │                   │  ⌀ FREED  │
    └───────────┘                   └───────────┘
      ① alloc                         ② free → tcache[n]

                  ptr ─┐                           ptr ─┐
    ┌───────────┐     │             ┌───────────┐     │
    │ ATTACKER  │◄ ─ ─┘  reclaim    │ ATTACKER  │◄────┘
    │  payload  │                   │  payload  │   use → control
    └───────────┘                   └───────────┘
      ③ realloc same-size             ④ use-after-free
```

# sin99xx · Warisjeet Singh

Security researcher. I find memory-safety and logic bugs in software people trust — then I write the tooling that finds the next one. The diagram above is `CVE-2026-48004`.

The handle is `sinner`. No competition. I only see me.

`web / api` &nbsp;·&nbsp; `llm app sec` &nbsp;·&nbsp; `virtualization internals` &nbsp;·&nbsp; `state-machine & logic bugs` &nbsp;·&nbsp; [`hackerone.com/sin99xx`](https://hackerone.com/sin99xx)

<br>

## CVEs — four assigned

No dupes, no triage wins. Each is a real primitive against shipping software.

| | target | class | what it gives you |
|---|---|---|---|
| **`CVE-2026-48004`** | QEMU | heap UAF | Freed object reachable from guest-controlled state in device emulation — the front half of a guest-to-host escape in virtualized, multi-tenant cloud. → fixed upstream [`5a8da7e`](https://github.com/qemu/qemu/commit/5a8da7e979f1f56b1cab82c2354833f309f1a78f) |
| **`CVE-2026-48840`** | Exim | pre-auth leak | Uninitialised-stack leak in the PROXY-protocol parser, reachable before auth. An ASLR-defeat primitive that weakens mitigations on internet-facing mail servers before anyone logs in. → [advisory](https://exim.org/static/doc/security/EXIM-Security-2026-05-19.1.txt) |
| **`CVE-2025-5009`** | Gemini · iOS | info disclosure | Snippet-share leaked the *entire* conversation history instead of the selected snippet. Private user data exposed in a consumer AI product at scale. |
| **`CVE-2026-4XXXX`** | Joomla | reserved | Disclosure pending. |

<br>

## Automated vulnerability research

I build AI-agent frameworks that drive zero-day discovery through large codebases — orchestration that triages reachable attack surface and surfaces candidates for a human to confirm. The CVEs above came out of this loop.

```
agents narrow the search.  the exploitation is mine.
```

<br>

## Bug bounty

Accepted findings across private and public programs on HackerOne.

```
KOHO ··········· rank #2
```

`KOHO` · `1win` · `DoorDash` · `Monero` · `Whoop` · `Eightfold` · private programs
— and a finding against the Bugcrowd platform itself.

<br>

```
$ sinner --status
still mid-journey. the work speaks.
```
