# Token contract

### Preparation

1. Update local ton.

    ```bash
    $ cd {path}/ton/build
    $ cmake -DCMAKE_BUILD_TYPE=Release ..
    $ make -j 4
    ```

2. Project folders tree 
    ```
     {path}/project -> ton-bin  -> lib-fift
                    -> ton_fungible_token
    ```

3. Copy fift and func into the ton-bin folder.

    ```bash
    $ cp {path}/ton/build/crypto/fift {path}/project/ton-bin
    $ cp {path}/ton/build/crypto/func {path}/project/ton-bin
    ```

4. Copy fift libs into the lib-fift folder.

    ```bash
    $ cp {path}/ton/crypto/fift/lib/* {path}/project/ton-bin/lib-fift
    ```

### Manual

1. Compile the app.

    ```bash
    $ cd  {path}/project/ton_fungible_token
    $ ./funccompile.sh 
    ```

2. Run and check tests.

    ```bash
    $ cd tests/
    $ ./fiftcompile.sh -s deploy-runvm.fif
    $ ./fiftcompile.sh -s transfer-runvm.fif
    $ ./fiftcompile.sh -s getbalance-runvm.fif
    $ ./fiftcompile.sh -s approve-transfer-runvm.fif
    ```
    
    If some of them doesn't pass make sure that you have latest TON version.

3. Deploy token contract.

    interface example: *deploy.fif \<file-base\> \<owner-address-or-file\> \<decimals\> \<supply\> [\<token-name\> \<token-symbol\>]*
    
    Example
    ```bash
    $ ./fiftcompile.sh -s deploy.fif wallets/contract-wallet @wallets/alice-wallet.addr 0 1000 TON TON
    ```
    Save output somewhere, we will use it later. Take a look on the next fields:

    - "Non-bounceable address (for init)" - Will use it as a {contract id}.
    - "Bounceable address (for later access)" - Will use it as a {contract id inited}.
    
4. Check contract status using the Lite Client.

    Copy deploy.boc (generated file into the lite-client) 
    ```bash
    $ cp {path}/project/ton_fungible_token/queries/deploy.boc {path}/ton/build/lite-client
    ```
    
    Run lite-client
    ```bash
    $ cd {path}/ton/build/lite-client
    $ ./lite-client -C ton-lite-client-test1.config.json -D db
    ```
    
    Get last block
    ```bash
    lite-client> last 
     ```
    
    Get account info 
    ```bash
    lite-client> getaccount {contract id}
    ```
    
    Publish contract
    ```bash
    lite-client> sendfile deploy.boc
    ```
    
     *Deposit some test grams in to the contract address.*
     - Use TestTonBot. (Telegram)
     - Send {contract id}.
     - Chose any amount.
      
    Update state
    ```bash
    lite-client> last
    ```
    
    We can see new information about contract if it deploy.
    ```bash
    lite-client> getaccount {contract id}
    ```
    
    Get token supply.
    ```bash
    lite-client> runmethod {contract id} getsupply
    ```

5. Prepare transfer query.
    
    ```bash
    $ cd {path}/project/ton_fungible_token
    ```
    
    interface example: *transfer.fif \<query-id\> \<dest-addr\> \<amount\>*
    
    ```bash
    $ ./fiftcompile.sh -s transfer.fif 1 @wallets/bob-wallet.addr 10
    ```

6. Get the token owner address.

    ```bash
    $ ./fiftcompile.sh -s tools/show-addr.fif wallets/alice-wallet
    ```

    Save output somewhere, we will use it later. Take a look on the next fields:
    
    - "Bounceable address (for later access)" - Will use it as a {owner id}.

7. Obtain the owner wallet seqno using the Lite Client.

    Update state
    ```bash
    lite-client> last
    ```

    We can see new information about contract when it is deployed.
    ```bash
    lite-client> getaccount {owner id}
    ```

    Get a nonce number. Result field is a nonce number. Will use it as a {seqno}.
    ```bash
    lite-client> runmethod {owner id} seqno
    ```

7. Wrap transfer message in the owner wallet query.

    **Important notice: _wallet should contain at least 1 Gram. It will be used to pay the fees to access token contract and remaining Grams will be returned back._**

    interface example: *wallet.fif \<owner-wallet\> \<contract-addr\> \<seqno\> \<amount-grams\> -B \<body-cell\> \<query-output\>*

    ```bash
    $ ./fiftcompile.sh -s tools/wallet.fif wallets/alice-wallet {contract id inited} {seqno} 1 -B queries/body-transfer.boc queries/token-transfer
    ```

    Copy into lite-client folder.
    ```bash
    $ cp queries/token-transfer.boc {path}/ton/build/lite-client 
    ```
    
    Run lite-client
    ```bash
    $ cd {path}/ton/build/lite-client
    $ ./lite-client -C ton-lite-client-test1.config.json -D db
    ```
    
    Get last block
    ```bash
    lite-client> last 
    ```
    
    Publish transfer query
    ```bash
    lite-client> sendfile token-transfer.boc
    ```
    
    Get last block
    ```bash
    lite-client> last
    ```
    
8. Get balance value.

    Decode address format. Will use it as {owner network} and {owner address}
    ```bash
    $ ./fiftcompile.sh -s tools/parse-address.fif {owner id}
    ```

    Take a look on the next fields:

    - "Network code is" - Will use it as a {owner network}.
    - "Address uint256 is" - Will use it as a {owner address}.

    Get the owner balance
    ```bash
    lite-client> runmethod {contract id inited} getbalance {owner network} {owner address}
    ```

### Deployment

Test contract is deployed at address kf_pV_iUiWRPOJhmI9WIOszOlZXF3YTLFInGNOFyZi7LLuit in the TON testnet2.

Token owner private key is contained in
```bash
wallets/alice-wallet.pk
```

Token owner address is contained in
```bash
wallets/alice-wallet.addr
```

Token owner network for lite-client 'getbalance' request is:
```bash
-1
```

Token owner address for lite-client 'getbalance' request is:
```bash
11940668616038190933510003492525825940569683466691253499213030567732558073356
```

### Usage notice

Token approvals may be attacked as described here: https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit

This issue should be mitigated with proper contract usage:

- get user balance before approving
- approve user with 0 value (reset approval)
- get user balance again - it should not change
- if balance was changed - transfer should be terminated - you've been attacked
- approve user to transfer

### Future improvements

1. Burnable and mintable tokens.

Contract may be extended to support dynamically changing token supply.

2. Token exchange.

Token exchange is easier to build as an extension to the token contract.

3. Non-fungible tokens.

We may add an extension to to create singleton tokens. They are non-exchangeable and contain some unique data attached.