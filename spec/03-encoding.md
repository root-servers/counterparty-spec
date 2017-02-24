# Encoding a transaction

## Preface

These instructions assume transaction data returned from the bitcoin core reference client as of version 0.9


## build the counterparty data

Build the transaction specific data.  Then encode the using one of the following methods

1. [OP_RETURN](#encoding-using-op_return)
2. [OP_CHECKSIG](#encoding-using-op_checksig)
3. [OP_CHECKMULTISIG](#encoding-using-op_checkmultisig)


### Encoding using OP_RETURN

The data is encoded as an OP_RETURN script in the transaction.

The OP_RETURN data begins with CNTRPRTY (8 bytes) and the remaining 72 bytes may be used for the transaction type and data.


```
434e545250525459|00000000|000000000004fadf000000174876e800000000000000000000000000
       |             |          |
       |             |          └── All of this is the operation data (maximum 68 bytes)
       |             └──────────────── This is the transaction type identifier (4 bytes)
       └───────────────────────────────── The string CNTRPRTY in hexadecimal (8 bytes)
```

EXAMPLE:
https://chain.so/api/v2/tx/BTC/8a1d4c5308b7b47134b64fec8dfcb501b38d8ac57c2b7e8f1666a0536e36927e



### Encoding using OP_CHECKSIG

(not written yet)


### Encoding using OP_CHECKMULTISIG

If necesary, pad the operation specific data with zeros to make it 49 bytes long.

The entire transaction data is 62 bytes long.  Begin by assembling the length of the data, the string CNTRPRTY, a transaction type id and the padded transaction data.

```
1c|434e545250525459|00000000|000000000004fadf000000174876e800000000000000000000000000000000000000000000000000000000000000000000
 |       |             |          |
 |       |             |          └── All of this is the operation data (49 bytes)
 |       |             └──────────────── This is the transaction type identifier (4 bytes)
 |       └───────────────────────────────── The string CNTRPRTY in hexadecimal (8 bytes)
 └──────────────────────────────────────────── The length of the counterparty data (1 btye)
```

Now, obfuscate this data using ARC4 encoding.  The key is the txid of the first transaction input for the new transaction you are building.

Split the 62 bytes of data into 2 parts of 31 bytes each.

For each 31 byte part, create a valid 33 byte public key.  To create a public key, append `0x02` or `0x03` at the beginning and guess the last 2 bytes that make the checksum valid.

```
NOTE: This is not ARC4 Encoded - need to fix this example
02|000000000004fadf000000174876e80|FF
 |             |                   |
 |             |                   └── Randomly guessed checksum byte
 |             └───────────────────────── 31 bytes of data
 └────────────────────────────────────────── Either 02 or 03
 ```

The last public key in the send transaction is the sender's public key.  This allows the sender to reclaim the BTC dust used in the transaction at a later time.





