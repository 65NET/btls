# 65NET fork of `0x676e67/btls`

Base: upstream tag `v0.5.6` (the newest published crate). Nothing else from
upstream `main` is taken.

## Branch topology, and what we actually track

```
main       mirror of 0x676e67/btls main. We do not build from it and do not
           merge into it. It exists so the fork can pull upstream.
ghostpaw   our baseline, cut at tag v0.5.6. Everything we build from descends
           from here; every change PRs into this branch, not into main.
```

**The thing we track is BoringSSL, not btls.** `65NET/boringssl`
`ghostpaw/head-plus-reverts` is the tracking branch: it moves when we choose to
move it, and each revert on it is a commit that says why we carry an upstream
deletion. `btls` itself is *pinned* at a release tag plus the rebase in this
file. Those are different relationships and the branch names should not blur
them.

### Why not base on upstream `main`

`main` is 99 commits ahead of `v0.5.6` and does some of this work already: it
splits `boringssl.patch` into ten numbered feature patches (`c7ada1f9`), moves
the submodule to `f1f2556a5` (2026-07-27, inside the Kyber/padding window), and
carries the two `btls` wrappers backported below. Rebasing onto it would make
future BoringSSL bumps smaller and better factored, and it is a real option.

It was not taken, for three reasons:

1. **Blast radius.** `main` also carries unreleased changes to
   `btls/src/ssl/x509/verify.rs`, new session APIs, refreshed certificate
   fixtures and two new certificate-validation patches
   (`bad-cert-verification.patch`, `relax-cert-validation.patch`). None of that
   is a BoringSSL upgrade, and none of it is verified by anything in this
   chain. "Upgrade BoringSSL" should not quietly become "adopt unreleased
   changes to certificate validation".
2. **It is unreleased and moving.** Dependabot is active on it. Pinning our
   baseline to a moving unreleased branch is the *opposite* of knowing what we
   track, even though tracking it looks tidier at a glance.
3. **`f1f2556a5` sits inside the clean window**, so a fork based on `main` would
   need no reverts today — and would therefore not record the two deletions
   anywhere. ADR 0001 rejected that pin for exactly this reason: it hands the
   same problem to whoever bumps next, with the measurement no longer fresh.

If we later decide the split patch series is worth adopting, that is its own
change with its own verification, on top of this one.

## What this fork changes

### 1. `btls-sys/deps/boringssl` moves to head plus two reverts

**Three upstream removals affect us. Two are reverted, one is adapted to.**
Which is which is a judgement call that no tooling makes for us, so it is
recorded here and in the commit messages rather than left to be re-derived:

The submodule now points at **`65NET/boringssl`**, branch
`ghostpaw/head-plus-reverts`:

```
805f6405  Revert "Remove old Kyber hybrid from the TLS stack"      (188ce3c13, 2026-08-03)
3c452508  Revert "Remove the old F5 padding workaround"            (eae46df1e, 2026-08-14)
4a925794  Start migrating HPKE to use EVP_KEM                      (upstream head, 2026-09-04)
```

`.gitmodules` deliberately carries **no `branch =` key**. The gitlink SHA above
is the record of what we build; a `branch` key would let
`git submodule update --remote` float BoringSSL off it silently, which is the
one thing a pinned vendored dependency must not do. Moving the submodule is a
commit, and that is the point.

`v0.5.6` vendored `91a66a59b` (2025-11-04), ten months behind. The upgrade is
what makes ML-DSA (`4a3cda40b`, 2026-04-23) and
`SSL_CTX_set_grease_sigalgs_enabled` (`29e593e29`, 2026-06-29) available, which
Chrome M150 and M152 respectively need.

The reverts are explicit commits, not a squashed import, because they record two
upstream deletions we deliberately carry:

* **Kyber.** `X25519Kyber768Draft00` (group `0x6399`) is the key share Chrome
  M124–M130 offers. It is also a **build** requirement: `btls` names
  `ffi::SSL_GROUP_X25519_KYBER768_DRAFT00`, and `boring-pq.patch` builds
  `P256Kyber768Draft00` on top of `crypto/kyber`.
* **`0015` padding.** BoringSSL pads a ClientHello landing in 256–511 bytes.
  Chrome M113–M118 hellos land there, so without it that shape is unreachable.

The third removal, `c5cbc0f91` "Remove SSL{_CTX}_set_aes_hw_override_for_testing"
(2026-08-24), is **adapted to, not reverted** — §2. The rule that separated
them: revert when upstream deleted something and left nothing in its place;
adapt when upstream deleted something and built a replacement that later commits
already depend on. Reverting `c5cbc0f91` was attempted and abandoned for exactly
that reason, so the branch carries two reverts and not three.

### 2. The patch stack is rebased onto that tree

`boringssl-windows.patch` is **deleted**: upstream's own fiat-p256 ASM guard is
now `(__ELF__ || __APPLE__) && OPENSSL_X86_64 && !OPENSSL_NANOLIBC`, which
already excludes MinGW.

The one semantically interesting rebase is the AES-hardware / TLS 1.3 cipher
ordering:

* Upstream `c5cbc0f91` dropped `has_aes_hw` from `ssl_create_cipher_list()` and
  removed the `aes_hw_override` fields, then introduced a first-class
  `SSLCipherPreferenceList tls13_cipher_list` on `SSLContext`/`SSL_CONFIG`
  (plus `SSL_CTX_set1_tls13_ciphers`).
* Reverting `c5cbc0f91` was tried and rejected: later upstream commits build on
  the new mechanism, so the revert conflicts with work we want.
* So the patch **adapts** instead. `has_aes_hw` is restored as a parameter on
  `ssl_create_cipher_list()` and added to `ssl_create_default_tls13_cipher_list()`;
  `aes_hw_override`/`aes_hw_override_value` are re-added as patch-owned fields;
  `SSL_CTX_set_aes_hw_override()` now recomputes `tls13_cipher_list`, because
  upstream materialises that list at `SSL_CTX` creation, before the override can
  be set.
* The patch's own `preserve_tls13_cipher_list` list storage and its
  `handshake_client.cc` cipher-emission block are **dropped**: upstream's
  `hs->config->tls13_cipher_list` does the same job natively.
  `ssl_create_preserve_tls13_cipher_list()` now writes into that list.

`btls` keeps `SSL_CTX_set_aes_hw_override` and
`SSL_CTX_set_preserve_tls13_cipher_list`, so both remain build requirements.

#### The one place the adaptation changes semantics

Before `c5cbc0f91` the AES-hardware override was **two bits read at handshake
time**, so it could not overwrite anything and the setters were
order-independent. Upstream now expresses the TLS 1.3 cipher order as a
materialised list, built once at `SSL_CTX` creation — so for the override to
reach the wire at all, the setter has to *recompute* that list. Recomputing can
overwrite, and overwriting is order-dependent. That is a real behavioural
difference and not a mechanical port.

It matters because upstream expresses compliance policies the same way, by
re-`Init`-ing the same list: an unguarded recompute would silently discard a
`cnsa_202407` list if the override happened to be set afterwards. The old code
honoured `compliance_policy == cnsa_202407 -> kCiphersCNSA` at handshake time
regardless of call order.

So the recompute is guarded. It runs only while the list is still the default:

```c
if (!preserve_tls13_cipher_list &&        // caller pinned the order
    !tls13_cipher_list_explicit &&        // caller named the list outright
    compliance_policy == ssl_compliance_policy_none) {
  ssl_create_default_tls13_cipher_list(&tls13_cipher_list, aes_hw_override_value);
}
```

`tls13_cipher_list_explicit` is a patch-owned bit set by
`SSL{_CTX}_set1_tls13_ciphers`. Both of those setters are already modified by
this patch, so the guard adds no new rebase surface. With it, a compliance
policy and an explicit cipher list each win over the override **in either
order**, which restores the order-independence the old code had.

Nothing `wreq` exposes reaches `set_compliance_policy` or
`set1_tls13_ciphers` today — but `btls` exposes `set_compliance_policy`
publicly, so this is one caller away, which is why it is guarded rather than
merely documented.

### 3. Two `btls` wrappers backported from upstream `main`

* `SslContextBuilder::set_grease_sigalgs_enabled` (upstream `d32ee495`)
* `SslContextBuilder::set_requested_trust_anchors` + `ExtensionType::TRUST_ANCHORS`
  (upstream `c23e35ee`)

Hand-applied rather than cherry-picked, to avoid dragging in unrelated `main`
changes.

## Verification

`ssl_test` on the rebased tree: 477/483 pass. The 6 failures are
`CipherRules`, `ClientHello`, `SigAlgs`, `SigAlgsList`,
`InvalidSignatureAlgorithm` and `SetGroupIdsWithFlags_DefaultGroups`. The first
five fail identically on the *unrebased* `v0.5.6` stack at its own pin (5/462),
because the patch stack changes cipher rules, hello shape and sigalg handling on
purpose. The sixth is a test that does not exist at the old pin; it asserts the
default group list, which `boring-pq.patch` changes by adding
`P256Kyber768Draft00`. No new failure class.
