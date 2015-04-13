# Decoding a transaction

## Preface

These instructions assume transaction data returned from the bitcoin core reference client as of version 0.9


## Process vouts

To decode a transaction, begin by iterating over each output (vout).  When finished iterating over all the vouts, we will have 2 pieces of data:

1. A destination addresses or an array of destination addresses
2. Some binary data called `counterpartydata`

For each vout, do the following:

### Step 1: Determine the type of output.

Look at the last operator in the script stack of the output.  This is found in `scriptPubKey.asm`.  This must be one of:

1. [OP_RETURN](#decoding-op_return)
2. [OP_CHECKSIG](#decoding-op_checksig)
3. [OP_CHECKMULTISIG](#decoding-op_checkmultisig)

### Step 2: Interpret the data for this output

Decode the data for this output.  See [decoding data](#decoding-a-vout).

The decoded data will either be a destination address or a `datachunk`.

If the decoded data is a destination address, then add this destination address to `destinations`.

If the decoded data is a `datachunk`, then append the `datachunk` to the `counterpartydata`.


## Decoding a vout

The decoded data will either be a destination address or a `datachunk`.

Begin by looking at the last operator in the script stack of the output.  This is found in `scriptPubKey.asm`.  Based on the type of operator, the data will be decoded in one of three ways.

### Decoding OP_RETURN

An OP_RETURN transaction contains the counterparty transaction data entirely encoded in the 40 byte OP_RETURN transaction.

To decode the data, do the following

1. Deobfuscate the OP_RETURN data using the ARC4 cipher.
2. Verify that the data chunk begins with CNTRPRTY

The data chunk beginning with the 9th byte (after CNTRPRTY) is the decoded data.

#### Example decoding OP_RETURN

`this example is not written yet`


### Decoding OP_CHECKSIG

not written yet


### Decoding OP_CHECKMULTISIG

An OP_CHECKMULTISIG transaction contains either 2 or 3 pubkeys in its script.  The first 1 or 2 pubkeys hold the encoded transaction data.  The last pubkey is a destination pubkey and does not contain counterparty data.

To decode the data, do the following

1. Make a combined data chunk.  Combine the 1 or 2 data pubkeys by strippping the first and last bytes of each and appending them together as binary data.
2. Deobfuscate this combined data chunk using the ARC4 cipher.
3. Strip the first byte from the combined data chunk
4. Verify that combined data chunk, begins with CNTRPRTY

The combined data chunk beginning with the 9th byte (after CNTRPRTY) is the decoded data.


#### Example decoding OP_CHECKMULTISIG

Given this hexadecimal representation of the script (ASM):

```
1 
0276d539826e5ec10fed9ef597d5bfdac067d287fb7f06799c449971b9ddf9fec6 
02af7efeb1f7cf0d5077ae7f7a59e2b643c5cd01fb55221bf76221d8c8ead92bf0 
02f4aef682535628a7e0492b2b5db1aa312348c3095e0258e26b275b25b10290e6 
3 
OP_CHECKMULTISIG
```

We will do the following steps:

1) Make a combined data chunk

Ignore the first and last bytes of each pubkey hash

```
02|76d539826e5ec10fed9ef597d5bfdac067d287fb7f06799c449971b9ddf9fe|c6 
xx                                                                xx 

+

02|af7efeb1f7cf0d5077ae7f7a59e2b643c5cd01fb55221bf76221d8c8ead92b|f0
xx                                                                xx
```
 
The combined data chunk is:

```
76d539826e5ec10fed9ef597d5bfdac067d287fb7f06799c449971b9ddf9fe
af7efeb1f7cf0d5077ae7f7a59e2b643c5cd01fb55221bf76221d8c8ead92b
```


2) Deobfuscate with ARC4

After deobfuscating, the data chunk is

```
1c434e54525052545900000000000000000004fadf000000174876e8000000
00000000000000000000000000000000000000000000000000000000000000
```


3) Strip the first byte from the combined data chunk
and
4) Verify that combined data chunk, begins with CNTRPRTY

```
1c|434e545250525459|00000000|000000000004fadf000000174876e800000000000000000000000000000000000000000000000000000000000000000000
|         |             |       |
|         |             |       └───── All of this is the transaction data (49 bytes)
|         |             └─────────────── This is the transaction type identifier (4 bytes)
|         └───────────────────────────────── the string CNTRPRTY in hexadecimal (8 bytes)
└─────────────────────────────────────────────── Ignore the first byte
```



## Decoding counterpartydata

After finished with the processing of the vouts, you will have the `destinations` and `counterpartydata`.

The counterparty data looks like this

```
434e545250525459|FFFFFFFF|xxxxxx...
       |            |       |
       |            |       └───── this data is different for each transaction type
       |            └──────────────── the transaction type identifier (4 bytes)
       └──────────────────────────────── the string CNTRPRTY (8 bytes)
```

The first 8 bytes of the `counterpartydata` must be the string CNTRPRTY (Or `434e545250525459` in hexadecimal).

The next 4 bytes of the contain a 4 byte integer representing the type of the transaction.  The types of transactions and ID numbers are

ID | Transaction type
---|-----------------
0  | [Send](#decoding-sends)
20 | Issuance
30 | Broadcast
.. | (this list is incomplete)


## Decoding sends

A send transaction contains 2 pieces of data.  An asset name and a quantity.

```
000000000004fadf|000000174876e800
       |               |
       |               └───── quantity (8 bytes)
       └────────────────── asset name (8 bytes)
```


The first 8 bytes contain an unsigned integer with the asset id.  This asset id is converted to an asset name.  See [decoding asset names](decoding-asset-names.md).

And the next 8 bytes contain an unsigned integer with the quantity of the send.




# Decoding Asset Names

An asset ID is a 64 bit number.  

There are 2 special asset IDs.  Asset id `0` is BTC and asset id `1` is XCP.

Asset IDs greater than 1 and less than 17576 (0x44A8) are invalid.

To convert an asset ID into a name, repeatedly divide the asset ID by 26, round down and convert each remainder into a character until the result of the division is 0.  The name is built from the last letter to the first letter.

```
Example of converting 000000000004fadf (326367) into a name

ABCDEFGHIJKLMNOPQRSTUVWXYZ
||||                    |└── 25
|||└───── 3             └─ 24
||└──── 2
|└─── 1
└── 0

326367 / 26 = 12552 remainder 15. 15 translates to P. The Asset is P.
12552 / 26  = 482   remainder 20. 20 translates to U. The Asset is UP.
482 / 26    = 18    remainder 14. 14 translates to O. The Asset is OUP.
18 / 26     = 0     remainder 18. 18 translates to S. The Asset is SOUP.
```
See [Javascript Example](https://gist.github.com/loon3/a714c7a85abe48d587bd)
