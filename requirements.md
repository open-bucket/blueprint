# Daemon

## Init
- User __must__ have `Node.js@10.0.0` installed and install `@open-bucket/daemon` globally using `npm`: `npm i -g @open-bucket/daemon`
- `Daemon` has __CLI__ and __TCP__ interface. User interacts with Daemon through __CLI__, Client interacts with Daemon through __TCP__ interface
- Daemon have __2 Primary Instances__: `Producer` & `Consumer`
    - `Producer`: provides functionality for producing storage space into the Network
    - `Consumer`: provides functionality for consuming storage space from the Network
- User __MUST__ init the `Daemon` with: `open-bucket init`. The `init` command will walk user through the following steps:
    - Authentication (Register/Login)
    - Define the Config file
        - Ask if user want to config `Producer`
        - Ask if user want to config `Consumer`
        - Ask user to allow the `Daemon` to start `Producer` | `Consumer` on startup
    - Ask user to allow the `Daemon` to start `Producer` | `Consumer`
- If user re-init `Daemon`, all current configs will be overridden

## Consuming storage space from the Network
- User __MUST__ start the `Comsumer` first by using: `open-bucket consumer start`. `Daemon` will ignore this command if the `Consumer` has already started
- User can consume storage from the Network with 2 ways:
    - __COPY__ files directly into the __local Consumer directory__. Based on the `Consumer` status, the following cases will be considered:
        - [If the `Consumer` status is __STARTED__]: the `Consumer` will detect file changes and automatically upload the unsynced files into the Network with __default availability is 2__
        - [If the `Consumer` status is __STOPPED__]: Nothing will happen. But when user start the `Consumer`, `Consumer` will ask if user wanted to upload the unsynced file to the Network
            - [If __YES__]: the `Consumer` will consume the storage space from the Network to storage files
            - [If __NO__]: the `Consumer` will delete the files on __local Consumer directory__
    - Use `open-bucket consumer add` command
- When user delete their files on __local Consumer directory__:
    - [If the file is __completely uploaded__ into the Network]: The files won't be deleted on the Network, it just got deleted locally
    - [If the file is __NOT completely uploaded__ into the Network]: the file will be removed on the Network
- A file is considered __completely uploaded into the Network__ when its __actual availability__ is matched with is __desired availability__ defined by user
- The __availability__ is calculated by the `Tracker`. `Tracker` will calculate the __availability__ based on the number of producer is producing shards of the file
- User can delete their file on the Network by using `open-bucket consumer rm` command
- User can download their file on the Network by using `open-bucket consumer cp` command
- Files produced into the Network will have their metadata saved by the `Tracker`. User can browse their files on the network by using `open-bucket consumer ls` command

## Providing storage space to the Network
- User __MUST__ start the `producer` first by using: `open-bucket producer start`. `Daemon` will ignore this command if the `Producer` has already started
- __Local Producer directory__ will have its limit defined by user.
    - If the limit is reached, `Producer` stop consuming data from the Network
    - If user config the limit to be lower than __local producer directory__ size, __local Producer directory__ directory will be truncated to match with the desired litmit
    - Invalid __limit__ field will disable the `Producer`. __limit__ value should be in format: `${produceSize} GB`
- Data in __local producer directory__ will be the __encypted shards__ of the files that have been uploaded into the Network
- Shards inside __local Producer directory__ __MUST NOT__ be modified, `Producer` will ensure the providing shards hasn't been modified. For each modified/deleted shard, `Producer` will ask the `Tracker` to remove current user from the list of the shard provider & remove the shard locally

# Client
- When user install the `Client` before installing the `Daemon`, `Client` will ask user permission to install the `Daemon`

# Payment
TODO

# Implementation
## Core
- Open Bucket will have all its configs saved to the Config file. On startup, Daemon | Producer | Consumer will init their config state from Config file
- Every HTTP Request to Daemon from Client don't need to have Authentication information (i.e. Authorization header) since Daemon can get that info directly from the Config file
- Uses [Inquirer.js](https://github.com/SBoudrias/Inquirer.js) for interactive command line user interfaces

### The Config file
```json
{
    "authentication": {
        "token": "User Authentication Token",
        "privateKey": "User private key"
    },
    "consumer": {
        "directory": "~/open-bucket/consume",
        "defaultAvailability": 4,        
        "port": 1234,
        "startOnStartup": false
    },
    "producer": {
        "directory": "~/open-bucket/produce",
        "limit": "1 GB",
        "port": 5678,
        "startOnStartup": false
    },
    "_updatedAt": "1525147564"
}
```
Default values will be used in place of missing configs, the fields start with `_` (e.g: `_status`) are INTERNAL fields.

### Watcher
- `Producer` & `Consumer` have their own Watcher (`PWatcher` for Producer, `CWatcher` for Consumer)
- The Watcher will be started or terminated by its `Consumer` | `Producer`
- Watcher extends Node.js EventEmitter
- Watcher takes responsibility for detecting the changes in files that contain state and emit approriate event for each change (e.g. When user change `limit` field of `consumer` in the
Config file, the CWatcher will detect the changes and emit `setLimit` event with the new limit)
- Use [Chokidar](https://github.com/vunguyenhung/chokidar) to detect file changes (e.g: webpack, nodemon,... )
     - Chokidar usage - Nodemon: https://github.com/vunguyenhung/nodemon/blob/master/lib/monitor/watch.js

### Init:
> Note: this feature only available on Daemon CLI & uses only interactive mode
- __Authentication__: Daemon CLI shows options for user to choose Register or Login
    - [If user choose Login]: Uses Daemon `login` functionality
    - [If user choose Register]: Uses Daemon `register` functionality
- __Define the Config file__: Daemon CLI shows options for user to choose whether they want to config `Producer` or `Consumer`:
    - [If user choose __Producer__]: Uses Producer `config` functionality
    - [If user choose __Consumer__]: Uses Consumer `config` functionality
- __Ask user to allow the `Daemon` to start `Producer` | `Consumer`__: Perform `open-bucket producer start`
#### Command: `open-bucket init`

### Login:
1) `Daemon` asks username & password from user
2) `Daemon` then send request to the `Tracker` along with username & password
3) After receiving response from the `Tracker`.
    - [If 200]: `Daemon` then save user auth to the Config file.
    - [If 400]: `Daemon` then show error from the Tracker to user.
#### Command: `open-bucket auth login [options]`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                    Disable interactive mode. In this case, user MUST specify all required fields
    -u, --username <yourUsername>   Specify username
    -p, --password <yourPassword>   Specify password
    ```
#### REST API: `POST /v1/auth/login`
- Request body:
    ```json
    {
        "username": "yourUsername",
        "password": "yourPassword"
    }
    ```
- Responses:
    - 200 - {"token": "JWT Token"}
    - 400 - {"msg": "Invalid username or password"}

### Register:
1) `Daemon` ask user basic information (username, password, email, etc...)
2) `Daemon` then send request to the `Tracker` along with user basic information
3) After receiving response from the `Tracker`:
    - [If 201]: Show Create success message to user
    - [If 400]: Show error message from the `Tracker` to user.
#### Command: `open-bucket auth register [options]`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                    Disable interactive mode. In this case, user MUST specify their info.
    -u, --username <yourUsername>   Specify username
    -p, --password <yourPassword>   Specify password
    -e, --email    <yourEmail>      Specify email
    ```
#### REST API: `POST /v1/auth/register`
- Request body:
    ```json
    {
        "username": "yourUsername",
        "password": "yourPassword",
        "email": "yourEmail"
    }
    ```
- Responses:
    - 201 - {"msg": "Account created"}
    - 400 - {"msg": "Duplicate username"}

### Config Authentication information:
Using this feature will change user authentication information on the Config file. Users takes all responsibility to validate their input.
#### Command: `open-bucket auth config [options]`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                        Disable interactive mode. In this case, user MUST specify username & password options
    -t, --token <yourToken>             Specify authentication token
    -p, --private-key <yourPrivateKey>  Specify private key
    ```
#### REST API: `POST /v1/auth/config`
- Request body:
    ```json
    {
        "token": "yourToken",
        "privateKey": "yourPrivateKey"
    }
    ```
- Responses:
    - 200 - {"msg": "Change authentication configuration successfully"}

## Consumer
### Start Consumer:
1) `Daemon` starts the `Consumer`.
2) `Consumer` starts the `PWatcher`.
#### Command: `open-bucket consumer start`
#### REST API: `POST /v1/consumer/start`
- Responses:
    - 200 - {"msg": "Consumer started"}
    - 200 - {"msg": "Consumer had already started"}

### Stop Consumer:
1) `Consumer` stops the `PWatcher`.
2) `Daemon` stops the `Consumer`.
#### Command: `open-bucket consumer stop`
#### REST API: `POST /v1/consumer/stop`
- Responses:
    - 200 - {"msg": "Producer stopped"}
    - 200 - {"msg": "Producer had already stopped"}

### Consumer storage space from the Network:
1) Copy the file to __local Consumer directory__.
2) `PWatcher` will detect the changes and notify Consumer to upload the file.
3) `Consumer` upload the file:
    - Get __defaultAvailability__ in the Config file
    - `Consumer` will attemp to encrypt & shard the file.
    - Then, Consumer tell the `Tracker` that it need to upload a file.
    - The `Tracker` then form __contracts__ between the user and available producers & returns to `Consumer` a list of producers.
    - After the `Tracker` returns a list of producers, `Consumer` upload file shards to those producers.
4) After the file has been completely uploaded into the network, Consumer then notify the user.
5) Finally Consumer delete the file if `deleteLocalOnComplete` option is specified and notify user.
#### Command: `open-bucket consumer add [options] </path/to/file>`
- Options:
    ```
    -d, --delete-on-complete        Delete the file on local Producer directory on upload complete
    ```
#### REST API: `POST /v1/producer/files/upload`
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
        "msg": "File filePath has been added",
        "id": "FileId"
    }
    ```

### Remove a file from the Network:
1) `Consumer` will atempt to validate inputs from user based on the following rules:
    - __fileId__: must be a string, this field will have higher priority over __remotePath__
    - __remotePath__: must be a valid path string, this field will be ignored if __fileId__ is specified
2) Delete the local file.
    - [If the local file is existing]: `PWatcher` will detect the changes & notify `Consumer` to trigger the removing file process
    - [If the local file is __NOT__ existing]: trigger the removing file process of the `Consumer` manually
3) `Consumer` delete the file on the network:
    - `Consumer` tells the `Tracker` that it need to remove the file from the network
    - The `Tracker` modify the contract between the user & producers
    - The `Tracker` will tell all consumers of the file to delete all the shards
    - Providers notify the `Tracker` after removing the shards successfully
    - After the `Tracker` confirm that all the shards has been deleted, it will notify to the `Consumer`
4) After receiving delete confirmation from the `Tracker`, `Consumer` will notify the `Client`
#### Command: `open-bucket consumer rm </path/to/file/from/Producer/dir | fileId>`
#### REST API: `POST /v1/consumer/files/remove`
- Request Body:
    ```json
    {
        "fileId": "fileId",
        "remotePath": "/path/to/file/from/remote/Consumer/dir"
    }
    ```
- Responses:
    - 200 - {"msg": "File `fileId` removed"}
    - 400 - {"msg": "File `fileId` not exists"}

### Download a file from the Network:
1) Producer will atempt to validate inputs from user based on the following rules:
    - __fileId__: must be a string, this field will have higher priority over __remotePath__
    - __remotePath__: must be a valid path string, this field will be ignored if __fileId__ is specified
    - __destination__: required, must be a valid path string
2) `Consumer` tell `Tracker` that it need to download all the shards of the file `fileId`
3) The `Tracker` then modify the contract between the user & producer, then return the list of producers to the `Consumer`
4) `Consumer` then will start receiving shards from the producers & report speed as well as process to the `Client`
5) After all shards are downloaded, `Consumer` will resemble the shards into original file
6) `Consumer` notify `Client` to notify user that the file is downloaded
#### Command: `open-bucket consumer cp </path/to/file/from/remote/Consumer/dir | fileId> </destination/dir>`
#### REST API: `POST /v1/consumer/files/download`
- Request Body:
    ```json
    {
        "fileId": "fileId",
        "remotePath": "/path/to/file/from/remote/Consumer/dir",
        "destination": "/path/to/local/dir"
    }
    ```
- Responses:
    - 200 - {"msg": "Downloading file `fileId | filePath` to `/destination/dir`"}
    - 400 - {"msg": "File `fileId | filePath` not exists"}

### Change Comsumer config:
1) `Consumer` will validate the input based on the following rules:
    - __directory__: current user MUST have `read & write permission` to this dir
    - __defaultAvailability__: must be a number, greater than 3.
    - __port__: must be a number
    - __startOnStartup__: must be `true` or `false`
2) After all fields has passed their validation rules, `Consumer` will attemp to do the following actions:
    - [If __directory__ is changed]: 
        - Move all current files in the old directory to the new one.
        - Change the Config file
    - [If __defaultAvailability__ is changed]: Change the Config file
    - [If __startOnStartup__ is changed]: 
        - Make user OS to run `open-bucket consumer start` on startup
        - Change the Config file  

If any of those steps failed, all the changes will be rolled back.
#### Command: `open-bucket consumer config`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -r, --directory <dirPath>              Specify local Producer directory
    -a, --default-availability <number>    Specify default availability used when upload file
    -s, --start-on-startup                 Specify to start Producer on startup
    ```
#### REST API: `POST /v1/consumer/configs`
- Request body:
    ```json
    {
        "directory": "~/open-bucket/consume",
        "defaultAvailability": 2,
        "port": 1234,
        "startOnStartup": true
    }
    ```
- Responses:
    - 200 - {"msg": "Producer configs applied"}
    - 400 - {"msg": "Invalid directory"}

### Change file metadata on Tracker:
1) `Consumer` will validate the input based on the following rules:
    - __fileId__: required, must be a string
    - __remotePath__: must be a valid path string
    - __availability__: must be a number
2) After all fields has passed their validation rules, `Producer` will attemp to do the following actions:
    - [If __remotePath__ is specified]: `Tracker` change file metadata
    - [If __availability__ is specified]: `Tracker` attempt to add/remove consumers of the file based on the new availability
3) Tracker notify `Producer` when the process is done.
#### Command: `open-bucket consumer edit [options] <fileId>`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -r, --remotePath <dirPath>             Specify new remote path to file
    -a, --availability  <number>           Specify new availability number of the file
    ```
#### REST API: `POST /v1/consumer/edit`
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

### Browse files on the network:
#### Command: `open-bucket consumer ls`
- Options:
    ```
    -l, --local                 Shows files metadata on local Producer directory
    ```
#### REST API: `GET /v1/consumer/files`
- Responses:
    - 200
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


## Producer
### Change Producer config:
1) `producer` will validate the input based on the following rules:
    - __directory__: current user MUST have `read & write permission` to this dir
    - __limit__: the value must have format: `${consumeSize} GB`
    - __startOnStartup__: must be `true` or `false`
2) After all fields has passed their validation rules, `Producer` will attempt to do the following actions:
    - [If __directory__ is changed]: 
        - Move all current files in the old directory to the new one.
        - Change the Config file
    - [If __limit__ is changed]: 
        - `Consumer` will attempt to truncate the dir
        - For each truncated shard, CWatcher will watch the changes and notify `Producer` to trigger the removing file process
    - [If __startOnStartup__ is changed]: 
        - Make user OS to run `open-bucket producer start` on startup
        - Change the Config file  

If any of those steps failed, all the changes will be rolled back.
#### Command: `open-bucket producer config`
> _This command uses __interactive mode__ by default._
- Options:
    ```
    -d, --detach                           Disable interactive mode.
    -r, --directory <dirPath>              Specify local Producer directory
    -l, --limit <number>                   Specify limit
    -s, --start-on-startup                 Specify to start Producer on startup
    ```
#### REST API: `POST /v1/producer/configs`
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
