# Daemon

## Init
- User __must__ have `Node.js` installed and install `@open-bucket/daemon` globally using `npm`: `npm i -g @open-bucket/daemon`
- `Daemon` will act as a `cli` and a `server`. The `cli` for user to easier to use, the `server` which expose `REST APIs` or `GraphQL endpoints` which will be used by the Client.
- User __can__ use `open-bucket [producer | consumer] start` command to start `Producer` or `Consumer` manually. Or they can config the `Daemon` to start on startup
- User __must__ init the `Daemon` with: `open-bucket init`. The `init` command will walk user through the following steps:
    - Authentication
    - Ask user whether they use `Producer` or `Consumer` functionality. User will need to defind the config for the one they've choose *YES* (Default values will be used in place of missing configs)
    - Configurations:
        ```json
        {
            "token": "User Authentication Token",
            "consumer": {
                "directory": "~/open-bucket/consume",
                "limit": "2 GB",
                "started": true,
                "startOnStartup": true,
                "disabled": true
            },
            "producer": {
                "directory": "~/open-bucket/produce",
                "started": true,
                "startOnStartup": true,
                "disabled": true
            }
        }
        ```
    - Instruct user to start the Producer or the Consumer
    - Ask user to allow the `Daemon` to start on startup
- User will able to change the configs by using: `open-bucket producer configs` or `open-bucket consumer configs`. The command will __open the config file in user's default editor__. When the config file is saved, the saved configs will be applied.

## Producing Files to the Network
- User can __copy__ files directly into the __local Producer directory__. Based on the Producer status, the following cases will be considered:
    - \[If the Producer is `started` \]: the Producer will detect the changes and automatically upload the unsynced files into the Network.
    - \[If the Producer is `stoped` \]: Nothing will happen at the time. But when user start the Producer, Producer will ask if user wanted to produce the unsynced file to the Network
        - \[If YES\]: the Producer will produce the files to the Network.
        - \[If NO\]: the Producer will delete the files on Producer directory & Delete the file in the network
- User can delete the file in __local Producer directory__:
    - \[If the file is __completely uploaded__ into the Network\]: nothing will happen.
    - \[If the file is __NOT completely uploaded__ into the Network\]: the file will be removed on the Network.
- Data produced into the Network will have its metadata saved by the Tracker.
    - User can browse their data on the network by using: `open-bucket producer ls`, add option `--local` to browse data on their local machine.
        - Result format:

            | id       | Path             | Last Modified | Size | Status        | Availability |
            | -------- | ---------------- | ------------- | ---- |:------------: |:------------:|
            | someUdid | path/to/file.txt | Apr 30 15:35  | 2 Kb |UPLOADING      |2             |

- A file is considered __completely uploaded into the Network__ when its __actual availability__ is matched with is __desired availability__ defined by user.
- The availability is calculated by the Tracker & based on the count of consumers in the Network that consume the file (this may change in the future).
- User can edit file's metadata on the Network by using: `open-bucket producer edit [ /path/to/file | fileId ]`. File's editable metadata will be opened on user's default editor & will have json format. The changes in file's metadata will be applied when user save. Example of file's editable metadata:  
    ```json
        {
            "path": "path/to/file.txt",
            "availability": 1
        }
    ```
- User can delete their file on the Network by using: `open-bucket producer rm [ /path/to/file | fileId ]`, this command will also delete the file on __local Producer directory__.

## Consuming Data from the Network
- Consumer directory will have its limit defined by user.
    - If it is reached, `Daemon` stop consuming data from the Network.
    - If user config the limit to be lower than current Consumer directory size, Consumer directory will be truncated to match with the desired litmit.
- If user config disable the Consumer, all data in __local Consumer directory__ will be deleted.

# Client
- When user install the `Client` before installing the `Daemon`, Client will ask user permission to install the `Daemon`.
- The Client will have all the functionality of `Daemon` (including Producer, Consumer, configs, etc...) visualized to the user.

# Implementation
- Research mechanism to detect file changes (e.g: webpack, nodemon,... )



