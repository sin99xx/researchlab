# JWTs for People Who Hunt Bugs, Not Blog Posts

## A whitepaper on JOSE semantics, verifier failure modes, and where real systems break

JWT is usually presented as a compact token format for authentication and authorization. That description is operationally useful and analytically weak. For a security engineer, a JWT is better modeled as a **serialized cryptographic object carrying attacker-controlled metadata**, where the decisive question is not whether the primitive is modern, but whether the verifier permits untrusted structure to influence trust before trust has been earned. JWT failures are rarely about “breaking SHA-256.” They are about finding the exact point where the implementation stops treating the token as hostile input and starts treating it as configuration. The JOSE stack—JWS, JWE, JWK, and JWA—was standardized across RFCs 7515, 7516, 7517, and 7518, with JWT itself in RFC 7519 and the hardening guidance in RFC 8725. ([RFC Editor][1])

That is the right frame for a 2026 reader. The interesting problems are not “what is a claim” or “how do I decode base64url.” The interesting problems are these: which fields are consulted before authenticity is established, which keys are reachable through attacker input, which algorithms are accepted because the token asked for them, which claims are interpreted outside their intended context, and which error paths leak structure under adaptive probing. The current IETF direction has only reinforced this posture: RFC 8725 remains the baseline BCP, `rfc8725bis` is active, and JOSE is also advancing draft work to formally deprecate JWS `none` and JWE `RSA1_5`, which is really the standards process admitting that verifier flexibility has historically been too generous. ([RFC Editor][2])

## 1. What a JWT actually is, before applications romanticize it

At the wire level, a compact JWT is not “a JSON object.” It is an ASCII string composed of base64url-encoded segments separated by literal dots. RFC 7519 defines JWT claims as a JSON object used as the payload of a JWS or the plaintext of a JWE. That byte-level definition matters because the security boundary is not the later parsed object model; it is the exact serialized input on which cryptographic operations are defined. ([RFC Editor][3])

For a signed JWT, the compact JWS form is:

`BASE64URL(UTF8(JWS_Header)) || '.' || BASE64URL(JWS_Payload) || '.' || BASE64URL(JWS_Signature)`

RFC 7515 defines the JWS Signing Input as the ASCII bytes of the first two transmitted segments joined by a dot. That means the signature is over bytes, not over semantic JSON equivalence, not over claims “as understood by the application,” and not over any normalized representation a library invents after parsing. If one forgets that, one begins reasoning about JWT at the wrong layer. ([RFC Editor][1])

For an encrypted JWT, compact JWE is:

`BASE64URL(Protected_Header) || '.' || BASE64URL(Encrypted_Key) || '.' || BASE64URL(Initialization_Vector) || '.' || BASE64URL(Ciphertext) || '.' || BASE64URL(Authentication_Tag)`

RFC 7516 defines those five fields and makes the protected header part of the authenticated data. In other words, even when the header is visible, it is still cryptographically bound to the ciphertext and tag. That is one of the more misunderstood aspects of JWE: the header is not merely descriptive metadata, it participates in the integrity model. ([IETF Datatracker][4])

## 2. JWS and JWE are not “formats”; they are trust transitions

In `HS256`, the signer computes an HMAC over the JWS Signing Input using a shared secret. In `RS256`, the signer hashes the same input and signs it using RSASSA-PKCS1-v1_5. In `ES256`, the signer generates an ECDSA signature over P-256 and serializes the `(r,s)` pair in JOSE’s fixed-width format. In all of those cases, the verifier’s obligation is very narrow: reconstruct the signing input exactly, apply the cryptographic check under a policy-approved algorithm and key, and reject on any mismatch. ([RFC Editor][1])

JWE is a different sort of machine. The verifier first has to recover or derive the Content Encryption Key according to `alg`, then apply the content encryption algorithm specified by `enc`, and only then release plaintext if authentication succeeds. Depending on the header, the CEK may be transported with RSA-OAEP, wrapped under AES Key Wrap, derived through ECDH-ES, or supplied directly in `dir` mode. Each accepted `alg` therefore expands the verifier’s state space. That is not merely interoperability; it is attack surface. ([IETF Datatracker][4])

This is why JWT should be taught as a system of trust transitions rather than a convenience token. JWS asks whether these bytes were authenticated under this algorithm and key. JWE asks whether CEK establishment, content decryption, and integrity checks all succeeded under exact rules. Nested JWT asks whether both layers were validated in the intended order. Systems get into trouble when they flatten those separate questions into a single boolean called “token valid.” ([RFC Editor][3])

## 3. The real bug-hunting thesis: untrusted metadata must not negotiate verification policy

The recurring JWT bugs all share one structural defect: the verifier allows attacker-controlled token metadata to influence the rules that are supposed to constrain the token. That is the common logic behind algorithm confusion, `kid` injection, `jwk` abuse, `jku`-driven key substitution, cross-token substitution, and differentiated decryption oracles. The mistake is not the existence of a header. The mistake is permitting the unauthenticated header to participate in trust negotiation. RFC 8725 is essentially a long formal argument for refusing that posture. ([RFC Editor][2])

A serious verifier therefore begins with policy that exists outside the token: accepted algorithms, accepted key classes, accepted issuers, accepted audiences, accepted token types, and accepted key lookup rules. The token may identify a candidate within that universe. It must not be allowed to widen the universe. The difference sounds philosophical until you review a target that reads `alg`, chooses a verification mode, reads `kid`, chooses a storage path, reads `jku`, chooses a network destination, and only then asks whether any of this was a good idea. That is not verification. That is adversary-assisted configuration. ([RFC Editor][2])

## 4. Algorithm choice is really trust-model choice

The standard HS256/RS256/ES256/PS256 comparison is too often presented as a performance table. For actual security work, these are distinct trust models disguised as algorithms. RFC 7518 defines the algorithms, but RFC 8725 is where the operational consequences become explicit. ([RFC Editor][2])

`HS256` is cryptographically fine if the key is actually a cryptographic key. The trouble is architectural: every verifier holding the HMAC secret can also mint tokens. In a distributed system, that collapses issuance and verification into the same privilege. RFC 8725 explicitly warns against using human-memorable passwords directly as HMAC keys because once a valid token is observed, weak deployments reduce to offline guessing. The failure mode is not exotic. It is a key-management failure wearing a JWT badge. ([RFC Editor][2])

`RS256` improves the trust model because verifiers only need the public key. But it also creates the exact precondition for the most famous JWT exploit family: if the verifier treats “key material” generically and allows the token to influence algorithm choice, RSA public-key bytes can be reinterpreted as an HMAC secret. That is not a weakness in RSA. It is type confusion at the verification boundary. RFC 8725’s guidance on caller-specified algorithms and key separation exists precisely to prevent that collapse. ([RFC Editor][2])

`ES256` is attractive because elliptic-curve signatures are smaller and efficient, but they are less forgiving of sloppy implementation. ECDSA depends on sound nonce generation, which is exactly why RFC 6979 standardized deterministic nonce generation. The point for a bug hunter is not to recite “ECDSA needs good randomness.” The point is to inspect whether the implementation got nonce discipline, JOSE signature serialization, side-channel behavior, and key validation right at the same time. ([IETF Datatracker][4])

`PS256` is generally the cleaner RSA signature choice when ecosystem support allows it. But even a superior signature scheme does not rescue a verifier that accepts the wrong token, from the wrong issuer, under the wrong key-selection pathway, for the wrong application context. Strong primitives do not compensate for weak verification semantics. ([IETF Datatracker][4])

## 5. The exploit classes that still matter

Algorithm confusion remains the canonical JWT bug because it exposes whether the verifier actually knows what kind of key it is holding. A public RSA key and an HMAC secret are not interchangeable in any real cryptographic model. Yet a verifier that reduces both to “bytes plus an algorithm named by the token” has erased that distinction itself. Once that type boundary is gone, the attacker does not need to break RSA; they only need the application to keep passing the same bytes into a library that now interprets them differently. That is why this class is still worth teaching to mature engineers. It is not old news. It is a lesson about trust-boundary typing. ([RFC Editor][2])

Weak HS256 deployments are less glamorous and often more practical. If the HMAC key is low entropy, the token becomes an offline cracking target. No interaction with the verifier is required after capture. One valid token plus a bad secret policy is enough. RFC 8725’s language on high-entropy keys is not ceremonial advice; it reflects a failure mode that remains common in real deployments. ([RFC Editor][2])

Header-driven key selection is where JWT starts to look less like cryptography and more like input injection wearing a suit. RFC 7515 defines `kid`, `jwk`, `jku`, `x5u`, and related parameters, while RFC 8725 warns that these values must not be trusted without strong policy constraints. If `kid` is spliced into SQL, mapped into a filesystem path, or treated as an unchecked selector into a key store, the token has opened a classic injection surface. If `jwk` is accepted as self-supplied trust or `jku` is allowed to fetch arbitrary remote keys, the signature check may still succeed while the verifier has been tricked into validating against attacker-chosen material. That is not signature bypass in the childish sense. It is verification-path compromise. ([RFC Editor][1])

Substitution attacks are what happen when systems confuse cryptographic validity with semantic legitimacy. A token may be perfectly signed and still be unacceptable: wrong audience, wrong issuer relationship, wrong token type, wrong protocol role, wrong relying party, wrong nested composition. RFC 8725 leans hard on issuer validation, audience validation, explicit typing, and mutually exclusive validation rules because a valid token for one purpose is routinely replayable into another unless the application binds it to context. This is the mature JWT bug class: the crypto passes, the authorization model fails. ([RFC Editor][2])

## 6. The advanced vectors serious assessors still care about

ECDH-ES in JWE is where casual JWT knowledge ends and actual cryptographic review begins. In that mode, the recipient combines a static private key with an attacker-influenced ephemeral public key from the header. If the implementation does not validate that point correctly, classic invalid-curve or subgroup-style attacks become relevant. RFC 8725 explicitly calls out the need for elliptic-curve input validation, and the JOSE ecosystem has already lived through real-world library fixes motivated by this class of issue. Once you realize that a token header is feeding a public-key operation, the review question writes itself: are we validating the peer point before scalar multiplication? ([RFC Editor][2])

`RSA1_5` remains important not because modern engineers should be using it, but because it teaches the right lesson about decryption-side behavior. The long history of Bleichenbacher-style oracle attacks against PKCS#1 v1.5 encryption is exactly why the JOSE standards track is moving to formally deprecate JWE `RSA1_5`, and why current IETF drafts reflect a narrower tolerance for legacy flexibility. The vulnerability is not “RSA bad.” It is that decryption plus distinguishable failure behavior plus attacker-controlled ciphertext is a dangerous combination, and standards bodies are finally treating that as a first-class design constraint rather than a footnote. ([IETF Datatracker][5])

## 7. What a hardened verifier actually believes

A hardened JWT acceptance pipeline is built on invariants, not convenience. The accepted algorithm set is defined by application policy before the token is consulted. Keys are typed and scoped: an RSA verification key is not “just bytes,” an HMAC secret is not “just bytes,” and a signing key is not silently reused as a decryption key. Header parameters are treated as hostile until independently constrained. Unknown critical parameters are not tolerated into success. Signature or decryption success is necessary and insufficient: issuer, audience, lifetime, token type, and context-specific profile rules are separate gates. And failure handling is uniform enough that the verifier does not become an oracle under probing. Those are not stylistic preferences. They are the minimum beliefs required to keep JWT from becoming a parser-mediated trust escalation mechanism. ([RFC Editor][2])

## 8. Final thesis

The enduring lesson of JWT is not that JSON is dangerous or that web developers should not touch cryptography. It is that **verification semantics are security-critical software**, and JWT puts a large amount of expressive power in precisely the layer where teams are tempted to trust attacker-controlled structure too early. The question that matters in every serious review is simple:

**What does the token get to influence before trust has been established?**

If the answer includes algorithm choice, key class, key source, token type, or authorization context, the system is probably review-worthy. That was true when RFC 8725 was published, and the direction of `rfc8725bis` and the deprecation work around `none` and `RSA1_5` suggests the standards process agrees. JWT is still useful. But in 2026, anyone serious about bug hunting should stop treating it as a token format and start treating it as what it really is: a hostile input language for verifiers. ([RFC Editor][2])

[1]: https://www.rfc-editor.org/rfc/rfc7515 "RFC 7515: JSON Web Signature (JWS)"
[2]: https://www.rfc-editor.org/rfc/rfc8725 "RFC 8725: JSON Web Token Best Current Practices"
[3]: https://www.rfc-editor.org/rfc/rfc7519 "RFC 7519: JSON Web Token (JWT)"
[4]: https://datatracker.ietf.org/wg/jose/about/ "Javascript Object Signing and Encryption (jose)"
[5]: https://datatracker.ietf.org/doc/draft-ietf-oauth-rfc8725bis/ "draft-ietf-oauth-rfc8725bis-04 - JSON Web Token Best Current Practices"
