# Filecoin API - Schema and usage guide

## ADDRESSES
The `addresses` table maps robust-form addresses (f1, f2, f3) with their ID address equivalent (f0) and contains information on the account type.

_(Explainer on addresses: [link](https://docs.filecoin.io/about-filecoin/how-filecoin-works/#addresses))_


| Column name  | Description|
| :------------ | :------------------------------------------------------------------------------------------------ |
| Robust        | Robust form for a Filecoin address (BLS, SECP256K1 and Actor addresses starting with f1, f2, f3)
| Short         | Short (ID) form for a Filecoin address, starting with f0
| Account type  | Account type (account, payment channel, miner, etc)|
| ID            | Additional field to aid with ordering and foreign key constraints |
<br/>

#### ▶️ **Example usage (SQL)**

* “Convert” an address (look up the address in the short/robust format)
```sql
SELECT robust from finance.addresses WHERE short = 'f025...';

SELECT short from finance.addresses WHERE robust = 'f3u4awxhbhz7jb...';
```

(See the “Transactions” section for an example of how to use addresses as an auxiliary table)

<br/>

#### ▶️ **Example usage (GraphQL)**

* “Convert” an address (look up the address in the short/robust format)
```
query AddressesGetRobustByShort {
  finance_addresses(where: {short: {_eq: "f025..."}}) {
    robust
  }
}

query AddressesGetShortByRobust {
  finance_addresses(where: {robust: {_eq: "f3u4awxhbhz7jb..."}}) {
    short
  }
}
```

<br/>

## TRANSACTIONS
The `transactions` table contains transactions data, formatted to fit a reporting model.

| Column name  | Description |
| :----------- | :------------------------------------|
| Height       | Tipset height at which the transaction’s block is located |
| Tx_timestamp | Tipset and block timestamp in UNIX format
| Tx_hash      | Transaction hash (identifier) |
| Tx_from      | Sender address |
| Tx_to        | Receiver address |
| Amount       | Transaction value |
| Status       | Transaction status (if the transaction was successful or not) |
| Tx_type      | Transaction type (method) |
| Tx_metadata _(optional)_       | Includes additional info on the Tx (currently this refers to the method of the Tx a fee belongs to) |
| Tx_params _(optional)_      | If any additional user-generated input has been added to the Tx _(currently parsed params are only available for Txs with the following format: PREFIX:WHATEVER )_ |

<br/>

#### ▶️ **Example usage (SQL)**
* Find all transactions in which a certain address has participated:
```sql
SELECT * FROM finance.transactions WHERE tx_from = 'f025...' OR tx_to = 'f025...';
```

* Find outward transactions from an address:
```sql
SELECT * FROM finance.transactions WHERE tx_from = 'f025...' OR tx_to = (SELECT robust from finance.addresses WHERE short = 'f025...');
```

(Note: for inward transactions, look up the address in the tx_to column)

* Filter transactions by date
```sql
WITH search_heights AS (
    SELECT height FROM finance.blocks_dates
    WHERE timestamp BETWEEN TO_DATE('2021-07-31', 'YYYY-MM-DD')
              AND TO_DATE('2021-08-01', 'YYYY-MM-DD')
)
SELECT * FROM finance.transactions
WHERE height IN (SELECT height FROM search_heights)
```

* Get transactions at a certain height (or filter by a single column):
```sql
SELECT * FROM finance.transactions WHERE height = 148888;
```

* Filter by a tx_params prefix:
```sql
SELECT * FROM finance.transactions WHERE tx_params LIKE 'SOMEPREFIX%';
```
<br/>

#### ▶️ **Example usage (GraphQL)**
* Find all transactions in which a certain address has participated:
```
query TransactionsGetByAddress {
  finance_transactions(where: {_or: [{tx_from: {_eq: "f3rrwmsdclcwsy3prwjd..."}}, {tx_from: {_eq: "f0869..."}}]}) {
    tx_from
    amount
    height
    status
    tx_hash
    tx_metadata
    tx_params
    tx_timestamp
    tx_to
    tx_type
  }
}
```

* Get transactions at a certain height (or filter by a single column):
```
query TransactionsGetByHeight {
  finance_transactions(where: {height: {_eq: "231866"}}) {
    amount
    height
    status
    tx_from
    tx_hash
    tx_metadata
    tx_params
    tx_to
    tx_type
  }
}
```

* Filter by a tx_params prefix:
```
query TransactionsGetByParams {
  finance_transactions(where: {tx_params: {_like: "SOMEPREFIX%"}}) {
    amount
    height
    status
    tx_from
    tx_hash
    tx_metadata
    tx_params
    tx_to
    tx_type
  }
}
```

<br/>


## VESTING
The `vesting` table contains vesting details for multisig wallets.

| Column name     | Description |
| :-----------    | :------------------------------------|
| Address         | Filecoin account address |
| Initial_balance | Inicial locked vesting amount
| Start_epoch     | Tipset height in which the vesting period starts
| Unlock_duration | How many block epochs it will take to unlock the full amount |
| Unlock_height   | Tipset height at which the full vesting amount will be released (calculated as `start_epoch + unlock_duration`) |

<br/>

#### ▶️ **Example usage (SQL)**
* Look up vesting data for an address
```sql
SELECT * FROM finance.vesting WHERE address = 'f096...'


// If we only know the robust version
SELECT * FROM finance.vesting WHERE address = 
(SELECT short FROM finance.addresses WHERE robust = 'f2bedy6otpq...');
```

<br/>

#### ▶️ **Example usage (GraphQL)**
* Look up vesting data for an address
```
query VestingGetByAddress {
  finance_vesting(where: {address: {_eq: "f096..."}}) {
    address
    initial_balance
    start_epoch
    unlock_duration
    unlock_height
  }
}
```

<br/>

## Auxiliary tables
## BLOCKS_DATES

Auxiliary table to simplify querying by block date (as the dates are stored in a human-readable format instead of a UNIX timestamp)

| Column name  | Description |
| :----------- | :------------------------------------|
| Height       | Tipset height |
| Block_date   | UTC date with the format `YYYY-MM-DD hh:mm:ss`|

<br/>

#### ▶️ **Example usage (SQL)**
* Filter transactions by date
```sql
WITH search_heights AS (
    SELECT height FROM finance.blocks_dates
    WHERE timestamp BETWEEN TO_DATE('2021-07-31', 'YYYY-MM-DD')
              AND TO_DATE('2021-08-01', 'YYYY-MM-DD')
)
SELECT * FROM finance.transactions
WHERE height IN (SELECT height FROM search_heights)
```
<br/>

## MSIG_TRANSACTIONS


| Column name  | Description |
| :----------- | :------------------------------------|
| Height       | Tipset height |
| Tx_hash      | Transaction hash (identifier) |
| Tx_from      | Sender address |
| Multisig      | Target multisig address |
| Proposed_method      | Method being proposed to be executed by the multisig |
| Signer_in    | (In `AddSigner` Txs) Signer that has been added to the multisig wallet |
| Signer_out   | (In `RemoveSigner` Txs) Signer that has been removed from the multisig wallet |
| Tx_type      | Multisig transaction type (method) |
| Tx_value     | Transaction value proposed or transferred |
| Status      | Transaction status (if the transaction was successful or not) |

<br/>

#### ▶️ **Example usage (SQL)**
* Find all multisigs an address is a signer of (or has been proposed as a signer)
```sql
SELECT * FROM finance.msig_transactions WHERE signer_in = 'f3vqxn4nq...';
```
* Find all multisigs an address is a signer of 
```sql
SELECT multisig FROM finance.msig_transactions WHERE signer_in = 'f3vqxn4nq...'
AND (tx_type IN ('AddSigner', 'SwapSigner'));
```
* Find all multisigs an address has been removed from
```sql
SELECT multisig FROM finance.msig_transactions WHERE signer_out = 'f3vqxn4nq...'
AND (tx_type IN ('RemoveSigner', 'SwapSigner'));
```

<br/>

#### ▶️ **Example usage (GraphQL)**
* Find all multisigs an address is a signer of (or has been proposed as a signer)
```
query MsigTransactionsGetByAddedSigner {
  finance_msig_transactions(where: {signer_in: {_eq: "f3vqxn4nq..."}}) {
    multisig
  }
}
```
* Find all multisigs an address is a signer of 
```
query MsigGetBySigner {
  finance_msig_transactions(where: {signer_in: {_eq: "f3vqxn4nq..."}, tx_type: {_in: ["AddSigner", "SwapSigner"]}}) {
    multisig
  }
}
```

* Find all multisigs an address has been removed from
```
query MsigTransactionsGetByRemovedSigner {
  finance_msig_transactions(where: {signer_out: {_eq: "f020..."}, tx_type: {_in: ["RemoveSigner", "SwapSigner"]}}) {
    multisig
  }
}
```