# Daemon

## Init
- User __must__ have `Node.js@10.0.0` installed and install `@open-bucket/daemon` globally using `npm`: `npm i -g @open-bucket/daemon`
- `Daemon` will act as a `cli` and a `server`. The `cli` for user to easier to use, the `server` which expose `REST APIs` or `GraphQL endpoints` which will be used by the Client.
- Daemon have __2 Primary Instances__: `Producer` & `Consumer`.
    - `Producer`: provides __upload__ functionality.
    - `Consumer`: provides __download__ functionality.
- User __must__ init the `Daemon` with: `open-bucket init`. The `init` command will walk user through the following steps:
    - Authentication: Register - Login
    - Define the Config file
    - Ask user to allow the `Daemon` to start `Producer` | `Consumer`
    - Ask user to allow the `Daemon` to start `Producer` | `Consumer` on startup
- If user re-init the Daemon, Daemon will have user confirm to override the current configs.
- User can init Producer | Consumer by using: `open-bucket [producer | consumer] init`. The same behavior will apply as when user re-init the Daemon.
- User can use `open-bucket [producer | consumer] start` command to start `Producer` | `Consumer` manually. Or they can config the `Daemon` to start them on startup.
- User can change the configs by using: `open-bucket [producer | consumer] configs`. The Config file's editable fields will be opened in user default editor.
    - When the config file is saved, the saved configs will be applied.
    - If user change diretory path: TODO
    - If user config the limit to be lower than current Consumer directory size, Consumer directory will be truncated to match with the desired litmit.
    - Example of config's editable fields (READONLY fields will be hidden):
    ```json
        {
            "authentication": {
                "token": "User Authentication Token",
                "privateKey": "User private key"
            },
            "consumer": {
                "directory": "~/open-bucket/consume",
                "limit": "2 GB",
                "startOnStartup": true
            },
            "producer": {
                "directory": "~/open-bucket/produce",
                "defaultAvailability": 2,
                "startOnStartup": true
            },
        }
    ```

## Producing Files to the Network
- User can produce files to the Network with 2 ways:
    - User can __copy__ files directly into the __local Producer directory__. Based on the Producer status, the following cases will be considered:
        - \[If the Producer status is `STARTED` \]: the Producer will detect the changes and automatically upload the unsynced files into the Network with the default availability is __2__.
        - \[If the Producer status is `STOPPED` \]: Nothing will happen at the time. But when user start the Producer, Producer will ask if user wanted to produce the unsynced file to the Network
            - \[If YES\]: the Producer will produce the files to the Network.
            - \[If NO\]: the Producer will delete the files on Producer directory
    - By using: `open-bucket producer add /path/to/file`. This command will copy the file to local Producer directory, upload the file to the Network, and delete the file on local Producer directory when the file is __completely uploaded__.
- When user delete their files on __local Producer directory__:
    - \[If the file is __completely uploaded__ into the Network\]: The files won't be deleted on the Network, it just got deleted locally.
    - \[If the file is __NOT completely uploaded__ into the Network\]: the file will be removed on the Network.
- A file is considered __completely uploaded into the Network__ when its __actual availability__ is matched with is __desired availability__ defined by user.
- The `availability` is calculated by the Tracker & based on the count of consumers in the Network that consume the file (this may change in the future).
- User can delete their file on the Network by using `open-bucket producer rm [ /path/to/file/from/Producer/dir | fileId ]`, this command will also delete the file on __local Producer directory__ (if exists).
- User can download their file on the Network by using: `open-bucket producer cp [/path/to/file/from/remote/Producer/dir | fileId] /destination/dir`
- Data produced into the Network will have its metadata saved by the Tracker.
    - User can browse their data on the network by using: `open-bucket producer ls`, add option `--local` to browse data on their local machine.
        - Result format will be in JSON:
            ```json
                {
                    "id": "fileUdid",
                    "path": "path/to/file/from/Producer/dir.txt",
                    "updatedAt": "Apr 30 15:35",
                    "size": "2 KB",
                    "status": "UPLOADING",
                    "availability": 2
                }
            ```
- User can edit file's metadata on the Network by using: `open-bucket producer edit [ /path/to/file/from/Producer/dir | fileId ]`. File's editable metadata will be opened on user's default editor & will have JSON format. The changes in file's metadata will be applied when user save. Example of file's editable metadata:
    ```json
        {
            "path": "path/to/file.txt",
            "availability": 2
        }
    ```

## Consuming Data from the Network
- Local Consumer directory will have its limit defined by user.
    - If the limit is reached, Consumer stop consuming data from the Network.
    - If user config the limit to be lower than current Consumer directory size, Consumer directory will be truncated to match with the desired litmit.
    - Invalid `limit` field will disable the Consumer. `limit` format should be: `${consumeSize} GB`
- Data in __local Consumer directory__ will be the __encypted shards__ of the files that have been produced into the Network.
- Shards inside consumer __MUST NOT__ be modified (including deleting), CWatcher will ensure the consuming shards hasn't been modified. For each modified/deleted shard, Consumer will ask the Tracker to remove current user from the list of the shard consumer & remove the shard locally.

# Client
- When user install the `Client` before installing the `Daemon`, Client will ask user permission to install the `Daemon`.
- The Client will have all the functionality of `Daemon` (including Producer, Consumer, configs, etc...) visualized to the user.

# Payment
TODO

# Implementation
## Core
- Open Bucket will have all its configs saved to the Config file. On startup, Daemon | Producer | Consumer will init their config state from Config file.

### Watcher
- Producer & Consumer have their own Watcher (`PWatcher` for Producer, `CWatcher` for Consumer)
- The Watcher will be started or terminated by its Consumer | Producer.
- Watcher extends Node.js EventEmitter.
- Watcher takes responsible for detecting the changes in files that contain state and emit approriate event for each change (e.g. When user change `limit` field of `consumer` in the
Config file, the CWatcher will detect the changes and emit `setLimit` event with the new limit)
- Use [Chokidar](https://github.com/vunguyenhung/chokidar) to detect file changes (e.g: webpack, nodemon,... )
     - Chokidar usage - Nodemon: https://github.com/vunguyenhung/nodemon/blob/master/lib/monitor/watch.js

### Init command:
> Command: `open-bucket init`
> REST API: `POST /v1/init` - Client must provide all necessary information: Authentication &Configs
- __Authentication__: User must provide username/password that they've registered on the Tracker.
- __Define the Config file__: Ask user whether they use `Producer` or `Consumer` functionality. User will need to define the config for the one they've choose __YES__ (Default values will be used in place of missing configs, the fields start with `_` (e.g: `_status`) are READONLY fields)
    - The Config file:
        ```json
        {
            "authentication": {
                "token": "User Authentication Token",
                "privateKey": "User private key"
            },
            "consumer": {
                "directory": "~/open-bucket/consume",
                "limit": "2 GB",
                "_status": "STARTED",
                "startOnStartup": true
            },
            "producer": {
                "directory": "~/open-bucket/produce",
                "_status": "STARTED",
                "defaultAvailability": 2,
                "startOnStartup": true
            },
            "_updatedAt": "1525147564"
        }
        ```
- __Ask user to allow the `Daemon` to start `Producer` | `Consumer`__: Perform `open-bucket producer start`
- __Ask user to allow the `Daemon` to start `Producer` | `Consumer` on startup__: Perform `open-bucket producer start` on startup (need to research this)

## Producer
### Start command:
> Command: `open-bucket producer start`
> REST API: `POST /v1/producer/start` - Client must be authenticated
- Start the Producer. After the Producer started, it will start the PWatcher.
- Finally, the Producer will edit its status in config file to `STARTED`.

### Stop command:
> Command: `open-bucket producer stop`
> REST API: `POST /v1/producer/stop` - Client must be authenticated
- Stop the PWatcher. After the PWatcher is stopped, the Producer will be terminated.
- Finally, the Producer will edit its status in config file to `STOPPED`.

## Consumer
TODO