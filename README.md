# Using VLS with Core Lightning on Regtest

This guide demonstrates how to set up two Core Lightning nodes using Validating Lightning Signer (VLS) as the signer on a Bitcoin regtest network. The nodes will be connected, a channel will be established, an invoice will be created on one node, and paid from the other using Docker containers. Outputs for key steps are included to verify success.

## Prerequisites

- **Docker** and **Docker Compose** installed.
- **Git** to clone the repository.
- **Basic familiarity** with Bitcoin and Lightning Network concepts.
- **System requirements**: At least 4GB RAM and 10GB disk space.

## Repository

Clone the VLS container repository:

```bash
git clone https://gitlab.com/lightning-signer/vls-container.git
cd vls-container
```

Ensure the repository contains:
- `docker-compose.two-nodes.yml`
- `lightningd-regtest-1.conf`
- `lightningd-regtest-2.conf`
- `.env`
- `bitcoind/`, `lightningd/`, `vlsd/` directories

## Setup

### Step 1: Configure Environment

Verify `.env` contains:

```bash
# bitcoin version 27.0
BITCOIN_VERSION=27.0
BITCOIN_SHA256SUMS_HASH=4bc7c97684a0bd1ba000f64f7a24c49302bcbd716eacb134b972188e0379a415
# core lightning version v24.11.1
CORE_LIGHTNING_REPO=https://github.com/ElementsProject/lightning.git
CORE_LIGHTNING_GIT_HASH=e6398e5b9aa1d54b42fc61db2ac35558f8e4f38a
# clboss version v0.14.1
CLBOSS_REPO=https://github.com/ZmnSCPxj/clboss
CLBOSS_GIT_HASH=b6795c43dc646225902845c81c4a4df3fea532e7
# cln plugins (2024-07-27)
CLN_PLUGINS_REPO=https://github.com/lightningd/plugins.git
CLN_PLUGINS_GIT_HASH=5e449468bd57db7d0f33178fe0dc867e0da94133
# txoo version 0.9.0
TXOO_REPO=https://gitlab.com/lightning-signer/txoo.git
TXOO_GIT_HASH=501eef7ace226f49e30e91d4c3520d34abe89fd7
# vls version v0.13.0
VLS_REPO=https://gitlab.com/lightning-signer/validating-lightning-signer.git
VLS_GIT_HASH=d1985121d1dfbbd50d2eeea42467f334105a2143
VLS_FEATURES=default
# lss version v0.4.0
LSS_REPO=https://gitlab.com/lightning-signer/validating-lightning-signer.git
LSS_GIT_HASH=d1985121d1dfbbd50d2eeea42467f334105a2143

IMAGE_TAG=regtest
TXOO_PUBLIC_KEY=02f6725f9c1c40333b67faea92fd211c18305036bdc03b073ce8e1b7f5627f4466
```

### Step 2: Configure Lightning Nodes

Create configuration files for both nodes.

**lightningd-regtest-1.conf**:

```bash
network=regtest
bitcoin-rpcuser=rpcuser
bitcoin-rpcpassword=VLSsigner1
bitcoin-rpcconnect=bitcoind-regtest
bitcoin-rpcport=38332
```

**lightningd-regtest-2.conf**:

```bash
network=regtest
bitcoin-rpcuser=rpcuser
bitcoin-rpcpassword=VLSsigner1
bitcoin-rpcconnect=bitcoind-regtest
bitcoin-rpcport=38332
```

Save these in the `vls-container` directory.

### Step 3: Docker Compose Configuration

Use the provided `docker-compose.two-nodes.yml` to define services for Bitcoin Core, Core Lightning nodes, and VLS. 

### Step 4: Start Containers

Build and start the containers:

```bash
docker compose -f docker-compose.two-nodes.yml up --build -d
```

## Funding the Nodes

### Step 5: Create and Fund Wallet

1. **Create Bitcoin Wallet**:

   ```bash
   docker container exec bitcoind-regtest bitcoin-cli createwallet default || true
   ```

   **Output**:

   ```
   {
     "name": "default"
   }
   ```

2. **Generate Mining Address**:

   ```bash
   docker container exec bitcoind-regtest bitcoin-cli getnewaddress
   ```

   **Output**:

   ```
   bcrt1q9yf4khj8yde79q3ccd57ulew8kdum3m757neg8
   ```

3. **Mine Blocks**:

   ```bash
   docker container exec bitcoind-regtest bitcoin-cli generatetoaddress 101 bcrt1q9yf4khj8yde79q3ccd57ulew8kdum3m757neg8
   ```

   **Output**:

   ```
   [
        "43251ca486e9071e8df010228cf7fcc00847dce8066f3c0ff7965dd19aa50a53",
        "6717f3d07d50f7c947a5f3ab172642841a167011014367b2fdcd2f94e283cffd",

        ...
        "36325b60e18d753c6f599a6de4f15e13bdd1b676f8b639ebd0d2c103c9ac52fc"
   ]
   ```

4. **Get Lightning Addresses**:

   ```bash
   docker container exec lightningd-regtest-1 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest1 newaddr
   docker container exec lightningd-regtest-2 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest2 newaddr
   ```

   **Output**:

   ```
   {
     "bech32": "bcrt1q4yeqtp029yz4442n7cf8kmnz8rz5wc7yeuh34m"
   }
   {
     "bech32": "bcrt1qgs8pp3vdzfgphq4mwj9mywv7dym0mkx5z734pk"
   }
   ```

5. **Send Funds**:

   ```bash
   docker container exec bitcoind-regtest bitcoin-cli sendtoaddress bcrt1q4yeqtp029yz4442n7cf8kmnz8rz5wc7yeuh34m 10
   docker container exec bitcoind-regtest bitcoin-cli sendtoaddress bcrt1qgs8pp3vdzfgphq4mwj9mywv7dym0mkx5z734pk 10
   ```

   **Output**:

   ```
   037f22c5b44a52cb5b5def8f9fc4a6cabfe08c521938482d2eb488c7b37232fe
   d3a2cfb9cc9c797268ec65643c9b90478bc5ce1198409cd954a0342caa492264
   ```

6. **Confirm Transactions**:

   ```bash
   docker container exec bitcoind-regtest bitcoin-cli generatetoaddress 1 bcrt1q9yf4khj8yde79q3ccd57ulew8kdum3m757neg8
   ```

   **Output**:

   ```
   [
        "59be07bdbb4dfa07e263149e3e98e64feee202cac62f0f5761edec0c72b5a572"
   ]
   ```

7. **Verify Funds**:

   ```bash
   docker container exec lightningd-regtest-1 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest1 listfunds
   docker container exec lightningd-regtest-2 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest2 listfunds
   ```

   **Output**:

   ```
   {
     "outputs": [
       {
         "txid": "037f22c5b44a52cb5b5def8f9fc4a6cabfe08c521938482d2eb488c7b37232fe",
         "output": 0,
         "value": 1000000000,
         "amount_msat": 1000000000000,
         "address": "bcrt1q4yeqtp029yz4442n7cf8kmnz8rz5wc7yeuh34m",
         "status": "confirmed",
         "reserved": false
       }
     ],
     "channels": []
   }
   {
     "outputs": [
       {
         "txid": "d3a2cfb9cc9c797268ec65643c9b90478bc5ce1198409cd954a0342caa492264",
         "output": 0,
         "value": 1000000000,
         "amount_msat": 1000000000000,
         "address": "bcrt1qgs8pp3vdzfgphq4mwj9mywv7dym0mkx5z734pk",
         "status": "confirmed",
         "reserved": false
       }
     ],
     "channels": []
   }
   ```

## Connecting Nodes and Opening a Channel

### Step 6: Connect Nodes

1. **Get Node IDs**:

   ```bash
   docker container exec lightningd-regtest-1 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest1 getinfo
   docker container exec lightningd-regtest-2 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest2 getinfo
   ```

   **Output**:

   ```
    {
        "id": "0379b2041e3e98cdb73bc0b495c36a0142453662278f0625165407fddde94f0db4",
        "alias": "VIOLENTSCAN",
        "color": "0379b2",
        "num_peers": 1,
        "num_pending_channels": 0,
        "num_active_channels": 1,
        "num_inactive_channels": 0,
        "address": [
            {
                "type": "ipv4",
                "address": "172.18.0.3",
                "port": 9735
            }
        ],
        "binding": [
            {
                "type": "ipv4",
                "address": "0.0.0.0",
                "port": 9735
            }
        ],
        "version": "v24.11.1",
        "blockheight": 114,
        "network": "regtest",
        "fees_collected_msat": 0,
        "lightning-dir": "/home/lightning/.lightning/regtest1/regtest",
        "our_features": {
            "init": "08a0880a8a59a1",
            "node": "88a0880a8a59a1",
            "channel": "",
            "invoice": "02000002024100"
        }
    }
    {
        "id": "03c1f5a62bf9859207eae8227029ce20f408a3a58a44e5da63ae7bd4eb8226657b",
        "alias": "BIZARREMONKEY",
        "color": "03c1f5",
        "num_peers": 1,
        "num_pending_channels": 0,
        "num_active_channels": 1,
        "num_inactive_channels": 0,
        "address": [
            {
                "type": "ipv4",
                "address": "172.18.0.4",
                "port": 9736
            }
        ],
        "binding": [
            {
                "type": "ipv4",
                "address": "0.0.0.0",
                "port": 9736
            }
        ],
        "version": "v24.11.1",
        "blockheight": 114,
        "network": "regtest",
        "fees_collected_msat": 0,
        "lightning-dir": "/home/lightning/.lightning/regtest2/regtest",
        "our_features": {
            "init": "08a0880a8a59a1",
            "node": "88a0880a8a59a1",
            "channel": "",
            "invoice": "02000002024100"
        }
    }
   ```

2. **Connect Node 1 to Node 2**:

   ```bash
   docker container exec lightningd-regtest-1 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest1 connect 03c1f5a62bf9859207eae8227029ce20f408a3a58a44e5da63ae7bd4eb8226657b lightningd-regtest-2 9736
   ```

   **Output**:

   ```
   {
     "id": "03c1f5a62bf9859207eae8227029ce20f408a3a58a44e5da63ae7bd4eb8226657b",
     "features": "08a0880a8a59a1",
     "direction": "out",
     "address": {
       "type": "ipv4",
       "address": "172.18.0.4",
       "port": 9736
     }
   }
   ```

3. **Verify Connection**:

   ```bash
   docker container exec lightningd-regtest-1 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest1 listpeers
   ```

   **Output**:

   ```
    {
        "peers": [
            {
                "id": "03c1f5a62bf9859207eae8227029ce20f408a3a58a44e5da63ae7bd4eb8226657b",
                "connected": true,
                "num_channels": 1,
                "netaddr": [
                    "172.18.0.4:9736"
                ],
                "features": "08a0880a8a59a1"
            }
        ]
    }
   ```

### Step 7: Open a Channel

1. **Fund Channel**:

   ```bash
   docker container exec lightningd-regtest-1 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest1 fundchannel 03c1f5a62bf9859207eae8227029ce20f408a3a58a44e5da63ae7bd4eb8226657b 1000000
   ```

   **Output**:

   ```
    {
        "tx": "02000000000101fe3272b3c788b42e2d483819528ce0bfcaa6c49f8fef5d5bcb524ab4c5227f030100000000fdffffff0240420f000000000022002071910004f25ccc7e6944487938d016f356c9130d48eb9de71d28dc99465a3ead1a878b3b00000000225120502f1cab090c231aee9f83b363725afeeb843b921f9443c32093b06721af07b0024730440220464bae5acdacf4d932b2504241c0743f224ec3aa404568c19864e786b161a81502207643affbfa973f0852e0393ad48ea4d4f2696f5e7b3a0031e917e620424a801c01210275eddcc499a1206ae1d1f144562bae5b74fc1adf971119343ae8b55e09f2096b6c000000",
        "txid": "881c5e54893ecb8d7748e3fc8d2fed7fe38d9d5cf0c4e8c740ffbfbd6fd0ef87",
        "channel_id": "87efd06fbdbfff40c7e8c4f05c9d8de37fed2f8dfce348778dcb3e89545e1c88",
        "channel_type": {
            "bits": [
                12,
                22
            ],
            "names": [
                "static_remotekey/even",
                "anchors/even"
            ]
        },
        "outnum": 0
    }
   ```

2. **Confirm Channel**:

   ```bash
   docker container exec bitcoind-regtest bitcoin-cli generatetoaddress 6 bcrt1q9yf4khj8yde79q3ccd57ulew8kdum3m757neg8
   ```

   **Output**:

   ```
    [
        "219bda411aeca9e28a3734cbd623eaeb33bf899d2ed612b278cf121494712047",
        "3c4ddb365e88cce8618484c94f18ba21680fb9d88255d4a8ebd5f0bfe9d8987f",
        "5ca2ef48c18fee63c4de465efaee93161d66dc11eb6e14efcf474bc08f7677e2",
        "7d457c3c7ea82d97290f9ecf5802a7be573e6e6102f2f873fec9d5db08ffa1c0",
        "5a8051e654d5b89763b7293c97416ead8f96902b3a4105b15795d3aa8df86684",
        "32adaa9cb93cc9ca5d6795faa20e2e441942cdf37854b8c614ec3fc7681deee0"
    ]
   ```

## Creating and Paying an Invoice

### Step 8: Create and Pay Invoice

1. **Create Invoice on Node 2**:

   ```bash
   docker container exec lightningd-regtest-2 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest2 invoice 100000 myinvoice2 "Test payment"
   ```

   **Output**:

   ```
   {
     "payment_hash": "44d3edf32be835d113545420d88dbc0dd1cc83cc352229576f70ca63343104a6",
     "expires_at": 1745432870,
     "bolt11": "lnbcrt1u1pnlla4xsp5nw5mc8z2dytpc739x2f3ts72y4xupmra9xlealtf8lk0t207a0zspp5gnf7muetaq6azy652ssd3rduphgueq7vx53zj4m0wr9xxdp3qjnqdq523jhxapqwpshjmt9de6qxqyjw5qcqp29qxpqysgqlxgc75mds750az3wclyszyemnz0m0fupuc8xnkaunhvu3nsw0geh05330gacet2rmvzlk5uakz3zwmx0xk6veuqua259a52dtv90xucppea89j",
     "payment_secret": "9ba9bc1c4a69161c7a25329315c3ca254dc0ec7d29bf9efd693fecf5a9feebc5",
     "created_index": 1
   }
   ```

2. **Pay Invoice from Node 1**:

   ```bash
   docker container exec lightningd-regtest-1 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest1 pay lnbcrt1u1pnlla4xsp5nw5mc8z2dytpc739x2f3ts72y4xupmra9xlealtf8lk0t207a0zspp5gnf7muetaq6azy652ssd3rduphgueq7vx53zj4m0wr9xxdp3qjnqdq523jhxapqwpshjmt9de6qxqyjw5qcqp29qxpqysgqlxgc75mds750az3wclyszyemnz0m0fupuc8xnkaunhvu3nsw0geh05330gacet2rmvzlk5uakz3zwmx0xk6veuqua259a52dtv90xucppea89j
   ```

   **Output**:

   ```
    {
        "payment_preimage": "ac633429ac85bef06016d33b2513db09dc0bac09b2096308d79365b3bb5a5924",
        "status": "complete",
        "amount_msat": 100000,
        "amount_sent_msat": 100000,
        "destination": "03c1f5a62bf9859207eae8227029ce20f408a3a58a44e5da63ae7bd4eb8226657b",
        "payment_hash": "44d3edf32be835d113545420d88dbc0dd1cc83cc352229576f70ca63343104a6",
        "created_at": 1744863587,
        "parts": 1
    }
   ```

3. **Verify Payment**:

   ```bash
   docker container exec lightningd-regtest-2 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest2 listinvoices
   ```

   **Output**:

   ```
    {
        "invoices": [
            {
                "label": "myinvoice2",
                "bolt11": "lnbcrt1u1pnlla4xsp5nw5mc8z2dytpc739x2f3ts72y4xupmra9xlealtf8lk0t207a0zspp5gnf7muetaq6azy652ssd3rduphgueq7vx53zj4m0wr9xxdp3qjnqdq523jhxapqwpshjmt9de6qxqyjw5qcqp29qxpqysgqlxgc75mds750az3wclyszyemnz0m0fupuc8xnkaunhvu3nsw0geh05330gacet2rmvzlk5uakz3zwmx0xk6veuqua259a52dtv90xucppea89j",
                "payment_hash": "44d3edf32be835d113545420d88dbc0dd1cc83cc352229576f70ca63343104a6",
                "amount_msat": 100000,
                "status": "paid",
                "pay_index": 1,
                "amount_received_msat": 100000,
                "paid_at": 1744863587,
                "payment_preimage": "ac633429ac85bef06016d33b2513db09dc0bac09b2096308d79365b3bb5a5924",
                "description": "Test payment",
                "expires_at": 1745432870,
                "created_index": 1,
                "updated_index": 1
            }
        ]
    }
   ```

## Verification

### Step 9: Collect Output

Verify the setup:

```bash
docker container exec bitcoind-regtest bitcoin-cli getwalletinfo
docker container exec lightningd-regtest-1 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest1 listfunds
docker container exec lightningd-regtest-2 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest2 listfunds
docker container exec lightningd-regtest-1 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest1 listpeers
docker container exec lightningd-regtest-2 lightning-cli --regtest --lightning-dir=/home/lightning/.lightning/regtest2 listinvoices
cat lightningd-regtest-1.conf
cat lightningd-regtest-2.conf
```

**Output**:

- **bitcoin-cli getwalletinfo**:

  ```
  {
    "walletname": "default",
    "walletversion": 169900,
    "balance": 50.00000000,
    ...
  }
  ```

- **lightning-cli listfunds** (Node 1):

  ```
  {
    "outputs": [],
    "channels": [
      {
        "peer_id": "03c1f5a62bf9859207eae8227029ce20f408a3a58a44e5da63ae7bd4eb8226657b",
        "connected": true,
        "state": "CHANNELD_NORMAL",
        "amount_msat": 1000000000,
        ...
      }
    ]
  }
  ```

- **lightning-cli listfunds** (Node 2):

  ```
  {
    "outputs": [],
    "channels": [
      {
        "peer_id": "0379b2041e3e98cdb73bc0b495c36a0142453662278f0625165407fddde94f0db4",
        "connected": true,
        "state": "CHANNELD_NORMAL",
        "amount_msat": 1000000000,
        ...
      }
    ]
  }
  ```

- **lightning-cli listpeers** (Node 1):

  ```
  {
    "peers": [
      {
        "id": "03c1f5a62bf9859207eae8227029ce20f408a3a58a44e5da63ae7bd4eb8226657b",
        "connected": true,
        "netaddr": [
          "172.18.0.4:9736"
        ],
        "channels": [
          {
            "state": "CHANNELD_NORMAL",
            ...
          }
        ]
      }
    ]
  }
  ```

- **lightning-cli listinvoices** (Node 2):

  ```
  {
    "invoices": [
      {
        "label": "myinvoice2",
        "bolt11": "lnbcrt1u1pnlla4xsp5...",
        "payment_hash": "44d3edf32be835d113545420d88dbc0dd1cc83cc352229576f70ca63343104a6",
        "amount_msat": 100000,
        "status": "paid",
        "description": "Test payment",
        ...
      }
    ]
  }
  ```

- **cat lightningd-regtest-1.conf**:

  ```
  network=regtest
  bitcoin-rpcuser=rpcuser
  bitcoin-rpcpassword=VLSsigner1
  bitcoin-rpcconnect=bitcoind-regtest
  bitcoin-rpcport=38332
  ```

- **cat lightningd-regtest-2.conf**:

  ```
  network=regtest
  bitcoin-rpcuser=rpcuser
  bitcoin-rpcpassword=VLSsigner1
  bitcoin-rpcconnect=bitcoind-regtest
  bitcoin-rpcport=38332
  ```

## Cleanup

### Step 10: Stop and Remove Containers

```bash
cd vls-container
docker compose -f docker-compose.two-nodes.yml down --volumes
docker network rm regtest_lightning || true
```

**Output**:

```
[+] Running 6/6
 ✔ Container lightningd-regtest-1  Removed
 ✔ Container lightningd-regtest-2  Removed
 ✔ Container vls-regtest-1         Removed
 ✔ Container vls-regtest-2         Removed
 ✔ Container bitcoind-regtest      Removed
 ✔ Network regtest_lightning       Removed
```

## Notes

- **Versions**: Bitcoin Core v27.0, Core Lightning v24.11.1, VLS v0.13.0.
- **VLS**: Acts as a remote signer via `--remote-hsmd-socket`.
- **Regtest**: Ideal for testing with no real funds.
- **Security**: Use strong credentials in production environments.

This setup ensures two Core Lightning nodes with VLS signers can connect, open a channel, and process payments on regtest, with Outputs to confirm each step.