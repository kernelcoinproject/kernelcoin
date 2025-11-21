# Mining

## CPU
- https://github.com/Kudaraidee/cpuminer-opt-kudaraidee/
- https://github.com/JayDDee/cpuminer-opt

Start a RPC daemon
```
mkdir -p ~/.kernelcoin 
cat > ~/.kernelcoin/kernelcoin.conf << EOF 
# enable p2p
listen=1
# debug log details about transactions
txindex=1
logtimestamps=1
# Enable RPC
server=1
rpcuser=mike
rpcpassword=x
rpcport=9332
rpcallowip=127.0.0.1
rpcallowip=192.168.0.0/16
rpcallowip=172.17.0.0/16
rpcbind=0.0.0.0
maxtxfee=999999
EOF
./kernelcoind
```

Setup a proxy to modify requests to add mweb
```
#!/usr/bin/env python3
import aiohttp, asyncio, json
from aiohttp import web

RPC_URL = "http://127.0.0.1:9332"
RPC_AUTH = aiohttp.BasicAuth("mike", "x")

async def handle(request):
    data = await request.json()
    if data.get("method") == "getblocktemplate":
        data["params"] = [{"rules": ["segwit", "mweb"]}]
    async with aiohttp.ClientSession(auth=RPC_AUTH) as s:
        async with s.post(RPC_URL, json=data) as r:
            result = await r.text()
    return web.Response(text=result, content_type="application/json")

app = web.Application()
app.router.add_post("/", handle)
web.run_app(app, port=9333)
```
```
python3 proxy.py
```

Generate a wallet address either via gui or cli
```
./kernelcoin-cli createwallet "main"
./kernelcoin-cli getnewaddress "" legacy
```
Mine away to your wallet address
```
./cpuminer-zen2 -a scrypt --url=http://127.0.0.1:9333 --user=mike --pass=x --coinbase-addr=K7tWowKEAPTQAXcJTo2z7qihAe74vak4ib
.\cpuminer-avx2.exe -a scrypt --url=http://127.0.0.1:9333 --user=mike --pass=x --coinbase-addr=kcn1q5vh476x8gplnmsyklyd0zqhw4akp7kzj72anvu 
```
My ryzen 5 3600 cpu gives me ~57kH/s

## GPU
1. Build your own mining pool

I'll spare you the effort, I did not have good luck with [yiimp](https://github.com/tpruvot/yiimp), [nomp](https://github.com/meterio/nomp), or [coiniumserv](https://github.com/bonesoul/CoiniumServ), or trying to write my own based off a [ravencoin proxy](https://github.com/kralverde/ravencoin-stratum-proxy)

I didn't explore https://github.com/Crypto-Expert/stratum-mining

With some tweaks I managed to get MiningCore working. 

```
docker run -it -p 4000:4000 -p 4066:4066 -p 4067:4067 -p 3333:3333 --rm ubuntu:22.04 bash

echo "America/Chicago" > /etc/timezone
ln -fs /usr/share/zoneinfo/America/Chicago /etc/localtime
apt update
apt install -y git dotnet-sdk-6.0 git cmake build-essential libssl-dev pkg-config libboost-all-dev libsodium-dev libzmq5 nano tzdata postgresql

git clone https://github.com/oliverw/miningcore.git
cd miningcore/src/Miningcore
BUILDIR=${1:-../../build}
echo "Building into $BUILDIR" 
dotnet publish -c Release --framework net6.0 -o $BUILDIR

pg_ctlcluster 14 main start
su postgres
psql
CREATE ROLE miningcore WITH LOGIN ENCRYPTED PASSWORD 'your-secure-password';
CREATE DATABASE miningcore OWNER miningcore;
\q
psql -d miningcore -f Persistence/Postgres/Scripts/createdb.sql
exit
cd /miningcore
# Change pool wallet address, update ip
cat > /miningcore/kernelcoin.json << EOF
{
  "api": {
      "enabled": true,
      "listenAddress": "127.0.0.1",
      "port": 4000,
      "rateLimit": {
        "disabled": true
      }
  },

  "logging": {
    "level": "info",
    "enableConsoleLog": true,
    "enableConsoleColors": true,
    "logDirectory": "logs"
  },

  "banning": {
    "enabled": false
  },

  "notifications": {
    "enabled": false
  },

  "persistence": {
    "postgres": {
      "host": "127.0.0.1",
      "port": 5432,
      "user": "miningcore",
      "password": "your-secure-password",
      "database": "miningcore"
    }
  },

  "paymentProcessing": {
    "enabled": true,
    "interval": 600,
    "shareRecoveryFile": "recovered-shares.txt"
  },

  "pools": [
    {
      "id": "kernelcoin",
      "enabled": true,

      "coin": "kernelcoin",

      "address": "KP8Usn6Mz7RnX3rDCvAVf5u1eiLMHGfsyP",

      "rewardRecipients": [], 

      "addressPrefixes": {
        "pubkey": [45],
        "script": [23, 25],
        "bech": "kcn"
      },

      "jobRebroadcastTimeout": 10,
      "blockRefreshInterval": 1000,
      "clientConnectionTimeout": 600,

      "blockTemplateRpcExtraParameters": {
        "rules": ["segwit", "mweb"]
      },

      "ports": {
        "3333": {
          "listenAddress": "0.0.0.0",
          "difficulty": 0.1,
          "name": "CPU/GPU Mining",
          "varDiff": {
            "minDiff": 0.1,
            "maxDiff": 50,
            "targetTime": 15,
            "retargetTime": 60,
            "variancePercent": 30
          }
        }
      },

      "daemons": [
        {
          "host": "192.168.0.186",
          "port": 9333,
          "user": "mike",
          "password": "x"
        }
      ],

      "coreWallet": {
        "host": "192.168.0.186",
        "port": 9333,
        "user": "mike",
        "password": "x"
      },

      "paymentProcessing": {
        "enabled": true,
        "minimumConfirmations": 2,
        "minimumPayment": 1,
        "payoutScheme": "PPLNS",
        "payoutSchemeConfig": {
          "factor": 2.0
        }
      }
    }
  ]
} 
EOF
sed -i '1r /dev/stdin' build/coins.json <<'EOF'
"kernelcoin": {
    "name": "Kernelcoin",
    "canonicalName": "Kernelcoin",
    "symbol": "KCN",
    "family": "bitcoin",
    "explorerBlockLink":"https://explorer.kernelcoin.io/block/{0}",
    "explorerTxLink": "https://explorer.kernelcoin.org/explorer/?search={0}",
    "explorerAccountLink": "https://explorer.kernelcoin.org/explorer/?search={0}",
    "website": "",
    "market": "",
    "twitter": "",
    "telegram": "",
    "discord": "",

    "coinbaseHasher": {
        "hash": "sha256d"
    },

    "headerHasher": {
        "hash": "scrypt",
        "args": [
            1024,
            1
        ]
    },

    "blockHasher": {
        "hash": "reverse",
        "args": [
            {
                "hash": "sha256d"
            }
        ]
    },

    "posBlockHasher": {
        "hash": "reverse",
        "args": [
            {
                "hash": "scrypt",
                "args": [
                    1024,
                    1
                ]
            }
        ]
    },

    "shareMultiplier": 65536,

    "addressPrefixes": {
        "pubkey": [45],
        "script": [23, 25],
        "bech": "kcn"
    }
},
EOF
./build/Miningcore -c kernelcoin.json
```
2. Spin up a proxy to modify requests from Miningcore to your local rpc daemon. You could probably patch miningcore somewhere as well https://github.com/jon4hz/miningcore-foss/commits/ltc-fix

```
cat > proxy.py << EOF
# Update ip/user/pass for local rpc
#!/usr/bin/env python3
import aiohttp
import asyncio
import json
from aiohttp import web

RPC_URL = "http://127.0.0.1:9332"
RPC_AUTH = aiohttp.BasicAuth("mike", "x")

async def forward_rpc(payload):
    async with aiohttp.ClientSession(auth=RPC_AUTH) as session:
        async with session.post(RPC_URL, json=payload) as resp:
            return await resp.text()

def patch_call(call):
    if not isinstance(call, dict):
        return call

    if call.get("method") == "getblocktemplate":
        # Miningcore sends params as:
        # [ { "capabilities": [...], "rules": [...], ... } ]
        if call.get("params") and isinstance(call["params"], list) and len(call["params"]) > 0:
            p = call["params"][0]

            if isinstance(p, dict):
                # Only replace the rules key
                p["rules"] = ["segwit", "mweb"]

    return call

async def handle(request):
    try:
        body = await request.text()
        data = json.loads(body)

        # Batch vs single
        if isinstance(data, list):
            patched = [patch_call(call) for call in data]
        else:
            patched = patch_call(data)

        response_text = await forward_rpc(patched)
        return web.Response(text=response_text, content_type="application/json")

    except Exception as e:
        print(str(e))
        err = {"error": "internal proxy error"}
        return web.Response(text=json.dumps(err), content_type="application/json", status=500)

app = web.Application()
app.router.add_post("/", handle)

web.run_app(app, port=9333)
EOF
python3 proxy.py
```
3. Mine

Test mining pool with cpuminer-opt
```
./cpuminer-zen2 -a scrypt -o stratum+tcp://127.0.0.1:3333 -u K7tWowKEAPTQAXcJTo2z7qihAe74vak4ib -p x
```

With an nvidia gpu I was able to use https://github.com/tpruvot/ccminer/releases to get 600kH/s on my rtx 3050

```
.\ccminer-x64.exe -a scrypt -o stratum+tcp://192.168.0.186:3333 -u kcn1qc6urf9kvx2m3xvzlnwvla09v9rlkfv0pn2lph2 -p x
```

With an amd gpu I was able to get 200kH/s on a very old 7790 Sapphire
```
cgminer.exe --scrypt -o stratum+tcp://192.168.0.186:3333 -u K7tWowKEAPTQAXcJTo2z7qihAe74vak4ib -p x
```

Additional flags to cgminer matter a lot for speed. 

```
-I 17 -g 1 -w 128 --thread-concurrency 8000
```

YMMV!

You might be able to use other software as well with extensive modifications :)

Kernelcoin is a fork from ltc 21.4 and currently has segwit enabled, but a nonactive mweb. Your pool/miner needs to be aware of gbt and mweb from 2022 to be able to mine properly.

## ASIC

I don't have one, but I imagine it'll work the same as a gpu, just connect it to your pool
