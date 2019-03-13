# 使用したコマンド


## ローカル環境でのプライベートネットの構築
- private-netディレクトリの作成
```
$ mkdir private-net
```

- プライベートネットの作成
```
$ geth --datadir ./private-net --nodiscover --maxpeers 0 init ./private-net/genesis.json 
```

- go-ethereumでのプライベートネットの起動（geth-start.sh）
```
$ geth --datadir ./private-net --networkid 15 \
--nodiscover --maxpeers 0 --mine --minerthreads 1 \
--rpc --rpcaddr "0.0.0.0" --rpccorsdomain "*" \
--rpcvhosts "*" --rpcapi "eth,web3,personal,net" \
--ipcpath   ./private-net/geth.ipc --ws --wsaddr "0.0.0.0" \
--wsapi "eth,web3,personal,net" --wsorigins "*" \
--unlock 0,1,2,3,4 --password ./private-net/password
```

- go-ethereumへの接続
```
$ geth attach http://localhost:8545
```

- ブロック高の確認
```
$ eth.blockNumber
```


## gethoによるプライベートネットの構築

- gethoノードへのリクエスト
```
$ curl -X POST https://smart-wolf-40088.getho.io/jsonrpc \
-H 'Content-Type:application/json' \
--data '{"jsonrpc":"2.0","method":"eth_accounts","params":[],"id": 1}' {"jsonrpc":"2.0","id":1,"result":["0xb1f407dcc37cdc0d5193c09f499d3766aa4c5743","0x836c0c0eb4368ada386b319e69eace9bf9a96418","0xc5693c0c32529b556b523c59918ed6f148b68beb"]}
```

- Truffleのサンプルをダウンロード
```
$ mkdir truffle-metacoin
$ cd truffle-metacoin
$ truffle unbox metacoin
```

- gethoノードへのコントラクトのデプロイ
```
$ truffle migrate --network getho
```

## ログによる実践的な動作確認

1. bootnode
2. test-geth1
3. test-geth2

### 事前準備
- 作業ディレクトリ作成
```
$ mkdir -p ./workspace/find-node
```

- 各データを保存するディレクトリ作成
```
$ mkdir -p ./workspace/find-node/bootnode \
./workspace/find-node/test-geth1 \
./workspace/find-node/test-geth2

```

- 各ログファイル作成
```
$ touch ./workspace/find-node/bootnode/bootnode.log \
./workspace/find-node/test-geth1/geth.log \
./workspace/find-node/test-geth2/geth.log
```

### bootnode設定

- bootnodeの秘密鍵生成と確認
```
$ cd ./workspace/find-node/bootnode/
$ bootnode --genkey boot.key
$ cat boot.key
???
```

- bootnode起動
```
$ cd ../../..
$ ls
workspace
$ bootnode --nodekey ./workspace/find-node/bootnode/boot.key \
--verbosity 9 2>> ./workspace/find-node/bootnode/bootnode.log
```