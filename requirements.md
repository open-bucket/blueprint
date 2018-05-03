# Daemon

## Init
- User __must__ have `Node.js@10.0.0` installed and install `@open-bucket/daemon` globally using `npm`: `npm i -g @open-bucket/daemon`
- `Daemon` has __CLI__ and __TCP__ interface. User interacts with Daemon through __CLI__, Client interacts with Daemon through __TCP__ interface
- Daemon have __2 Primary Instances__: `Producer` & `Consumer`
    - `Producer`: provides functionality for producing files into the Network
    - `Consumer`: provides functionality for consuming data from the Network
- User __MUST__ init the `Daemon` with: `open-bucket init`. The `init` command will walk user through the following steps:
    - Authentication (Register/Login)
    - Define the Config file
        - Ask if user want to config `Producer`
        - Ask if user want to config `Consumer`
        - Ask user to allow the `Daemon` to start `Producer` | `Consumer` on startup
    - Ask user to allow the `Daemon` to start `Producer` | `Consumer`
- User can init `Producer` | `Consumer` by using: `open-bucket [producer | consumer] init`
- If user re-init `Daemon`, `Producer` or `Consumer`. All current configs will be overridden

## Producing Files to the Network
- User __MUST__ start the `Producer` first by using: `open-bucket producer start`. `Daemon` will ignore this command if the `Producer` has already started
- User can produce files to the Network with 2 ways:
    - __COPY__ files directly into the __local Producer directory__. Based on the `Producer` status, the following cases will be considered:
        - \[If the `Producer` status is __STARTED__\]: the `Producer` will detect file changes and automatically upload the unsynced files into the Network with __default availability is 2__
        - \[If the `Producer` status is __STOPPED__\]: Nothing will happen. But when user start the `Producer`, `Producer` will ask if user wanted to produce the unsynced file to the Network
            - \[If __YES__\]: the `Producer` will produce the files to the Network
            - \[If __NO__\]: the `Producer` will delete the files on __local Producer directory__
    - Use `open-bucket producer add /path/to/file`. This command will:
        1) Copy the file to __local Producer directory__
        2) Upload the file to the Network
        3) Delete the file on __local Producer directory__ after the file is __completely uploaded__
- When user delete their files on __local Producer directory__:
    - \[If the file is __completely uploaded__ into the Network\]: The files won't be deleted on the Network, it just got deleted locally
    - \[If the file is __NOT completely uploaded__ into the Network\]: the file will be removed on the Network
- A file is considered __completely uploaded into the Network__ when its __actual availability__ is matched with is __desired availability__ defined by user
- The __availability__ is calculated by the `Tracker`. `Tracker` will calculate the __availability__ based on the number of consumer consuming shards of the file.
- User can delete their file on the Network by using `open-bucket producer rm [/path/to/file/from/Producer/dir | fileId]`, this command will also delete the file on __local Producer directory__ (if exists).
- User can download their file on the Network by using: `open-bucket producer cp [/path/to/file/from/remote/Producer/dir | fileId] /destination/dir`
    - \[If the file's existing locally\]: `Producer` will copy it to the destination
    - \[If the file's __NOT__ existing locally \]: `Producer` will start downloading
- Files produced into the Network will have their metadata saved by the `Tracker`.
    - User can browse their files on the network by using: `open-bucket producer ls`. Result in JSON format:
        ```json
            [
                {
                    "id": "fileUdid",
                    "path": "path/to/file/from/remote/Producer/dir.exe",
                    "localPath": "path/to/file/from/local/Producer/dir.exe",
                    "updatedAt": "2018-05-02T13:41:37.666Z",
                    "size": "2 GB",
                    "status": "UPLOADING",
                    "actualAvailability": 1,
                    "availability": 2
                },
                {
                    "id": "fileUdid1",
                    "path": "path/to/file/from/remote/Producer/dir1.txt",
                    "localPath": "path/to/file/from/local/Producer/dir1.txt",
                    "updatedAt": "2018-05-02T13:45:32.612Z",
                    "size": "5 KB",
                    "status": "AVAILABLE",
                    "actualAvailability": 2,
                    "availability": 2
                },
            ]
        ```
- User can apply changes to `Producer` by using: `open-bucket producer apply /path/to/input/file.json`
    - Input file __MUST__ be on JSON format.
    - Input file __MUST__ have `type` field. `Producer` will base on the `type` field and some other fields (if required) to apply the changes.
    - `Producer` only takes valid field, invalid fields will be ignored.
    - Input file that has valid field with invalid value will be rejected.
    - User can apply changes to file's metadata by using an Input file has `"type": "File"` (with the required field to be `id` or `path`). A sample of the file input content in JSON:
        ```json
            {
                "type": "File",
                "id": "fileId",
                "path": "path/to/file.txt",
                "availability": 2
            }
        ```
    - User can apply changes on `Producer` configs by using an Input file has `"type": "ProducerConfig"`. A sample of the file input content in JSON:
        ```json
            {
                "type": "ProducerConfig",
                "directory": "~/open-bucket/produce",
                "defaultAvailability": 2,
                "startOnStartup": true
            }
        ```
        - If user change diretory path: All the files inside the old `Producer` directory will be moved to the new path.

## Consuming Data from the Network
- User __MUST__ start the `Consumer` first by using: `open-bucket consumer start`. `Daemon` will ignore this command if the `Consumer` has already started.
- __Local Consumer directory__ will have its limit defined by user.
    - If the limit is reached, `Consumer` stop consuming data from the Network.
    - If user config the limit to be lower than __local Consumer directory__ size, __local Consumer directory__ directory will be truncated to match with the desired litmit.
    - Invalid __limit__ field will disable the `Consumer`. __limit__ value should be in format: `${consumeSize} GB`
- Data in __local Consumer directory__ will be the __encypted shards__ of the files that have been produced into the Network.
- Shards inside __local Consumer directory__ __MUST NOT__ be modified, `Consumer` will ensure the consuming shards hasn't been modified. For each modified/deleted shard, `Consumer` will ask the Tracker to remove current user from the list of the shard consumer & remove the shard locally.
- User can apply changes to `Consumer` by using: `open-bucket consumer apply /path/to/input/file.json`
    - The same rules applied as Producer input file.
    - User can apply changes to `Consumer` configs by using an Input file has `"type": "ConsumerConfig"`. A sample of the file input content in JSON:
        ```json
            {
                "type": "ConsumerConfig",
                "directory": "~/open-bucket/consume",
                "limit": "2 GB",
                "startOnStartup": true
            }
        ```
        - If user change diretory path: All the shards inside the old `Consumer` directory will be moved to the new path.

# Client
- When user install the `Client` before installing the `Daemon`, `Client` will ask user permission to install the `Daemon`

# Payment
TODO

# Implementation
## Core
- Open Bucket will have all its configs saved to the Config file. On startup, Daemon | Producer | Consumer will init their config state from Config file.
- Every HTTP Request to Daemon from Client don't need to have Authentication information (i.e. Authorization header) since Daemon can get that info directly from the Config file.

### Watcher
- `Producer` & `Consumer` have their own Watcher (`PWatcher` for Producer, `CWatcher` for Consumer)
- The Watcher will be started or terminated by its `Consumer` | `Producer`.
- Watcher extends Node.js EventEmitter.
- Watcher takes responsibility for detecting the changes in files that contain state and emit approriate event for each change (e.g. When user change `limit` field of `consumer` in the
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
### Start Producer:
1) `Daemon` starts the `Producer`.
2) `Producer` starts the `PWatcher`.
3) `Daemon` change the `Producer` status in config file to `STARTED`.
#### Command: `open-bucket producer start`
#### REST API: `POST /v1/producer/start`
- Responses:
    - 200 - {"msg": "Producer started"}
    - 200 - {"msg": "Producer has already started"}
    - 401 - {"msg": "Invalid token"}

### Stop Producer:
1) `Producer` stops the `PWatcher`.
2) `Daemon` stops the `Producer`.
3) `Daemon` changes `Producer` status in config file to `STOPPED`.
#### Command: `open-bucket producer stop`
#### REST API: `POST /v1/producer/stop`
- Responses:
    - 200 - {"msg": "Producer stopped"}
    - 200 - {"msg": "Producer has already stopped"}

### Produce a file to the Network:
1) Copy the file to __local Producer directory__.
2) `PWatcher` will detect the changes and upload the file.
3) `Producer` will communicate with the `Tracker` to upload the file.
4) The `Tracker` will __notify__ `Producer` when the file has completely uploaded into the network.
5) When `Producer` got notification from `Tracker`, it will delete the file if `deleteLocalOnComplete` option is TRUE.
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
1) Delete the file locally (if exists)
2) `Producer` will ask the `Tracker` to remove the file.
3) `Tracker` will tell all consumer of the file to delete all the shards
#### Command: `open-bucket producer rm [/path/to/file/from/Producer/dir | fileId]`
#### REST API: `DELETE /v1/producer/files/:fileId`
- Responses:
    - 200 - {"msg": "File `fileId` removed"}
    - 400 - {"msg": "File `fileId` not exists"}

### Download a file from the Network:
1) `Daemon` must make sure it has permission to write to the destination directory
2) `Daemon` tell `Tracker` that it need to download all the shards of the file `fileId`
3) `Daemon` then will start receiving shards from the `Tracker` & report speed as well as process to the `Client`
4) After all shards are downloaded, `Daemon` will resemble the shards into original file
5) `Daemon` notify Client to notify user that the file is downloaded
#### Command: `open-bucket producer cp [/path/to/file/from/Producer/dir | fileId] /destination/dir`
#### REST API: `GET /v1/producer/files/:fileId`
- Request Header:
    - destination: "/destination/dir"
- Responses:
    - 200 - {"msg": "Downloading file `fileId | filePath` to `/destination/dir`"}
    - 400 - {"msg": "File `fileId | filePath` not exists"}

## Consumer
TODO