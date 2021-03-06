## Objectives

###  Migrate data to another network
#### Get the best

Ideally data can be exported, all servers run on nudb and updated to the latest version

- [x] copy data to new server, install rippled on it (data server). configure several validators to connect to it.

- [x] idem but data server runs with --import to NuDB
-- takes 11 hrs to complete

- [] idem but other servers run with nuDb from the start

- [x] (failed) idem but servers run with latest version from the start

- [] idem but remove server's own signature from cfg

- [] during run time remove data server

- [] during run time update each server


#### Minor things
- [] Copy the data to new server install latest rippled, load it, make it connect to other servers.

- [x] Validators over 5. Peers should increase

- [] Deploy AMI image from an r snapshot, load it, make it connect to other servers. See Anthony's note

- [x] Test it with existing account to do transaction

- [] Update to latest rippled version during runtime.

- [] Stress test

- [x] Run `--load` with Nudb configuration from the start (failed)

- [x] Run `--net` with Nudb (data node will still use RocksDB)

- - [x] After it syncs delete RocksDB data node

- [] Copy the data from NuDB node and paste it to a new server. Run `--load`

- [] copy the data to GCP instance

- [] make an image of the instance

- [] run test on GCP

### Steps (v.070)

##### Copy-pasting existing data at `/var/lib/rippled` to a new empty server and install rippled.

**Starting data server**
- Deploy AMI mbcu-ubuntu1604-r1JanData-installscript070
- run `./rippled-setup.sh`
- run `/opt/ripple/bin/validator-keys create_keys` to generate validator keys
- run `/opt/ripple/bin/validator-keys create_token --keyfile /home/ubuntu/.ripple/validator-keys.json` (pay attention to user directory)
- note each server's validator_key and validator_token, add it to `/opt/rippled/etc/rippled.cfg`
- copy the db folder to `/var/lib/rippled/` (not needed for AMI mbcu-ubuntu16014-rippled-preinstall)
- - one file `random.seed` 's permission had to be changed to write `chmod +r`
- modify run parameter `/usr/lib/systemd/system/rippled.service`
- - data server is run with param `--quorum 1` and `--load`
- for ubuntu, clear `User` and `Group` from `/usr/lib/systemd/system/rippled.service`
- run it with `sudo systemctl restart rippled.service`
- logs
- - `sudo journalctl` to see if service related issues.
- - `sudo systemctl status rippled` to see rippled execution issues.
- - `tail -f /var/log/rippled/debug.log` for rippled log
- test a command like `/opt/rippled/bin/rippled server_info`
- - there's wait time after `Watchdog: Launching child 1`. after this is done debug.log will be created
- - if rippled is running, then executing rippled will result in
```
Terminating thread rippled: main: unhandled St13runtime_error 'Unable to open/create RocksDB: IO error: lock /var/lib/rippled/db/rocksdb/LOCK: Resource temporarily unavailable'
```
- - initially any command will return below. it could be ok debug.log doesn't show any error.
```
{
   "error" : "internal",
   "error_code" : 69,
   "error_message" : "Internal error.",
   "error_what" : "no response from server"
}
```
- loading data takes a lot of time. loading january data takes 3.5 hrs
```
2018-Feb-09 00:32:36 Application:NFO Loading specified Ledger
2018-Feb-09 00:32:37 Application:NFO Loading ledger F0F211C05C509156205C73D245F7824995A83BB1C69EDC9953FCE61917B00566 seq:3713121
2018-Feb-09 03:24:35 TimeKeeper:NFO SNTP: Alarm condition 45.127.112.2:123
2018-Feb-09 03:48:58 LedgerMaster:DBG Ledger 3713121 accepted :F0F211C05C509156205C73D245F7824995A83BB1C69EDC9953FCE61917B00566
2018-Feb-09 03:48:59 Amendments:ERR Unsupported amendment 9178256A980A86CF3D70D0260A7DA6402AAFE43632FDBCB88037978404188871 activated.
2018-Feb-09 03:48:59 LedgerMaster:ERR One or more unsupported amendments activated: server blocked.
2018-Feb-09 03:48:59 NetworkOPs:NFO STATE->tracking
2018-Feb-09 03:48:59 OrderBookDB:DBG Advancing from 0 to 3713121
```
- open files problem
```
2018-Feb-08 05:18:29 Application:NFO Loading ledger F0F211C05C509156205C73D245F7824995A83BB1C69EDC9953FCE61917B00566 seq:3713121
2018-Feb-08 05:18:31 NodeObject:ERR IO error: /var/lib/rippled/db/rocksdb/088158.sst: Too many open files
2018-Feb-08 05:18:31 NodeObject:WRN Unknown status=105
2018-Feb-08 05:18:31 Application:FTL Data is missing for selected ledger
2018-Feb-08 05:18:31 Application:ERR The specified ledger could not be loaded.
2018-Feb-08 05:18:31 Application:DBG Received signal: 2
```

- - for ubuntu it looks like max open files is 1024 (r1 centos is 8000) https://underyx.me/2015/05/18/raising-the-maximum-number-of-file-descriptors

**Joel Katz guide**
- https://forum.ripple.com/viewtopic.php?f=2&t=16207

If you're going to make three validators with a quorum of 2, run the validation_create command three times. Take the three public keys and use that as the UNL on each server. Take the three private keys and configure one of them on each validator.

You then have the challenge of coldstarting your blockchain. The easiest way to do that is to start one validator with "--quorum 1". Let it stabilize, and then add the other two validators also with "--quorum 1". Once all validators are stable, restart them one at a time without any "--quorum" command to return to the configured quorum of 2.

Note that a 2 of 3 configuration does give you fault tolerance but it does not protect you against even a single malicious/broken validator.

- - when we restart a server without `quorum` the log is stuck at this

2018-Feb-09 06:39:28 ValidatorList:WRN New quorum of 7 exceeds the number of trusted validators (6)

so we set the quorum explicitly (4 for 6 validators)

-https://forum.ripple.com/viewtopic.php?f=2&t=15608

The "validation_public_key" is what all servers should add to their list of validators. The "validation_seed" is what you should add to one server's configuration.

The selection of a validation quorum is a bit complex. For large numbers of validators (greater than 15 or so), 80% of the UNL size works well. For smaller numbers, you have to make tradeoffs between fault tolerance and attack resistance. For example, with three validators, two validators give you good fault tolerance but almost no resistance to a malicious validator. Three validators gives you no fault tolerance, but resistance to a malicious validator.

**Deploying validators**

- wait until data server loads the data.
```
"build_version" : "0.70.0",
"complete_ledgers" : "506533-3713523",
"hostid" : "ip-172-31-58-223",
```
- run parameter `--quorum 1` and `--net`
- all servers are started simultaneously
- it takes around 5 minutes for the log to move from
```
Server:NFO Opened 'port_ws_public' (ip=127.0.0.1:5005, ws)
2018-Feb-08 04:00:02 NetworkOPs:WRN We are not running on the consensus ledger
```
- from time to time check `rippled peers` to see other peers the server listens to
- restarting a server will cause `Peer:WRN [006] onReadMessage: short read` on the other servers' log

**Deploying stocks**
- download and install Rippled
- remove `[validator_key]` and `[validator_token]`
- run with `--net` and `--quorum` with appropriate value

**Clusters**
- copied `[validators]` to `[node_clusters]`
- check with `rippled peers` and look at `clusters`

**NuDB Config**
```
type=NuDB
path=/var/lib/rippled/db/nudb
```

**Deploying data node with type=nuDb from the beginning**
- when original data were stored in RocksDB this doesn't work

**Importing data --import**
- use data server
- configuration is NuDB
- run --import
- takes 11 hours
```
[import_db]
type=RocksDB
path=/var/lib/rippled/db/rocksdb


[node_db]
type=NuDB
path=/var/lib/rippled/db/nudb
open_files=2000
filter_bits=12
cache_mb=256
file_size_mb=8
file_size_mult=2
online_delete=ledger_history
advisory_delete=0
```


#### 0.81
- ledger close is stuck
```
"id" : 1,
"result" : {
   "info" : {
      "amendment_blocked" : true,
      "build_version" : "0.81.0",
      "complete_ledgers" : "3713121",
      "hostid" : "ip-172-31-54-84",
      "io_latency_ms" : 1,
```
compare it with 0.70 which has range and moves backward
```
"build_version" : "0.70.0",
"complete_ledgers" : "2796120-3713222",
"hostid" : "ip-172-31-58-223",

"build_version" : "0.70.0",
"complete_ledgers" : "2721120-3713230",
"hostid" : "ip-172-31-58-223",

"build_version" : "0.70.0",
"complete_ledgers" : "2588620-3713243",
"hostid" : "ip-172-31-58-223",
```

#### Anthony's method: creating an AMI image of current EC2 instance and deploy it to a new network

Replication Test(A):
1. Create Image from servers r1-r3
2. Lunch Instances from images created from step 1(Note: Do not start rippled services)
3. Modify rippled config file(ripple.cfg) for each server in step 1.
  - change value in [ips_fixed] with the IP's from servers in step 1.
4. start rippled services in step 1 simultaneously (starting from r1 copy to r3 copy)
5. verify that the services are running by checking "systemctl status rippled" and "rippled server_info"
6. if rippled service is running run the command on each servers "rippled peers" this should return at least 2 peers
  with details for each server including the closed ledgers. If service is not running check configuration and restart service

Test:
  Restarting service one by one
  A. Restating service without changing in ripple.service file
    1. reboot rippled service in server 1
    2. check remaining server status using "rippled server_info" and "rippled peers"
      - on "rippled server_info" response check for completed ledgers if no ledgers are missing
      and compare with other servers to verify if ledgers and synching
      - on "rippled peers" command check for peers and check if ledgers are the same
    3. repeat step 1 and 2 until all servers are tested

  B. Restating service after changing ripple.service file
    1. modify  server 1  rippled.file by changing "--quorum 3" to "--quorum 2"
    2. enter command "sudo systemctl daemon-reload"
    3. reboot rippled service
    4. check remaining server status using "rippled server_info" and "rippled peers"
      - on "rippled server_info" response check for completed ledgers if no ledgers are missing
      and compare with other servers to verify if ledgers and synching
      - on "rippled peers" command check for peers and check if ledgers are the same
    3. repeat step 1 and 2 until all servers are tested
