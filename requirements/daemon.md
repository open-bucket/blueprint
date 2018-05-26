# Prerequisite
- Install `Node.js@10.0.0`
- Install `@open-bucket/daemon` globally using `npm` (i.e `npm i -g @open-bucket/daemon`)

# Architecture
- Daemon has __CLI__ and __TCP__ interface. User interacts with Daemon through __CLI__, Client interacts with Daemon through __TCP__ interface
- Daemon have __3 Primary Instances__: Producer, Consumer & Wallet
    - Producer: provides functionality for __producing storage space__ to Open Bucket Network (OBN)
    - Consumer: provides functionality for __consuming storgae space__ from OBN
    - Wallet: provides functionality for managing user's Etherum accounts.
- Uses [Inquirer.js](https://github.com/SBoudrias/Inquirer.js) for interactive command line user interfaces
### Watchers
- Producer & Consumer have their own Watcher (PWatcher for Producer, CWatcher for Consumer)
- Watchers are started or terminated by its parent
- Watcher extends Node.js EventEmitter
- Watcher takes responsibility for detecting file changes and emit approriate event for each change (i.e when user add new file to __Consumer space__, CWatcher emit `fileAdded` event)
- Use [Chokidar](https://github.com/vunguyenhung/chokidar) to detect file changes (e.g: webpack, nodemon,... )
     - Chokidar usage - Nodemon: https://github.com/vunguyenhung/nodemon/blob/master/lib/monitor/watch.js

## Ethereum
- Daemon uses [web3.js](https://github.com/ethereum/web3.js/) to interact with Ethereum Network
- Daemon stores user's Ethereum account in its self hosted wallet using [this](https://github.com/ConsenSys/eth-lightwallet)
### Smart Contract
- Only Tracker create & modify state of the contract.
- Contract is initialized with consumer address.
- Consumer can __pay__ Ether to the contract.
- Consumer can __withdraw__ Ether from the contract.
    - If the remaining Ether is not enough to renew on new month, the contract will be terminated on 48 hours.
    - If the remaining Ether is not enough to pay consumers for a download request, the download request will be denied.
- Producers can __withdraw__ Ether from the contract.
    - When a producer doesn't serve a file for 2 days, Tracker will automatically remove it from the producer list of that file.
- Tracker keep track of the data flow in OB Network & transfer Ether from consumer to producers inside a contract.
    - For each download, Tracker transfer an ammount of Ether (based on GB downloaded) from consumer to producers.
    - For each month, Tracker transfer an amount of Ether (based on GB stored) from consumer to producers.
- Producer & consumers can terminate the contract at anytime through Tracker.

# Funtionality

## Daemon
### Configs
1) [Optional] Add default Ethereum account to Wallet.
2) Define config file:
    - [Optional] Request user define Consumer config.
    - [Optional] Request user define Producer config.
3) Ask user to allow Daemon to start Producer & Consumer

### Config file
```json
{
    "wallet": {
        "secretPath": "path/to/your/seed/password"
    },
    "consumer": {
        "directory": "~/open-bucket/producer-space"
    },
    "producer": {
        "directory": "~/open-bucket/consumer-space",
        "size": "5GB"
    }
}
```

## Consumer
### Start Consumer:
1) Read the config file.
2) [Optional] Request user to add default Ethereum account to Wallet.
3) Start Consumer

### Stop Consumer:
1) Destroy all current connection from Open Bucket producer
2) Stop Consumer

### Consume space from OBN:
1) [Optional] Start the Consumer.
2) [Optional] Copy the file to __Consumer space__
3) Ask user to provide information:
    - User's Ethereum account (choose from Wallet)
    - [Optional] File tier: Free | Basic | Premium
    - [Optional] Desired availability
    - [Optional] The amount of time the file exists inside OBN. 
4) Tracker create the Smart Contract & charge user.
5) Consumer start the upload the file process.
6) After the file has been completely uploaded into the network, Consumer change the status of the file to UPLOADED.
7) Consumer delete the file locally

### Remove a file OBN
1) Validate input including: __fileId__, __path__
2) Tracker close the contract
3) Send delete message to all related producers when the contract is closed
4) Clean up all resources related to the file on Tracker
5) Notify Consumer after Tracker received delete confirmation message from all consumers

## Download data from OBN:
1) Validate inputs
2) Consumer ask Tracker shards information of the specified file & receive the list of available consumers
3) Consumer start file downloading process.
4) After the file download process is completed, Tracker transfer Ether from consumer to producers inside the Smart Contract

## Change Consumer config:
1) Validate inputs
2) Consumer apply the changes

## Change file metadata on Tracker:
1) Validate inputs
2) Send request to Tracker
3) Tracker applies the changes
3) Tracker notify Daemon when done

## Producer
### Start Producer:
1) Read the config file.
2) [Optional] Request user to add default Ethereum account to Wallet.
3) [Optional] Request user to choose producer Ethereum account from Wallet
4) Start Producer.
#### Notes:
- When a producer receives a shard, it send shard's MD5 hash to Tracker for confirmation. Tracker will increase the file's availability if the condition are matched
- When a producer doesn't serve a file for 2 days, Tracker will automatically remove it fron the producer list of that file.

## Change Producer config:
1) Validate inputs
2) Producer apply the changes