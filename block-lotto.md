![Simple Ledger Protocol](images/SLP-logo-solid-200.png)

# Simple Ledger Protocol (SLP) Block Lotto

#### Version: 0.2
#### Date published: October 18, 2026

## Purpose

This specification describes a protocol for an on-chain lottery game where the amount a player wins is determined by the relationship between a series of numbers chosen by the player (ticket) and the hash of a future block. The lottery is provably fair because, in general, a miner would not withold broadcasting a block in order to "cheat" a player(s), and it is impossible to know the hash of any as-yet unmined block.

The payout of winnings is done programatically, in the form of SLP tokens, using the [Simple Ledger Protocol (SLP) Self Mint](https://github.com/badger-cash/slp-self-mint-protocol/blob/a6ca53561af3b0e8574f5dbb4f318791a78915db/selfmint-specification.md). In this way, all necessary transactions can be standardized.

## Background

[Academic studies have concluded](https://eprint.iacr.org/2015/1015.pdf) that the block hashes generated through Nakamoto Consensus can serve, effectively, as a public source for random number generation of the type needed in games such as lottery. Lottery-style games, where players pick numbers before a drawing, are particularly susceptible to cheating and manipulation by the entity offering the game and drawing the winning numbers. It is for this reason that major state-run lotteries still pick their numbers with mechanical devices that spin physical balls with numbers printed on them, and do the drawings on live television.

The less a lottery operator pays out in winnings, the greater the profit for the operator. Therefore, the operator has an incentive to manipulate the drawing of the winning numbers to his benefit. Using (a portion of) the hash of a future block as the winning numbers ensures that manipulation of the winning numbers in impossible. Encoding such a game in Bitcoin Script, along with a known pay table, ensures that the resulting lottery is provably fair, with a degree of provable fairness that cannot be achieved by any other existing lottery system.

## Specification

### Considerations

This protocol uses the data of an entire transaction as an element in the unlocking script (scripSig) of another transaction. The primary considerations in the design of the Block Lotto protocol are: the limit on the number of script elements allowed in an unlocking script; and the maximum allowed size (in bytes) of any given element in an unlocking script.

### Ticket Transaction

The first necessary transaction represents a lottery ticket sent from the game operator ("authorizer") to the player. This ticket transaction defines the required parameters for play and represents a single play instance. The parameters are encoded in different locations within the transaction.

#### Ticket Transaction Inputs

The first input (`vin 0`) of the ticket transaction must be sent from the P2PKH address associated with the same public key used to sign the oracle data for the result payout. This allows the public key element of the unlocking script of the ticket transaction to be used programatically in the redemption/payout transaction.

Any additional inputs are allowed, but additional inputs may increase the size of the ticket transaction in a way that makes the ticket unredeemable, becasue it exceeds maximum script element size limits in the redeem transaction.

#### Ticket Transaction Outputs

A ticket transaction requires two outputs. 

##### OP_RETURN Output
The first output `vout 0` is an OP_RETURN. 

First a signature (`authsig`) must be generated. The public key used is the authorizer public key used to sign `vin 0`. The message signed is the serialization of:

* `stamp_outpoint` - The outpoint of the UTXO used in `vin 0`
* `redeem_outputs` - The serialized outputs of a redeem transaction with the maximum payout to the player. This can include an output for an affiliate that generated the ticket sale
* `raised_bits` - Maximum valid proof-of-work target `bits` with same encoding as is used in block header. This is the work floor the redeem script demands of the block containing the ticket. It must be close to the network difficulty at issue: a floor far easier than the network would let the issuer redeem the ticket against a block it forged itself. Covenants V1 and V2 add `0x040000` to the tip's compact bits (1.03x to 6x easier, depending on the mantissa); V3 doubles the tip's target (always one bit easier). A buyer verifies the floor against the chain tip before accepting a ticket (see Covenant Versions)
* `player_numbers` - The numbers chosen by the player. Each number is 1 byte and has a value in the range of `0x01` to `0x7f`. Any number outside of this range will make the ticket irredeemable as it breaks minimum encoding script rules. The numbers are serialized. The numbers take part in the outcome only through the ticket's txid, which they are part of

The data in the OP_RETURN script is a serialization of:

* `redeem_outputs`
* `raised_bits`
* `player_numbers`
* `authsig`

This output, when the ticket transaction is used in the redeem/payout script, serves as a self-mint authorization code.

##### Self-Mint Stamp Output

The second output in the ticket transaction `vout 1` is a self-mint stamp with a scriptPubKey corresponding to the mint vault for the token that will be paid out upon a win (and/or to an affiliate). This output, as is typical of a self-mint stamp, should contain enough value such that the resulting UTXO can be used as the sole input in the redeem/payout transaction.

### Redeem Transaction

Redemption of the ticket transaction, resulting in a payout to the player of zero or more tokens, follows the [Simple Ledger Protocol (SLP) Self Mint](https://github.com/badger-cash/slp-self-mint-protocol/blob/a6ca53561af3b0e8574f5dbb4f318791a78915db/selfmint-specification.md) specification. The self-mint unlock script performs the following validation functions:

* Verify the hash of the block that includes the ticket transaction. Covenants V1 and V2 rely on the authorizer to provide a signed oracle result correlating the ticket transaction hash with the hash of the block the ticket transaction was mined into. Covenant V3 verifies a Merkle branch from the ticket's txid to the merkle root in the block header instead, and needs no oracle
* Verify that the block header meets the `raised_bits` target committed in the ticket
* Hash the ticket transaction hash and the block hash together to get the random number for this play instance. This assures that all tickets have unique random numbers generated, even if many tickets have identical `player_numbers` and are mined into the same block
* Validate that `vout 1` of ticket transaction corresponds to `vin 0` of redeem transaction, and that it is the only input
* Calculate payout amount
* Validate that the outputs of redeem transaction correspond to `redeem_outputs` modifed to reflect proper payout amount

In order to perform these functions, the (P2SH) unlock script of `vin 0` of the redeem transaction must be provided the following elements:

* Raw data of ticket transaction
* Block header for block that ticket transaction was mined into
* Redeem transaction preimage (V1), or its first 4 bytes and the bytes from `hashSequence` through `sequence` (V2, V3): the covenant rebuilds the rest around the `hashPrevouts` and `hashOutputs` it expects
* Authorizer oracle signature correlating block hash and ticket transaction hash (V1, V2), or the Merkle branch of the ticket transaction (V3)
* Player public key (corresponding to payout address)
* Player signature for `vin 0`

In order to construct a valid redeem transaction, the player will need to calculate the payout, using the data above and the pay table for the game, and then substitute the player payout amount in the redeem transaction SLP MINT OP_RETURN to reflect the winning amount. In the case of a loss, the payout amount is zero, expressed as an 8 bit integer.

#### Payout Table

The random number is `HASH256(ticket_txid || block_hash)`. Its last four bytes each contribute their low nibble to a sum `s` in the range 0 to 60. The payout is the maximum payout committed in `redeem_outputs` divided by:

| `s` | divisor |
|---|---|
| 0 | 1 (maximum) |
| 1 to 4 | 2 |
| 5 to 6 | 4 |
| 7 to 11 | 8 |
| 12 to 35 | 16 |
| 36 to 60 | no payout |

### Covenant Versions

Three redeem scripts implement the protocol. They share the ticket format, the oracle-free V3 included, so the same ticket data can be issued into any of them. Each has its own P2SH address and therefore its own SLP token, since the token's genesis commits to the mint vault's script hash. Every redeem script is bound by the 201 non-push opcode limit and the 1650 byte standard scriptSig limit.

#### V1

156 non-push opcodes. The oracle signature covers `ticket_txid || block_hash`. Unlocking data, bottom of stack first: player signature, player public key, oracle signature, preimage, block header, raw ticket transaction.

#### V2

88 non-push opcodes, same oracle and outcomes as V1. The savings: the paytable is a 61 byte lookup table indexed by `s` instead of a chain of conditionals; `s` is computed as `(random[28..32) AND 0f0f0f0f) mod 255`, which equals the sum of the four low nibbles because 256 is 1 modulo 255; and the preimage is rebuilt by the covenant from the pieces the redeemer supplies, so that the closing signature check compares `hashPrevouts` and `hashOutputs` without separate equality checks. Unlocking data, bottom of stack first: player public key, player signature without its hashtype byte, preimage bytes 0 to 4, preimage bytes 36 to (length - 40), oracle signature, block header, raw ticket transaction. The sighash type is fixed to `ALL | FORKID` and the locktime to zero.

#### V3 (oracle-free)

199 non-push opcodes. In place of the oracle signature the redeemer supplies a Merkle branch from the ticket's txid to the `merkleRoot` of the block header, up to 15 levels, so a V3 ticket is redeemable from any block holding at most 32,768 transactions. An `OP_CODESEPARATOR` before the closing signature check means the redeem script itself is not part of the preimage the redeemer supplies, which is what keeps a 15 level proof under the scriptSig limit (1590 bytes at full depth).

Unlocking data, bottom of stack first: player public key, player signature without its hashtype byte, preimage bytes 0 to 4, preimage bytes 36 to (length - 40), block header, then the proof from level 15 down to level 1, then the raw ticket transaction. Each present level is `prefix suffix flag`, where one of `prefix`/`suffix` is the 32 byte sibling hash and the other is empty, placing the sibling on the correct side, and `flag` is `OP_1`. An absent level is a single `OP_0`. Level 1 has no flag: a block holding the ticket also holds a coinbase, so the branch is never empty. The covenant computes `HASH256(prefix || node || suffix)` for each present level and requires the result to equal the header's merkle root.

The depth limit is a property of the covenant, not of the chain. The largest block eCash has produced since its fork holds a few hundred transactions; a 32 MB block of nothing but 413 byte tickets would hold 77,481, which is beyond 15 levels. Tickets are the only traffic that could fill a block that far, and they are issued by the authorizer, which therefore caps the number of V3 tickets it issues against one chain tip (30,000 in the reference implementation) so that no block can be too deep for its own tickets.

Because no oracle is involved, the only statement about the chain that the authorizer still signs is `raised_bits`. The buyer verifies it: on receiving a ticket, the reference client fetches the current tip and refuses a ticket whose floor is more than ten times easier than the tip's target (V3 tickets are issued at exactly two times; V1 and V2 tickets fall between 1.03 and 9 times).

## Reference Implementations

### Clients
None currently

### Libraries
None currently

### Sample Transactions

[Ticket Transaction](https://explorer.e.cash/tx/faf3c10450560081f9080d86488c5195932b810062fb14153f0ccaf4a99d2ad1)

[Redeem/Payout Transaction](https://explorer.e.cash/tx/52a6cdc351595ff405c073cbd3515c7b9188a37849f9acd72fbbb96eebd2efa8)

### Code

Reference code for creating covenant scripts using [bcash](https://github.com/badger-cash/bcash) (Badger.cash fork) JavaScript library

#### Redeem Mint Vault

```js
const buildOutScript = (authPubKey) => {

    const script = new bcash.Script()

        .pushSym('3dup')
        .pushSym('hash256')
        .pushSym('dup')
        .pushSym('rot')
        .pushSym('hash256')
        .pushSym('cat')
        .pushInt(6)
        .pushSym('roll')
        .pushSym('over')
        .pushData(authPubKey)
        .pushSym('dup')
        .pushSym('toaltstack')
        .pushSym('checkdatasigverify') // Verify tx+block signature
        .pushSym('hash256') // Random number

        // Begin dissecting preimage
        .pushSym('rot')
        .pushInt(4)
        .pushSym('split')
        .pushSym('nip')
        .pushInt(32)
        .pushSym('split')
        .pushInt(3)
        .pushSym('roll')
        .pushData(Buffer.from('01000000', 'hex'))
        .pushSym('cat')
        .pushSym('hash256')
        .pushSym('rot')
        .pushSym('equalverify') // Validate single input is from index 0 of ttx outputs
        .pushSym('size')
        .pushInt(40)
        .pushSym('sub')
        .pushSym('split')
        .pushSym('nip')
        .pushInt(32)
        .pushSym('split')
        .pushSym('drop') // Output hash

        // Begin dissecting ttx
        .pushSym('rot')
        .pushInt(5)
        .pushSym('split')
        .pushSym('nip')
        .pushInt(36)
        .pushSym('split')
        .pushInt(1)
        .pushSym('split')
        .pushSym('swap')
        .pushSym('split')
        .pushSym('nip')
        .pushInt(13)
        .pushSym('split')
        .pushSym('nip')
        .pushInt(1)
        .pushSym('split')
        .pushSym('swap')
        .pushData(Buffer.from('00', 'hex'))
        .pushSym('cat')
        .pushSym('split')
        .pushSym('drop')
        .pushInt(3)
        .pushSym('split')
        .pushSym('nip')
        .pushInt(147)
        .pushSym('split')
        .pushSym('rot')
        .pushInt(2)
        .pushSym('pick')
        .pushSym('cat')
        .pushSym('fromaltstack')
        .pushSym('checkdatasigverify')
        .pushInt(139)
        .pushSym('split')
        .pushInt(4)
        .pushSym('split')

        // Begin random number modification
        .pushInt(3)
        .pushSym('split')
        .pushSym('swap')
        .pushInt(2)
        .pushSym('split')
        .pushSym('swap')
        .pushInt(1)
        .pushSym('split')
        .pushInt(7)
        .pushSym('roll')

        // Modulo calculate and sum that solves for signs
        .pushInt(31)
        .pushSym('split')
        .pushSym('rot')
        .pushSym('cat')
        .pushInt(16)
        .pushSym('mod')
        .pushSym('swap')

        .pushInt(30)
        .pushSym('split')
        .pushInt(3)
        .pushSym('roll')
        .pushSym('cat')
        .pushInt(16)
        .pushSym('mod')
        .pushSym('swap')

        .pushInt(29)
        .pushSym('split')
        .pushInt(4)
        .pushSym('roll')
        .pushSym('cat')
        .pushInt(16)
        .pushSym('mod')
        .pushSym('swap')

        .pushInt(28)
        .pushSym('split')
        .pushSym('nip')
        .pushInt(4)
        .pushSym('roll')
        .pushSym('cat')
        .pushInt(16)
        .pushSym('mod')

        .pushSym('add')
        .pushSym('add')
        .pushSym('add')

        .pushSym('rot')
        .pushInt(73)
        .pushSym('split')
        .pushSym('swap')
        .pushInt(65)
        .pushSym('split')
        .pushSym('reversebytes')
        .pushSym('bin2num')
        .pushInt(3)
        .pushSym('roll')
        .pushSym('tuck')

    // Payout Calculation
    paytable = [1, 5, 7, 12]

    for (let i = 0; i < paytable.length; i++) {

        script.pushInt(paytable[i])
            .pushSym('greaterthanorequal')
            .pushSym('if')
            .pushInt(2)
            .pushSym('div')
            .pushSym('endif')
        // Do OP_OVER unless final and then do OP_SWAP
        if (i === paytable.length - 1) {
            script.pushSym('swap')
        } else {
            script.pushSym('over')
        }
    }

    script.pushInt(36)
        .pushSym('greaterthanorequal')
        .pushSym('if')
        .pushSym('drop')
        .pushData(Buffer.from('00', 'hex'))
        .pushSym('endif')

        .pushInt(8)
        .pushSym('num2bin')
        .pushSym('reversebytes')
        .pushSym('rot')
        .pushSym('cat')
        .pushSym('cat')
        .pushSym('hash256')
        .pushSym('rot')
        .pushSym('equalverify')
        .pushInt(3)
        .pushSym('split')
        .pushSym('dup')
        .pushInt(32)
        .pushSym('swap')
        .pushSym('sub')
        .pushData(Buffer.from('00', 'hex'))
        .pushSym('swap')
        .pushSym('num2bin')
        .pushInt(3)
        .pushSym('roll')
        .pushSym('hash256')
        .pushSym('rot')
        .pushSym('split')
        .pushSym('rot')
        .pushSym('equalverify')
        .pushSym('size')
        .pushInt(3)
        .pushSym('sub')
        .pushSym('split')
        .pushSym('nip')
        .pushData(Buffer.from('00', 'hex'))
        .pushSym('cat')
        .pushSym('bin2num')
        .pushSym('swap')
        .pushData(Buffer.from('00', 'hex'))
        .pushSym('cat')
        .pushSym('bin2num')
        .pushSym('lessthanorequal')
        .pushSym('verify')

    // Begin preimage validation
    // Anyone can spend
    .pushSym('sha256')
    .pushSym('3dup')
    .pushSym('rot')
    .pushSym('size')
    .pushSym('1sub')
    .pushSym('split')
    .pushSym('drop')
    .pushSym('swap')
    .pushSym('rot')
    .pushSym('checkdatasigverify')
    
    .pushSym('drop')
    .pushSym('checksig');
    
    // compile and return
    return script.compile();
}

```

#### Redeem Mint Vault (V3, oracle-free)

```js
const { Script } = bcash

const buildOutScriptV3 = (authPubKey) => {

    // Divisor applied to the max payout for each nibble sum 0..60. Sums of 36
    // and above are zeroed before the lookup, so those entries are never read.
    const divisors = Buffer.alloc(61, 1)
    for (let sum = 1; sum < 36; sum++) {
        if (sum >= 12) divisors[sum] = 16
        else if (sum >= 7) divisors[sum] = 8
        else if (sum >= 5) divisors[sum] = 4
        else divisors[sum] = 2
    }

    // Unlocking data, bottom to top:
    //   pk, sig (no hashtype byte), preimage[0..4), preimage[36..len-40),
    //   header,
    //   level 15: prefix suffix flag ... level 2: prefix suffix flag,
    //   level 1: prefix suffix,
    //   ticket
    // A present level carries the 32-byte sibling as prefix or suffix (the
    // other empty) and flag OP_1; an absent level is a single OP_0. Level 1 is
    // unconditional: a block holding the ticket has at least two transactions.
    // The preimage is signed from the OP_CODESEPARATOR onward, so the script
    // itself is not part of the preimage the redeemer supplies.
    const script = new Script()

        // txid, then the Merkle branch up to the header's root
        .pushSym('dup')
        .pushSym('hash256')                 // ... hdr levels tkt txid
        .pushSym('swap')
        .pushSym('toaltstack')              // ... hdr levels txid          alt: tkt
        .pushSym('swap')
        .pushSym('cat')
        .pushSym('cat')
        .pushSym('hash256')                 // level 1: prefix||txid||suffix

    for (let level = 2; level <= 15; level++) {
        script.pushSym('swap')
            .pushSym('if')
            .pushSym('swap')
            .pushSym('cat')
            .pushSym('cat')
            .pushSym('hash256')
            .pushSym('endif')
    }

    script.pushSym('over')                  // pk sig v mid hdr root' hdr
        .pushInt(36)
        .pushSym('split')
        .pushSym('nip')
        .pushInt(32)
        .pushSym('split')
        .pushSym('drop')
        .pushSym('equalverify')             // pk sig v mid hdr
        .pushSym('hash256')                 // pk sig v mid bh
        .pushSym('fromaltstack')
        .pushSym('dup')
        .pushSym('hash256')                 // pk sig v mid bh tkt txid
        .pushSym('dup')
        .pushData(Buffer.from('01000000', 'hex'))
        .pushSym('cat')
        .pushSym('hash256')                 // ... bh tkt txid hpExp
        .pushInt(4)
        .pushSym('roll')                    // pk sig v bh tkt txid hpExp mid
        .pushInt(4)
        .pushSym('roll')                    // pk sig v tkt txid hpExp mid bh
        .pushInt(3)
        .pushSym('roll')
        .pushSym('over')
        .pushSym('cat')
        .pushSym('hash256')                 // pk sig v tkt hpExp mid bh rnd

        // Sum of the low nibbles of the last four random bytes (256 = 1 mod 255)
        .pushInt(28)
        .pushSym('split')
        .pushSym('nip')
        .pushData(Buffer.from('0f0f0f0f', 'hex'))
        .pushSym('and')
        .pushSym('bin2num')
        .pushInt(255)
        .pushSym('mod')                     // pk sig v tkt hpExp mid bh sum

        // Ticket: stamp outpoint, auth code payload and its signature
        .pushInt(4)
        .pushSym('roll')                    // pk sig v hpExp mid bh sum tkt
        .pushInt(5)
        .pushSym('split')
        .pushSym('nip')
        .pushInt(36)
        .pushSym('split')                   // ... sum prevout rest
        .pushInt(1)
        .pushSym('split')
        .pushSym('swap')
        .pushSym('split')
        .pushSym('nip')                     // skip the vin 0 scriptSig by its length byte
        .pushInt(17)
        .pushSym('split')
        .pushSym('nip')                     // skip sequence, vout count, value, script length, 6a4cda
        .pushInt(147)
        .pushSym('split')                   // ... sum prevout payload authsig+tail
        .pushSym('size')
        .pushInt(36)
        .pushSym('sub')
        .pushSym('split')
        .pushSym('drop')                    // ... sum prevout payload authsig
        .pushSym('rot')
        .pushInt(2)
        .pushSym('pick')
        .pushSym('cat')
        .pushData(authPubKey)
        .pushSym('checkdatasigverify')      // ... bh sum payload
        .pushInt(139)
        .pushSym('split')
        .pushInt(4)
        .pushSym('split')
        .pushSym('drop')                    // pk sig v hpExp mid bh sum outputs bits

        // Proof of work: blockhash >> 8*(exponent-3) must not exceed the mantissa
        .pushInt(3)
        .pushSym('split')
        .pushInt(3)
        .pushSym('sub')                     // ... bh sum outputs mantissa exp-3
        .pushInt(4)
        .pushSym('roll')
        .pushSym('swap')
        .pushSym('split')
        .pushSym('nip')
        .pushSym('bin2num')                 // ... sum outputs mantissa hashTop
        .pushSym('swap')
        .pushData(Buffer.from('00', 'hex'))
        .pushSym('cat')
        .pushSym('bin2num')
        .pushSym('lessthanorequal')
        .pushSym('verify')                  // pk sig v hpExp mid sum outputs

        // Payout
        .pushInt(65)
        .pushSym('split')
        .pushInt(8)
        .pushSym('split')                   // ... sum prefix max suffix
        .pushSym('swap')
        .pushSym('reversebytes')
        .pushSym('bin2num')                 // ... sum prefix suffix max
        .pushInt(3)
        .pushSym('roll')                    // ... prefix suffix max sum
        .pushSym('tuck')
        .pushInt(36)
        .pushSym('greaterthanorequal')
        .pushSym('if')
        .pushSym('drop')
        .pushInt(0)
        .pushSym('endif')                   // ... prefix suffix sum max'
        .pushData(divisors)
        .pushSym('rot')
        .pushSym('split')
        .pushSym('nip')
        .pushInt(1)
        .pushSym('split')
        .pushSym('drop')                    // ... prefix suffix max' divisor
        .pushSym('div')                     // ... prefix suffix payout
        .pushInt(8)
        .pushSym('num2bin')
        .pushSym('reversebytes')
        .pushSym('swap')
        .pushSym('cat')
        .pushSym('cat')
        .pushSym('hash256')                 // pk sig v hpExp mid hashOutputs

        // Reassemble the preimage: locktime 0, SIGHASH_ALL | SIGHASH_FORKID
        .pushData(Buffer.from('0000000041000000', 'hex'))
        .pushSym('cat')
        .pushSym('cat')
        .pushSym('cat')
        .pushSym('cat')                     // pk sig preimage

        // The preimage must be this transaction's; scriptCode starts here
        .pushSym('codeseparator')
        .pushSym('sha256')
        .pushSym('over')
        .pushData(Buffer.from('41', 'hex'))
        .pushSym('cat')
        .pushInt(3)
        .pushSym('pick')
        .pushSym('checksigverify')
        .pushSym('rot')
        .pushSym('checkdatasig');

    return script.compile();
}
```

#### Payout Calculation

This code calculates the payout using the same alogrithm as the scripts above.

```js
const { 
    Hash256: hash256
} = require('bcrypto');
const { U64 } = require('n64');

const calculatePayout = (ttxHash, blockHash, playerChoiceBytes, maxPayoutBufBE) => {

    const combineHashes = Buffer.concat([ttxHash, blockHash]);
    const randomNumber = hash256.digest(combineHashes);
    let payoutNum = parseInt(U64.fromBE(maxPayoutBufBE).toString());

    let modSum = 0;


    for (let i = 0; i < playerChoiceBytes.length; i++) {
        
        const pOffset = 3 - i
        const playerByte = playerChoiceBytes.slice(pOffset, pOffset + 1)
        const offset = 31 - i
        const randomByte = randomNumber.slice(offset, offset + 1)
        const numBuf = Buffer.concat([randomByte, playerByte])
        const number = numBuf.readInt16LE()
        modSum += number % (4 * playerChoiceBytes.length)
    }

    // Paytable zero index pays max amount, the rest divide by 2 and greater than final pays zero
    const playerWinningsTier = [
        { threshold: 0, multiplier: 16},
        { threshold: 4, multiplier: 8},
        { threshold: 6, multiplier: 4},
        { threshold: 11, multiplier: 2},
        { threshold: 35, multiplier: 1},
    ];
    const paytable = playerWinningsTier.map(obj => obj.threshold)
    for (let i = 0; i < paytable.length; i++) {
        if (modSum > paytable[i]) {
            if (i === paytable.length - 1)
                payoutNum = 0;
            else
                payoutNum = payoutNum / 2;
        }
    }

    const actualPayout = U64.fromInt(payoutNum);
    return actualPayout.toBE(Buffer)
}
```

## Changelog

### 0.2

Add the Payout Table and the Covenant Versions section describing V1, V2 and the oracle-free V3, including V3's unlocking data, depth limit, issuance cap and the buyer's verification of `raised_bits`. Correct the reference paytable from `[1, 5, 7, 16]` with a `2x` threshold of 15 to `[1, 5, 7, 12]` and 11, which is what the deployed covenant has always used. Correct `player_numbers` from "1 bit" to 1 byte and state that the numbers reach the outcome only through the txid. Add the V3 reference script.
