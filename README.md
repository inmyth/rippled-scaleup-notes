## Objectives

###  Migrate data to another network
- [] Copy the data to new server install latest rippled, load it, make it connect to other servers.

- [] Deploy AMI image from an r snapshot, load it, make it connect to other servers. See Anthony's note

- [] Test it with existing account to do transaction


### Steps

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
- run it with `sudo systemctl enable rippled.service`
- if it doesn't work check
- - `sudo journalctl` to see if there's owner or group issues.
- - `sudo systemctl status rippled` to see rippled runtime issues.
- - on ubuntu, I made it run by deleting Group and User in rippled.service, created an empty file for log `/var/log/rippled/debug.log` then rebooted the server
- check it with `tail -f /var/log/rippled/debug.log` to see if rippled works fine or not
- after a while test a command like `/opt/rippled/bin/rippled server_info`
**Deploying other servers**
- data server should finish loading first before running other servers one-by-one
- before running service it's better to run rippled directly to see what error comes out
- in case of "cannot access /var/lib/rippled/db" use `chown -R ubuntu /var/lib/rippled`
- run parameter `--quorum 1` and `--net`





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
