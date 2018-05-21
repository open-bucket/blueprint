_Conventions_:
- `[*]`: Likely to change in the future
- `[Optional]`: Can be skipped if some conditions are met

# Daemon
## Init:
1) [Optional] Add default Ethereum account to Wallet: Uses Wallet `add account` functionality
    - If the default Ethereum account is already exists, skip this step
2) Define config file:
    - Request user define Producer config: Uses Producer `config` functionality.
    - Request user define Consumer config: Uses Consumer `config` functionality.
3) Ask user to allow Daemon to start Producer & Consumer: Perform `open-bucket producer start` & `open-bucket consumer start`
### Notes:
- If user re-init `Daemon`, all current configs will be overridden
### Command: `open-bucket init`

# Wallet
TODO

# Producer
## Start Producer:
1) Read the config file.
2) [Optional] Request user to add default Ethereum account to Wallet.
    - If the default Ethereum is already exists, skip this step
3) Start Producer:
    - Daemon starts the Producer.
    - Producer starts PWatcher.
    - Producer get the file list by sending request to Tracker. The file list then will be compared (by MD5 hash) with the files inside __local Producer directory__:
    - If there're __new files__: Add the new files to the in-memory file list & change their status to __PENDING__.
    - If there're __missing files__: Check the current status of the file on Tracker:
        - If it is __UPLOADED__: do nothing. 
        - If it is __UPLOADING__: TODO
### Command: `open-bucket producer start`
<!-- TODO: add options to override config values -->
### REST API: `POST /v1/producer/start`
- Responses:
    - 200 - {"msg": "Producer started"}
    - 200 - {"msg": "Producer had already started"}

## Stop Producer:
1) Destroy all current connection from Open Bucket consumers
2) Stop Producer
    - Producer stops the PWatcher.
    - Daemon stops the Producer.
### Command: `open-bucket producer stop`
### REST API: `POST /v1/producer/stop`
- Responses:
    - 200 - {"msg": "Producer stopped"}
    - 200 - {"msg": "Producer had already stopped"}

## Produce data to the Network:
1) [Optional] Start Producer.
    - If the Producer is already started, skip this step.
2) [Optional] Copy the file to __local Producer directory__.
    - If it is existing in there (check by MD5 hash), skip this step
    - PWatcher detects the changes and notify the Producer.
    - Producer add the new file to the in-memory file list and mark its status as __PENDING__.
3) Ask the user to provide information to create an `Ethereum Smart Contract` for the file:
    - User's Ethereum account (choose from wallet)
    - [Optional] File tier: Free | Basic | Premium
    - [Optional] Desired availability
    - [Optional] The amount of time the file exists inside the network. The Ethereum account that user provided will be charged accordingly.
4) Producer create the Smart Contract & Charge user.
    - Tracker create the Ethereum contract with __producer Ethereum address__
    - Tracker pay Ether to the contract on behave of user
    - Tracker notify user that the contract has been created & waiting for consumers to sign
    - [*] When Tracker receives enough consumer registrations for the file, it add all consumer address to the contract & return user with the list of consumers
5) Producer start the upload the file process
    - Producer attempts to encrypt & shard the file with user Ethereum private key.
    - Producer of upload the shards to consumers (each shard has a torrent file)
6) After the file has been completely uploaded into the network,  Producer send message to Tracker to change the status of the file to UPLOADED & save file shards info.
7) [Optional] Producer saves shards infomation into its local storage.
8) Producer delete the file locally.
### Notes: When user delete their files on __local Producer directory__:
- If the file is __completely uploaded__ into the Network: The files won't be deleted on the Network, it just got deleted locally
- If the file is __NOT completely uploaded__ into the Network: TODO
- A file is considered __completely uploaded into the Network__ when its __actual availability__ is matched with is __desired availability__ defined by user
- The __availability__ is calculated by Tracker. Tracker calculates the __availability__ based on the number of consumer consuming shards of the file inside the network. For example: file A with size 5GB is splitted into 5 shards (1GB each). file A's availability = 5 only when every shard of file A is served by 5 consumer inside the network.
- User can delete their file on the Network by using `open-bucket producer rm` command
- User can download their file on the Network by using `open-bucket producer cp` command
- Files produced into the Network will have their metadata saved by the `Tracker`. User can browse their files on the network by using `open-bucket producer ls` command
### Command: `open-bucket producer upload [options] </path/to/file>`
### REST API: `POST /v1/producer/files/upload`
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

## Remove a file from the Network:
1) Producer attempts to validate inputs from user based on the following rules:
    - __fileId__: must be a string, this field will have higher priority over __path__
    - __path__: must be a valid path string, this field will be ignored if __fileId__ is specified
2) Delete the local file.
    - [If the local file is existing]: `PWatcher` will detect the changes & notify `Producer` to trigger the removing file process
    - [If the local file is __NOT__ existing]: trigger the removing file process of the `Producer` manually
3) Tracker close the contract. The contract then will not be able to receive Ether anymore.
4) Send delete message to all related consumer
5) Clean up all resources related to the file on Tracker
6) Notify Producer after Tracker received delete confirmation message from all consumers
### Notes:
- The Producer can withdraw their remaining Ether from the contract
- The Consumers can withdraw their earned Ether from the contract
### Command: `open-bucket producer rm </path/to/file/from/Producer/dir | fileId>`
### REST API: `POST /v1/producer/files/remove`
- Request Body:
    ```json
    {
        "fileId": "fileId",
        "remotePath": "/path/to/file/from/remote/Producer/dir"
    }
    ```
- Responses:
    - 200 - {"msg": "File `fileId` removed"}
    - 400 - {"msg": "File `fileId` not exists"}

## Download data from the Network:
1) Producer validates inputs based on the following rules:
    - __fileId__: must be a string, this field will have higher priority over __path__
    - __path__: must be a valid path string, this field will be ignored if __fileId__ is specified
    - __destination__: required, must be a valid path string to a dir. Current user must have __read & write permission__ to this dir
2) Producer ask Tracker shards information of the specified file & receive the list of available consumers
3) Producer start file downloading process.
    - Download all the shards
    - Resemble the shard & decrypt the file
    - Send hash of the file to Tracker for confirmation
4) After the file download process is completed, Tracker transfer Ether from producer to consumers inside the Smart Contract
### Command: `open-bucket producer cp </path/to/file/ | fileId> </destination/dir>`
### REST API: `POST /v1/producer/files/download`
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

## Change Producer config:
1) Producer validates inputs based on the following rules:
    - __directory__: current user MUST have _
    _read & write permission__ to this dir
    - __defaultAvailability__: must be a number
    - __startOnStartup__: must be `true` or `false`
2) After all fields has passed their validation rules, Producer does the following actions:
    - If __directory__ is changed: 
        - Move all current files in the old directory to the new one.
        - Change the Config file
    - If __defaultAvailability__ is changed: Change the Config file
    - If __startOnStartup__ is changed: 
        - Make user OS to run `open-bucket producer start` on startup
        - Change the Config file  

If any of those steps failed, all the changes will be rolled back.
### Command: `open-bucket producer config`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -r, --directory <dirPath>              Specify local Producer directory
    -a, --default-availability <number>    Specify default availability used when upload file
    -s, --start-on-startup                 Specify to start Producer on startup
    ```
### REST API: `POST /v1/producer/configs`
- Request body:
    ```json
    {
        "directory": "~/open-bucket/produce",
        "defaultAvailability": 2,
        "startOnStartup": true
    }
    ```
- Responses:
    - 200 - {"msg": "Producer configs applied"}
    - 400 - {"msg": "Invalid directory"}

## Change file metadata on Tracker:
1) Producer validates the input based on the following rules:
    - __fileId__: required, must be a string
    - __path__: must be a valid path string
    - __availability__: must be a number
2) Next, Tracker does the following actions:
    - If __path__ is specified: Tracker change file metadata
    - If __availability__ is specified: Tracker add/remove consumers of the file based on the new availability
3) Tracker notify Daemon
### Command: `open-bucket producer edit [options] <fileId>`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -r, --remotePath <dirPath>             Specify new remote path to file
    -a, --availability  <number>           Specify new availability number of the file
    ```
### REST API: `POST /v1/producer/edit`
- Request body:
    ```json
    {
        "directory": "~/open-bucket/produce",
        "defaultAvailability": 2,
        "startOnStartup": true
    }
    ```
- Responses:
    - 200 - {"msg": "File's metadata has changed"}
    - 400 - {"msg": "Invalid directory"}

## Browse files on the network:
### Command: `open-bucket producer ls`
- Options:
    ```
    -l, --local                 Shows files metadata on local Producer directory
    ```
### REST API: `GET /v1/producer/files`
- Responses:
    - 200
    ```json
        [
            {
                "id": "fileUdid",
                "path": "path/to/file/from/remote/Producer/dir.exe",
                "size": "2 GB",
                "status": "UPLOADING",
                "actualAvailability": 1,
                "availability": 2,
                "hash": "file MD5 hash"
            },
            {
                "id": "fileUdid1",
                "path": "path/to/file/from/remote/Producer/dir1.txt",
                "size": "5 KB",
                "status": "AVAILABLE",
                "actualAvailability": 2,
                "availability": 2,
                "hash": "file MD5 hash"
            },
        ]
    ```


# Consumer
## Contract termination process:
TODO

## Start Consumer:
1) Read the config file
2) [Optional] Request user to add default Ethereum account to Wallet
    - If the default Ethereum is already exists, skip this step
3) [Optional] Request user to choose consumer Ethereum account from Wallet
    - If user already add it in Consumer config, skip this step
4) Start Consumer:
    - Daemon starts the Consumer.
    - Consumer starts the CWatcher.
### Notes:
- Daemon ignore this command if Consumer has already started
- Data in __local Consumer directory__ are the __shards__ of the encypted files
- Shards inside __local Consumer directory__ __MUST NOT__ be modified. If any shard is modified/deleted, the following actions will be triggered:
    - Consumer triggers the __contract termination process__ related of that shard.
    - The `Tracker` will add a new consumer to match with the file's desired availability defined by the Producer.
    - User can withdraw their earned Ether from the contract.
### Command: `open-bucket consumer start`
### REST API: `POST /v1/consumer/start`
- Responses:
    - 200 - {"msg": "Consumer started"}
    - 200 - {"msg": "Consumer had already started"}

## Change Consumer config:
1) Consumer validates the input based on the following rules:
    - __directory__: current user MUST have `read & write permission` to this dir
    - __limit__: the value must have format: `${consumeSize} GB`
    - __account__: must be valid account identifier inside the self hosted wallet.
    - __startOnStartup__: must be `true` or `false`
2) Next, `Consumer` does the following actions:
    - If __directory__ is changed:
        - Move all current files in the old directory to the new one.
        - Change the Config file
    - If __limit__ is changed: 
        - If user config the limit to be lower than __local Consumer directory__ size, __local Consumer directory__ directory will be truncated to match with the desired litmit. All contract related to the truncated shards will be terminated.
        - If the limit is reached, Consumer stop consuming data from the Network
    - If __account__ is changed: TODO
    - If __startOnStartup__ is changed: 
        - Make user OS to run `open-bucket consumer start` on startup
        - Change the Config file  

If any of those steps failed, all the changes will be rolled back.
### Command: `open-bucket consumer config`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -r, --directory <dirPath>              Specify local Producer directory
    -l, --limit <number>                   Specify limit
    -s, --start-on-startup                 Specify to start Producer on startup
    ```
### REST API: `POST /v1/consumer/configs`
- Request body:
    ```json
    {
        "directory": "~/open-bucket/produce",
        "limit": "2 GB",
        "startOnStartup": true
    }
    ```
- Responses:
    - 200 - {"msg": "Consumer configs applied"}
    - 400 - {"msg": "Invalid directory"}


# Tracker
- Tracker keep track of all available Producer & Consumer.
- Tracker assign each consumer a `nice` point. `nice` points is calculated based on consumer's availability & network speed.
- Tracker rank file based on their tier, files with tier 3 (Premium) have the highest priority.