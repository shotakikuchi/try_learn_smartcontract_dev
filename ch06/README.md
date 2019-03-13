## 6-1 データモデリング

- ユーザーテーブルのCREATE文
```sql
CREATE TABLE IF NOT EXISTS `users` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `nickname` VARCHAR(255) DEFAULT NULL,
  `mail` VARCHAR(255) DEFAULT NULL,
  `ethaddress` VARCHAR(255) NOT NULL,
  `encrypted_password` VARCHAR(255) DEFAULT NULL,
  `salt` VARCHAR(255) DEFAULT NULL,
  `updated_at` DATETIME NULL DEFAULT CURRENT_TIMESTAMP,
  `created_at` DATETIME NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE INDEX `mail_UNIQUE` (`mail` ASC),
  UNIQUE INDEX `ethaddress_UNIQUE` (`ethaddress` ASC)
)
ENGINE=InnoDB
DEFAULT CHARSET=utf8
AUTO_INCREMENT=1
COMMENT = 'master table of users';
```

- ルームテーブルのCREATE文
```sql
CREATE TABLE IF NOT EXISTS `rooms` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `owner_id` BIGINT UNSIGNED NOT NULL,
  `owner_address` VARCHAR(255) DEFAULT NULL,
  `title` VARCHAR(255) DEFAULT NULL,
  `description` TEXT DEFAULT NULL,
  `event_code` VARCHAR(255) NOT NULL,
  `address` VARCHAR(255) DEFAULT NULL,
  `create_tx_hash` VARCHAR(255) DFAULT NULL,
  `is_private` TINYINT(1) DEFAULT 0,
  `wei_balance` BIGINT UNSIGNED DEFAULT 0,
  `wei_price` BIGINT UNSIGNED DEFAULT 0,
  `start_time` DATETIME DEFAULT NULL,
  `end_time` DATETIME DEFAULT NULL,
  `active` TINYINT(1) DEFAULT 1,
  `updated_at` DATETIME NULL DEFAULT CURRENT_TIMESTAMP,
  `created_at` DATETIME NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`owner_id`),
  UNIQUE INDEX `event_code_UNIQUE` (`event_code` ASC),
  CONSTRAINT `fk_rooms_users`
    FOREIGN KEY (`owner_id`)
    REFERENCES `users` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION
)
ENGINE=InnoDB
DEFAULT CHARSET=utf8
COMMENT='master table of rooms';
```

- 質問テーブルのCREATE文
```sql
CREATE TABLE IF NOT EXISTS `questions` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `room_id` BIGINT UNSIGNED NOT NULL,
  `address` VARCHAR(255) NOT NULL,
  `owner_id` BIGINT UNSIGNED DEFAULT NULL,
  `body` TEXT DEFAULT NULL,
  `adopted` TINYINT(1) DEFAULT 0,
  `updated_at` DATETIME NULL DEFAULT CURRENT_TIMESTAMP,
  `created_at` DATETIME NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  CONSTRAINT `fk_question_rooms`
    FOREIGN KEY (`room_id`)
    REFERENCES `rooms` (`id`)
    ON DELETE NO ACTION
    ON DELETE NO ACTION
)
  ENGINE=InnoDB
  DEFAULT CHARSET=utf8
  COMMENT='master table of question';
```

- 預託・報酬支払トランザクションテーブルのCREATE文
```sql
-- Deposition transaction table. 
CREATE TABLE IF NOT EXISTS `tx_deposits` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `room_id` BIGINT UNSIGNED NOT NULL,
  `tx_hash` VARCHAR(255) DEFAULT NULL,
  `confirmed` TINYINT(1) DEFAULT 0,
  `success` TINYINT(1) DEFAULT 0,
  `sender` VARCHAR(255) NOT NULL DEFAULT '0x0',
  `receiver` VARCHAR(255) NOT NULL DEFAULT '0x0',
  `wei_amount` BIGINT UNSIGNED NOT NULL DEFAULT 0,
  `updated_at` DATETIME NULL DEFAULT CURRENT_TIMESTAMP,
  `created_at` DATETIME NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  INDEX (`room_id`)
)
  ENGINE=InnoDB
  DEFAULT CHARSET=utf8
  COMMENT='transaction table of deposits';


-- Payments transaction table.
CREATE TABLE IF NOT EXISTS `tx_payments` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `question_id` BIGINT UNSIGNED NOT NULL,
  `tx_hash` VARCHAR(255) DEFAULT NULL,
  `confirmed` TINYINT(1) DEFAULT 0,
  `success` TINYINT(1) DEFAULT 0,
  `sender` VARCHAR(255) NOT NULL DEFAULT '0x0',
  `receiver` VARCHAR(255) NOT NULL DEFAULT '0x0',
  `wei_amount` BIGINT UNSIGNED NOT NULL DEFAULT 0,
  `updated_at` DATETIME NULL DEFAULT CURRENT_TIMESTAMP,
  `created_at` DATETIME NULL DEFAULT CURRENT_TIMESTAMP
  PRIMARY KEY (`id`),
  INDEX (`question_id`)
)
  EINGINE=InnoDB
  DEFAULT CHARSET=utf8
  COMMENT='transaction table of payments';
```


## 6-3 Truffleフレームワークによる開発準備

- Truffleプロジェクト用のディレクトリ作成
```
$ mkdir myproject
```

- Truffleプロジェクトの新規作成
```
$ cd myproject
$ truffle init
```

- Truffle Developの起動
```
$ truffle develop
```

- 通常のコンパイル
```
$ truffle compile
```

- Truffleコンソールでのコンパイル
```
truffle(devgelop) > compile
```

- Truffleコンソールでのデプロイ
```
truffle(develop) > migrate
```

- truffle-config.jsでのネットワーク設定
```js
module.exports = {
    networks: {
    
            development: {
                host: '127.0.0.1',
                port: 8545,
                network_id: 15,
                gas: 4700000
            },
            ganache: {
                host: '127.0.0.1',
                port: 7545,
                network_id: '*'
            }
        }
}
```

- Truffleコンソール起動コマンド
```
$ truffle console --network <truffleo-config.jsで設定したネットワーク名>
```

- developmentネットワークを指定してTruffleコンソールを起動
```
$ truffle console --network development 
```

- 「development」ネットワーク面を省略してTruffleコンソールを起動
```
$ truffle console
```

- ganacheネットワークを指定してTruffleコンソールを起動
```
$ truffle console --network ganache
```

## 6-4 コントラクトの実装

### OpenZeppelinのインストール

- package.jsonの作成
```
$ npm init -y
```

- OpenZeppelinのインストール
```
$ npm install openzeppelin-solidity@1.12.0
```

### コントラクトの作成
- Roomコントラクト(Room.sol)
```solidity
pragma solidity ^0.4.24;

import "openzeppelin-solidity/contracts/lifecycle/Pausable.sol";
import "openzeppelin-solidity/contracts/lifecycle/Destructible.sol";
import "./Activatable.sol";

contract Room is Destructible, Pausable , Activatable{

    mapping(uint256 => bool) public rewardSent;

    event Deposited(
        address indexed _depositor,
        uint256 _depositedValue
    );

    event RewardSent(
        address indexed _dest,
        uint256 _reward,
        uint256 _id
    );

    event RefundedToOwner(
        address indexed _dest,
        uint256 _refundedBalance
    );

    constructor(address _creator) public payable {
        owner = _creator;
    }

    function deposit() external payable whenNotPaused {
        require(msg.value > 0);
        emit Deposited(msg.sender, msg.value);
    }

    function sendReward(uint256 _reward, address _dest, uint256 _id) external onlyOwner {
        require(!rewardSent[_id]);
        require(_reward > 0);
        require(address(this).balance >= _reward);
        require(_dest != address(0));
        require(_dest != owner);

        rewardSent[_id] = true;
        _dest.transfer(_reward);
        emit RewardSent(_dest, _reward, _id);
    }

    function refundToOwner() external whenNotActive onlyOwner {
        require(address(this).balance > 0);

        uint256 refundedBalance = address(this).balance;
        owner.transfer(refundedBalance);
        emit RefundedToOwner(msg.sender, refundedBalance);
    }
}
```

- RoomFactory.sol
```solidity
pragma solidity ^0.4.24;

import "openzeppelin-solidity/contracts/lifecycle/Pausable.sol";
import "openzeppelin-solidity/contracts/lifecycle/Destructible.sol";
import "./Room.sol";

contract RoomFactory is Destructible, Pausable {

    event RoomCreated(
        address indexed _creater,
        address _room,
        uint256 _depositValue
    );

    function createRoom() external payable whenNotPaused {
        address newRoom = (new Room).value(msg.value)(msg.sender);
        emit RoomCreated(msg.sender, newRoom, msg.value);
    }
}
```

- Activatable.sol
```solidity
pragma solidity ^0.4.24;

import "openzeppelin-solidity/contracts/ownership/Ownable.sol";

contract Activatable is Ownable {
    event Deactivate(address indexed _sender);
    event Activate(address indexed _sender);

    bool public active = false;

    modifier whenActive() {
        require(active);
        _;
    }

    modifier whenNotActive() {
        require(!active);
        _;
    }

    function deactivate() public whenActive onlyOwner {
        active = false;
        emit Deactivate(msg.sender);
    }

    function activate() public whenNotActive onlyowner {
        active = true;
        emit Activate(msg.sender);
    }
}
```

### コンパイル

- Solidityのコンパイル
```
$ truffle compile
```

- Solidity全体のコンパイル
```
$ truffle compile --all
```

### プライベートネットへのデプロイ

- マイグレーションファイルの作成(2_deploy_room_factory.js)
```javascript
const RoomFactory = artifacts.require('./RoomFactory.sol');

module.exports = deployer => {
    deployer.deploy(RoomFactory);
    
};
```

- Ganacheへデプロイ
```
$ truffle migrate --network ganache
```

- 全てのマイグレーションファイルを実行してGanacheへデプロイ
```
$ truffle migrate --reset --network ganache
```

- コンパイルとマイグレーションの同時実行
```
$ truffle migrate --reset --compile-all --network ganache
```

## プライベートネットでの動作確認

### デプロイとコントラクトオブジェクトの作成

- Gethプライベートネットへのデプロイ
```
$ truffle migrate --reset
```

- マイニングの停止
```
> miner.stop()
true
```

- コントラクトオブジェクトを定義
```
> var roomFactory = eth.contract(先ほど取得したABI).at('コントラクトアドレス')
```

- コントラクト情報を表示
```
> roomFactyory

{
  abi: [{
      constant: false,
      inputs: [],
      name: "unpause",
      outputs: [],
      payable: false,
      signature: "0x3f4ba83a",
      stateMutability: "nonpayable",
      type: "function"
      
      ...
      
       address: "0x642B549f4820fca59113936c84E76AD3481325d4",
        transactionHash: null,
        OwnershipRenounced: function(),
        OwnershipTransferred: function(),
        Pause: function(),
        RoomCreated: function(),
        Unpause: function(),
        allEvents: function(),
        createRoom: function(),
        destroy: function(),
        destroyAndSend: function(),
        owner: function(),
        pause: function(),
        paused: function(),
        renounceOwnership: function(),
        transferOwnership: function(),
        unpause: function()
      }

```


### コントラクトの状態を呼び出し(call)

- paused関数をcall
```
> roomFactory.paused.call()
false
```

- paused関数をcall
```
> roomFactory.paused()
false
```

### コントラクトの状態変更(transaction)

- sendTransactionを使ってSolidityで定義した関数を呼び出す
```
<コントラクトオブジェクト>.<関数名>.sendTransaction(引数, {from: 呼び出し元アドレwす, gas: ガスリミット, value: 送金額})

```

- createRoom関数をtransactionとして呼び出す
```
> roomFactory.createRoom.sendTransaction({from: eth.accounts[0], gas: 100000, value: web3.toWei(0.1, 'ether')})
"0xb43d93a356b5887b1a1912a44a0a52bf08cf4dcbcaa4f7706d791c61315ee6c4"
```

### 結果の確認

- トランザクションの確認
```
> eth.getTransaction('0xb43d93a356b5887b1a1912a44a0a52bf08cf4dcbcaa4f7706d791c61315ee6c4')

{
  blockHash: "0x3c585e3ada03faf4650d65b78f8d0887d647a6fba3f351faa1eecb1a4e97ee85",
  blockNumber: 2183,
  from: "0x1854520d659d17832e916a3eed81a4580c2e9b24",
  gas: 100000,
  gasPrice: 1000000000,
  hash: "0xb43d93a356b5887b1a1912a44a0a52bf08cf4dcbcaa4f7706d791c61315ee6c4",
  input: "0x3be272aa",
  nonce: 16,
  r: "0xe59cfb7848c332c4ea7737b9dd8b56cc9983a09b4543318f66f865dea721b737",
  s: "0x172cf56586c814b99f1ccf54274680b7e805f01b73fadb0d4b9b85fe9dfeef26",
  to: "0x642b549f4820fca59113936c84e76ad3481325d4",
  transactionIndex: 0,
  v: "0x41",
  value: 100000000000000000
}

```

