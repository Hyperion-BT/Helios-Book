# Example: Always succeeds

Start a nodejs repl on your development computer, and import the Helios library:
```bash
$ cd ./helios
$ nodejs
```
```javascript
> var helios; import("./helios.js").then(m=>{helios=m});
```

Compile the Always Succeeds script into its JSON representation:
```javascript
> console.log(helios.Program.new("spending always_succeeds func main() -> Bool {true}").compile().serialize())

{"type": "PlutusScriptV1", "description": "", "cborHex" :" 581358110100002223333573464945262498992601"}
```

Start an interactive shell in the *cardano-node* container and copy the content of the JSON representing the script:
```bash
$ docket exec -it <container-id> bash

> mkdir -p /data/scripts
> cd /data/scripts

> echo '{
  "type": "PlutusScriptV1", 
  "description": "", 
  "cborHex": "581358110100002223333573464945262498992601"
}' > always-succeeds.json

```

Generate the script address:
```bash
> cardano-cli address build \
  --payment-script-file /data/scripts/always-succeeds.json \
  --out-file /data/scripts/always-succeeds.addr \
  --testnet-magic $TESTNET_MAGIC_NUM

> cat /data/scripts/always-succeeds.addr

addr_test1wzlmzvrx48rnk9js2z6c0gnul2063hl2ptadw9cdzvvq7vgy4qmsu
```

We need a datum, which can be chosen arbitrarily in this case:
```bash
> DATUM_HASH=$(cardano-cli transaction hash-script-data --script-data-value "42")
> echo $DATUM_HASH

9e1199a988ba72ffd6e9c269cadb3b53b5f360ff99f112d9b2ee30c4d74ad88b
```

We also need to select some UTxOs as inputs to the transaction. At this point we should have one UTxO sitting in wallet 1. We can query this using the following command:
```bash
> cardano-cli query utxo \
  --address $(cat /data/wallets/wallet1.addr) \
  --testnet-magic $TESTNET_MAGIC_NUM

TxHash             TxIx  Amount
-------------------------------------------------------------
4f3d0716b07d75...  0     1000000000 lovelace + TxOutDatumNone
```
`4f3d...` is the transaction id. The UTxO id in this case is `4f3d...#0`.

We now have everything we need to build a transaction and submit it.

Let's send 2 tAda (2 million lovelace) to the script address:
```
> TX_BODY=$(mktemp)
> cardano-cli transaction build \
  --tx-in 4f3d...#0 \
  --tx-out $(cat /data/scripts/always-succeeds.addr)+2000000 \
  --tx-out-datum-hash $DATUM_HASH \
  --change-address $(cat /data/wallets/wallet1.addr) \
  --testnet-magic $TESTNET_MAGIC_NUM \
  --out-file $TX_BODY \
  --babbage-era

Estimated transaction fee: Lovelace 167217

> TX_SIGNED=$(mktemp)
> cardano-cli transaction sign \
  --tx-body-file $TX_BODY \
  --signing-key-file /data/wallets/wallet1.skey \
  --testnet-magic $TESTNET_MAGIC_NUM \
  --out-file $TX_SIGNED

> cardano-cli transaction submit \
  --tx-file $TX_SIGNED \
  --testnet-magic $TESTNET_MAGIC_NUM

Transaction successfully submitted
```

If you check the wallet 1 payment address balance after a few minutes you will noticed that it has decreased by 2 tAda + fee. Note the left-over UTxO id, we will need it to pay fees when retrieving funds.


You can also try to check the balance of the script address:
```bash
> cardano-cli query utxo \
  --address $(cat /data/scripts/always-succeeds.addr) \
  --testnet-magic $TESTNET_MAGIC_NUM

...
```
The table should list at least one UTxO with your specific datum hash.

We can now try and get our funds back from the script by building, signing and submitting another transaction:
```bash
> PARAMS=$(mktemp) # most recent protocol parameters
> cardano-cli query protocol-parameters --testnet-magic $TESTNET_MAGIC_NUM > $PARAMS

> TX_BODY=$(mktemp)
> cardano-cli transaction build \
  --tx-in <fee-utxo> \ # used for tx fee
  --tx-in <script-utxo> \
  --tx-in-datum-value "42" \
  --tx-in-redeemer-value <arbitrary-redeemer-data> \
  --tx-in-script-file /data/scripts/always-succeeds.json \
  --tx-in-collateral <fee-utxo> \ # used for script collateral
  --change-address $(cat /data/wallets/wallet1.addr) \
  --tx-out $(cat /data/wallets/wallet1.addr)+2000000 \
  --out-file $TX_BODY \
  --testnet-magic $TESTNET_MAGIC_NUM \
  --protocol-params-file $PARAMS \
  --babbage-era

Estimated transaction fee: Lovelace 178405

> TX_SIGNED=$(mktemp)
> cardano-cli transaction sign \
  --tx-body-file $TX_BODY \
  --signing-key-file /data/wallets/wallet1.skey \
  --testnet-magic $TESTNET_MAGIC_NUM \
  --out-file $TX_SIGNED

> cardano-cli transaction submit \
  --tx-file $TX_SIGNED \
  --testnet-magic $TESTNET_MAGIC_NUM

Transaction successfully submitted
```

If you now check the balance of wallet 1 you should see two UTxOs, and the total value should be your starting value minus the two fees you paid. 

Note that *collateral* is only paid if you submit a bad script. Cardano-cli does extensive checking of your script though, and should prevent you from submitting anything faulty. So *collateral* is only really paid by malicious users.
