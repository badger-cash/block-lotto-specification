![Simple Ledger Protocol](images/SLP-logo-solid-200.png)

# Simple Ledger Protocol (SLP) Block Lotto

#### Version: 0.1
#### Date published: April 15, 2024

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
* `raised_bits` - Maximum valid proof-of-work target `bits` with same encoding as is used in block header
* `player_numbers` - The numbers chosen by the player. Each number is 1 bit and has a value in the range of `0x01` to `0x7f`. Any number outside of this range will make the ticket irredeemable as it breaks minimum encoding script rules. The numbers are serialized

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

* Verify the hash of the block that includes the ticket transaction. This can potentially be performed by using a merkle proof against the block header, but script size limitations currently require another method is used. Currently, the method used is for the authorizer to provide a signed oracle result correlating the ticket transaction hash with the hash of the block the ticket transaction was mined into
* Hash the ticket transaction hash and the block hash together to get the random number for this play instance. This assures that all tickets have unique random numbers generated, even if many tickets have identical `player_numbers` and are mined into the same block
* Validate that `vout 1` of ticket transaction corresponds to `vin 0` of redeem transaction
* Calculate payout amount
* Validate that the outputs of redeem transaction correspond to `redeem_outputs` modifed to reflect proper payout amount

In order to perform these functions, the (P2SH) unlock script of `vin 0` of the redeem transaction must be provided the following elements:

* Raw data of ticket transaction
* Block header for block that ticket transaction was mined into
* Redeem transaction preimage
* Authorizer oracle signature correlating block hash and ticket transaction hash
* Player public key (corresponding to payout address)
* Player signature for `vin 0`

In order to construct a valid redeem transaction, the player will need to calculate the payout, using the data above and the pay table for the game, and then substitute the player payout amount in the redeem transaction SLP MINT OP_RETURN to reflect the winning amount. In the case of a loss, the payout amount is zero, expressed as an 8 bit integer.

## Reference Implementations

### Clients
None currently

### Libraries
None currently

### Sample Transactions

[Ticket Transaction](https://explorer.e.cash/tx/361198ada49c1928e107dd93ab7bac53acbef208b0c0e8e65b4e33c3a02a32b6)

[Redeem/Payout Transaction](https://explorer.e.cash/tx/32a716ffe3c88f6db6316dd513faee30330f938b4a0eff068bacc64890f9f862)

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
    paytable = [1, 5, 7, 16]

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

#### Payout Calculation

This code calculates the payout using the same alogrithm as the script above.

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
        { threshold: 15, multiplier: 2},
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
