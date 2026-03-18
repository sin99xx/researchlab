# 1/ RFC 6531 (SMTPUTF8) for Bug Hunters

## Unicode email is not a “support more characters” feature. It is an identity-parsing problem.

If you are reading RFC 6531 like a product manager, it says: email should support Unicode. If you are reading it like a bug hunter, it says something much more useful: **the Internet mail stack now has more than one valid representation of “the same-looking address,” and different components are allowed to understand different subsets of it**. RFC 6531 is the SMTP extension that lets servers advertise support for internationalized addresses and internationalized headers, sitting on top of the framework in RFC 6530 and the header-format changes in RFC 6532. It is still the standards-track SMTPUTF8 spec today. ([rfc-editor.org][1])

The first thing worth burning into memory is this: **the domain part and the local part are not the same problem**. RFC 6530 says the domain part was already internationalized via IDNA, while the local part was the missing piece. That means systems often inherit one canonicalization story for domains and a completely different one for mailbox local-parts, then accidentally smear those two stories together in application code. That is where real bugs start. ([rfc-editor.org][2])

At the protocol level, SMTPUTF8 is not magic. A server advertises support, the client uses the `SMTPUTF8` parameter on `MAIL FROM` when the envelope or message actually needs it, and servers offering SMTPUTF8 **must** also advertise `8BITMIME`. If the server does **not** offer SMTPUTF8, an SMTPUTF8-aware client must not send internationalized addresses and must not send a message containing internationalized headers. That means the protocol explicitly creates a split world: UTF-8-aware hops and legacy ASCII-only hops. Any product that assumes “we accepted the address once, therefore the rest of the pipeline agrees with it” is already on thin ice. ([rfc-editor.org][3])

That split world is the bug surface.

## The real attack surface is parser disagreement, not Unicode itself

Most people reduce SMTPUTF8 risk to homograph phishing. That is the most boring part of the problem. The better question is: **which components in the system think two addresses are equal, and which components do not?** RFC 6532 says UTF-8 can now appear directly in header field bodies, local parts, quoted strings, domains, and even Message-IDs, while header field names remain ASCII. It also says NFC normalization **should** be used for UTF-8 syntax, while NFKC **should not** be used because it can destroy information needed to spell names correctly. That sounds like internationalization guidance. For security engineers, it is a warning label: multiple layers are going to normalize differently unless someone was unusually disciplined. ([datatracker.ietf.org][4])

That gives you the first high-value bug class: **signup/login mismatch**. One code path accepts `用户@例子.公司`, another lowercases or normalizes it, a third converts only the domain to A-label form, a fourth stores the mailbox in NFC, and a fifth compares raw bytes. If the product uses email as a primary identity key, invitation target, password-reset target, allowlist entry, certificate SAN, or tenant boundary, these disagreements are often enough to get duplicate accounts, account recovery confusion, or authorization keyed to the wrong principal. RFC 9598 is especially useful here because it makes the comparison rules painfully explicit for certificate matching: the domain gets compared in LDH/A-label form, the local-part stays UTF-8, and the local-part must **not** be case-folded or normalized during setup. ([rfc-editor.org][5])

That should immediately make you suspicious of any product that proudly says “we canonicalize all email addresses by lowercasing and normalizing them.”

## The best bug: systems that canonicalize the whole address as if it were a domain

This is probably the cleanest SMTPUTF8 finding pattern.

Developers learned, correctly, that domains are case-insensitive and that internationalized domains often need IDNA handling. Then they extrapolated, incorrectly, that the whole mailbox should be treated that way. RFC 9598’s comparison rules are useful because they crystallize what the standards are *not* saying: domain handling and local-part handling are separate; local-parts are compared as UTF-8 and are not to be transformed by normalization or case folding in that setup step. In the certificate world, two `SmtpUTF8Mailbox` values are equivalent only on an exact octet-for-octet match after the comparison setup, and an `SmtpUTF8Mailbox` will never match a traditional ASCII `rfc822Name`. ([rfc-editor.org][5])

So the bug-hunter move is obvious: find any place where the app:

* lowercases the whole email,
* applies NFKC to the whole email,
* strips accents or performs compatibility folding,
* converts U-labels/A-labels inconsistently,
* or treats ASCII `rfc822Name` and Unicode `SmtpUTF8Mailbox` as interchangeable.

That is not harmless cleanup. That is identity rewriting.

## SMTPUTF8 creates cross-layer drift even when the product UI “doesn’t support Unicode email”

A lot of companies think they are safe because their signup form rejects non-ASCII email. That is not enough.

RFC 6531 and RFC 6532 matter anywhere the system **receives**, **parses**, **relays**, **logs**, **verifies**, or **matches** email addresses, not just where users type them into a registration form. SMTPUTF8 changes what can appear in the SMTP envelope and in message headers, including trace fields and reply contexts, and RFC 6532 expands where UTF-8 can live in header bodies and addresses. So even “ASCII-only” products often touch internationalized addresses indirectly through inbound mail, CRM sync, ticketing systems, SSO assertions, X.509 SAN matching, mailing-list software, support tooling, or admin consoles that trust email-shaped strings. ([rfc-editor.org][3])

That produces a very practical class of bugs: **input path A rejects Unicode, but backend path B later ingests or trusts it anyway**. The frontend thinks the identity space is ASCII. The mail pipeline or admin pipeline does not. The resulting gap is where impersonation, duplicate invitations, reset-link confusion, or policy bypass often show up.

## Downgrade and legacy compatibility are where things get lossy

One of the most useful signals in this whole area is that the standards themselves admit the compatibility story is messy. RFC 6857 exists because legacy POP/IMAP clients do not necessarily understand internationalized messages, and it describes post-delivery downgrading as one of the “least bad” options rather than a perfect one. The document is explicit that none of the plausible approaches preserve enough information to guarantee lossless reply behavior in the general case. That is not reassuring language. That is the standards track telling you there are edge cases where representations change and semantics drift. ([rfc-editor.org][6])

Bug-hunter translation: any mail-driven workflow that crosses modern and legacy systems is a candidate for **address mutation bugs**. If a message gets downgraded, wrapped, surrogate-generated, re-encoded, or surfaced through a client that only partially understands UTF-8 mail, look for:

* broken reply-to identity,
* mismatch between displayed sender and actionable sender,
* policy checks on a downgraded surrogate instead of the original mailbox,
* signed-content or header-integrity assumptions breaking when rewritten,
* and message-routing decisions made on one representation while audit logs or UI show another. ([rfc-editor.org][6])

The trick is not to think “can I send Unicode mail.” The trick is to think “where does this company convert, degrade, or compare mailbox identities across incompatible layers.”

## Byte length and character length are not the same anymore

RFC 6532 makes a small-looking change that is actually good bug fuel: the classic 998 limit is treated in **octets**, while the 78 recommendation remains about **characters**. In plain ASCII those are nearly the same story. In UTF-8 they are not. ([datatracker.ietf.org][4])

That opens the usual parser boundary class of bugs:

* storage limits enforced in characters on one side and bytes on another,
* truncation in logs or databases that breaks mailbox equality,
* header folding or transport libraries that assume ASCII byte width,
* and validation code that approves “short” values which later overflow a byte-based boundary downstream. ([datatracker.ietf.org][4])

These do not always become sexy account-takeover bugs. But in systems where email is a unique key, a routing key, or a security principal, byte/character mismatch is exactly the kind of boring engineering fault that can be turned into identity confusion.

## Encoded-word handling is another parser differential mine

RFC 6532 is very clear that once you are using this extension, MIME encoded-words generally **should not** be used for new header generation, and processors that decode encoded-words must not create syntactically invalid fields. That sounds like hygiene. It is also a clue that mixed legacy/modern parsing around encoded headers is still a place where implementations can disagree. ([datatracker.ietf.org][4])

That gives you another strong place to test:

* systems that decode display headers before applying policy,
* systems that transform encoded-words into UTF-8 and then reparse,
* and systems that trust one parser’s idea of the sender while another parser enforces policy on a different reconstruction.

Whenever email security decisions are made after multiple parse/decode/render steps, SMTPUTF8 and RFC 6532 make those disagreements more likely, not less.

## The most underrated finding: invite/reset/allowlist confusion

If a company uses email as identity, then SMTPUTF8 is not merely “mail transport support.” It is an **authorization surface**.

The best targets are:

* invitation systems,
* password reset targeting,
* domain-based access control,
* tenant membership keyed by email,
* certificate SAN matching,
* HR/IdP sync,
* and support/admin tooling that impersonates or searches by email.

The testing mindset is simple:

1. Find where the app stores the canonical email.
2. Find where it compares email.
3. Find where it routes mail.
4. Find where it displays email.
5. Find where it trusts email from another subsystem.

If those five places do not share the same model of Unicode mailbox equivalence, you may have a bug.

And the standards strongly suggest they often won’t. RFC 6531 extends SMTP transport rules, RFC 6532 extends header syntax and recommends NFC while warning against NFKC, RFC 6857 admits downgrading is lossy, and RFC 9598 nails down comparison behavior in certificate land in a way many product stacks do not mirror. ([rfc-editor.org][3])

## Final thesis

RFC 6531 is not interesting because “email supports Unicode now.”

It is interesting because it turns email addresses into a **cross-protocol identity object with multiple valid representations and multiple failure modes**. The moment a company uses email as a security boundary, SMTPUTF8 becomes bug-hunter material. The winning question is not “does the app support internationalized email?” The winning question is:

**Which subsystems think two Unicode email addresses are the same, and which subsystems silently disagree?** ([rfc-editor.org][1])


[1]: https://www.rfc-editor.org/info/rfc6531 "Information on RFC 6531 » RFC Editor"
[2]: https://www.rfc-editor.org/rfc/rfc6530 "RFC 6530: Overview and Framework for Internationalized Email"
[3]: https://www.rfc-editor.org/rfc/rfc6531 "RFC 6531: SMTP Extension for Internationalized Email"
[4]: https://datatracker.ietf.org/doc/html/rfc6532 "
            
                RFC 6532 - Internationalized Email Headers
            
        "
[5]: https://www.rfc-editor.org/in-notes/rfc9598.pdf "RFC 9598: Internationalized Email Addresses in X.509 Certificates"
[6]: https://www.rfc-editor.org/rfc/rfc6857.html "RFC 6857: Post-Delivery Message Downgrading for Internationalized Email Messages"
