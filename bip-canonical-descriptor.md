```
  BIP: ?
  Layer: Applications
  Title: Canonical Form for Output Script Descriptors
  Authors: Craig Raw <craig@sparrowwallet.com>
  Status: Draft
  Type: Specification
  Assigned: ?
  License: BSD-2-Clause
  Requires: 327, 341, 379, 380, 381, 382, 383, 384, 385, 386, 387, 389, 390, 392, 393, 394
```

## Abstract

This document defines a canonical form for output script descriptors: a deterministic textual
encoding such that two descriptors that differ only in how they are written — in spelling, in
encoding, and in the order of elements whose order carries no meaning — produce the same string, and
therefore the same checksum. Descriptors that describe the same output scripts by different means,
such as a public key written directly rather than derived from an extended key, are not brought
together by canonicalization and remain distinct.

The canonical form is intended for backup, display, export and comparison. It does not change how
descriptors are parsed, what scripts they produce, or what any existing implementation must accept
as input. The document also recommends which of several equivalent descriptors an exporter should
produce, and describes how later documents that define new descriptor expressions can specify their
canonical form.

## Copyright

This BIP is licensed under the BSD 2-clause license.

## Motivation

For any wallet that is not single-signature, the output script descriptor is the backup. It is
printed, engraved, photographed as a QR code, and handed to people who will need it long after the
software that produced it has been replaced. Users are routinely asked to compare one against
another: to check that a restored wallet matches the original, that each participant in a multisig
holds the same wallet, or that a descriptor recovered from an heir's records is the one that was
written down.

Descriptors do not currently support that use. The same wallet has many valid textual
representations, differing in such things as the hardened derivation marker, the case of
hexadecimal, and — for `sortedmulti()` and its relatives — the order in which the key expressions
are written. Each representation has its own [BIP 380](bip-0380.mediawiki) checksum, so the checksum
cannot be used to confirm that two backups describe the same wallet, and a user comparing two
strings by eye cannot conclude anything from the fact that they differ.

Implementations have responded to this individually, and they do not agree. Some preserve the order
in which the key expressions were supplied. Others sort them, but by different criteria: by the
public keys the expressions derive at some index, or by the text of the expressions themselves, with
or without their key origin information. Two implementations that both sort therefore still produce
different strings for the same wallet, and an implementation that sorts produces a different string
from one that does not. The problem is not that any of these choices is wrong — each is reasonable
for the purpose it was made for — but that no choice is specified, so none of them interoperate.

Key order also has consequences beyond display. A [BIP 388](bip-0388.mediawiki) wallet policy is a
descriptor template together with an ordered key information vector, and the proof of registration a
signing device returns commits to the policy it approved, so permuting the keys yields a policy for
which that proof no longer holds. Implementations that register wallets with such devices already
need a stable order, and have each invented one.

## Specification

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT" and "MAY" in this document are to be
interpreted as described in RFC 2119.

### Applicability

This document defines a function from a descriptor to a string. A descriptor in canonical form is
also a valid descriptor as defined by [BIP 380](bip-0380.mediawiki) and its extensions, and
describes exactly the same set of output scripts as the descriptor it was produced from.

Canonicalization is an explicit operation, applied to a descriptor for the purposes described below.
Parsers MUST continue to accept descriptors in any valid form, whether or not they are canonical,
and implementations MUST NOT reject a descriptor for being non-canonical.

Canonicalization derives no keys. Where the same key can be written as extended keys at different
depths, with the derivation steps between them moved into or out of key origin information, the
results describe the same output scripts but have different canonical forms. Choosing which
descriptor to export is therefore a separate step, described under Exporting, which precedes
canonicalization.

The requirements of this document depend on the purpose for which a descriptor is used:

* An implementation that compares descriptors under this document MUST compare their canonical
  forms.
* An implementation that presents a descriptor as a backup, or for a person to verify against
  another copy, MUST present its canonical form. Where the implementation is the exporter of that
  descriptor, it SHOULD first follow the recommendations under Exporting.
* An implementation that exports a descriptor to another implementation SHOULD follow the
  recommendations under Exporting and export the canonical form of the result. It MAY instead
  export a form that a consumer known not to accept the canonical form can parse.
* Other display, such as while a descriptor is being edited, is not constrained by this document.
  An implementation SHOULD present the canonical form where it presents a descriptor as the wallet's
  descriptor.

Where a descriptor has no canonical form, an implementation reports why, as described under
Preconditions, and does not compare it under this document or present it as canonical. The
recommendations under Exporting apply to a descriptor an implementation produces from a wallet it
holds, whether it then presents it as a backup, displays it, or transfers it to another
implementation. They never apply to a descriptor it received from elsewhere and only displays or
compares. An implementation conforms to this document only if it meets the requirements for each
purpose for which it uses descriptors.

### Preconditions

A descriptor has a canonical form only if all of the following hold. An implementation that is asked
to canonicalize a descriptor that does not satisfy them MUST report the reason rather than emit a
string.

1. The descriptor parses as an output script descriptor per [BIP 380](bip-0380.mediawiki) and its
   extensions, its checksum, if one is present, is valid, and every expression in it is within the
   scope defined under Extensions.
2. After an extended public key, a key expression performs no hardened derivation, unless its final
   step is a hardened wildcard (`/*h` or `/*'`). A final step such as `/5h`, or a hardened element
   of a multipath tuple, does not itself qualify for the exception.

A descriptor excluded by the second condition has an equivalent in which the extended public key has
been derived to the last hardened step and the steps up to it moved into its key origin information,
the derivation described in the "Normalization of Key Expressions with Hardened Derivation" section
of [BIP 380](bip-0380.mediawiki). The two describe the same wallet, so accepting both would leave
exactly the ambiguity this document exists to remove. Deriving that key requires the corresponding
private keys, so it is a separate operation, performed when the descriptor is exported by a party
holding those keys, and is never performed by canonicalization, which derives no keys.

This condition is deliberately narrower than that section, which says that additional derivation
"is not necessary" when the final derivation is hardened, leaving an exporter free to derive further
or not. Where the final step is a hardened wildcard, this document leaves the key expression as
written, and accepts it including any hardened steps before the wildcard. Where the final step is a
single hardened index, the equivalent above exists and an exporter may produce it, so only that
equivalent is accepted; accepting both would give one key two canonical forms.

That section also predates multipath descriptors, and [BIP 389](bip-0389.mediawiki) does not say how
one is normalized, so this document applies the condition to each branch in turn. Where the excluded
hardened step is within or after the tuple, deriving through it gives a different extended public
key on each branch, and the equivalent is a separate descriptor for each branch. Each of those
descriptors has a canonical form; the multipath descriptor does not.

### Canonicalization

Canonicalization proceeds bottom-up: sub-expressions are canonicalized before the expression that
contains them.

#### Encoding

These rules apply to the descriptor as a whole.

1. The descriptor MUST NOT contain whitespace.
2. A numeric value MUST be written without leading zeros. Hexadecimal encodes bytes rather than
   numbers and is not affected.
3. An uppercase character MUST NOT appear outside a substring in a case-sensitive encoding, and
   such substrings MUST be written as supplied. Among the documents this one requires, the only
   case-sensitive encoding is Base58, which encodes extended keys, WIF private keys, and the Base58
   addresses of `addr()` expressions ([BIP 385](bip-0385.mediawiki)). A later document may
   introduce others, as described under Extensions.
4. Where [BIP 379](bip-0379.md) defines a miniscript fragment or wrapper as syntactic sugar for
   another expression, the sugared form MUST be written: `pk(KEY)`, for example, and not
   `c:pk_k(KEY)`. The `c:` wrapper is absorbed only where it is applied directly to `pk_k` or
   `pk_h`. Wrappers applied in succession are written as that document writes them, as one run of
   letters before a single colon, so `t:` applied to `v:pk(KEY)` is `tv:pk(KEY)`. Where both `l:`
   and `u:` apply, as in `or_i(0,0)`, `l:` MUST be written.

#### Key expressions

1. Key origin information MUST NOT be added where it was not present, and MUST NOT be removed where
   it was; either would make the result depend on state outside the descriptor, and would move the
   key expression in the ordering below. Where present, key origin information is written as
   `[fingerprint]` or `[fingerprint/path]`.
2. Hardened derivation steps, in both key origin information and derivation suffixes, MUST be
   written with `h`. The apostrophe form MUST NOT be used.
3. The bytes of a key MUST NOT be changed, whether it is an extended key, a private key or a raw
   public key. This rule fixes the key material only; the case of the characters that encode it is
   governed by the Encoding rules above. The one exception is a raw public key written as the key of
   a `tr()` or `rawtr()` expression, or as a key within a tapscript. A compressed public key in one
   of those positions is converted to the x-only key with the same x coordinate (specified for
   `tr()` in [BIP 386](bip-0386.mediawiki) and for `rawtr()` in [BIP 394](bip-0394.mediawiki)),
   and MUST be written as that x-only key of 64 hexadecimal characters. The participant keys of a
   `musig()` expression are not converted, because [BIP 327](bip-0327.mediawiki) aggregates plain
   public keys in compressed format.
4. Canonicalization does not convert a descriptor to a public one, and an implementation MUST NOT
   treat a canonical descriptor as one from which private key material has been removed.
5. Multipath derivation, where present, MUST be retained in the form defined by
   [BIP 389](bip-0389.mediawiki), and MUST NOT be expanded into separate descriptors. This rule
   constrains canonicalization only; an exporter may still choose to export separate descriptors,
   as described under Exporting.

#### Ordering

Three expressions take a set of key expressions in which the written order has no effect on the
scripts produced. Two are script expressions; the third, `musig()`, is itself a key expression:

* `sortedmulti()`, defined in [BIP 383](bip-0383.mediawiki)
* `sortedmulti_a()`, defined in [BIP 387](bip-0387.mediawiki)
* `musig()`, defined in [BIP 390](bip-0390.mediawiki)

In these three expressions, the key expressions MUST be written in ascending order under the
comparison rule below.

The two children of a branch in a `tr()` script tree are likewise unordered:
[BIP 341](bip-0341.mediawiki) sorts a pair of child hashes before hashing them, so exchanging the
children of any branch leaves the merkle root, the output script and every control block unchanged.
The children of each branch MUST therefore be written in ascending order under the comparison rule,
applied to each branch in turn. Only the children of a branch may be exchanged: moving a script to a
different depth in the tree changes the merkle root, and MUST NOT be done.

The key/value pairs of an annotation expression, as defined in [BIP 393](bip-0393.mediawiki), are
also unordered: that document states that their order is not significant for interpretation. The
pairs MUST be written in ascending order of their keys under the comparison rule. Annotations MUST
NOT be added or removed.

The elements of a multipath tuple are positional — [BIP 389](bip-0389.mediawiki) expands the first
element into the first descriptor, the second into the second, and so on, which wallets commonly use
for receiving and change addresses — and MUST NOT be reordered, despite the resemblance of `<0;1>`
to a set.

In every other position the written order is meaningful, or is not known to be meaningless, and MUST
be preserved exactly as given, unless a later document specifies otherwise as described under
Extensions. This includes, but is not limited to, the key expressions of `multi()` and `multi_a()`,
and the arguments of miniscript fragments.

#### Comparison rule

Two expressions are compared by the value of each character of their canonical text in turn. Every
character of a descriptor comes from the character set defined in [BIP 380](bip-0380.mediawiki),
which lies entirely within the ASCII range, so character order and byte order are the same. If the
two texts are equal up to the length of the shorter, the shorter sorts first.

Two identical expressions compare equal, and may be written in either order; the result is the same
string.

#### Serialization

The canonicalized expression tree is serialized, and a [BIP 380](bip-0380.mediawiki) checksum
computed over the resulting string is appended, separated by `#`. Any checksum present in the input
is discarded.

### Exporting

The recommendations in this section govern which descriptor an exporter produces, before it is
canonicalized. They are not part of canonicalization, which never applies them. The only key
derivation they call for is the normalization of hardened derivation described in
[BIP 380](bip-0380.mediawiki), extended under Preconditions to a final step that is a single
hardened index.

An exporter SHOULD write each extended public key as the key at the last hardened derivation step
in its path from the master key, as described in the "Normalization of Key Expressions with
Hardened Derivation" section of [BIP 380](bip-0380.mediawiki), and SHOULD NOT derive it further.
Unhardened steps that follow the last hardened step are written after the extended key, not moved
into its key origin information. A wallet following the [BIP 44](bip-0044.mediawiki) family of
derivation paths, for example, is exported with its account-level extended public key, as
`[fingerprint/84h/0h/0h]xpub.../0/*` and not `[fingerprint/84h/0h/0h/0]xpub.../*`. Where a key's
path contains no hardened step, the extended key is the master key. This rule does not apply where
the final step is a hardened wildcard, which [BIP 380](bip-0380.mediawiki) and the preconditions
above leave as written. An exporter that holds only an extended public key derived further than
this cannot derive the key above it, and exports the key it holds.

An extended private key SHOULD be written as the exporter holds it, with the derivation steps from
it written after it, and SHOULD NOT be derived further, since from the descriptor alone a derived
private key cannot recover its parent or that parent's other children. A wallet holding a master
private key, for example, is exported as `xprv.../84h/0h/0h/0/*` and not as
`[fingerprint/84h/0h/0h]xprv.../0/*`. The normalization described in [BIP 380](bip-0380.mediawiki)
concerns only descriptors exported without private keys.

Where a wallet's receiving and change output scripts are described by two descriptors differing
only in one derivation step at the same position in each key expression, they SHOULD be exported as
one multipath descriptor as defined by [BIP 389](bip-0389.mediawiki), so that one canonical string
describes the whole wallet. When they are so exported, the first element of each tuple MUST be the
step of the receiving descriptor and the second the step of the change descriptor, following the
convention of that document. A wallet using `0` for receiving and `1` for change is therefore
written `<0;1>`, never `<1;0>`: the elements of a tuple are positional, so the two would be
different descriptors with different canonical forms. The recommendation to export one multipath
descriptor applies where the resulting multipath descriptor has a canonical form; where it does
not, because of a hardened step within or after the tuple, each branch is exported as its own
canonical descriptor. Where the consumer is known not to support multipath descriptors, the two
descriptors SHOULD instead be exported separately, each in canonical form; see Backward
Compatibility. An exporter MAY also offer the separate descriptors on request.

### Extensions

The canonical form is defined for descriptors whose expressions are all defined in the documents
listed in this document's `Requires` header, and for expressions defined in later documents that
specify their canonical form by reference to this one. A descriptor containing any other expression
does not satisfy the preconditions, and has no canonical form, even where an implementation is able
to parse it.

A document that defines a new descriptor expression, or extends an existing one, SHOULD include a
section specifying its canonical form, which states:

1. Which arguments, if any, are unordered, and how they are ordered. Such arguments SHOULD be
   ordered by the comparison rule of this document.
2. Any case-sensitive encoding it introduces, and the substrings it applies to.
3. Any equivalent spellings it permits, and which of them is canonical.
4. Any conversion of key material, in the manner of the conversion of compressed public keys to
   x-only keys above.
5. Test vectors giving the canonical form of descriptors that use it.

Where such a section is silent, the rules of this document apply unchanged: arguments keep the
order in which they were written, key material is not converted, and the Encoding rules apply to the
expression's text.

A document that extends descriptors SHOULD NOT change the canonical form of any descriptor that
could be written before it, so that a descriptor once in canonical form remains in canonical form.

## Rationale

**Why order by the text of the key expression, rather than by the public keys it derives?**
[BIP 383](bip-0383.mediawiki) sorts the keys that go into the output script, after derivation, at
each index. That order is a property of the index, not of the wallet: two key expressions can derive
keys that sort one way at index 0 and the other way at index 1. An ordering defined by derived keys
is therefore only well-defined once an index is also fixed by convention, and two implementations
that both "sort by the derived keys" will still disagree unless they have chosen the same index.
Ordering by the text requires no derivation, no elliptic curve arithmetic, and no choice of index,
and is defined identically for ranged and non-ranged descriptors. It also produces an order that a
person can verify by reading the descriptor from left to right, which is the property this document
exists to provide; no one can check by eye that a set of extended keys is in the order of the public
keys they derive at some index.

**Why is key origin information part of the sort key?** Because it is written first, ordering by the
text of the key expression orders by the master key fingerprint, then by the derivation path, then
by the key itself. Fingerprints are how users and signing devices identify cosigners, so this
produces an order that a person can relate to the participants of the wallet, and one that can be
checked against a list of fingerprints without handling extended keys. Because lowercase hexadecimal
characters sort in ASCII in the same relative order as the values they encode, ordering fingerprints
as text is the same as ordering them as bytes. [BIP 129](bip-0129.mediawiki) requires a signer to
display the position of its own key in the policy during setup, which it justifies for scripts in
which key order matters. For `sortedmulti()` that position otherwise reflects only the order in
which the keys happened to be written. When a device is given a descriptor in canonical form whose
key expressions carry key origin information, the position follows from the fingerprints, so a user
can predict it and check it against what the device shows.

**Why compare derivation paths as text rather than numerically?** Path components are compared as
characters, so `/9h` sorts after `/48h`. Numeric comparison would be closer to how a reader thinks
about paths, but it requires parsing the path and defeats the property that the ordering can be
checked by reading the string. Cosigners in a multisig almost always share the same derivation path,
in which case the comparison never reaches it. The text rule is chosen for determinism and
checkability rather than for reading order.

**Why rule out whitespace and leading zeros?** [BIP 380](bip-0380.mediawiki) does not say whether
either is valid, and parsers differ. Where a parser accepts one, it is another way of writing the
same descriptor, and the canonical form does not use it. Where a parser rejects it, the descriptor
does not parse and the rule has nothing to act on.

**Why the sugared miniscript forms?** [BIP 379](bip-0379.md) defines them as equivalent to the
expressions they stand for, so the two differ only in spelling and one had to be chosen. The sugared
form is the shorter.

**Why `h` rather than `'`?** Both are accepted, and one had to be chosen.
[BIP 380](bip-0380.mediawiki) writes its own specification with `h` and describes the apostrophe as
the alternative indicator. The apostrophe also requires escaping in shells and in JSON, is altered
by systems that substitute typographic quotation marks, and is silently removed by some shell
quoting, which converts hardened derivation into unhardened derivation with no visible sign. `h` has
no such failure modes.

**Why is `multi()` not reordered?** The order of keys in a `multi()` expression is the order in
which they appear in the output script, and therefore determines the addresses. Reordering it would
describe a different wallet.

**Which descriptors does canonicalization not bring together?** An extended key can often be written
at more than one depth. A key expression with unhardened steps after the extended key describes the
same scripts as one in which the extended key has been derived further and those steps moved into
its key origin information, and the same is true of hardened steps where private key material is
present, and of hardened steps that precede a hardened wildcard after an extended public key, which
[BIP 380](bip-0380.mediawiki) leaves in place. Unifying these would require deriving keys, which
canonicalization does not do, so they remain distinct. The precondition on hardened derivation after
an extended public key covers the cases where an exporter holding the private keys derives the key
further: where [BIP 380](bip-0380.mediawiki) calls for it, and where the final step is a single
hardened index. This document applies it to each branch of a multipath descriptor. The
recommendation on export depth covers unhardened steps, so conforming exporters agree on the depth
of extended public keys except where the final step is a hardened wildcard. Other differences
remain even between conforming exporters, because they change what the descriptor conveys rather
than how it is written: the presence or absence of key origin information, the presence of private
key material and the depth at which it is held, and a public key written directly rather than as an
extended key with its derivation.

**Why export the extended public key at the last hardened step?** Unhardened derivation needs only
the extended public key, so a canonicalizer could derive through unhardened steps and remove the
ambiguity itself. It does not, for two reasons. Doing so would require elliptic curve arithmetic in
an operation that is otherwise purely textual. More importantly, the result would be worse for
comparison: the account-level extended key is the one that signing devices display and export,
that cosigners exchange, and that users recognize, while a key derived further is one none of them
has seen. Deriving in the other direction, towards the master key, is impossible without private
keys. The last hardened step is the depth that leaves the familiar key in place, and it is the
depth [BIP 380](bip-0380.mediawiki) already prescribes when hardened steps must be removed. This
document adds only that the exporter stops there. The rule concerns extended public keys only:
deriving an extended private key discards, from the descriptor alone, the ability to derive its
parent's other children, so an extended private key is written as held. Two descriptors that hold
private keys at different depths convey different private key material, and are among the
differences that remain distinct, described above.

**Why are the children of a script tree branch ordered?** For the same reason as the key expressions
above: the order carries no information. [BIP 341](bip-0341.mediawiki) sorts each pair of child
hashes before hashing them, noting that "we do not actually care about the position of specific
scripts in the tree; only that they are actually committed to", so two descriptors that differ only
in the order of a branch's children describe the same output script and are indistinguishable once
spent. Exchanging children preserves the depth of every script and every merkle path, which is why
the rule is stated per branch rather than over the leaves as a whole.

**Why are annotations kept, and ordered?** An annotation records what is needed to recover the
wallet's funds, so removing it would change what the descriptor conveys, as removing a private key
would. [BIP 393](bip-0393.mediawiki) gives the order of the pairs no meaning, but notes that
reordering them changes the checksum, which is the problem this document addresses. Two descriptors
for the same wallet whose annotation values differ, because one was exported later than the other,
convey different things and have different canonical forms.

**Why is private key material preserved rather than removed?** Removing it would change what the
descriptor conveys, and for some descriptors it would leave nothing usable behind: the watch-only
form of an `sp()` descriptor, as defined in [BIP 392](bip-0392.mediawiki), carries the silent
payment scan private key, because outputs cannot be detected without it. A descriptor holding
private key material and the public descriptor derived from it are different things, and have
different canonical forms.

**Why is `musig()` included?** [BIP 390](bip-0390.mediawiki) requires that the participant keys be
sorted before aggregation precisely so that the written order cannot matter, noting that this "aids
with recovery where the descriptor was not backed up as users will not need to also have backed up,
or guess, the correct order of keys". The same reasoning applies to the written form.

**Why not expand multipath descriptors?** Expanding a multipath descriptor produces two strings
where the wallet is one thing, which is precisely the comparison problem this document addresses.
Retaining the [BIP 389](bip-0389.mediawiki) form keeps one wallet to one canonical string. The
exception is a multipath descriptor that the second precondition excludes because of a hardened step
within or after the tuple: such a wallet is written as one canonical descriptor per branch.

**Is this a claim that a descriptor has one true form?** No. This document defines one form for one
purpose — comparing and backing up — and leaves every other use of descriptors untouched. In
particular, implementations that use descriptor text as an internal key for stored state are not
required to change that state to canonical form, and are advised not to; see Backward Compatibility.

**Is the canonical form a wallet identifier?** It is not intended as one. Two descriptors with the
same canonical form describe the same output scripts, but the converse does not hold: descriptors
for the same wallet can differ in annotations, which change between exports, in the depth at which
an extended key is written, in the presence of key origin information or private key material, and
in whether receiving and change are written as one multipath descriptor or two. Its checksum is
not an identifier either: [BIP 380](bip-0380.mediawiki) designs it to detect transcription errors,
and at 40 bits it is not collision-resistant. A wallet identifier must give the same value for every
descriptor of the same wallet, and must resist collisions. The first requires deriving keys, which
canonicalization does not do. Such an identifier can nonetheless be defined on top of the canonical
form, by first normalizing a descriptor to remove the differences above and then hashing its
canonical form; this document leaves that to a separate specification.

**Why must later expressions opt in to a canonical form?** An implementation that applied the rules
of this document to an expression it did not know would have to guess whether its arguments were
unordered, whether any of its text was case-sensitive, and whether it had equivalent spellings. Two
implementations that guessed differently, or one that guessed before the expression's defining
document said otherwise and one that did not, would produce different canonical forms for the same
descriptor, which is the ambiguity this document exists to remove. Refusing to canonicalize an
expression whose canonical form has not been specified keeps every canonical form that an
implementation does produce stable across implementations and over time. This document cannot
bind later ones, so it can only recommend what they state; a later document that states nothing
leaves its expressions without a canonical form rather than with an inconsistent one.

**Why not use [BIP 129](bip-0129.mediawiki) (BSMS) instead?** BSMS establishes a multisig wallet
among several participants and gives each the same descriptor record. It solves the setup problem,
and is complementary to this document, but it does not define how the descriptor within that record
is written, and it is not what a user holds years later when comparing a printed backup against a
wallet they have just restored.

## Backward Compatibility

This document introduces no consensus change and no change to the scripts or addresses that any
descriptor produces. Every spelling canonicalization chooses is one that
[BIP 380](bip-0380.mediawiki) and its extensions define and use in their own examples, so a
canonicalized descriptor is accepted by any parser that conforms to them, and a conforming
implementation that ignores this document continues to interoperate. A parser that accepts only some
of the spellings those documents allow, such as only the apostrophe as the hardened derivation
marker, may accept a descriptor and reject its canonical form; an implementation exporting to such
a consumer may export a form it accepts, as described under Applicability.

Three compatibility considerations apply to implementations that adopt it.

An implementation that begins exporting descriptors in canonical form will, for any descriptor that
was not already in canonical form, produce a different string and a different checksum than it
produced before. Users comparing a new export against a backup taken previously will see a
difference that does not indicate an error. Implementations SHOULD note the change where users will
encounter it.

An implementation that uses descriptor text as a key for persistent state — most importantly the key
information vector of a [BIP 388](bip-0388.mediawiki) wallet policy and the proof of registration a
signing device has returned for it — will invalidate that state if it switches the state to
canonical form, requiring the user to approve the wallet on the device again. Such implementations
SHOULD leave their internal identity function unchanged and apply the canonical form only where a
descriptor is exported, displayed or compared. Nothing in this document requires the two to be the
same function.

The recommendation to export a wallet as a single multipath descriptor can produce a descriptor that
an existing consumer cannot read. Multipath descriptors are a later addition than the separate
receiving and change descriptors they replace, and software that parses the latter may not parse
the former. The recommendation nonetheless applies unless the consumer is known not to support
[BIP 389](bip-0389.mediawiki), because the consumer of a backup is usually not known when it is
made, and is more likely to be later software than earlier. Where a consumer does not support
multipath descriptors, the failure is visible: the descriptor is rejected when parsed, not
misinterpreted. A multipath descriptor with two-element tuples is converted to the equivalent pair
of descriptors by replacing every tuple with its first element in one descriptor and its second
element in the other, then canonicalizing each, which also recomputes its checksum. Replacing the
tuples alone is not always enough: two key expressions that are identical up to their tuples, such
as `xpub.../<0;1>/*` and `xpub.../<1;0>/*`, can change their relative order, so an unordered set
that was in canonical order before the conversion may not be after it.

## Test Vectors

Each vector gives one or more descriptors and the canonical form of each. Input descriptors are
shown without a checksum; canonical forms include the checksum computed over the canonical string.

### Ordering of `sortedmulti()` key expressions

The following two descriptors, which differ in the order of their key expressions and in the
hardened derivation marker, have the same canonical form.

```
wsh(sortedmulti(2,[b7868815/48'/0'/0'/2']xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh/<0;1>/*,[1cf0bf7e/48'/0'/0'/2']xpub6FL8FhxNNUVnG64YurPd16AfGyvFLhh7S2uSsDqR3Qfcm6o9jtcMYwh6DvmcBF9qozxNQmTCVvWtxLpKTnhVLN3Pgnu2D3pAoXYFgVyd8Yz/<0;1>/*,[4fc1dd4a/48'/0'/0'/2']xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj/<0;1>/*))
```

```
wsh(sortedmulti(2,[4fc1dd4a/48h/0h/0h/2h]xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj/<0;1>/*,[b7868815/48h/0h/0h/2h]xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh/<0;1>/*,[1cf0bf7e/48h/0h/0h/2h]xpub6FL8FhxNNUVnG64YurPd16AfGyvFLhh7S2uSsDqR3Qfcm6o9jtcMYwh6DvmcBF9qozxNQmTCVvWtxLpKTnhVLN3Pgnu2D3pAoXYFgVyd8Yz/<0;1>/*))
```

Canonical form of both:

```
wsh(sortedmulti(2,[1cf0bf7e/48h/0h/0h/2h]xpub6FL8FhxNNUVnG64YurPd16AfGyvFLhh7S2uSsDqR3Qfcm6o9jtcMYwh6DvmcBF9qozxNQmTCVvWtxLpKTnhVLN3Pgnu2D3pAoXYFgVyd8Yz/<0;1>/*,[4fc1dd4a/48h/0h/0h/2h]xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj/<0;1>/*,[b7868815/48h/0h/0h/2h]xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh/<0;1>/*))#jecfc8v2
```

Note that ordering by the extended keys alone, ignoring key origin information, would give a
different result: `xpub6EebM...`, `xpub6FA5r...`, `xpub6FL8F...`.

### `multi()` is not reordered

```
wsh(multi(2,[b7868815/48'/0'/0'/2']xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh/<0;1>/*,[1cf0bf7e/48'/0'/0'/2']xpub6FL8FhxNNUVnG64YurPd16AfGyvFLhh7S2uSsDqR3Qfcm6o9jtcMYwh6DvmcBF9qozxNQmTCVvWtxLpKTnhVLN3Pgnu2D3pAoXYFgVyd8Yz/<0;1>/*,[4fc1dd4a/48'/0'/0'/2']xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj/<0;1>/*))
```

Canonical form, with the order unchanged and only the hardened derivation marker rewritten:

```
wsh(multi(2,[b7868815/48h/0h/0h/2h]xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh/<0;1>/*,[1cf0bf7e/48h/0h/0h/2h]xpub6FL8FhxNNUVnG64YurPd16AfGyvFLhh7S2uSsDqR3Qfcm6o9jtcMYwh6DvmcBF9qozxNQmTCVvWtxLpKTnhVLN3Pgnu2D3pAoXYFgVyd8Yz/<0;1>/*,[4fc1dd4a/48h/0h/0h/2h]xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj/<0;1>/*))#nnhw487l
```

### Raw public keys and hexadecimal case

```
sh(sortedmulti(1,03ACD484E2F0C7F65309AD178A9F559ABDE09796974C57E714C35F110DFC27CCBE,022F01E5E15CCA351DAFF3843FB70F3C2F0A1BDD05E5AF888A67784EF3E10A2A01))
```

Canonical form:

```
sh(sortedmulti(1,022f01e5e15cca351daff3843fb70f3c2f0a1bdd05e5af888a67784ef3e10a2a01,03acd484e2f0c7f65309ad178a9f559abde09796974c57e714c35f110dfc27ccbe))#6chqugeh
```

### Uppercase fingerprints and leading zeros

The fingerprint is hexadecimal and is lowercased, the leading zero is removed from the derivation
step `00'`, and the hardened derivation marker is rewritten. This applies where the parser accepts
the input; see the Rationale on leading zeros.

```
wpkh([DEADBEEF/84'/0'/00']xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL/0/*)
```

Canonical form:

```
wpkh([deadbeef/84h/0h/0h]xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL/0/*)#kf3v6fpx
```

A leading zero in a miniscript argument is likewise removed.

```
wsh(and_v(v:pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),older(0144)))
```

Canonical form:

```
wsh(and_v(v:pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),older(144)))#5nhqcn6v
```

### `sortedmulti_a()` within `tr()`

```
tr(50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0,sortedmulti_a(2,a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd,669b8afcec803a0d323e9a17f3ea8e68e8abe5a278020a929adbec52421adbd0))
```

Canonical form, with the internal key untouched and the tapscript keys ordered:

```
tr(50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0,sortedmulti_a(2,669b8afcec803a0d323e9a17f3ea8e68e8abe5a278020a929adbec52421adbd0,a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd))#6k4ze8h5
```

### Compressed public keys in taproot positions

A compressed public key written as the key of a `tr()` expression, or as a key within a tapscript,
is converted to the x-only key with the same x coordinate.

```
tr(03acd484e2f0c7f65309ad178a9f559abde09796974c57e714c35f110dfc27ccbe,pk(022f01e5e15cca351daff3843fb70f3c2f0a1bdd05e5af888a67784ef3e10a2a01))
```

Canonical form:

```
tr(acd484e2f0c7f65309ad178a9f559abde09796974c57e714c35f110dfc27ccbe,pk(2f01e5e15cca351daff3843fb70f3c2f0a1bdd05e5af888a67784ef3e10a2a01))#2tfqq2d8
```

The same conversion applies to the key of a `rawtr()` expression.

```
rawtr(03acd484e2f0c7f65309ad178a9f559abde09796974c57e714c35f110dfc27ccbe)
```

Canonical form:

```
rawtr(acd484e2f0c7f65309ad178a9f559abde09796974c57e714c35f110dfc27ccbe)#80xfhdgw
```

Keys are converted before they are ordered, since canonicalization proceeds bottom-up. Here the
compressed keys are already in ascending order, but their x-only forms are not, so conversion
reverses them.

```
tr(50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0,sortedmulti_a(1,02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9,03acd484e2f0c7f65309ad178a9f559abde09796974c57e714c35f110dfc27ccbe))
```

Canonical form:

```
tr(50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0,sortedmulti_a(1,acd484e2f0c7f65309ad178a9f559abde09796974c57e714c35f110dfc27ccbe,f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9))#67f4uppl
```

### `musig()`

```
tr(musig([b7868815/48h/0h/0h/2h]xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh,[1cf0bf7e/48h/0h/0h/2h]xpub6FL8FhxNNUVnG64YurPd16AfGyvFLhh7S2uSsDqR3Qfcm6o9jtcMYwh6DvmcBF9qozxNQmTCVvWtxLpKTnhVLN3Pgnu2D3pAoXYFgVyd8Yz)/0/*)
```

Canonical form:

```
tr(musig([1cf0bf7e/48h/0h/0h/2h]xpub6FL8FhxNNUVnG64YurPd16AfGyvFLhh7S2uSsDqR3Qfcm6o9jtcMYwh6DvmcBF9qozxNQmTCVvWtxLpKTnhVLN3Pgnu2D3pAoXYFgVyd8Yz,[b7868815/48h/0h/0h/2h]xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh)/0/*)#c32wpjxj
```

### `musig()` participant keys are not converted

The participants are ordered, but keep the compressed serialization they were given.

```
tr(musig(03dff1d77f2a671c5f36183726db2341be58feae1da2deced843240f7b502ba659,02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9,023590a94e768f8e1815c2f24b4d80a8e3149316c3518ce7b7ad338368d038ca66))
```

Canonical form:

```
tr(musig(023590a94e768f8e1815c2f24b4d80a8e3149316c3518ce7b7ad338368d038ca66,02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9,03dff1d77f2a671c5f36183726db2341be58feae1da2deced843240f7b502ba659))#nu38trla
```

### Single-key descriptor

```
wpkh([f6bb4c63/84'/0'/0']xpub6BosfCnifzxcFwrSzQiqu2DBVTshkCXacvNsWGYJVVhhawA7d4R5WSWGFNbi8Aw6ZRc1brxMyWMzG3DSSSSoekkudhUd9yLb6qx39T9nMdj/<0;1>/*)
```

Canonical form:

```
wpkh([f6bb4c63/84h/0h/0h]xpub6BosfCnifzxcFwrSzQiqu2DBVTshkCXacvNsWGYJVVhhawA7d4R5WSWGFNbi8Aw6ZRc1brxMyWMzG3DSSSSoekkudhUd9yLb6qx39T9nMdj/<0;1>/*)#kcpz3ljl
```

### Hardened steps before a hardened wildcard

The key expression ends in a hardened wildcard, so the hardened steps before it are accepted and
only the hardened derivation marker is rewritten.

```
pkh([deadbeef/0'/1'/2]xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL/3'/4'/5'/*')
```

Canonical form:

```
pkh([deadbeef/0h/1h/2]xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL/3h/4h/5h/*h)#u7dt9330
```

A hardened multipath tuple is likewise accepted where the final step is a hardened wildcard.

```
pkh(xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL/<0';1'>/*')
```

Canonical form:

```
pkh(xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL/<0h;1h>/*h)#nfcvtcww
```

### Key expressions without key origin information

Key origin information is neither added nor removed. Where it is absent, the key expressions are
compared from the extended keys onwards.

```
wsh(sortedmulti(2,xpub6FL8FhxNNUVnG64YurPd16AfGyvFLhh7S2uSsDqR3Qfcm6o9jtcMYwh6DvmcBF9qozxNQmTCVvWtxLpKTnhVLN3Pgnu2D3pAoXYFgVyd8Yz/<0;1>/*,xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj/<0;1>/*))
```

Canonical form:

```
wsh(sortedmulti(2,xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj/<0;1>/*,xpub6FL8FhxNNUVnG64YurPd16AfGyvFLhh7S2uSsDqR3Qfcm6o9jtcMYwh6DvmcBF9qozxNQmTCVvWtxLpKTnhVLN3Pgnu2D3pAoXYFgVyd8Yz/<0;1>/*))#9dx350ct
```

### Mixed key types

Key expressions of different types are ordered by the same comparison rule, with no grouping by
type. A compressed public key begins with `02` or `03`, and a digit sorts before `[`, which sorts
before the `x` of a bare extended public key.

```
wsh(sortedmulti(2,xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj/0/*,[b7868815/48'/0'/0'/2']xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh/0/*,03acd484e2f0c7f65309ad178a9f559abde09796974c57e714c35f110dfc27ccbe))
```

Canonical form:

```
wsh(sortedmulti(2,03acd484e2f0c7f65309ad178a9f559abde09796974c57e714c35f110dfc27ccbe,[b7868815/48h/0h/0h/2h]xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh/0/*,xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj/0/*))#2470v4q0
```

An x-only public key may begin with a digit or with a letter from `a` to `f`, so x-only keys do not
sort together: one beginning with a digit sorts before a key expression with key origin
information, and one beginning with a letter sorts after it.

```
tr(50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0,sortedmulti_a(2,xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj/0/*,a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd,[b7868815/48'/0'/0'/2']xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh/0/*,669b8afcec803a0d323e9a17f3ea8e68e8abe5a278020a929adbec52421adbd0))
```

Canonical form:

```
tr(50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0,sortedmulti_a(2,669b8afcec803a0d323e9a17f3ea8e68e8abe5a278020a929adbec52421adbd0,[b7868815/48h/0h/0h/2h]xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh/0/*,a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd,xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj/0/*))#8krfv705
```

### Private key material and case-sensitive encodings

The hexadecimal spend key is lowercased. The scan key is a WIF-encoded private key, whose Base58
encoding is case-sensitive, and is written as supplied.

```
sp(L4rK1yDtCWekvXuE6oXD9jCYfFNV2cWRpVuPLBcCU2z8TrisoyY1,0260B2003C386519FC9EADF2B5CF124DD8EEA4C4E68D5E154050A9346EA98CE600)
```

Canonical form:

```
sp(L4rK1yDtCWekvXuE6oXD9jCYfFNV2cWRpVuPLBcCU2z8TrisoyY1,0260b2003c386519fc9eadf2b5cf124dd8eea4c4e68d5e154050a9346ea98ce600)#sjd9eg3d
```

### The children of a `tr()` script tree branch are ordered

The two leaves are exchanged, which leaves the merkle root and the output script unchanged.

```
tr(50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0,{pk(a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),pk(669b8afcec803a0d323e9a17f3ea8e68e8abe5a278020a929adbec52421adbd0)})
```

Canonical form:

```
tr(50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0,{pk(669b8afcec803a0d323e9a17f3ea8e68e8abe5a278020a929adbec52421adbd0),pk(a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd)})#q4qxv5lf
```

### A nested `tr()` script tree

The children of the inner branch are exchanged, and then the children of the outer branch, where the
leaf sorts before the branch because `p` precedes `{`. Every script stays at its original depth.

```
tr(50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0,{{pk(f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9),pk(a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd)},pk(669b8afcec803a0d323e9a17f3ea8e68e8abe5a278020a929adbec52421adbd0)})
```

Canonical form:

```
tr(50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0,{pk(669b8afcec803a0d323e9a17f3ea8e68e8abe5a278020a929adbec52421adbd0),{pk(a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),pk(f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9)}})#regxyp2f
```

### Miniscript syntactic sugar

The `pk` and `pkh` fragments, on their own and under another wrapper:

```
wsh(and_v(vc:pk_k(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),c:pk_h(02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9)))
```

Canonical form:

```
wsh(and_v(v:pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),pkh(02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9)))#grwmwe8a
```

The `t:` wrapper, joined to the wrapper already present:

```
wsh(and_v(v:pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),1))
```

Canonical form:

```
wsh(tv:pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd))#q9z64efh
```

The `and_n` fragment:

```
wsh(andor(pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),older(144),0))
```

Canonical form:

```
wsh(and_n(pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),older(144)))#kgs6932n
```

The `l:` wrapper:

```
wsh(or_d(pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),and_v(v:pk(02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9),or_i(0,older(144)))))
```

Canonical form:

```
wsh(or_d(pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),and_v(v:pk(02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9),l:older(144))))#yx6xrsnq
```

The `u:` wrapper:

```
wsh(or_d(pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),and_v(v:pk(02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9),or_i(older(144),0))))
```

Canonical form:

```
wsh(or_d(pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),and_v(v:pk(02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9),u:older(144))))#lnt2evdy
```

Where both `l:` and `u:` apply, `l:` is written:

```
wsh(or_d(pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),or_i(0,0)))
```

Canonical form:

```
wsh(or_d(pk(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),l:0))#uuktqzxq
```

Within a tapscript, the sugared forms are written and the compressed keys are converted to x-only
keys:

```
tr(50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0,and_v(vc:pk_k(03a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),c:pk_h(02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9)))
```

Canonical form:

```
tr(50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0,and_v(v:pk(a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd),pkh(f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9)))#3ce6akgq
```

### Annotations

The annotation pairs are written in ascending order of their keys.

```
wpkh([deadbeef/84h/0h/0h]xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL/0/*)?gl=30&bh=800000
```

Canonical form:

```
wpkh([deadbeef/84h/0h/0h]xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL/0/*)?bh=800000&gl=30#v35n46d2
```

### Bech32 address in `addr()`

```
addr(BC1QW508D6QEJXTDG4Y5R3ZARVARY0C5XW7KV8F3T4)
```

Canonical form:

```
addr(bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4)#uyjndxcw
```

### `spscan` key expression

```
sp(SPSCAN1QU6D9S9LFD3A99NCKPJW7AS602LG0950WVCFWG7G4KAKHSP32R57QX4853D0YLM42UEWYDGX6XGZ0V20HGTHSK2KR84F96JLS3Q0JYWKTRV8US5)
```

Canonical form:

```
sp(spscan1qu6d9s9lfd3a99nckpjw7as602lg0950wvcfwg7g4kakhsp32r57qx4853d0ylm42uewydgx6xgz0v20hgthsk2kr84f96jls3q0jywktrv8us5)#p8gyfucr
```

### Descriptors with no canonical form

Each of the following is a valid descriptor that the second precondition excludes. No vector is
given for the scope condition in the first precondition, since every expression currently defined
is within the scope set out under Extensions; a descriptor using an expression defined later, whose
defining document does not specify its canonical form, would also have no canonical form.

Hardened steps after an extended public key, with a final step that is not hardened:

```
pkh(xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL/3h/4h/5h/*)
```

A final step that is a single hardened index:

```
pkh(xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL/3h/5h)
```

A hardened multipath tuple, whether or not it is the final step:

```
pkh(xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL/<0h;1h>/*)
```

```
pkh(xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL/<0h;1h>)
```

## Reference Implementation

To be provided.
