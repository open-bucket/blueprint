# Daemon

## Init
- User __must__ have `Node.js@10.0.0` installed and install `@open-bucket/daemon` globally using `npm`: `npm i -g @open-bucket/daemon`
- `Daemon` will act as a `cli` and a `server`. The `cli` for user to easier to use, the `server` which expose `REST APIs` or `GraphQL endpoints` which will be used by the Client.
- Daemon have __2 Primary Instances__: `Producer` & `Consumer`.
    - `Producer`: provides __upload__ functionality.
    - `Consumer`: provides __download__ functionality.
- User __MUST__ init the `Daemon` with: `open-bucket init`. The `init` command will walk user through the following steps:
    - Authentication: Register - Login
    - Define the Config file
    - Ask user to allow the `Daemon` to start `Producer` | `Consumer`
    - Ask user to allow the `Daemon` to start `Producer` | `Consumer` on startup
- If user re-init the Daemon, Daemon will have user confirm to override the current configs.
- User can init Producer | Consumer by using: `open-bucket [producer | consumer] init`. The same behavior will apply as when user re-init the Daemon.

## Producing Files to the Network
- User __MUST__ start the Producer first by using: `open-bucket producer start`. Daemon will ignore this command if the Producer has already started.
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
    - \[If the file's existing locally\]: Producer will copy it to the destination
    - \[If the file's __NOT__ existing \]: Producer will start downloading process.
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
- User can apply changes into Producer by using: `open-bucket producer apply /path/to/input/file.json`
    - Input file __MUST__ be on JSON format.
    - Producer only takes valid field of input file. Invalid fields will be ignored.
    - Input file that has valid field with invalid value will be rejected.
    - Input file __MUST__ have `type` field. Producer will base on the `type` field and some other fields (if required) to apply the changes.
    - User can apply changes on file's metadata by using an Input file has `"type": "File"` (with the required field to be `id` or `path`). A sample of the file input content in JSON:
        ```json
            {
                "type": "File",
                "id": "fileId",
                "path": "path/to/file.txt",
                "availability": 2
            }
        ```
    - User can apply changes on Producer configs by using an Input file has `"type": "ProducerConfig"`. A sample of the file input content in JSON:
        ```json
            {
                "type": "ProducerConfig",
                "directory": "~/open-bucket/produce",
                "defaultAvailability": 2,
                "startOnStartup": true
            }
        ```
        - If user change diretory path: All the files inside the old Producer directory will be moved to the new path.
## Consuming Data from the Network
- User __MUST__ start the Consumer first by using: `open-bucket consumer start`. Daemon will ignore this command if the Consumer has already started.
- Local Consumer directory will have its limit defined by user.
    - If the limit is reached, Consumer stop consuming data from the Network.
    - If user config the limit to be lower than current Consumer directory size, Consumer directory will be truncated to match with the desired litmit.
    - Invalid `limit` field will disable the Consumer. `limit` format should be: `${consumeSize} GB`
- Data in __local Consumer directory__ will be the __encypted shards__ of the files that have been produced into the Network.
- Shards inside consumer __MUST NOT__ be modified (including deleting), Consumer will ensure the consuming shards hasn't been modified. For each modified/deleted shard, Consumer will ask the Tracker to remove current user from the list of the shard consumer & remove the shard locally.
- User can apple changes to Consumer by using: `open-bucket consumer apply /path/to/input/file.json`
    - The same rules applied as Producer input file.
    - User can apply changes to Consumer configs by using an Input file has `"type": "ConsumerConfig"`. A sample of the file input content in JSON:
        ```json
            {
                "type": "ConsumerConfig",
                "directory": "~/open-bucket/consume",
                "limit": "2 GB",
                "startOnStartup": true
            }
        ```
        - If user change diretory path: All the shards inside the old Consumer directory will be moved to the new path.

# Client
- When user install the `Client` before installing the `Daemon`, Client will ask user permission to install the `Daemon`.
- The Client will have all the functionality of `Daemon` (including Producer, Consumer, configs, etc...) visualized to the user.

# Payment
TODO

# Implementation
## Core
- Open Bucket will have all its configs saved to the Config file. On startup, Daemon | Producer | Consumer will init their config state from Config file.
- Every REST API endpoints don't need to have Authentication information (i.e. Authorization header) since Daemon can get that info directly from the Config file.

### Watcher
- Producer & Consumer have their own Watcher (`PWatcher` for Producer, `CWatcher` for Consumer)
- The Watcher will be started or terminated by its Consumer | Producer.
- Watcher extends Node.js EventEmitter.
- Watcher takes responsible for detecting the changes in files that contain state and emit approriate event for each change (e.g. When user change `limit` field of `consumer` in the
Config file, the CWatcher will detect the changes and emit `setLimit` event with the new limit)
- Use [Chokidar](https://github.com/vunguyenhung/chokidar) to detect file changes (e.g: webpack, nodemon,... )
     - Chokidar usage - Nodemon: https://github.com/vunguyenhung/nodemon/blob/master/lib/monitor/watch.js

### Init command:
- __Authentication__: User must provide username/password that they've registered on the Tracker.
- __Define the Config file__: Ask user whether they use `Producer` or `Consumer` functionality. User will need to define the config for the one they've choose __YES__ (Default values will be used in place of missing configs, the fields start with `_` (e.g: `_status`) are READONLY fields)
    - The Config file:
        ```json
        {
            "authentication": {
                "token": "User Authentication Token",
                "privateKey": "User private key"
            },
            "producer": {
                "directory": "~/open-bucket/produce",
                "_status": "STARTED",
                "defaultAvailability": 2,
                "startOnStartup": true
            },
            "consumer": {
                "directory": "~/open-bucket/consume",
                "limit": "2 GB",
                "_status": "STARTED",
                "startOnStartup": true
            },
            "_updatedAt": "1525147564"
        }
        ```
- __Ask user to allow the `Daemon` to start `Producer` | `Consumer`__: Perform `open-bucket producer start`
- __Ask user to allow the `Daemon` to start `Producer` | `Consumer` on startup__: Perform `open-bucket producer start` on startup (need to research this)
#### Command: `open-bucket init`
#### REST APIs:
1) `POST /v1/auth/login`
    - Request body:
        ```json
        {
            "username": "Username",
            "password": "Password"
        }
        ```
    - Responses:
        - 200 - {"token": "JWT Token"}
        - 400 - {"msg": "Invalid username or password"}
2) `POST /v1/auth/configs`
    - Request body:
        ```json
        {
            "token": "User Authentication Token",
            "privateKey": "User private key"
        }
        ```
    - Responses: 200 - {"msg": "Authentication configs applied"}
3) `POST /v1/producer/configs`
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
        - 400 - {"msg": "Invalid \`directory\`"}
4) `POST /v1/consumer/configs`
    - Request body:
        ```json
        {
            "directory": "~/open-bucket/consume",
            "limit": "2 GB",
            "startOnStartup": true
        }
        ```
    - Responses:
        - 200 - {"msg": "Consumer configs applied"}
        - 400 - {"msg": "Invalid \`directory\`"}

## Producer
- Tracker __MUST__ have a mechanism to notify Producer when a file is completely uploaded into the Network. (TODO)
### Start Producer:
1) Daemon starts the Producer.
2) Producer starts the PWatcher.
3) Daemon change the Producer status in config file to `STARTED`.
#### Command: `open-bucket producer start`
#### REST API: `POST /v1/producer/start`
- Responses:
    - 200 - {"msg": "Producer started"}
    - 200 - {"msg": "Producer has already started"}
    - 401 - {"msg": "Invalid token"}

### Stop Producer:
1) Producer stops the PWatcher.
2) Daemon stops the Producer.
3) Daemon changes Producer status in config file to `STOPPED`.
#### Command: `open-bucket producer stop`
#### REST API: `POST /v1/producer/stop`
- Responses:
    - 200 - {"msg": "Producer stopped"}
    - 200 - {"msg": "Producer has already stopped"}

### Produce a file to the Network:
1) Copy the file to local Producer directory.
2) PWatcher will detect the changes and upload the file.
3) Producer will communicate with the Tracker to upload the file.
4) The Tracker will __notify__ Producer when the file has completely uploaded into the network.
5) When Producer got notification from Tracker, it will delete the file if `deleteLocalOnComplete` option is TRUE.
#### Command: `open-bucket producer add /path/to/file`
#### REST API: `POST /v1/producer/files`
- Request Body:
```json
{
    "path": "/path/to/file",
    "deleteLocalOnComplete": true
}
```
- Responses:
    - 200
    ```json
    {
        "msg": "File `filePath` added",
        "id": "FileId"
    }
    ```
### Delete a file from the Network:
2) Delete the file locally (if exists)
3) Producer will ask the Tracker to remove the file.
#### Command: `open-bucket producer rm [/path/to/file/from/Producer/dir | fileId]`
#### REST API: `DELETE /v1/producer/files/:fileId`
- Responses:
    - 200 - {"msg": "File `fileId` removed"}
    - 400 - {"msg": "File `fileId` not exists"}

### Download a file from the Network:
1) TODO
#### Command: `open-bucket producer cp [/path/to/file/from/Producer/dir | fileId] /destination/dir`
#### REST API: `GET /v1/producer/files/:fileId`
- Request Header:
    - destination: "/destination/dir"
- Responses:
    - 200 - {"msg": "File `fileId | filePath` removed"}
    - 400 - {"msg": "File `fileId | filePath` not exists"}

## Consumer
TODO