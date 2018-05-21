# Prerequisite
- Install `Node.js@10.0.0`
- Install `@open-bucket/daemon` globally using `npm` (i.e `npm i -g @open-bucket/daemon`) 

# Architecture
- Daemon has __CLI__ and __TCP__ interface. User interacts with Daemon through __CLI__, Client interacts with Daemon through __TCP__ interface
- Daemon have __3 Primary Instances__: Producer, Consumer & Wallet
    - Producer: provides functionality for producing data into Open Bucket (OB) Network
    - Consumer: provides functionality for consuming data from OB Network
    - Wallet: provides functionality for managing user's Etherum accounts.
- Uses [Inquirer.js](https://github.com/SBoudrias/Inquirer.js) for interactive command line user interfaces
### Watchers
- Producer & Consumer have their own Watcher (PWatcher for Producer, CWatcher for Consumer)
- Watchers are started or terminated by its parent
- Watcher extends Node.js EventEmitter
- Watcher takes responsibility for detecting file changes and emit approriate event for each change (i.e when user add new file to __local Producer dir__, PWatcher emit `fileAdded` event)
- Use [Chokidar](https://github.com/vunguyenhung/chokidar) to detect file changes (e.g: webpack, nodemon,... )
     - Chokidar usage - Nodemon: https://github.com/vunguyenhung/nodemon/blob/master/lib/monitor/watch.js

## Ethereum
- Daemon uses [web3.js](https://github.com/ethereum/web3.js/) to interact with Ethereum Network
- Daemon stores user's Ethereum account in its self hosted wallet using [this](https://github.com/ConsenSys/eth-lightwallet)
### Smart Contract
- Only Tracker create & modify state of the contract.
- Contract is initialized with producer address.
- Producer can __pay__ Ether to the contract.
- Producer can __withdraw__ Ether from the contract.
    - If the remaining Ether is not enough to renew on new month, the contract will be terminated on 48 hours.
    - If the remaining Ether is not enough to pay consumers for a download request, the download request will be denied.
- Consumers can __withdraw__ Ether from the contract.
- Tracker keep track of the data flow in OB Network & transfer Ether from producer to consumers inside a contract.
    - For each download, Tracker transfer an ammount of Ether (based on GB downloaded) from producer to consumers.
    - For each month, Tracker transfer an amount of Ether (based on GB stored) from producer to consumers.
- Producer & consumers can terminate the contract at anytime through Tracker.

# Funtionality

## Daemon
### Init
1) [Optional] Add default Ethereum account to Wallet.
2) Define config file:
    - [Optional] Request user define Producer config.
    - [Optional] Request user define Consumer config.
3) Ask user to allow Daemon to start Producer & Consumer

### Config file
```json
{
    "producer": {
        "directory": "~/open-bucket/produce",
        "startOnStartup": false
    },
    "consumer": {
        "directory": "~/open-bucket/consume",
        "limit": "5 GB",
        "account": "user Ethereum account identifier",
        "startOnStartup": false
    }
}
```

## Producer
### Start Producer:
1) Read the config file.
2) [Optional] Request user to add default Ethereum account to Wallet.
3) Start Producer

### Stop Producer:
1) Destroy all current connection from Open Bucket consumers
2) Stop Producer

### Produce data to the Network:
1) [Optional] Start the Producer.
2) [Optional] Copy the file to __local Producer directory__
3) Ask user to provide information:
    - User's Ethereum account (choose from Wallet)
    - [Optional] File tier: Free | Basic | Premium
    - [Optional] Desired availability
    - [Optional] The amount of time the file exists inside the OB network. 
4) Producer create the Smart Contract & charge user.
5) Producer start the upload the file process.
6) After the file has been completely uploaded into the network, Producer change the status of the file to Producer.
7) Producer delete the file locally

### Remove a file from the Network
1) Validate input including: __fileId__, __path__
2) [Optional] Delete the file on local Producer directory
3) Tracker close the contract
4) Send delete message to all related consumer when the contract is closed
5) Clean up all resources related to the file on Tracker
6) Notify Producer after Tracker received delete confirmation message from all consumers

## Download data from the Network:
1) Validate inputs
2) Producer ask Tracker shards information of the specified file & receive the list of available consumers
3) Producer start file downloading process.
4) After the file download process is completed, Tracker transfer Ether from producer to consumers inside the Smart Contract

## Change Producer config:
1) Validate inputs
2) Producer apply the changes

## Change file metadata on Tracker:
1) Validate inputs
2) Send request to Tracker
3) Tracker applies the changes
3) Tracker notify Daemon when done

## Consumer
### Start Consumer:
1) Read the config file.
2) [Optional] Request user to add default Ethereum account to Wallet.
3) [Optional] Request user to choose consumer Ethereum account from Wallet
4) Start Consumer.
#### Notes:
- When a consumer receives a shard, it send shard's MD5 hash to Tracker for confirmation. Tracker will increase the file's availability if the condition are matched

## Change Consumer config:
1) Validate inputs
2) Consumer apply the changes