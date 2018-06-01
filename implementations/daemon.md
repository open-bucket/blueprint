# Layout:
- [Prerequisite](#prerequisite)
- [Architecture](#architecture)
- [Wallet](#wallet)
    - [Wallet init](#wallet-init)
    - [Wallet config](#wallet-config)
- [Consumer](#consumer)
    - [Create Consumer](#create-comsumer)
    - [Start Consumer](#start-consumer)
    - [Stop Consumer](#stop-consumer)
    - [Upload a file to OBN](#consume-storage-from-obn)
    - [Remove a file from OBN](#remove-a-file-from-obn)
    - [Download data from OBN](#download-data-from-obn)
    - [Apply new Consumer config](#apply-new-consumer-config)
    - [Change file metadata on Tracker](#change-file-metadata-on-tracker)
    - [Browse files on OBN](#browse-files-on-obn)
    - [Consumer contract termination process](#consumer-contract-termination-process)
    - [Kill Consumer](#kill-consumer)
- [Producer](#producer)
    - [Create Producer](#create-producer)
    - [Start Producer](#start-producer)
    - [Apply new Producer config](#apply-new-producer-config)
    - [Producer contract termination process](#producer-contract-termination-process)
    - [Kill Producer](#kill-producer)
- [Tracker](#tracker)
    - [Tracker Contract termination process](#tracker-contract-termination-process)

_Conventions_:
- `[*]`: Likely to change in the future
- `[Optional]`: Can be skipped if some conditions are met

# Prerequisite
- Install `Node.js@10.0.0`
- Install `@open-bucket/daemon` globally using `npm` (i.e `npm i -g @open-bucket/daemon`)

# Architecture
- Daemon has __CLI__ and __TCP__ interface (webserver). User interacts with Daemon through __CLI__, Client interacts with Daemon through __TCP__ interface.
- Daemon have __3 Primary Instances__: Producer, Consumer & Wallet
    - Producer: provides functionality for __producing storage space__ to Open Bucket Network (OBN)
    - Consumer: provides functionality for __consuming storage space__ from OBN
    - Wallet: provides functionality for managing user's Etherum accounts.
- Producer & Consumer has their own __CLI__ & __TCP__ interface. 
    - Producer expose __TCP__ through a webserver on port `8080` by default.
    - Consumer expose __TCP__ through a webserver on port `8081` by default.
#### Consumer & Producer
- User first must to create Consumer/Producer & provide their configs. Each Consumer/Producer has its own config file. And that config shouldn't be changed
- Each Consumer/Producer has ID is the Eth address assigned to it on its creation phase.
- The path & name to save each config file is configs/consumers/EthAddress1.json
- Consumer/Producer (if needed) will get user's Eth Wallet information from Wallet
- __availability__ of a file is calculated by Tracker. Tracker calculates the __availability__ based on the number of producer serving shards of the file inside OBN. For example: file A with size 5GB is splitted into 5 shards (1GB each). file A's availability = 5 only when every shard of file A is served by 5 consumer inside the network.

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


# Wallet
## Init Wallet
Check the wallet config `secretPath` to see if user has defined the path to their secret & the secret is valid (i.e. login to Eth Network).
- If all steps are succeeded, init done.
- If there's error, ask user to add wallet.
    - User has 2 options to add their Wallet to the Daemon:
    Notify user: `Daemon will NOT save your information & transfer your information. All your info will be erased when the Daemon is turned off.`
    - Create new wallet
        - [Optional] Ask user to input `entropy`. User can skip this.
        - Provide user the generated `seed` with the notice: `Please write it down on paper or in a password manager, you will need it to access your wallet. Do not let anyone see this seed or they can take your Ether. `
        - Promt user to input their password with the notice: `Please enter a password to encrypt your seed while the Daemon is functioning`
    - Restore their existing wallet. Options:
        - input the `seed` & `password`
        - config path to secret file
### Command: `obn wallet init`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -s, --seed <string>                    Specify Seed of your Eth wallet
    -p, --password <string>                Specify password of your Eth wallet
    ```
### REST API: `POST /v1/wallet/init`
- Request body:
    ```json
    {
        "seed": "your seed",
        "password": "your password"
    }
    ```
- Responses:
    - 200 - {"msg": "Wallet initialised"}
    - 400 - {"msg": "Invalid information"}


## Wallet config
1) Request user define the __path__ to their `seed` & `password`. The file should contain texts that has format:
```json
{
    "seed": "your seed",
    "password": "your password"
}
```
### Command: `obn wallet config`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -p, --secret-file-path <string>        Specify path to your secret file.
    ```
### REST API: `POST /v1/wallet/config`
- Request body:
    ```json
    {
        "path": "path/to/your/secret/file"
    }
    ```
- Responses:
    - 200 - {"msg": "Applied new Wallet config"}
    - 400 - {"msg": "Invalid information"}

# Consumer
## Create Consumer:
1) User provide configs for new consumer. Consumer validates inputs based on the following rules:
    - __address__: Must be a valid Eth address
    - __directory__: current user MUST have __read & write permission__ to this dir
    - values with empty value (`null`, `undefined`, `''`) will be ignored
2) Consumer calls to tracker to save new Consumer with the provided information. Tracker returns the ConsumerId.
3) Write information to file.
### Consumer config file format:
```json
{
    "address": "",
    "directory": "~/open-bucket/consumer-space"
}
```
### Command: `obn consumer create`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -a, --address <dirPath>                Specify user Eth Address
    -r, --directory <dirPath>              Specify Consumer space directory
    ```
### REST API: `POST /v1/consumer/create`
- Request body:
    ```json
    {
        "address": "",
        "directory": "~/obn/consumer-space"
    }
    ```
- Responses:
    - 200 - {"msg": "Consumer ${ID} is created"}
    - 400 - {"msg": "Invalid directory"}


## Start Consumer:
1) User provide/choose available Consumer config file
2) Read the choosen config file.
3) [Optional] Init Wallet
4) Start Consumer:
    - Start Consumer webserver
    - Start Webtorrent client
*Notes: we assume users don't modify the consumer-space*
5) Daemon send back to user the ConsumerID in the config file
### Command: `obn consumer start`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -c, --config <configPath>              Specify path to consumer config file
    ```
### REST API: `POST /v1/consumer/start`
- Request body:
    ```json
    {
        "config": "new/path/to/consumer/config"
    }
    ```
- Responses:
    - 200 - {"msg": "Consumer ${consumerID} has started"}
    - 200 - {"msg": "Consumer ${consumerID} had already started"}

## Stop Consumer:
1) User stop a Consumer by providing the ConsumerID
2) Stop Consumer
    - Destroy all current connection from OBN producers
    - Stop Webtorrent client
    - Stop Consumer webserver
### Command: `obn consumer stop`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -i, --id <consumerId>                  Specify consumerId to stop
    ```
### REST API: `POST /v1/consumer/stop`
- Responses:
    - 200 - {"msg": "Consumer stopped"}
    - 200 - {"msg": "Consumer had already stopped"}

## Upload a file to OBN:
1) Consumer validates inputs based on the following rules:
    - __filePath__: required, must be a string,
    - __fileTier__: Free | Basic | Premium
    - __duration__: The amount of time the file exists inside the network. The Ethereum account that user provided will be charged accordingly. This value will be `Infinite` with Basic plan.
2) Start Prepare Ethereum contract process
    - Tracker create the Ethereum contract
        - Tracker add __consumer Ethereum address__ to the contract as the 1st party
        - Tracker add __its own Ethereum address__ as the modifier
    - Tracker notify user that the contract has been created & waiting for user to fulfill their role (pay Eth to the contract)
3) [Optional] Start payment process
    - [Optional] Init Wallet
    - Ask user permission for charging Ether on their Wallet. 
        - If YES, charge them. 
        - If NO, user must send Eth to the contract address to fulfill his role in the contract
4) When 1st party fulfill their role, Tracker starts producer matching process
    - [*] When Tracker receives enough producer registrations for the file, it add all producer address to the contract
    - When the contract is valid (all parties fulfill their role), Tracker connects Consumer & Producers when every they connect to Tracker. (TODO) 
5) Start Consumer
    - If Consumer is already started, skip this step.
6) Start Prepare file process
    - Consumer encrypts the file with user Ethereum private key.
    - Consumer shard the encrypted file with user Ethereum private key.
7) Consumer start the upload the file process
    - Consumer connects to Tracker to let the Tracker knows that it is ready for seeding the shards. The Tracker returns the fileId & producer list
    - Consumer uploads the shards to producers (each shard has a torrent file)
8) [*] After the file has been completely uploaded into OBN, Consumer send message to Tracker to change the status of the file to __UPLOADED__ & save file shards info.
9) [*] Consumer saves shards infomation into its local storage.
10) Consumer delete the shards locally.
### Command: `obn consumer upload [options] </path/to/file>`
### REST API: `POST /v1/consumer/files/upload`
- Request Body:
    ```json
    {
        "path": "/path/to/file"
    }
    ```
- Responses:
    - 200
    ```json
    {
        "msg": "File filePath is uploading",
        "id": "FileId"
    }
    ```

## Remove a file from OBN:
1) Consumer validates inputs based on the following rules:
    - __fileId__: must be a string, this field will have higher priority over __path__
    - __path__: must be a valid path string, this field will be ignored if __fileId__ is specified
2) Tracker close the contract. The contract won't be able to receive Ether anymore.
3) Send delete message to all related producers.
4) Clean up all resources related to the file on Tracker.
5) Notify Consumer after Tracker received delete confirmation message from all producers.
### Notes:
- Consumer can withdraw their remaining Ether from the contract
- Producers can withdraw their earned Ether from the contract
### Command: `obn consumer rm </path/to/file/from/Producer/dir | fileId>`
### REST API: `POST /v1/consumer/files/remove`
- Request Body:
    ```json
    {
        "fileId": "fileId",
        "path": "/path/to/file/"
    }
    ```
- Responses:
    - 200 - {"msg": "File `fileId` removed"}
    - 400 - {"msg": "File `fileId` not exists"}

## Download data from OBN:
1) Consumer validates inputs based on the following rules:
    - __fileId__: must be a string, this field will have higher priority over __path__
    - __path__: must be a valid path string, this field will be ignored if __fileId__ is specified
    - __destination__: required, must be a valid path string to a dir. Current user must have __read & write permission__ to this dir
2) Consumer ask Tracker shards information of the specified file & receive the list of available producers
3) Consumer start file downloading process.
    - Download all the shards
    - Resemble the shard & decrypt the file
    - Send hash of the file to Tracker for confirmation
4) After the file download process is completed, Tracker transfer Ether from consumer to producers inside the Smart Contract
### Command: `obn consumer cp </path/to/file/ | fileId> </destination/dir>`
### REST API: `POST /v1/consumer/files/download`
- Request Body:
    ```json
    {
        "fileId": "fileId",
        "path": "/path/to/file/",
        "destination": "/path/to/local/dir"
    }
    ```
- Responses:
    - 200 - {"msg": "Downloading file `fileId | filePath` to `/destination/dir`"}
    - 400 - {"msg": "File `fileId | filePath` not exists"}

## Change file metadata on Tracker:
1) Consumer validates inputs based on the following rules:
    - __fileId__: required, must be a string
    - __path__: must be a valid path string
    - __availability__: must be a number
2) Tracker does the following actions:
    - If __path__ is specified: Tracker change file metadata
    - If __availability__ is specified: Tracker add/remove producers of the file based on the new availability
3) Tracker notify Daemon
### Command: `obn consumer edit [options] <fileId>`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -p, --path <dirPath>                   Specify new path of the file
    -a, --availability  <number>           Specify new availability number of the file
    ```
### REST API: `POST /v1/consumer/edit`
- Request body:
    ```json
    {
        "path": "new/path/to/file",
        "availability": 2,
    }
    ```
- Responses:
    - 200 - {"msg": "File's metadata has changed"}
    - 400 - {"msg": "Invalid directory"}

## Browse files on OBN:
### Command: `obn consumer ls`
### REST API: `GET /v1/consumer/files`
- Responses:
    - 200
    ```json
        [
            {
                "id": "fileUdid",
                "path": "path/to/file.exe",
                "size": "2 GB",
                "status": "UPLOADING",
                "actualAvailability": 1,
                "availability": 2,
                "hash": "file MD5 hash"
            },
            {
                "id": "fileUdid1",
                "path": "path/to/file.txt",
                "size": "5 KB",
                "status": "UPLOADED",
                "actualAvailability": 2,
                "availability": 2,
                "hash": "file MD5 hash"
            },
        ]
    ```

## Consumer contract termination process:
TODO

# Producer
## Create Producer
1) Producer validates the input based on the following rules:
    - __directory__: current user MUST have `read & write permission` to this dir
    - __size__: the value must have format: `${consumeSize}${unit}`
    - __account__: must be valid account identifier inside the self hosted wallet.
    - values with empty value (`null`, `undefined`, `''`) will be ignored
2) Producer calls to tracker to save new Producer with the provided information. Tracker returns the ProducerId.
3) Write information to file.
### [*] Producer config file format
```json
{
    "address": "",
    "directory": "~/open-bucket/producer-space",
    "size": "5GB"
}
```
### Command: `obn producer create`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -r, --directory <dirPath>              Specify Producer space directory
    -z, --size <number>                    Specify size of Producer space directory
    ```
### REST API: `POST /v1/producer/create`
- Request body:
    ```json
    {
        "directory": "~/obn/producer-space",
        "size": "2 GB"
    }
    ```
- Responses:
    - 200 - {"msg": "Producer ${ID} is created"}
    - 400 - {"msg": "Invalid directory"}


## Start Producer:
1) Read the config file
2) Init Wallet
3) Start Producer:
    - Starts the Producer.
    - Starts Webtorrent Client
### Notes:
- Daemon ignore this command if Producer has already started
- Data in __Producer space__ are the __shards__ of the encypted files
*Notes: We assume users don't modify the producer space*
### Command: `obn producer start`
### REST API: `POST /v1/producer/start`
- Responses:
    - 200 - {"msg": "Producer started"}
    - 200 - {"msg": "Producer had already started"}

## Producer contract termination process:
TODO

# Tracker
- Tracker keep track of all available Producer & Consumer.
- Tracker assign each producer a `nice` point. `nice` points is calculated based on producer's availability & network speed.
- Tracker rank file based on their tier, files with tier 3 (Premium) have the highest priority.


## Tracker Contract termination process:
TODO
