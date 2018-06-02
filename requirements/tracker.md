# Table Of Content 
- [What is tracker?](#what-is-tracker?)
- [How does it work ?](#how-does-it-work?)

# What is tracker?
Tracker receives tracking request from cost center, It keeps a map of torrents and providers, frequently checks the availability, upload speed, etc ... of torrents on providers. That info is consumed by [Market](), [Cost Center](), [Client]().

|Torrent|Provider|
|:---|:---|
|Torrent 1|Provider A <br> Provider B <br> Provider C
|Torrent 2|Provider B <br> Provider C <br> Provider D
|...

# How does it work ?
After a period of time, tracker create a download progress for each torrents with only one provider, so tracker can measure availability and upload speed of a provider per torrent.

|Download progress|Torrent|Provider|Status|Speed
|---|---|---|---|---
|Progress 1.1|Torrent A|Provider A|Seeder|100kb/s
|Progress 1.2|Torrent A|Provider B|Seeder|200kb/s
|Progress 1.3|Torrent A|Provider C|Offline|0kb/s
|Progress 2.1|Torrent B|Provider B|Seeder|200kb/s
|Progress 2.2|Torrent B|Provider C|Leecher|100kb/s
|Progress 2.3|Torrent B|Provider D|Offline|0kb/s
