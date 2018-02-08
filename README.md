## Objectives

###  Migrate data to another network
- [] Copy the data to new server install latest rippled, load it, make it connect to other servers.

- [] Deploy AMI image from an r snapshot, load it, make it connect to other servers. See Anthony's note

- [x] Test it with existing account to do transaction

- [] Use latest rippled version.

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
- id debug.log is missing wait for some moments. if it's still missing reboot the instance
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
- looks like Jan data will get stuck with
`Application:NFO Loading ledger F0F211C05C509156205C73D245F7824995A83BB1C69EDC9953FCE61917B00566 seq:3713121`
- in the old server this part only took 6 minutes
```
2018-Feb-06 06:38:32 Application:NFO Loading ledger F0F211C05C509156205C73D245F7824995A83BB1C69EDC9953FCE61917B00566 seq:3713121
2018-Feb-06 06:43:02 LedgerMaster:DBG Ledger 3713121 accepted :F0F211C05C509156205C73D245F7824995A83BB1C69EDC9953FCE61917B00566
2018-Feb-06 06:43:02 OrderBookDB:DBG Advancing from 0 to 3713121
2018-Feb-06 06:43:02 OrderBookDB:DBG OrderBookDB::update>
2018-Feb-06 06:43:02 ManifestCache:NFO Manifest: AcceptedNew;Pk: nHB1gBYNwE7JQYUybTuZwHhUiVZggfEXgVgCQqCXyQDyKAc68x7J;Seq: 1;
2018-Feb-06 06:43:02 ManifestCache:DBG Manifest: Stale;Pk: nHB1gBYNwE7JQYUybTuZwHhUiVZggfEXgVgCQqCXyQDyKAc68x7J;Seq: 1;OldSeq: 1;
2018-Feb-06 06:43:02 ValidatorList:DBG Loading configured trusted validator list publisher keys
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

**Deploying other servers**

- before running service it's better to run rippled directly to see what error comes out
- in case of "cannot access /var/lib/rippled/db" use `chown -R ubuntu /var/lib/rippled`
- run parameter `--quorum 1` and `--net`
- it takes around 5 minutes for the log to move from
```
Server:NFO Opened 'port_ws_public' (ip=127.0.0.1:5005, ws)
2018-Feb-08 04:00:02 NetworkOPs:WRN We are not running on the consensus ledger
```
- at this point it running `rippled peers` will show the other non-data servers
- restarting a server will cause `Peer:WRN [006] onReadMessage: short read` on the other servers' log



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
