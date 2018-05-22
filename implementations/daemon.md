_Conventions_:
- `[*]`: Likely to change in the future
- `[Optional]`: Can be skipped if some conditions are met

# Daemon
## Init:
1) [Optional] Add default Ethereum account to Wallet: Uses Wallet `add account` functionality
    - If the default Ethereum account is already exists, skip this step
2) Define config file:
    - Request user define Consumer config: Uses Consumer `config` functionality.
    - Request user define Producer config: Uses Producer `config` functionality.
3) Ask user to allow Daemon to start Producer & Consumer: Perform `obn producer start` & `obn consumer start`
### Notes:
- If user re-init `Daemon`, all current configs will be overridden
### Command: `obn init`

# Wallet
TODO

# Consumer
## Start Consumer:
1) Read the config file.
2) [Optional] Request user to add default Ethereum account to Wallet.
    - If the default Ethereum is already exists, skip this step
3) Start Consumer:
    - Daemon starts the Consumer.
    - Consumer starts PWatcher.
    - Consumer get the file list by sending request to Tracker. The file list then will be compared (by MD5 hash) with the files inside __Consumer space__:
    - If there're __new files__: Add the new files to the in-memory file list & change their status to __PENDING__.
    - If there're __missing files__: Check the current status of the file on Tracker:
        - If it is __UPLOADED__: do nothing. 
        - If it is __UPLOADING__: TODO
### Command: `obn consumer start`
<!-- TODO: add options to override config values -->
### REST API: `POST /v1/consumer/start`
- Responses:
    - 200 - {"msg": "Consumer has started"}
    - 200 - {"msg": "Consumer had already started"}

## Stop Consumer:
1) Destroy all current connection from OBN producers
2) Stop Consumer
    - Consumer stops the CWatcher.
    - Daemon stops the Consumer.
### Command: `obn consumer stop`
### REST API: `POST /v1/consumer/stop`
- Responses:
    - 200 - {"msg": "Consumer stopped"}
    - 200 - {"msg": "Consumer had already stopped"}

## Consume storage from OBN:
1) [Optional] Start Consumer.
    - If the Consumer is already started, skip this step.
2) [Optional] Copy the file to __Consumer space__.
    - If it is existing in there (check by MD5 hash), skip this step
    - CWatcher detects the changes and notify the Consumer.
    - Consumer add new file to the in-memory file list and mark its status as __PENDING__.
3) Ask the user to provide information to create an `Ethereum Smart Contract` for the file:
    - User's Ethereum account (choose from wallet)
    - [Optional] File tier: Free | Basic | Premium
    - [Optional] Desired availability
    - [Optional] The amount of time the file exists inside the network. The Ethereum account that user provided will be charged accordingly.
4) Tracker create the Smart Contract & charge user.
    - Tracker create the Ethereum contract with __consumer Ethereum address__
    - Tracker pay Ether to the contract on behave of user
    - Tracker notify user that the contract has been created & waiting for producers to sign
    - [*] When Tracker receives enough producer registrations for the file, it add all producer address to the contract & return user with the list of producers
5) Consumer start the upload the file process
    - Consumer encrypts & shard the file with user Ethereum private key.
    - Consumer uploads the shards to producers (each shard has a torrent file)
6) After the file has been completely uploaded into OBN, Consumer send message to Tracker to change the status of the file to __UPLOADED__ & save file shards info.
7) [*] Consumer saves shards infomation into its local storage.
8) Consumer delete the shards locally.
### Notes: When user delete a shard on __Consumer space__:
- If the shard is __NOT completely uploaded__ into the Network: TODO
- A shard is considered __completely uploaded into the Network__ when its __actual availability__ is matched with is __desired availability__ defined by user
- __availability__ of a file is calculated by Tracker. Tracker calculates the __availability__ based on the number of producer serving shards of the file inside OBN. For example: file A with size 5GB is splitted into 5 shards (1GB each). file A's availability = 5 only when every shard of file A is served by 5 consumer inside the network.
- User can delete their file on the Network by using `obn consumer rm` command
- User can download their file on the Network by using `obn consumer cp` command
- Files produced into the Network have their metadata saved by the `Tracker`. User can browse their files on the network by using `obn consumer ls` command
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

## Change Consumer config:
1) Consumer validates inputs based on the following rules:
    - __directory__: current user MUST have _
    _read & write permission__ to this dir
    - __defaultAvailability__: must be a number
    - __startOnStartup__: must be `true` or `false`
2) After all fields has passed their validation rules, Consumer does the following actions:
    - If __directory__ is changed: 
        - Move all current shards in the old directory to the new one.
        - Change the Config file
    - If __defaultAvailability__ is changed: Change the Config file
    - If __startOnStartup__ is changed: 
        - Make user OS to run `obn consumer start` on startup
        - Change the Config file  

If any of those steps failed, all the changes will be rolled back.
### Command: `obn consumer config`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -r, --directory <dirPath>              Specify Consumer space
    -a, --default-availability <number>    Specify default availability used when upload file
    -s, --start-on-startup                 Specify to start Consumer on startup
    ```
### REST API: `POST /v1/consumer/configs`
- Request body:
    ```json
    {
        "directory": "~/obn/consumer-space",
        "defaultAvailability": 2,
        "startOnStartup": true
    }
    ```
- Responses:
    - 200 - {"msg": "Consumer configs applied"}
    - 400 - {"msg": "Invalid directory"}

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
## Start Producer:
1) Read the config file
2) [Optional] Request user to add default Ethereum account to Wallet
    - If the default Ethereum is already exists, skip this step
3) [Optional] Request user to choose producer Ethereum account from Wallet
    - If user already add it in Producer config, skip this step
4) Start Producer:
    - Daemon starts the Producer.
    - Producer starts the PWatcher.
### Notes:
- Daemon ignore this command if Producer has already started
- Data in __Producer space__ are the __shards__ of the encypted files
- Shards inside Producer space __MUST NOT BE MODIFIED/DELETED__. If any shard is modified/deleted, the following actions will be triggered:
    - Producer triggers the __contract termination process__ related of that shard.
    - The Tracker adds new producers to match with the file's desired availability defined by the Consumer.
    - User can withdraw their earned Ether from the contract.
### Command: `obn producer start`
### REST API: `POST /v1/producer/start`
- Responses:
    - 200 - {"msg": "Producer started"}
    - 200 - {"msg": "Producer had already started"}

## Change Producer config:
1) Producer validates the input based on the following rules:
    - __directory__: current user MUST have `read & write permission` to this dir
    - __limit__: the value must have format: `${consumeSize} GB`
    - __account__: must be valid account identifier inside the self hosted wallet.
    - __startOnStartup__: must be `true` or `false`
2) Next, Producer does the following actions:
    - If __directory__ is changed:
        - Move all current files from the old directory to the new one.
        - Change the Config file
    - If __limit__ is changed: 
        - If user config the limit to be lower than __Producer space__ size, __Producer space__ directory will be truncated to match with the desired litmit. All contract related to the truncated shards will be terminated.
        - If the limit is reached, Producer stop consuming data from the Network
    - If __account__ is changed: TODO
    - If __startOnStartup__ is changed: 
        - Make user OS to run `obn producer start` on startup
        - Change the Config file. 

If any of those steps failed, all the changes will be rolled back.
### Command: `obn producer config`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -r, --directory <dirPath>              Specify Producer space directory
    -l, --limit <number>                   Specify limit
    -s, --start-on-startup                 Specify to start Producer on startup
    ```
### REST API: `POST /v1/producer/configs`
- Request body:
    ```json
    {
        "directory": "~/obn/producer-space",
        "limit": "2 GB",
        "startOnStartup": true
    }
    ```
- Responses:
    - 200 - {"msg": "Producer configs applied"}
    - 400 - {"msg": "Invalid directory"}

## Producer contract termination process:
TODO


# Tracker
- Tracker keep track of all available Producer & Consumer.
- Tracker assign each producer a `nice` point. `nice` points is calculated based on producer's availability & network speed.
- Tracker rank file based on their tier, files with tier 3 (Premium) have the highest priority.

## Contract termination process:
TODO