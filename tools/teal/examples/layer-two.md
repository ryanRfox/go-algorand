## Variables
DATA=betanetdata
ACCOUNT_A=E33Y6VPY7LIXADZ5WMDKBHGBPRBMDLKMPJ7QBRCBUXBUL5722MCIEMCBOU # demo wallet
ACCOUNT_B=RZW5QPI4X7NZXUKFJZXOU3F5QQR7Q7MW5ZG3467XDWTMGO3YDUW3YCIJPQ # remote
ACCOUNT_MSIG=2J42MCHEYN2HBGWLYGBTFHTONFX6IJGD2JMGCTKBKT5OXOJYGHW76GVBRU # ACCOUNT_A & ACCOUNT_B
AMOUNT_INITIAL=100000
EVIDENCE_ACCOUNT=4YUPKVEMZA3IMQO57QIM56EZZPHXXG4OMX6JH44ITFPEC3BC6QHSAOSQQA


# Initialization
Multisig Escrow Account (MEA) Generation
MEA Timeout Liquidation Generation
// Trigger Contract Generation
// Closeout Contract Generation
MEA Funding

## Multisig Escrow Account (MEA) Generation
A & B cooperate to form a multisig account to escrow funds within and pay fees from
  `goal account multisig new -T 2 $ACCOUNT_A $ACCOUNT_B -d $DATA`

## MEA Timeout Liquidation Generation
A & B agree on the roundFirstValid N defining when the MEA may be liquidated unilaterally
A & B cooperate to generate, group and sign
  send X from MEA to A roundFirstValid N multisig(AB)
  send Y from MEA close to B first_valid N multisig(AB)
A & B retain the MEA Liquidation TXg for future use if Closeout is not triggered

## Trigger Contract Generation
A & B cooperate to generate `trigger.teal` approving send from MEA close to Trigger Contract (TODO: to Evidence Contract)
  single TX
  from MEA
  close to Trigger Contract (TODO: Evidence Contract)
A & B cooperate to sign `trigger.teal` and generate `trigger.lsig`
  // see https://developer.algorand.org/docs/asc-tutorial
  `goal clerk multisig signprogram -p trigger.teal -a ACCOUNT_A -A TRIGGER_TEAL_ADDRESS -o trigger.lsig -d $DATA`
  `goal clerk multisig signprogram -L trigger.lsig -a ACCOUNT_B -d $DATA`
A & B retain `trigger.lsig` offline. They may create an unsiged transaction at any time to trigger the closeout process.

## MEA Funding
A & B cooperate to generate, sign and broadcast grouped TX
  send X + Fees to MEA from A
  send Y + Fees to MEA from B
  `goal clerk send -a $AMOUNT_INITIAL -f $ACCOUNT_A -t $ACCOUNT_MSIG -o funding_a.utx -d $DATA`
  `goal clerk send -a $AMOUNT_INITIAL -f $ACCOUNT_B -t $ACCOUNT_MSIG -o funding_b.utx -d $DATA`
  `cat funding_a.utx funding_b.utx > funding.utxc`
  `goal clerk group -i funding.utxc -o funding.utxg`
  `goal clerk split -i funding.utxg -o funding.utx`
  `goal clerk sign -i funding-0.utx -o funding-0.stx -d $DATA` # signed by ACCOUNT_A
  msgpacktool -d -b32 < funding-1.utx > funding-1.json
  nano funding-1.json
  ~/go/bin/msgpacktool -e < funding-1.json > funding-1.utx
  `goal clerk sign -i funding-1.utx -o funding-1.stx -d $DATA` # signed by ACCOUNT_B
  ~/go/bin/msgpacktool -d < funding-1.stx > funding-1.json
  nano funding-1.json
  msgpacktool -e < funding-1.json > funding-1.stx
  `cat funding-0.stx funding-1.stx > funding.stxg`
  `goal clerk rawsend -f funding.stxg -d $DATA`

# Update Contract (signed by AB)
// allows send from MEA to Evidence Contract if arg_0 is EC arg_1 is Msig_AB(EC) arg_2 is pk(AB) 
A & B coorperate to create new Evidence Contracts for each new state, defining X' & Y' and Msig(EC address)
This allows A or B to broadcast a TX using the Msig(EC address) to send from MEA to a specific EC
Once funds are in an EC, A & B may 1) cooperativley close 2) timeout to A(X') & B(Y') 3) provide evidence of higher index EC: send close to MEA

# DEPRICATED: Trigger Contract (signed by AB)
// allows 1) send from TC to Evidence Contract if arg_0 is EC arg_1 is Msig_AB(EC) arg_2 is pk(AB) 
// allows 2) send from MEA TO TC
send from MEA close to TC

# Evidence Contract
TMPL_INDEX // index of this contract
Cooperative Close
Evidence Evaluation
  validation checks
    utx from TC to EC  // send from TC to EC_X    // EC_X is multisig(AB) signed below
    multisig(AB) send 0 from MEA to TC note EC_X 
    int TMPL_INDEX
    gtxn_1 note
    btoi
    >
  check index
Timeout (prior to MEA timeout)

## Evidence Expiration
  // Sign the Evidence Contract
  ./goal clerk compile evidence_contract.teal
  sudo ./goal clerk multisig signprogram -p evidence_contract.teal -a $ACCOUNT_A -A $ACCOUNT_MSIG -o evidence_contract.lsig -d $DATA
  sudo chmod 777 evidence_contract.lsig
  ./msgpacktool -d -b32 < evidence_contract.lsig > evidence_contract.json
  nano evidence_contract.json

  ~/go/bin/msgpacktool -e -b32 < evidence_contract.json > evidence_contract.lsig
  ./goal clerk multisig signprogram -L evidence_contract.lsig -a $ACCOUNT_B -d $DATA
  ~/go/bin/msgpacktool -d -b32 < evidence_contract.lsig > evidence_contract.json
  nano evidence_contract.json
  
  ./msgpacktool -e -b32 < evidence_contract.json > evidence_contract.lsig
  // END

  TEST:
  // Fund $ACCOUNT_MSIG
  ./goal clerk send -a 203000 -t $ACCOUNT_MSIG -d $DATA
  ./goal account balance -a $ACCOUNT_MSIG -d $DATA
  ./goal clerk inspect claim.stx 
  ./goal clerk dryrun -t claim.stx -d $DATA
  ./goal clerk rawsend -f claim.stx -d $DATA


  // Claim from MEA to EC
  ./goal clerk send -a 0 -f $ACCOUNT_MSIG -t $EVIDENCE_ACCOUNT -c $EVIDENCE_ACCOUNT -d $DATA -o claim.utx
  ./goal clerk sign -i claim.utx -o claim.stx -L evidence_contract.lsig -d $DATA
  FAIL: unable to sign a 

  TEST:
  // Redeem from $EVIDENCE_ACCOUNT using EC
  `goal clerk send -a 100000 -t $ACCOUNT_A -f $ACCOUNT_MSIG -o expiration_a.utx -d $DATA`
  `goal clerk send -a 100000 -t $ACCOUNT_B -c $ACCOUNT_B -f $ACCOUNT_MSIG -o expiration_b.utx -d $DATA`
  `cat expiration_a.utx expiration_b.utx > expiration.utxc`
  `goal clerk group -i expiration.utxc -o expiration.utxg`
  `goal clerk sign -i expiration.utxg -L evidence_contract.lsig -o expiration.stxg -d $DATA` # signed by EC LSig
  `goal clerk rawsend -f expiration.stxg -d $DATA`
  PASS

## Claim Evaluation
arg_0           // data (byte contract_address)
arg_1           // signature (byte signature_A)
arg_3           // pubkey (byte contract_pubkey)
ed25519verify   // check if contract_address is signed by A
arg_0           // data (byte contract_address)
arg_2           // signature (byte signature_B)
arg_3           // pubkey (byte contract_pubkey)
ed25519verify   // check if contract_address is signed by B
&&              // ensure both A & B signed contract_address
// allow a transaction from MEA closeto contract_address

## Dispute Evaluation
// check sigA
arg_0            // data (int dispute_index)
arg_1            // signature (byte signature_A_or_B)
byte $PUBKEY_A   // pubkey of A
ed25519verify    // check if A signed contract_index passed as arg_0
// check sigB
arg_0            // data (int dispute_index)
arg_1            // signature (byte signature_A_or_B)
byte $PUBKEY_B   // pubkey of B
ed25519verify    // check if B signed contract_index passed as arg_0
// evaluate sigs
+                // add results of signature verification checks
int 1            // put 1 on the stack
==               // ensure one signature is valid
// evaluate contract_index
arg_0            // load data (int index)
btoi             // convert byte to int
int $TMPL_INDEX  // index of this program
>                // check if arg_0 (new_index) is greater than TMPL_INDEX
&&               // check if both the signed new_index newer than this program's index
// allow a grouped transaction (close this dispute account to multisig escrow account)


## Cooperative Close 
TODO: allow MEA to multisig(AB): gtx send X' from TC to A & send Y' close to B
  Issue: multisig(AB) does not own funds in TC
    Solution? AB cooperate to sign a program spending from TC to Evidence Contract Index with each Update
  Issue: Trigger Contract must define where it can send funds to.
TODO: allow MEA to multisig(AB): send from CC close to Evidence Contract Sequence

## Cooperative Close
cooperatively create and sign a group TX send X'/Y' from CC to A/B signed(AB)  

retain the Closeout Contract and Evidence Contract
send from MEA to CC
  send from CC to Evidence Contract
    cooperative close
    newer evidence
    timeout

## Dispute Evidence
check evidence
send close to MEA from CC

#### Tests
multisig to sign a TEAL program
goal clerk multisig signprogram -p trigger.teal -a account_a -A address_of_TEAL_program -o trigger.lsig -d $DATA
goal clerk multisig signprogram -L trigger.teal -a account_b -o trigger.lsig -d $DATA

# Post

They agree to fund MEA with some amount (X + Y + Fees). The cooperatively create, sign and retain offline a grouped transaction set to return funds at an agreed to time in the future: `send X/Y from MEA to A/B firstvalid`. This is a failsafe in case either may attempt to grief the other. Now, they are willing to cooperate in signing and broadcasting a grouped transaction set to fund the MEA: `send X/Y+Fees to MEA`. 

Once the MEA is funded, they can perform any number of offline updates to the proportion of X+Y due to A/B. Cooperatively they can broadcast any closing transaction from MEA 


#### Calculate Merkle Tree

#### Argument serialization
arg[0] = total_leaves (int)
arg[1] = leaf_1_hash (bytes)
arg[2] = leaf_1_position (int 0=left | 1=right)
arg[n] = leaf_n_hash (bytes)
arg[n+1] = leaf_n_position (int 0=left | 1=right)

arg_0  // left
arg_1  // right
+      // concatinate
sha256 // hash the concatination
// 

TEST:
arg_0
byte base64 NmI4NmIyNzNmZjM0ZmNlMTlkNmI4MDRlZmY1YTNmNTc0N2FkYTRlYWEyMmYxZDQ5YzAxZTUyZGRiNzg3NWI0Yg==
==
byte base64 MA==
sha256
byte base64 NmI4NmIyNzNmZjM0ZmNlMTlkNmI4MDRlZmY1YTNmNTc0N2FkYTRlYWEyMmYxZDQ5YzAxZTUyZGRiNzg3NWI0Yg==
==


1
sha256 = 6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b
encode b64 = NmI4NmIyNzNmZjM0ZmNlMTlkNmI4MDRlZmY1YTNmNTc0N2FkYTRlYWEyMmYxZDQ5YzAxZTUyZGRiNzg3NWI0Yg==

2 = d4735e3a265e16eee03f59718b9b5d03019c07d8b6c51f90da3a666eec13ab35


TEST ed25519verify:
for (data A, signature B, pubkey C) verify the signature of (“ProgData” || program_hash || data) against the pubkey => {0 or 1}

data = index (int)
signature = multisig(AB)
pubkey = pubkey(AB)

