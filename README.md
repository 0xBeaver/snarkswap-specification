# Snarkswap protocol v0.1

## AMM

### Uniswap compatibility

* Protocol should implement all Uniswap AMM features.
* Users can run `swapInTheDark` by consuming 2 notes and creating 2 notes with the below SNARK condition.
* `swapInTheDark` darkens the token pair's updated reserve ratio and freeze `mint`, `sync`, `skim`, and `swap` functions.
* The dark trader should stake token to use `swapInTheDark` function.
* Anyone can undarken the pair in anytime.
* Any undarkener can take the staked token but the darkener has 10 minutes of prioritized period.
* Fee is proportional to the difficulty.

## ZKP

### Uniswap math

Snarkswap is completely an extended version of Uniswap. It uses the same mathematics but hide some values. Under the zk-SNARK, the signals will follow the Uniswap K math:

**Statement 1: Uniswap K**
```
fee0 = isToken0In ? token0In*fee : 0
fee1 = isToken1In ? token1In*fee : 0

newReserve0' = newReserve0 - fee0
newReserve1' = newReserve1 - fee1
reserve0 * reserve1 <= newReserve0' * newReserve1'
```

### hRatio is the SNARK to hunt

Instead of revealing the real reserve values `reserve0`, `reserve1`, trader reveals `hRatio` which is a hash of `reserve0`, `reserve1`, and a `salt`. Then others should guess them.

**Statement 2: hRatio**
```
hRatio = hash(reserve0, reserve1, salt)
```

Here using the poseidon hash looks a good option for snark.

### Hint is the mask and hReserves.

Hunting the `hRatio` without any information is absolutely an impossible task. Therefore, we should give some hints for the hunting and they are the mask and hReserves. You can just think like "If the n-th bit is masked, that bit of `hReserve` may differ to `reserve`. By ther way, otherwise `hReserve` will be same with `reserve`". As a result, this adds some uncertainty to confuse the front-runner. 

Mask is a 224 bits value and the first 112 bits are `mask0` for `reserve0` and the last 112 bits are `mask1` for `reserve1`.  Here's the example of uncertainty mask. We'll call the masked reserve as `hReserve` as it's a hiding.

```
newReserve0 = '00011101101111...001101'  |  newReserve1 = '00000101101111...101'
mask0       = '00011110000000...000001'  |  mask1       = '00000111100000...000'
---------------------------------------- |  ------------------------------------
hReserve0   = '00000101101111...001100'  |  hReserve1 s = '00000000001111...101'

```
**Statement 3: Mask**
```
(newReserve0 ^ hReserve0) | mask0 == mask0 
(newReserve1 ^ hReserve1) | mask1 == mask1
```



### No money is printed out during the swap.

To hide the swap details, the fund transfer should be pended until the ratio gets revealed. So, Snarkswap uses note pool and a swap transaction reveals the note hashes instead of their details.

**Statement 4: Note**
```
notehash = hash(tokenAddress, tokenAmount, publicKey.x, publicKey.y, salt)
```

A swap consumes 2 notes and creates 2 outputs. To make sure there was no money printed out, the updated reserve should satisfy the following condition

**Statement 5: Fund is safu**

```
token0AmountIn = [sourceA, sourceB]
    .filter(note => note.tokenAddress == token0)
    .reduce((acc, note) => acc + note.tokenAmount)
token1AmountIn = [sourceA, sourceB]
    .filter(note => note.tokenAddress == token1)
    .reduce((acc, note) => acc + note.tokenAmount)
token0AmountOut = [outputA, outputB]
    .filter(note => note.tokenAddress == token0)
    .reduce((acc, note) => acc + note.tokenAmount)
token1AmountOut = [outputA, outputB]
    .filter(note => note.tokenAddress == token1)
    .reduce((acc, note) => acc + note.tokenAmount)
 
reserve0 + token0AmountIn - token0AmountOut == newReserve0
reserve1 + token1AmountIn - token1AmountOut == newReserve1
```

**Statement 6: EdDSA**

To consume the notes, it should have a valid EdDSA signature for the transaction hash.

```
txHash = hash(
    sourceAHash,
    sourceBHash,
    outputAHash,
    outputBHash,
    hRatioSalt // the salt used in calculating hRatio.
)
pubkey == sourceA.pubKey
pubkey == sourceB.pubKey
pubkey == EdDSA(txHash, signature)
```

### Range check
**Statement 7: range check**
```
reserve0 <= (1 << 112)
reserve1 <= (1 << 112)
newReserve0 <= (1<< 112)
newReserve1 <= (1 << 112)
address0 <= (1 << 160)
address1 <= (1 <<160)
mask <= (1<< 224)
sourceA.amount <= (1<< 239) // prevent overflow
sourceB.amount <= (1<< 239) // prevent overflow
outputA.amount <= (1<< 239) // prevent overflow
outputB.amount <= (1<< 239) // prevent overflow
```
