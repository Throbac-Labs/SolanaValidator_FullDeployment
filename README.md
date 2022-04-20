# SolanaValidator_FullDeployment

End to end deployment of a testnet Solana Validator. This tutorial borrows heavily from [agjell's](https://github.com/agjell/sol-tutorials/blob/master/setting-up-a-solana-devnet-validator.md), with some updates, tweaks, and expansions.


## Overview

1. Server Specs
2. Set Up Ubuntu Server, Firewall & User
3. Properly Partition Hard Drive
4. Download & Configure Solana Cluster
5. Create & Configure Validator Accounts
6. Create Startup Script & System Services
7. Set Up Log Rotation
8. Start Validator
9. Staking
10. Monitoring

## Server Specs

Using a server rented through Lumen via the Solana Server Program.

CPU 32 cores @ 2.9 GHz (Intel w/ AVX-512F & SHA-NI instruction sets) 
  - Storage 1 x 480 GB SDD + 2 x 1 TB NVMe 
  - Memory 256GB 
  - Network 2x 10 Gbps with unlimited free transfers
  - OS: Ubuntu

## Set Up Ubuntu Server, Firewall & User

First we need to do is configure our Ubuntu server. Throbac rents a server provided by Lumen with Ubuntu pre-installed. Most bare metal providers will do the same.

The first thing to do after logging into a fresh installation is install updates:

```
sudo apt update && sudo apt upgrade --assume-yes
```

Solana does not require root privileges, and it’s considered poor practice to run Solana as “root". It’s also considered good to practice the principle of least privilege. This basically means that no user, process or application should ever have higher privileges than they need to fulfill their intended purpose. Therefore we'll create a new user, “sol", which will be running the validator service as a regular user:

```
sudo adduser sol
```

Next we'll create the firewall. Be carefull to properly add all the allow rules (particularly SSH) before enabling the firewall so as not to lock yourself out of the server. Our firewall rules are based on the minimum necessary to operate a validator. Certain ports are specific to testnet, so take care to update based on your desired cluster.

```
sudo ufw allow 22/tcp \
sudo ufw allow "OpenSSH" \
sudo ufw allow 80 \
sudo ufw allow from MY-LOCAL-IP \ # (optional)
sudo ufw allow 8000:8020/udp \
sudo ufw allow 8000/tcp \
sudo ufw allow 8001/tcp \
sudo ufw allow 8899/tcp \
sudo ufw allow 8900/tcp \
sudo ufw enable \
sudo ufw status 
```
Output should look as follows:

```
     To                         Action      From 
     --                         ------      ---- 
[ 1] 22/tcp                     ALLOW IN    Anywhere 
[ 2] 80/tcp                     ALLOW IN    Anywhere 
[ 3] Anywhere                   ALLOW IN    MY-LOCAL-IP # (optional)
[ 4] 8000:8020/tcp              ALLOW IN    Anywhere 
[ 5] 8000:8020/udp              ALLOW IN    Anywhere 
[ 6] 8900/tcp                   ALLOW IN    Anywhere 
[ 7] 8899/tcp                   ALLOW IN    Anywhere 
[ 8] 22/tcp (v6)                ALLOW IN    Anywhere (v6) 
[ 9] 80/tcp (v6)                ALLOW IN    Anywhere (v6) 
[10] 8000:8020/tcp (v6)         ALLOW IN    Anywhere (v6) 
[11] 8000:8020/udp (v6)         ALLOW IN    Anywhere (v6) 
[12] 8900/tcp (v6)              ALLOW IN    Anywhere (v6) 
[13] 8899/tcp (v6)              ALLOW IN    Anywhere (v6)
```

## Properly Partition Disks

This was one of the biggest hurdles for us in deployment. Solana is a performance intensive blockchain, and understanding proper partitioning is paramount to your server performing successfully. This partition is based on the Lumen server Throbac rents. It has 256GB of RAM, 2x 1TB NVMe's, and 1x 480GB NVMe pre partitioned with our Ubuntu OS.

Make sure you are logged in as a user with root privileges for this entire section.

Our original Root Filesystem & Device list:
```
df - h
Filesystem           Size  Used Avail Use% Mounted on 
udev                 126G     0  126G   0% /dev 
tmpfs                 26G  2.6M   26G   1% /run 
/dev/mapper/os-root   55G   20G   33G  38% / 
tmpfs                126G     0  126G   0% /dev/shm 
tmpfs                5.0M     0  5.0M   0% /run/lock 
tmpfs                126G     0  126G   0% /sys/fs/cgroup 
/dev/nvme0n1p1       511M  5.3M  506M   2% /boot/efi 
/dev/loop0            62M   62M     0 100% /snap/core20/1376 
/dev/loop1            68M   68M     0 100% /snap/lxd/22526 
/dev/loop2            44M   44M     0 100% /snap/snapd/15177 
tmpfs                 26G     0   26G   0% /run/user/1000

sudo ls blk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT 
loop0         7:0    0  61.9M  1 loop /snap/core20/1376 
loop1         7:1    0  67.9M  1 loop /snap/lxd/22526 
loop2         7:2    0  43.6M  1 loop /snap/snapd/15177 
nvme0n1     259:0    0 447.1G  0 disk 
├─nvme0n1p1 259:1    0   512M  0 part /boot/efi 
└─nvme0n1p2 259:2    0    50G  0 part 
  └─os-root 253:0    0    50G  0 lvm  / 
nvme1n1     259:3    0 894.3G  0 disk 
nvme2n1     259:4    0 894.3G  0 disk 
```
#### First, we resize the OS Disk to allow more breathing room for our OS.
```
sudo lsblk 
sudo growpart /dev/nvme0n1 2 
sudo lsblk 
df -h 
sudo pvresize /dev/nvme0n1p2 
sudo lvextend -r -l +80%FREE /dev/mapper/os-root 
sudo lsblk 
df -h
```
#### Second, we partition a new disk that we will eventually use to store the Ledger Data.

Ledgerdb Partition:
```
sudo parted /dev/nvme1n1 mklabel gpt
sudo blkid /dev/nvme1n1
sudo parted /dev/nvme1n1 mkpart primary ext4 0% 100%
sudo lsblk
sudo mkfs.ext4 /dev/nvme1n1p1
sudo mkdir /mnt/ledgerdb
sudo mount /dev/nvme1n1p1 /mnt/ledgerdb
sudo blkid /dev/nvme1n1p1
```
Use the UUID from the above and add it to your /etc/fstab file.
```
sudo nano /etc/fstab
------> UUID=546cd562-dab7-45dc-ac38-7f2f4021a895 /mnt/ledgerdb ext4 defaults,nofail 0 0
```
Save & exit. Then unmount and remount the drive to make sure everything is configured properly.
```
sudo umount /mnt/ledgerdb
sudo mount -a
ls /mnt/ledgerdb 
```
You should see "lost+found" in the ledgerdb directory. Now make sure the "sol" user has ownership over the directory where the drive is mounted.
```
sudo chown sol:sol /mnt/ledgerdb
```
#### Third, we make partitions for the Accounts DB.

This step is a bit more involved, as the Solana documentation recommends creating a ramdisk partition, as well as adding a significant amount of swap to help with performance. Here are some [resources](https://medium.com/swlh/using-parted-mkfs-ext4-and-etc-fstab-to-prepare-an-additional-drive-6a4b7257ed5d) we found [helpful](https://dade2.net/kb/how-to-extend-filesystem-on-linux/) when researching how best to partition our devices. Check [agjell's guide](https://github.com/agjell/sol-tutorials/blob/master/setting-up-a-solana-devnet-validator.md) for more detailed instructions on RAM disk.

Accountsdb Partition:
```
sudo parted /dev/nvme2n1 mklabel gpt   
sudo blkid /dev/nvme2n1   
sudo parted /dev/nvme2n1 mkpart primary ext4 0% 60%   
sudo lsblk   
sudo mkfs.ext4 /dev/nvme2n1p1   
sudo mkdir /mnt/accountsdb   
sudo mount /dev/nvme2n1p1 /mnt/accountsdb   
sudo blkid /dev/nvme2n1p1
```
Use the UUID from the above and add it to your /etc/fstab file.
```
sudo nano /etc/fstab   
------> UUID=9dec2abf-6c82-4750-b6c6-fee190520e31 /mnt/accountsdb ext4 defaults,nofail 0 0  
```
Save & exit. Then unmount and remount the drive to make sure everything is configured properly.
```
sudo umount /mnt/accountsdb   
sudo mount -a   
ls /mnt/accountsdb 
```
You should see "lost+found" in the accountsdb directory. Now make sure the "sol" user has ownership over the directory where the drive is mounted.
```
sudo chown sol:sol /mnt/accountsdb 
```
Now its time to create the RAM disk. First, we make a directory inside our newly created partition.
```
sudo mkdir /mnt/accountsdb/ramdisk 
```
Then we edit /etc/fstab to create our 300GB tmpfs partition. Make sure to add it to the end of the file.
```
sudo nano /etc/fstab 
tmpfs /mnt/accountsdb/ramdisk tmpfs rw,noexec,nodev,nosuid,noatime,size=300G,user=sol 0 0
```
Save and exit. Then we'll attempt to mount all drives to assure it's properly configured.
```
sudo mount --all --verbose 
```
Make sure the owner of the directory is "sol".
```
sudo chown sol:sol /mnt/accountsdb/ramdisk 
```
Now we will create a 250GB swap to extend the capabilities of our RAM. Some info on [Swap Space](https://opensource.com/article/18/9/swap-space-linux-systems). 

To start, we'll list our currently existing swaps, then remove them to clear the way for our new swap.
```
sudo swapon --show 
sudo swapoff /swap.img ## make sure this is consistent with your machine
sudo rm /swapfile.img 
sudo sed --in-place '/swap.img/d' /etc/fstab 
```
Then we'll partition a new device that the swap will live on.

/mnt/swap Partition:
```
sudo parted /dev/nvme2n1 mkpart primary ext4 60% 100%    
sudo lsblk 
sudo mkfs.ext4 /dev/nvme2n1p2 
sudo mkdir /mnt/swap    
sudo mount /dev/nvme2n1p2 /mnt/swap    
sudo blkid /dev/nvme2n1p2    
```
Use the UUID from the above and add it to your /etc/fstab file.
```
sudo nano /etc/fstab
------> UUID=7adc89ed-a054-483f-857e-e1baff18e1cf /mnt/swap ext4 defaults,nofail 0 0 
```
Save & exit. Then unmount and remount the drive to make sure everything is configured properly.
```
sudo umount /mnt/swap  
sudo mount -a    
ls /mnt/swap
```
You should see "lost+found" in the /mnt/swap directory. This will stay under the root user.

Finally, we'll create the swapfile. Since it is a large file, it can take several minutes to complete, so don't be surprised when the command hangs.
```
sudo dd if=/dev/nvme2n1p2 of=/swapfile bs=1M count=250K
```
Now we need to set the proper permissions and create the swapfile.
```
sudo chmod 0600 /swapfile 
sudo mkswap /swapfile 
```
Then we add the changes to our /etc/fstab to persist them. 
```
echo '/swapfile none swap sw 0 0' | sudo tee --append /etc/fstab > /dev/null 
```
Next we decrease the swapiness to make our 256GB RAM the primary target. More info [here](https://access.redhat.com/solutions/103833).
```
echo 'vm.swappiness=1' | sudo tee --append /etc/sysctl.conf > /dev/null 
sudo sysctl --load 
```
And finish by turning our swap on and confirming it has been correctly configured.
```
sudo swapon --all --verbose 
sudo swapon --show 
sudo free -h 
```
Phew! Deep breath. Now let's keep going.

## Install Solana

You can either install the prebuilt binaries or build your own binaries. This tutorial contains instructions to install the prebuilt binaries. However, there is a tutorial on building binaries from source, which you can find [here](https://github.com/agjell/sol-tutorials/blob/master/building-solana-from-source.md).
```
! Perform all tasks as user “sol”
```
First download the binaries.
```
sh -c "$(curl -sSfL https://release.solana.com/v1.10.8/install)"
# take note of path, add to ~/.profile of non-root sudo user also
### ex / export PATH="/home/sol/.local/share/solana/install/active_release/bin:$PATH"
```
Configure for testnet.
```
solana config set --url testnet
#take note of output
Config File: /home/sol/.config/solana/cli/config.yml 
RPC URL: https://api.testnet.solana.com 
WebSocket URL: wss://api.testnet.solana.com/ (computed) 
Keypair Path: /home/sol/.config/solana/id.json 
Commitment: confirmed
```


## Create & Configure Validator Accounts

The account structure in Solana can feel complicated at first blush, so our hope is to simplify it here focusing on the perspective of the validator. There are several accounts you will need to operate your validator. Some are necessarily "hot" wallets, with their keypairs stored on the device, while others should absolutely NOT be stored on your device. These keys should be kept in a safe place following private key management best practices.

Account 1: Authorized Withdrawer

Do NOT store this keypair on your device. This wallet is the key to our validator, it controls all accounts, allows for key rotation, comission changes, and withdrawals of rewards.
```
solana-keygen new --no-outfile
solana airdrop 1 5Kd2YFmsSFctVxBTuPfupA8oCAs42kbBmFBZrYKQ8jKm (use your pubkey)
```

Account 2: Validator Identity

This account pays for transaction fees and vote costs. Since it is consistently signing transactions, it is necessarily a "hot" wallet, and thus we have elected to keep the keypair on our device. It is our understanding that it is best to keep no more than a few days worth of vote costs and tx fees, making sure to set alerts and top up as it approaches a "minimum balance" you can define based on your comfort level (2.5 Sol for Throbac).
```
solana-keygen new --outfile ~/validator-keypair.json
solana config set --keypair ~/validator-keypair.json
```
Make a transfer from your Authorized Withdrawer to your Validator Identitiy. You will have to enter your keypair and passphrase for the Authorized Withdrawer, so type carefully! (You can fund it with airdrops too).
```
solana transfer --allow-unfunded-recipient \ 
  --fee-payer ASK \ 
  --from ASK ~/validator-keypair.json .5
```
Account 3: Vote Account

This account submits your validator's vote on chain. It is also the public key that folks will use when they delegate to your validator. It is also necessarily a "hot" wallet and we have elected to keep the keypair on our device.
```
solana-keygen new --outfile ~/vote-account-keypair.json
```
Here's the format for designating a Vote Account:
```
solana create-vote-account <ACCOUNT_KEYPAIR> <IDENTITY_KEYPAIR> <WITHDRAWER_PUBKEY> --commission <PERCENTAGE> --config <FILEPATH>
# and our example
solana create-vote-account ~/vote-account-keypair.json ~/validator-keypair.json 5Kd2YFmsSFctVxBTuPfupA8oCAs42kbBmFBZrYKQ8jKm --commission 10 \
--fee-payer ASK
```

## Create Startup Script & System Services

Make sure you're logged in as user "Sol".
```
su - sol
nano ~/start-validator.sh
# start-validator.sh script:
#!/bin/bash 
exec solana-validator \ 
 --entrypoint entrypoint.testnet.solana.com:8001 \ 
 --entrypoint entrypoint2.testnet.solana.com:8001 \ 
 --entrypoint entrypoint3.testnet.solana.com:8001 \
 --known-validator BwVDYeT9sUadojNc1JeFz66FktsUHGiFQa7LeuNxSBdh \ 
 --known-validator 5D1fNXzvv5NjV1ysLjirC4WY92RNsVH18vjmcszZd8on \ 
 --known-validator Ft5fbkqNa76vnsjYNwjDZUXoTWpP7VYm3mtsaQckQADN \ 
 --known-validator eoKpUABi59aT4rR9HGS3LcMecfut9x7zJyodWWP43YQ \ 
 --known-validator 2eCPrXeWo9Cg79cK6eyJdaCoiMDKJMPq7sAhXSP3spQk \
 --expected-genesis-hash 4uhcVJyU9pJkvQyS88uRDiswHXSCkY3zQawwpjk2NsNY \
 --dynamic-port-range 8000-8020 \ 
 --rpc-port 8899 \
 --only-known-rpc \
 --wal-recovery-mode skip_any_corrupted_record \ 
 --identity ~/validator-keypair.json \ 
 --vote-account ~/vote-account-keypair.json \ 
 --log ~/log/validator.log \ 
 --accounts /mnt/accountsdb/ramdisk \ 
 --ledger /mnt/ledgerdb/ledger \ 
 --limit-ledger-size 100000000 \
 --no-poh-speed-test \
 --skip-poh-verify \  
 --no-port-check
 ```
Next make the start-validator.sh script executable.
```
chmod +x ~/start-validator.sh
```
And create a directory for the log file as referenced above.
```
mkdir ~/log
```
Now for the system services.

Log in as a non-root user with sudo privileges.
```
su - "YOUR-USER"
```
Create our validator.service.
```
sudo nano /etc/systemd/system/validator.service
# validator.service file
[Unit] 
Description=Solana Validator 
After=network.target 
Wants=systuner.service 
StartLimitIntervalSec=0 
[Service] 
Type=simple 
Restart=on-failure 
RestartSec=1 
LimitNOFILE=1000000 
LogRateLimitIntervalSec=0 
User=sol 
Environment=PATH=/bin:/usr/bin:/home/sol/.local/share/solana/install/active_release/bin 
Environment=SOLANA_METRICS_CONFIG=host=https://metrics.solana.com:8086,db=testnet,u=scratch_writer,p=topsecret 
ExecStart=/home/sol/start-validator.sh 
[Install] 
WantedBy=multi-user.target
```
Save and exit. Now for our system tuner service.
```
sudo nano /etc/systemd/system/systuner.service
# systuner.service
[Unit] 
Description=Solana System Tuner 
After=network.target 
[Service] 
Type=simple 
Restart=on-failure 
RestartSec=1 
LogRateLimitIntervalSec=0 
ExecStart=/home/sol/.local/share/solana/install/active_release/bin/solana-sys-tuner --user sol 
[Install] 
WantedBy=multi-user.target
```
Releoad the systemctl daemon
```
sudo systemctl daemon-reload
```
## Set up Log Rotation

! Perform as user with root privileges

“Logrotate” takes care of the log rotation for us. It automatically creates a new log at 00:00 and deletes the excess. To activate logrotate for the validator log we need to create a configuration file for it:
```
sudo nano /etc/logrotate.d/solana
```
And paste the following:
```
/home/sol/log/validator.log {
  su sol sol
  daily
  rotate 7
  missingok
  postrotate
    systemctl kill -s USR1 validator.service
  endscript
}
```
To load the new configuration I need to restart the logrotate service:
```
sudo systemctl restart logrotate
```

## Start the Validator

Optional: After completing all the steps above you can reboot you server (sudo reboot to verify fstab has been set up correctly. Check [agjell's guide](https://github.com/agjell/sol-tutorials/blob/master/setting-up-a-solana-devnet-validator.md#starting-the-validator) on this.

System Tuner first, as our Validator service will use this.
```
sudo systemctl enable --now systuner.service
sudo systemctl status systuner.service
```
Then the Validator Service.
```
sudo systemctl enable --now validator.service
sudo systemctl status validator.service
```
Some useful commands to see if everything is working (supposing you see "Active" on both status checks above):
```
# as non-root user with sudo privileges
sudo journalctl -u validator.service -f 
sudo journalctl -u systuner.service -f 
```
```
# as "sol" user
solana-validator --ledger /mnt/ledgerdb/ledger monitor 
grep --ignore-case --extended-regexp 'error|warn' ~/log/validator.log
solana catchup ~/validator-keypair.json --our-localhost
```
If you're more than 3-4K blocks behind after running "solana catchup" it's likely your validator will not catch up to the chain. You can restart your validator, look in the solana discord for new known validators for testnet, and experiment with different snapshot setups to optimize.

## Staking

To delegate stake to your vote account and thus begin actual validation, you can self-delegate using the below guide.

We'll need a few acounts to pull this off. 

- Account 1 - Stake Authority Account
- Account 2 - Stake Account
- Account 3 - Withdrawal Authority Account (optional, but encouraged)

Best practice would say its good to not overlap your Authorized Withdrawer with your Stake & Withdrawal Authority accounts, the accounts that will control your Stake Account. For this example we will be using the same Stake & Withrawal Authority accounts.

Account 1: Stake Authority Account

This account keypair should NOT be stored on your device, as it controls the withdrawal account as well as actions that take place in the stake account.
```
solana-keygen new --no-outfile
solana airdrop 1 EYyixhf94DmteFJP3Yr6bZpb3x7wjJUxcpmCsiJKjcNT (pub key of generated account)
```
Account 2: Stake Account

This is the account that will contain the delegated sol and generate rewards. Its a placeholder account and does not have the ability to take any action itself, so it is safe to store the keypair on your device. All "ASK" keypairs we will manually enter the details for our Stake Authority account.
```
solana-keygen new --no-passphrase -o stake-account.json
```
To create the stake account:
```
# format -- solana create-stake-account [FLAGS] [OPTIONS] <STAKE_ACCOUNT_KEYPAIR> <AMOUNT>
solana create-stake-account --from ASK stake-account.json 1.5 \   
    --stake-authority ASK --withdraw-authority ASK \   
    --fee-payer ASK
```
Check your account details:
```
solana stake-account stake-account.json
```
To delegate your stake (make sure to select your previously created vote account):
```
# format -- solana delegate-stake [FLAGS] [OPTIONS] <STAKE_ACCOUNT_ADDRESS> <VOTE_ACCOUNT_ADDRESS>
solana delegate-stake --stake-authority ASK stake-account.json 8Jkrkcd6UCYV1PEDofdVgw7DGHwfn8CU7MeH7tfCoqu1 \ 
--fee-payer ASK
```
Check your stake account details again. Remember, there is a warm up and cool down period to staking. Once submitted, however, you should see your validator begin to submit votes + txs.
```
solana stake-account stake-account.json
```

## Monitoring

We'll be using the [solana mission control set up](https://github.com/Chainflow/solana-mission-control/blob/main/docs/prereq-manual.md), with a few tweaks. Shout out to Chris Remus @ Chainflow for his work on this.

Make sure you're logged in as a non-root user with Sudo privileges.

First, we'll need to install Go.
```
cd ~
curl -OL https://go.dev/dl/go1.18.linux-amd64.tar.gz
sha256sum go1.18.linux-amd64.tar.gz
sudo tar -C /usr/local -xvf go1.18.linux-amd64.tar.gz
```
Then we'll add the gobin path to our ~/.profile so we can enter our commands from anywhere.
```
sudo nano ~/.profile
export PATH=$PATH:/usr/local/go/bin
export PATH=$PATH:/home/USERNAME/go/bin
source ~/.profile
```
Second, we'll install Prometheus. Throbac uses Grafana Cloud, so our implementation includes a "remote write" condition that exports all data to our grafana cloud instance. For info on Grafana Cloud, click [here](https://grafana.com/products/cloud/).
```
wget https://github.com/prometheus/prometheus/releases/download/v2.34.0/prometheus-2.34.0.linux-amd64.tar.gz
tar -xvf prometheus-2.34.0.linux-amd64.tar.gz
```
Next we'll copy the config files to our gobin and the configuration prometheus.yml to our desired directory.
```
mkdir monitoring
cp prometheus-2.34.0.linux-amd64/prometheus /home/USERNAME/go/bin
cp prometheus-2.34.0.linux-amd64/prometheus.yml /home/USERNAME/monitoring
cd monitoring
```
Then we'll edit our prometheus.yml to include our remote write, as well as node_exporter and the solana-mc service, both of which we will create momentarily.
```
nano prometheus.yml 
# edits (carefull of spacing and line breaks)
  - job_name: 'node_exporter' 
      static_configs: 
      - targets: ['localhost:9100'] 
  - job_name: 'solana' 
    static_configs: 
      - targets: ['localhost:1234'] 
# You'll need to create a grafana account, create a monitoring stack, and then look at prometheus details in your stack. 
# From there you can create a remote write username + pw. All this can be accomlished through the gui.
remote_write:  
- url: https://prometheus-prod-10-prod-us-central-0.grafana.net/api/prom/push  
  basic_auth:  
    username: USE GRAFANA CLOUD CREATED USERNAME  
    password: USE GRAFANA CLOUD CREATED PASSWORD
```
Now to create the prometheus.service.
```
sudo nano /lib/systemd/system/prometheus.service
#copy & paste
[Unit] 
Description=Prometheus 
After=network-online.target 
[Service] 
Type=simple 
ExecStart=/home/USERNAME/go/bin/prometheus --config.file=/home/throbackevin/monitoring/prometheus.yml 
Restart=always 
RestartSec=3 
LimitNOFILE=4096 
[Install] 
WantedBy=multi-user.target
```
Save & exit. Reload the daemon and start the service.
```
sudo systemctl daemon-reload 
sudo systemctl enable prometheus.service 
sudo systemctl start prometheus.service
sudo systemctl status prometheus.service
sudo journalctl -u prometheus.service -f
```
Third, we'll install & configure node_exporter.
```
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.3.1.linux-amd64.tar.gz
```
Copy it to the gobin.
```
cp node_exporter-1.3.1.linux-amd64/node_exporter ~/go/bin/node_exporter
```
Create the service file.
```
sudo nano /lib/systemd/system/node_exporter.service
# copy & paste
[Unit] 
Description=Node_exporter 
After=network-online.target 
[Service] 
Type=simple 
ExecStart=/home/USERNAME/go/bin/node_exporter 
Restart=always 
RestartSec=3 
LimitNOFILE=4096 
[Install] 
WantedBy=multi-user.target
```
Start the service.
```
sudo systemctl daemon-reload
sudo systemctl enable node_exporter.service
sudo systemctl start node_exporter.service
```
Finally, Solana-mc. This is a service that will export information about our validator to Grafana.

Clone the solana mission control repo and cd into it.
```
git clone https://github.com/Chainflow/solana-mission-control
cd solana-mission-control
```
Copy the config.toml and edit it to your liking.
```
cp example.config.toml config.toml
nano config.toml
#change "telegram alerts" to "false" -- we run all our alerts through grafana cloud
```
Export the Solana Binary.
```
export SOLANA_BINARY_PATH="/home/sol/.local/share/solana/install/active_release/bin"
```
Copy our config.toml to our home directory so it can be picked up by the service.
```
cp config.toml /home/USERNAME/config.toml
```
Build the solana-mc script in our gobin.
```
go build -o /home/USERNAME/go/bin/solana-mc
```
Create the solana-mc service file.
```
sudo nano /lib/systemd/system/solana_mc.service
#copy&paste
[Unit] 
Description=Solana-mc 
After=network-online.target 
[Service] 
User=USERNAME
Environment="SOLANA_BINARY_PATH=/home/sol/.local/share/solana/install/active_release/bin/solana" 
ExecStart=/home/USERNAME/go/bin/solana-mc 
WorkingDirectory=/home/USERNAME
Restart=always 
RestartSec=3 
LimitNOFILE=4096 
[Install] 
WantedBy=multi-user.target
```
Reload the daemon and start the service.
```
sudo systemctl daemon-reload
sudo systemctl enable solana_mc.service
sudo systemctl start solana_mc.service
sudo systemctl status solana_mc.service
sudo journalctl -u solana_mc.service -f
```
Once these are all successfully up, you can log into your Grafana Cloud and add the following dashboards:
    14738: Validator monitoring metrics dashboard. 
    14739: Summary dashboard. 
    13445: System monitoring metrics dashboard.

Alert Rules will be uploaded as an additional file to this directory. 
