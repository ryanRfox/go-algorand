# Hash Time Lock Contract (HTLC) for Algorand Standard Assets (ASA)

## Functionality

This contract implements a hashed timelock contract (HTLC) for holding Algorand Standard Assets (ASA). It is intended to be used as a "contract only" account, not as a "delegated contract" account. In other words, this contract should not be signed by a spending key. 

The contract will approve transactions from itself under three circumstances:

  1. If `txn` is an "opt-in" transaction for `TMPL_ASA_ID`.
  2. If `txn.FirstValid` is greater than `TMPL_TIMEOUT`, then funds may be closed out to `TMPL_OWN`.
  3. If an argument `arg_0` is passed to the script such that `TMPL_HASHFN(arg_0)` is equal to `TMPL_HASHIMG`, then funds may be closed out to `TMPL_RCV`. 
  
The idea is that by knowing the preimage to `TMPL_HASHIMG`, ASA funds may be released to `TMPL_RCV` (Scenario 3). Alternatively, after some timeout round `TMPL_TIMEOUT`, all remaining funds may be closed back to their original owner, `TMPL_OWN` (Scenario 2). Note that Scenario 3 may occur up until Scenario 2 occurs, even if `TMPL_TIMEOUT` has already passed. As a "contract only" account, Scenario 1 is required to initialize the account by performing an "opt-in" transaction prior to funding it with an amount of `TMPL_ASA_ID`.

## Parameters

  - `TMPL_RCV`: the address to send funds to when the preimage is supplied
  - `TMPL_HASHFN`: the specific hash function (`sha256` or `keccak256`) to use
  - `TMPL_HASHIMG`: the image of the hash function for which knowing the preimage under `TMPL_HASHFN` will release funds
  - `TMPL_TIMEOUT`: the round after which funds may be closed out to `TMPL_OWN`
  - `TMPL_OWN`: the address to refund funds to on timeout
  - `TMPL_ASA_ID`: the assetid of the ASA the contract may hold
  - `TMPL_FEE`: maximum fee of any transactions approved by this contract
  
## Code overview
    ( (Scenario 1) OR  ( (        Scenario 2           ) OR  (      Scenario 3     ) ) AND Fee)
    ( (  Opt-in  ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )

### Scenario 1 - Opt-In for ASA

    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    __^^^^^^^^^^_____________________________________________________________________________

An account is required to perform an "opt-in" transaction to enable holding each ASA asset. Because this is a "contract only" account, the contract itself must define and later sign the "opt-in" transaction prior to any ASA being sent to it. This contract account must be funded with enough algos to pay for the "opt-in" transaction and maintain the minimum balance thereafter. This section will allow the "opt-in" transaction.

Ensure transaction type is AssetTransfer
```
   txn TypeEnum
   int AssetTransfer
   ==
```
Ensure AssetSender is not specified to prevent clawback
```
   txn AssetSender
   global ZeroAddress
   ==
   &&
```
Ensure AssetCloseTo is not specified (note: will be specified for HTLC)
```
   txn AssetCloseTo
   global ZeroAddress
   ==
   &&
```
Ensure opt-in is a zero-value asset transfer
```
   txn AssetAmount
   int 0
   ==
   &&
```
Ensure opt-in is for the specified ASA `TMPL_ASA_ID`
```
   txn XferAsset
   int TMPL_ASA_ID
   ==
   &&
```
At this point the contract has evaluated if the transaction is a valid "opt-in" transaction. The next section will branch forward if TRUE (a valid "opt-in" transaction), else will proceed to the HTLC section for further evaluation.

#### Branch to HTLC
    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    _____________^^^_________________________________________________________________________

Duplicate the top item on the stack which represents opt-in validation
 ```
   dup
 ```
Pop opt-in validation from the stack and branch if not zero
```
   bnz opt-in_pass
```
### Payment Scenarios

Next are the two payment scenarios which comprise the HTLC portion of the contract account. 

### Scenario 2 - Time Lock

    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    ___________________^^___________________________^^_______________________________________

The Time Lock scenario will be executed by `TMPL_OWN` twice. First to claim any remaining ASA Assets and close the asset account. Second, `TMPL_OWN` will claim all remaining Algos from the contract account and close it. However, neither of these claim transactions will be approved until the `TMPL_TIMEOUT` has expired.

Pop the FALSE opt-in validation from top of stack
```
   pop
```
#### Claim Algos

    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    _________________________________________^^^^^___________________________________________

Ensure only `TMPL_OWN` may claim Algo balance
```
   txn CloseRemainderTo
   addr TMPL_OWN
   ==
```   
Ensure transaction Receiver is `TMPL_OWN`
```
   txn Receiver
   addr TMPL_OWN
   == 
   &&
```

#### Claim ASA

    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    __________________________________^^^____________________________________________________

Ensure only `TMPL_OWN` may claim ASA balance
```
   txn AssetCloseTo
   addr TMPL_OWN
   ==
```   
Ensure transaction Receiver is `TMPL_OWN`
```
   txn AssetReceiver
   addr TMPL_OWN
   == 
   &&
 ```

#### ASA or Algos
    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    ______________________________________^^_________________________________________________

Allow `TMPL_OWN` to claim either ASA or Algo balances
```
   ||
```

#### Timeout Period 
    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    _____________________^^^^^^^^^^__________________________________________________________

Ensure the time lock period has passed
```
   txn FirstValid
   int TMPL_TIMEOUT
   >
   &&
```

### Branch to Hash Lock
    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    ___________________________________________________^^^___________________________________

Duplicate the top item on the stack which represents time lock validation
```
   dup
```

Pop time lock validation from the stack and branch if not zero
```
   bnz time-lock_pass
```

### Scenario 3 - Hash Lock
    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    _______________________________________________________^^^^^^^^^^^^^^^^^^^^^^^___________

The Hash Lock portion of the contract is the most interesting, as it allows `TMPL_RCV` to provide the preimage as `arg_0` with their AssetTransfer transaction to claim the ASA balance and close the asset account.

Pop the FALSE time lock validation from top of stack
```
   pop
```   
Ensure only TMPL_RCV may close asset balance 
```
   txn AssetCloseTo
   addr TMPL_RCV
   ==
```
Ensure only TMPL_RCV may claim asset balance 
```
   txn AssetReceiver
   addr TMPL_RCV
   ==
   &&
```

Ensure `arg_0` is 32 bytes in length
```
   arg 0
   len
   int 32
   ==
   &&
```

Ensure `arg_0` is the preimage for `TMPL_HASHIMG` under `TMPL_HASHFN`
```
   arg 0
   TMPL_HASHFN
   byte b64 TMPL_HASHIMG
   ==
   &&
```
### Scenario Evaluations

The next two sections provide re-entrance from an earlier successful branch. This effectively yields an OR statement where TRUE would be on the stack from reentry. 

### Either Hash Lock or Time Lock
    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    ___________________________________________________^^^___________________________________

Re-entrance if successful Time Lock
```
   time-lock_pass:
```

### Either Opt-in or HTLC
    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    _____________^^^_________________________________________________________________________
Re-entrance if successful Opt-In for ASA
```
   opt-in_pass:
```

### Fee
    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    ____________________________________________________________________________________^^^__

Ensure fee is reasonable
```
   txn Fee
   int 1000000
   <
```

### Final Validation
    ( ( Opt-in ) bnz ( ( Time Lock: ( ASA || Algos ) ) bnz ( Hash Lock: ASA Only ) ) && Fee )
    _________________________________________________________________________________^^______
Valid transaction and appropriate Fee
```
&&
```

## Walkthrough

Below is a walkthrough using `goal` to construct and broadcast the transactions. Use the [ASA HTLC Template](https://github.com/ryanRfox/go-algorand/blob/template-asa-htlc/tools/teal/templates/asa-htlc.teal.tmpl) to define your own escrow contract, fund it, and redeem the balances in each of the three scenarios defined above.

### Environment Variables

  - `$ESCROW_ACCOUNT`: the contract account defined by the TEAL code (see Step 2 below to obtain this value)
  - `$OWNER`: the address to refund funds to on timeout
  - `$RECEIVER`: the address to send funds to when the preimage is supplied
  - `$ASSET_ID`: the assetid of the ASA the contract may hold (Owner must hold a balance of this ASA already)
  - `$AMOUNT`: the amount of `$ASSET_ID` the `$OWNER` will send to `$ESCROW_ACCOUNT` and `$RECEIVER` will later claim
  - `$PREIMAGE`: the byte64 encoded preimage to submit when satisfying the Hash Lock (must be 32 bytes)
  - `$INIT_FUNDS`: an amount of MicroAlgos to fund the contract account with to pay for transactions (recommend 203000)
  - `$DATA`: Your local `--datadir` for `goal` 

### Escrow Account Creation and Funding (Owner)

#### 1. (Owner) Create the TEAL program `asa-htlc.teal` for use as the ASA HTLC escrow account

See [ASA HTLC Template](https://github.com/ryanRfox/go-algorand/blob/template-asa-htlc/tools/teal/templates/asa-htlc.teal.tmpl) to configure the required _parameters_.

#### 2. (Owner) Compile the TEAL program to determine the escrow account address. Set the `$ESCROW_ACCOUNT` environment variable.

    goal clerk compile asa-htlc.teal 

#### 3. (Owner) Fund escrow account with MicroAlgos to cover future fees and minimum balance

    goal clerk send -a $INIT_FUNDS -f $OWNER -t $ESCROW_ACCOUNT -d $DATA 

#### 4. (Owner) Create an unsigned opt-in transaction from/to the escrow account for specified ASA

    goal asset send -o asa-htlc-opt-in-escrow.utxn  -a 0 --assetid $ASSET_ID -f $ESCROW_ACCOUNT -t $ESCROW_ACCOUNT -d $DATA

#### 5. (Owner) Sign the opt-in transaction using the `asa-htlc.teal` file as the logicSig

    goal clerk sign -i asa-htlc-opt-in-escrow.utxn -o asa-htlc-opt-in-escrow.stxn -p asa-htlc.teal -d $DATA

#### 6. (Owner) Braodcast the signed opt-in transaction to the network

    goal clerk rawsend -f asa-htlc-opt-in-escrow.stxn -d $DATA

#### 6. (Owner) Send an amount of the specified ASA to the escrow account

    goal asset send -a $AMOUNT --assetid $ASSET_ID -f $OWNER -t $ESCROW_ACCOUNT -d $DATA 

#### 7. (Owner) Make the TEAL program available to Receiver 

### Claiming From Escrow using Hash Lock (Receiver)

#### 8. (Receiver) Compile the TEAL program to determine the escrow account address

    goal clerk compile asa-htlc.teal

#### 9. (Receiver) Review the TEAL program for correctness

 * Ensure Time Lock expiration meets your needs
 * Ensure Hash Lock contains expected $AMOUNT of expected $ASSET_ID
 * Ensure Hash Lock defines your $RECEIVER account
 * Ensure the $ESCROW_ACCOUNT is funded properly

#### 10. (Receiver) Create an unsigned opt-in transaction from $RECEIVER to enable holding $ASSET_ID

    goal asset send -o asa-htlc-opt-in-receiver.utxn  -a 0 --assetid $ASSET_ID -f $RECEIVER -t $RECEIVER -d $DATA

#### 11. (Receiver) Sign the unsiged opt-ing transaction

    goal clerk sign -i asa-htlc-opt-in-receiver.utxn -o asa-htlc-opt-in-receiver.stxn -d $DATA

#### 12. (Receiver) Send the signed transaction to the network for validation

    goal clerk rawsend -f asa-htlc-opt-in-receiver.stxn -d $DATA

#### 13. (Receiver) Create an unsigned AssetSend transaction which will close balance to $RECEIVER for $ASSET_ID

    goal asset send -o asa-htlc-hash-lock.utxn -a 0 --assetid $ASSET_ID -f $ESCROW_ACCOUNT -t $RECEIVER -c $RECEIVER -d $DATA 

#### 14. (Receiver) Sign the unsiged AssetSend transaction using the preimage (encoded)

    goal clerk sign -i asa-htlc-hash-lock.utxn -o asa-htlc-hash-lock.stxn -p asa-htlc.teal --argb64 $PREIMAGE -d $DATA

#### 15. (Receiver) Send the signed transaction to the network for validation

    goal clerk rawsend -f asa-htlc-hash-lock.stxn -d $DATA

### Claiming From Escrow using Time Lock (Owner)

#### 16. (Owner) Wait for Time Lock to expire, first redeem remaining ASA (will fail if Receiver already claimed them) then redeem MicroAlgos from escrow account

    goal asset send -o asa-htlc-time-lock-asa.utxn -a 0 --assetid $ASSET_ID -f $ESCROW_ACCOUNT -t $OWNER -c $OWNER -d $DATA
    goal clerk sign -i asa-htlc-time-lock-asa.utxn -o asa-htlc-time-lock-asa.stxn -p asa-htlc.teal -d $DATA
    goal clerk rawsend -f asa-htlc-time-lock-asa.stxn -d $DATA

    goal clerk send -o asa-htlc-time-lock-algo.utxn -a 0 -f $ESCROW_ACCOUNT -t $OWNER -c $OWNER -d $DATA
    goal clerk sign -i asa-htlc-time-lock-algo.utxn -o asa-htlc-time-lock-algo.stxn -p asa-htlc.teal -d $DATA
    goal clerk rawsend -f asa-htlc-time-lock-algo.stxn -d $DATA

