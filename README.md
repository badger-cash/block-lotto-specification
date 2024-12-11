# SIMPLE LEDGER PROTOCOL (SLP) BLOCK LOTTO SPECIFICATION
[Specification](block-lotto.md) for an on-chain lottery game where the amount a player wins is determined by the relationship between a series of numbers chosen by the player (ticket) and the hash of a future block. The lottery is provably fair because, in general, a miner would not withold broadcasting a block in order to "cheat" a player(s), and it is impossible to know the hash of any as-yet unmined block.

# MERKLE DRAW SPECIFICATION
[Specification](merkle-draw.md) for a random, lottery-style, drawing where chances are represented by SHA256 hashes.