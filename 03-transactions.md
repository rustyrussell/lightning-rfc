# BOLT #3: Bitcoin Transaction and Script Formats

This details the exact format of on-chain transactions, which both sides need to agree on to ensure signatures are valid.  That is, the funding transaction output script, commitment transactions and the HTLC transactions.

## Transaction input and output ordering

Lexicographic ordering as per BIP 69.

## Funding Transaction Output

* The funding output script is a pay-to-witness-script-hash<sup>[BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#witness-program)</sup> to:
   * `0 2 <key1> <key2> 2 OP_CHECKMULTISIG`
* Where `<key1>` is the numerically lesser of the two DER-encoded `funding-pubkey` and `<key2>` is the greater.

## Commitment Transaction
* version: 2
* locktime: lower 24 bits are the obscured commitment transaction number.
* txin count: 1
   * txin[0] outpoint: `txid` and `output_index` from `funding_created` message
   * txin[0] sequence: lower 24 bits are upper 24 bits of the obscured commitment transaction number.
   * txin[0] script bytes: 0
   * txin[0] witness: `<signature-for-key1>` `<signature-for-key-2>`

The 48-bit commitment transaction number is obscured by XOR with the lower 48 bits of:

    SHA256(payment-basepoint from open_channel || payment-basepoint from accept_channel)

This obscures the number of commitments made on the channel in the
case of unilateral close, yet still provides a useful index for both
nodes (who know the payment-basepoints) to quickly find a revoked
commitment transaction.

### Commitment Transaction Outputs

The amounts for each output are rounded down to whole satoshis.  If this amount is less than the `dust-limit-satoshis` set by the owner of the commitment transaction, the output is not produced (thus the funds add to fees).

To allow an opportunity for penalty transactions in case of a revoked commitment transaction, all outputs which return funds to the owner of the commitment transaction (aka "local node") must be delayed for `to-self-delay` blocks.  This delay is done in a second stage HTLC transaction.

The reason for the separate transaction stage for HTLC outputs is so that HTLCs can time out or be fulfilled even though they are within the `to-self-delay` `OP_CHECKSEQUENCEVERIFY` delay.  Otherwise the required minimum timeout on HTLCs is lengthened by this delay, causing longer timeouts for HTLCs traversing the network.

#### To-Local Output

This output sends funds back to the owner of this commitment transaction, thus must be timelocked using OP_CSV.  The output is a version 0 P2WSH, with a witness script:

	to-self-delay OP_CHECKSEQUENCEVERIFY OP_DROP <local-delayedkey> OP_CHECKSIG

It is spent by a transaction with nSequence field set to `to-self-delay` (which can only be valid after that duration has passed), and witness script `<local-delayedsig>`.

#### To-Remote Output

This output sends funds to the other peer, thus is a simple P2PKH to `<remotekey>`.

#### Offered HTLC Outputs

This output sends funds to a HTLC-timeout transaction after the HTLC timeout, or to the remote peer on successful payment preimage.  The output is a P2WSH, with a witness script:

    <remotekey> OP_SWAP
        OP_SIZE 32 OP_EQUAL
    OP_NOTIF
        # To me via HTLC-timeout tx (timelocked).
        OP_DROP 2 OP_SWAP <localkey> 2 OP_CHECKMULTISIGVERIFY
    OP_ELSE
        # To you with preimage.
        OP_HASH160 <ripemd-of-payment-hash> OP_EQUALVERIFY
        OP_CHECKSIGVERIFY
    OP_ENDIF

The remote node can redeem the HTLC with the scriptsig:

    <remotesig> <payment-preimage>

Either node can use the HTLC-timeout transaction to time out the HTLC once the HTLC is expired, as show below.

#### Received HTLC Outputs

This output sends funds to the remote peer after the HTLC timeout, or to an HTLC-success transaction with a successful payment preimage. The output is a P2WSH, with a witness script:

    <remotekey> OP_SWAP
        OP_SIZE 32 OP_EQUAL
    OP_IF
        # To me via HTLC-success tx.
        OP_HASH160 <ripemd-of-payment-hash> OP_EQUALVERIFY
        2 OP_SWAP <localkey> 2 OP_CHECKMULTISIGVERIFY
    OP_ELSE
        # To you after timeout.
        OP_DROP <locktime> OP_CHECKLOCKTIMEVERIFY OP_DROP
        OP_CHECKSIGVERIFY
    OP_ENDIF

To timeout the htlc, the remote node spends it with the scriptsig:

    <remotesig> 0

To redeem the HTLC, the HTLC-success transaction is used as detailed below.

## HTLC-Timeout and HTLC-Success Transaction
These HTLC transactions are almost identical, except the HTLC-Timeout transaction is timelocked.  This is also the transaction which can be spent by a valid penalty transaction.

* version: 2
* txin: the commitment transaction HTLC output.
* locktime: 0 for HTLC-Success, `htlc-timeout` for HTLC-Timeout.
* txin count: 1
   * txin[0] outpoint: `txid` of the commitment transaction and `output_index` of the matching HTLC output for the HTLC transaction.
   * txin[0] sequence: 0
   * txin[0] script bytes: 0
   * txin[0] witness stack: `<localsig> <remotesig> 0` (HTLC-Timeout) or `<localsig> <remotesig> <payment-preimage>` (HTLC-success).
* txout count: 1
   * txout[0] amount: the HTLC amount minus fees (see below)
   * txout[0] script: version 0 P2WSH with witness script as below.

The witness script for the output is:

    OP_IF
        # Penalty transaction
        <revocation-pubkey>
    OP_ELSE
        `to-self-delay`
        OP_CSV
        OP_DROP
        <local-delayedkey>
    OP_ENDIF
    OP_CHECKSIG

To spend this via penalty, the remote node uses a witness stack `<revocationsig> 1` and to collect the output the local node uses an input with nSequence `to-self-delay` and a witness stack `<local-delayedsig> 0`

# Key Derivation

Each commitment transaction uses a unique set of keys; `<localkey>` and `<remotekey>`.  The HTLC-success and HTLC-timeout transactions use `<local-delayedkey>` and `<revocationkey>`.  These are changed every time depending on the
`per-commitment-point`.

Keys change because of the desire for trustless outsourcing of
watching for revoked transactions; a "watcher" should not be able to
determine what the contents of commitment transaction is, even if
given the transaction ID to watch for and can make a resonable guess
as to what HTLCs and balances might be included.  Nonetheless, to
avoid storage for every commitment transaction, it can be given the
`per-commit-secret` values (which can be stored compactly) and the
`revocation-basepoint` and `delayed-payment-basepoint` to regnerate
the scripts required for the penalty transaction: it need only be
given (and store) the signatures for each penalty input.

Changing the `<localkey>` and `<remotekey>` every time ensures that commitment transaction id cannot be guessed: Every commitment transaction uses one of these in its output script.  Splitting the `<local-delayedkey>` which is required for the penalty transaction allows that to be shared with the watcher without revealing `<localkey>`; even if both peers use the same watcher, nothing is revealed.

Finally, even in the case of normal unilateral close, the HTLC-success
and/or HTLC-timeout transactions do not reveal anything to the
watcher, as it does not know the corresponding `per-commit-secret` and
cannot relate the `<local-delayedkey>` or `<revocationkey>` with
their bases.

For efficiency, keys are generated from a series of per-commitment secrets which are generated from a single seed, allowing the receiver to compactly store them (see [below](#efficient-per-commitment-secret-storage)).

## `localkey`, `remotekey`, `local-delayedkey` and `remote-delayedkey` Derivation

These keys are simply generated by addition from their base points:

	pubkey = basepoint + SHA256(per-commit-point || basepoint)*G

The `localkey` uses the local node's `payment-basepoint`, `remotekey`
uses the remote node's `payment-basepoint`, the `local-delayedkey`
uses the local node's `delayed-payment-basepoint`, and the
`remote-delayedkey` uses the remote node's
`delayed-payment-basepoint`.

The correspoding private keys can be derived similarly if the basepoint
secrets are known (ie. `localkey` and `local-delayedkey` only):

    secretkey = basepoint-secret + SHA256(basepoint || commit-number)

## `revocationkey` Derivation

The revocationkey is a blinded key: the remote node provides the base,
and the local node provides the blinding factor which it later
reveals, so the remote node can use the secret revocationkey for a
penalty transaction.

The `per-commit-point` is generated using EC multiplication:

	per-commit-point = per-commit-secret * G

And this is used to derive the revocation key from the remote node's
`revocation-basepoint`:

	revocationkey = revocation-basepoint * SHA256(revocation-basepoint || per-commit-point) + per-commit-point*SHA256(per-commit-point || revocation-basepoint)

This construction ensures that neither the node providing the
basepoint nor the node providing the per-commit-point can know the
private key without the other node's secret.

### Per-commitment Secret Requirements

A node MUST select an unguessable 256-bit seed for each connection,
and MUST NOT reveal the seed.  Up to 2^48-1 per-commitment secrets can be
generated; the first secret used MUST be index 281474976710655, and
then the index decremented.

The I'th secret P MUST match the output of this algorithm:

    generate_from_seed(seed, I):
        P = seed
        for B in 0 to 47:
            if B set in I:
                flip(B) in P
                P = SHA256(P)
        return P

Where "flip(B)" alternates the B'th least significant bit in the value P.

The receiving node MAY store all previous per-commitment secrets, or
MAY calculate it from a compact representation as described below.

### Efficient Per-commitment Secret Storage

The receiver of a series of secrets can store them compactly in an
array of 49 (value,index) pairs.  This is because given a secret on a
2^X boundary, we can derive all secrets up to the next 2^X boundary,
and we always receive secrets in descending order starting at
0xFFFFFFFFFFFF.

In binary, it's helpful to think of any index in terms of a *prefix*,
followed by some trailing zeroes.  You can derive the secret for any
index which matches this *prefix*.

For example, secret 0xFFFFFFFFFFF0 allows us to derive secrets for
0xFFFFFFFFFFF1 through 0xFFFFFFFFFFFF inclusive. Secret 0xFFFFFFFFFF08
allows us to derive secrets 0xFFFFFFFFFF09 through 0xFFFFFFFFFF0F
inclusive.

We do this using a slight generalization of `generate_from_seed` above:

    # Return I'th secret given base secret whose index has bits..47 the same.
    derive_secret(base, bits, I):
        P = base
        for B in 0 to bits:
            if B set in I:
                flip(B) in P
                P = SHA256(P)
        return P

We need only save one secret for each unique prefix; in effect we can
count the number of trailing zeros, and that determines where in our
storage array we store the secret:

    # aka. count trailing zeroes
    where_to_put_secret(I):
		for B in 0 to 47:
			if testbit(I) in B == 1:
				return B
        # I = 0, this is the seed.
		return 48

We also need to double-check that all previous secrets derive correctly,
otherwise the secrets were not generated from the same seed:

    insert_secret(secret, I):
		B = where_to_put_secret(secret, I)

        # This tracks the index of the secret in each bucket as we traverse.
		for b in 0 to B:
			if derive_secret(secret, B, known[b].index) != known[b].secret:
				error The secret for I is incorrect
				return

        # Assuming this automatically extends known[] as required.
		known[B].index = I
		known[B].secret = secret

Finally, if we are asked to derive secret at index `I`, we need to
figure out which known secret we can derive it from.  The simplest
method is iterating over all the known secrets, and testing if we
can derive from it:

	derive_old_secret(I):
		for b in 0 to len(secrets):
		    # Mask off the non-zero prefix of the index.
		    MASK = ~((1 << b)-1)
			if (I & MASK) == secrets[b].index:
				return derive_secret(known, i, I)
	    error We haven't received index I yet.

This looks complicated, but remember that the index in entry `b` has
`b` trailing zeros; the mask and compare is just seeing if the index
at each bucket is a prefix of the index we want.

# References

# Authors

FIXME




