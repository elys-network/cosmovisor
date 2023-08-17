# Running an Elys Network node

## Purpose of this Guide

The below tutorial describes how to set up and operate a node for Elys Network (be it a validator node or not).<br>
It teaches how to leverage **Cosmovisor** to automate the upgrades and achieve an appropriate quality of service.<br>
It also focuses on implementing a basic level of security by following a number of keys principles and good practices in Unix system administration.

If you have no prior experience, it is **strongly** advised to start by running a node on a testnet: as there is no actual money involved, it is appropriate for experimenting, learning and possibly crashing everything without any actual consequences.

You can get support and information in our official [Discord](https://t.co/YTlBEofNOt).

# **1. Hardware Requirements**

At this time, an Elys node runs fine on a machine with a **4-core CPU and 8G memory**. Going forward, better specs may be required to accomodate for faster block times and increased on-chain activity.<br>
Disk space depends on the purpose of the node (validator, archive...) and some configuration items; it is recommended to have at least **100G** of available space at disposal.<br>
Most importantly, the disk type should be **NVMe**: this is the fastest option in terms of I/O (= writing and reading speeds). Regular SSD is likely to be too slow.

⚠ It is paramount that the disk does not cause a bottleneck, preventing the node from processing its data fast enough: this will cause it to skip blocks and affect its uptime.

Lastly, and given the large amount of data that a node sends and receives, a fast internet connection is required: fiber optics recommended, 500MB upload/download speed.

A node can run on multiple types of machines: Cloud VM, dedicated server/baremetal, or even a home server as long as the internet connection complies with the requirements.

# **2. Server Setup**

Nodes typically run on Linux, and the below instructions are specifically meant for Debian-based systems like Ubuntu. The commands should be adjusted for other distributions.

## ***2.1 Logging into the server***

When deploying a server, you will often log in with the root account by default, which is <u>not recommended</u> in production. The below steps will describe how to create a new user with less privileges and prevent the ability to log in as root.

**The below instructions are simplified, and implement basic security practices**. For more details you can check [this page](https://www.cyberciti.biz/faq/how-to-set-up-ssh-keys-on-linux-unix/), or [this one](https://www.ssh.com/academy/ssh/putty/windows/puttygen) if you use Windows and PuTTY.

- Log into the server, using the credentials that were provided when deploying it (either username/password, or username/ssh key):
  
  <code>$ ssh root@SERVER_IP</code>
  
- **As root**, create a new user, which we will call **elys**:<br>

  <code># adduser elys --system --group --shell /bin/bash</code><br>
  <code># passwd elys</code>
  
- Set a password, preferably a strong one. This is the password you will need to type when requesting elevated privileges later.
  
- On the computer that you will use to log into the node server — assuming it runs Linux —, type the following:<br>

  <code>$ ssh-keygen -t ed25519</code>
  
- Hit Enter on the next prompts — set up your email, passphrase, etc. or not as you see fit.<br>

  <code>$ ssh-copy-id -i ~/.ssh/id_ed25519 validator@server_ip</code>
  
- Type the password you set for user **elys**, and the key will be transferred over.
  
- Now, you can verify that everything works as expected by logging in as user **elys**: this time, no password should be required.<br>

  <code>$ ssh elys@SERVER_IP</code>

- If it worked, log out and in again as **root**:

  <code>$ ssh root@SERVER_IP</code>

- We will now allow the **elys** to obtain superuser privileges, i.e., to become root.<br>Then, we'll disable the root login altogether.

  <code># passwd -d root</code>  --> this removes the password for the root user.

  ⚠ One important detail is that if you misplace or overwrite your id_ed25519.pub, you’ll be locked out of the server. And if someone else has it and knows its IP, they’ll have access to said server.

  <code># echo “elys ALL=(ALL:ALL) ALL” >> /etc/sudoers.d/10-elys</code> --> this allows the **elys** user to acquire elevated privileges.

  Verify that it works by switching to the **elys** user with <code># su elys</code>, then by requesting the superuser privileges:

  <code>$ sudo su</code>

  You will be prompted to enter the password you defined earlier, and if it works as intended you will be root again -- you can verify by checking the shell prompt, which should start with **#**.

- Next up, we will prevent the **root** user from logging into the server. It will therefore only be accessible from within the **elys** user — if anyone tries to brute force the root password, he will never succeed as the server will simply not allow this user to connect remotely:
 
  <code># vi /etc/ssh/sshd_config</code> (or use <code>nano</code> or whatever your favorite text editor is if <code>vi</code> is not installed yet).

- Edit the line “PermitRootLogin” — uncomment it by removing the initial # and switch the value to “no”:
 
  <code>PermitRootLogin no</code>

- It is recommended to also disable the password authentication with:
 
  <code>PasswordAuthentication no</code> --> if someone gets hold of the **elys** user password, he won't be able to access the servcer anyway.

- Save and exit the file, then restart the ssh service:

  <code># systemctl restart sshd</code>

- Now if you attempt to log into the server as root, you will remain locked out.

## ***2.2 Preparing the server***

We need to prepare the server so that it has the ability to run the node, by installing required packages. As **root**, run:

<code># apt update && apt upgrade -y && apt install -y build-essential glances jq vim git && apt autoremove -y</code>

Most distributions ship with all the other important packages preinstalled. If some are actually missing, the errors are usually quite explicit so you’ll just need to install them.

<code>Go</code> is also required.<br>
⚠ The needed Go version can change over time, and this needs to be verified to select the correct one. Currently, Go version <code>1.20.x</code> is needed. 

*See [here](https://go.dev/doc/install) how to download and install.*

Once Go is installed, we need to allow the **elys** user to run it, by telling him where it is located, typically under <code>/usr/local/go/bin</code>:

<code># echo -e ‘cd ~ \nexport PATH=/usr/local/go/bin:~/go/bin:$PATH’ >> /home/elys/.bashrc</code>

## ***2.3 Firewall***

The node must obviously have access to the internet to run, and external nodes should also be able to connect to it. However, not all interfaces should be remotely accessible.<br>
We will therefore configure the server firewall to that aim.<br>

Typically, only 2 ports should allow inbound connections: SSH (22) and the node's peer-to-peer port, which is 26656 by default.

⚠ If you are using a cloud VM, the firewall is often managed from the provider interface itself. Refer to its documentation.

If you are running a baremetal server, this is done through <code>ufw</code>, as **root**:

<code>ufw enable</code> ⚠ Don't log out now.<br>
<code>ufw allow 22</code> ⚠ This opens port 22 to all remote hosts. You may want to restrict to known IPs if you are confident you'll never need to log in from any other network.<br>
<code>ufw allow 26656</code>

If you plan on providing RPC, GRPC or REST services, you will also need to allow 26657, 9090 and/or 1317, but you'll probably also need to set up a rate limiter to prevent abuse.<br>
Heavy queries to these interfaces can put a significant burden on the server and if it is validator node, cause it to miss blocks. It is recommended to keep these ports closed in this case.

# **3. Installing and running the node**

Now that everything is set, we can deploy the node, as the **elys** user.<br>

- Let us start by cloning the relevant GitHub repository:

<code>$ git clone https://github.com/elys-network/elys.git</code><br>
<code>$ cd elys</code>

- We need to install the correct version -- this needs to be checked in the documentation of Discord. Currently, it's <code>v0.9.0</code>:

  <code>git checkout v0.9.0</code>

- Now we can install the node:

  <code>make install</code>

⚠ If the above command terminated with an error, carefully read the output as it explains what went wrong and give you hints on how to resolve the issue.<br>
If it is related to a missing package, you need to acquire the root privileges with <code>sudo su</code> and install it.<br>
Once done, type <code>exit</code> or hit <code>Ctrl+d</code> to return to the **elys** user, navigate to the <code>elys</code> directory and attempt the installation again.

## **3.1 Installing Cosmovisor**

Cosmovisor is a tool developed by the Cosmos SDK team that allows nodes to be upgraded in a seamless and nearly completely automated fashion.<br>
As it is important to upgrade the node on time to avoid being jailed, it is highly recommended to use Cosmovisor.

The full documentation is [here](https://docs.cosmos.network/v0.45/run-node/cosmovisor.html) for reference.

Let’s start with installing it:

<code>$ go install github.com/cosmos/cosmos-sdk/cosmovisor/cmd/cosmovisor@latest</code>

Now we’ll create the relevant directories:

<code>$ mkdir -p ~/.elys/cosmovisor/genesis/bin</code>
<code>$ mkdir -p ~/.elys/cosmovisor/upgrades</code>

If you going to sync your node from scratch you have to [build](https://github.com/svv28/cosmovisor/blob/main/tutorial_validator_setup.md#3-installing-and-running-the-node) Elys node with <code>v0.1.0</code> tag. And then copy the genesis binary to that first directory:

<code>$ cp ~/go/bin/elysd ~/.elys/cosmovisor/genesis/bin/</code>

**It should be noted** that if you decide to use [node's data balse snapsot](https://github.com/svv28/cosmovisor/blob/main/tutorial_validator_setup.md#using-a-quicksync-snapshot) or use a [Statesync](https://github.com/svv28/cosmovisor/blob/main/tutorial_validator_setup.md#using-statesync) you have to use latest binaries (currently, it's <code>v0.9.0</code>), which has to be placed into cosmovisor `upgrades` folder.


For now, that’s it. Let’s continue with the setup.

## **3.2 Initializing the node**

This will create the relevant directories and files of the node.

<code>$ elysd init "moniker" --chain-id elystestnet-1</code>

The "moniker" can be whatever you want, it is of no importance. The chain id, however, must be correct and adjusted to the network that you want to join (in this example, the first Elys testnet).<br>
The following will be created under <code>/home/elys/</code>:

<code>.elys 
├── config 
│ ├── addrbook.json 
│ ├── app.toml 
│ ├── client.toml 
│ ├── config.toml 
│ ├── genesis.json 
│ ├── node_key.json 
│ └── priv_validator_key.json 
└── data 
 └── priv_validator_state.json</code>

Before moving on to the creation of the validator, the node needs to be started and synchronized.<br>
In most cases, the chain will have started long ago and catching up from block 1 would take ages.<br>
A couple mechanisms exist to bootstrap a node from a recent height, allowing it to quickly get in sync.<br>
But first off, some configuration is required so that the node can connect to the rest of the chain.<br>
With your favorite text editor, open <code>$ .elys/config/config.toml</code> and update the following items (we’ll assume here that you will use the default ports 26656 and 26657):

  **In the [p2p] section**<br>
  <code>external_address = “your_actual_ip:26656”</code> → this allows other nodes to connect to yours. You can change the default ports (26656 for the p2p, 26657 for the rpc), but if so, make sure to change them everywhere in that file.<br>
  <code>seeds = “…”</code>  → find one or several seeds (in the documentation or the chain’s Discord / Telegram) and add them here, comma-separated. Otherwise you can find persistent_peers and list them in the line that follows, but seeds are preferable.

  You can also update the below limits so that the node can connect to more peers:<br>
  <code>max_num_inbound_peers = 60</code><br>
  <code>max_num_outbound_peers = 40</code>

  **In the [tx_index] section**<br>
  If the node is purely a validator, the index is not required. Disabling it helps reduce the disk usage. If you intend to run an archive node or query historical data, keep it as it is.<br>
  <code>indexer = “null”</code>

  Now edit <code>$ .elys/config/app.toml</code>:

  **At the top of the file**<br>
  *Usually it is required to specify something here:*
  <code>minimum-gas-prices = “…”</code> → check the documentation, Discord or Telegram for the correct value.
  <code>pruning = “default”</code> → you can customize this, including setting “everything”, but unless you know what you are doing it is best left as it is.

  **in the [grpc] section**<br>
  If you won't run an IBC relayer or provide a gRPC service, you probably won’t have much use for this. Otherwise, keep it enabled and keep in mind you'll need to configure the firewall.<br>
  <code>enable = false</code>

  **In the [state-sync] section**<br>
  If you plan on providing a RPC to allow other operators to statesync from your node, you can configure this to make a snapshot every 2000 blocks for example. Else, leave it to 0.<br>
  <code>snapshot-interval = 0</code>

  Now we’ll create a systemd service `elys.service` to control the node via cosmovisor. First, create the local service file by executing the following:
```
tee $HOME/elys.service > /dev/null <<EOF      
[Unit]
Description=Elys Network node
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.elys"
Environment="DAEMON_NAME=elysd"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF
```
Second, we'll create a symlink of this local file to the `/etc/systemd/system` using the **root** access:
```
sudo ln -s $HOME/elys.service /etc/systemd/system
```
And lastly, enable `elys` service file:
```
sudo systemctl daemon-reload
sudo systemctl enable elys.service
```

The *Environment* items instruct Cosmovisor how to perform. They are all self-explanatory except maybe these two items:

<code>Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"</code>

This means that it won’t be allowed to automatically download the new upgrades. This is out of caution: if an issue occurs when downloading the binary, Cosmovisor would restart anyway.<br> 
Another possibility is if somehow the chain repository was hacked and a malicious binary pushed, it would avoid that the node automatically starts using it.<br>
While not a likely situation, it is best to keep it to “false”.

<code>Environment="UNSAFE_SKIP_BACKUP=true"</code>

By default, Cosmovisor makes a full backup of the data folder when an upgrade takes place — if the upgrade fails and the data becomes corrupted, it allows to restore the backup and attempt the process again.
However, this can take a long time and use up a lot of disk space.<br>
This flag disables this backup.

## **3.3 Upgrading**

Chains are regularly upgraded to implement new features or resolve issues. The upgrades **must** be applied at the right time, i.e. the correct height, and in a timely manner to maintain a good uptime.<br>
This is where Cosmovisor is useful, although it must be set up to apply said upgrade.

For an upgrade called v1.0.0 (just an example. Ensure to use the actual upgrade tag, and always replace the user and binary path accordingly.):

    $ cd ~/elys
    $ git fetch
    $ git checkout v1.0.0
    $ make build
    
    $ mkdir -p ~/.elys/cosmovisor/upgrades/v1.0.0/bin
    $ cp ~/elys/build/elysd ~/.elys/cosmovisor/upgrades/v1.0.0/bin/

And that’s it. When the height is reached, Cosmovisor will automatically swap the binary and restart.

This is particularly useful when the upgrade time isn’t convenient, e.g. occurring in the middle of the night — although it’s always best to be around in case something goes wrong.

 ## **3.4 Running the node**

Now we can prepare the node to allow it to start and quickly synchronize with the rest of the chain.

 ###  Using a Quicksync Snapshot

This is an archive containing the data of a node up to a certain height, which can be downloaded and extracted in <code>$ .elys/</code>, replacing the current data folder.
Many teams provide such snapshots -- check for availability in the documentation or Discord.
Follow the instructions to deploy it, paying attention to the presence of the wasm folder: it **has** to be there.

There are multiple sizes available, depending on the start and end blocks in the snapshots and their type (full, archive, pruned, containing the index or not).
Once deployed, the node will start catching up from the last block of the snapshot, which should ideally be recent.

 ###  Using Statesync

 This is a little more complicated. You need to find a RPC node and specify some details in <code>config.toml</code>.<br>
 In short, these RPCs are nodes that are configured to take regular snapshots of their data — not the same kind than above — , and serve them to the network.

   - **The RPC**: basically an IP and a port. There are plenty, or very few, depending on the network. You can ask in Discord or Telegram for the information.
   - **Trusted height and hash**: once this RPC is obtained, you can query it to get the current height:
   
       <code>curl rpc_IP:rpc_PORT/status | jq .result.sync_info.latest_block_height</code>

   - With this current height, for example **15427931**, you can get the trusted height and its trusted hash — basically, the oldest block from which your node will accept to restore a snapshot from the RPC.<br>
Typically we take a height that is 2000 to 5000 blocks below the current one, in this case say **15426000**. Now we can get its hash:

       <code>curl rpc_IP:rpc_PORT/status | jq -r .result.block_id.hash</code>

   - This returns something like “_D56FF01D27CD36D24EA97F069BCA5F81E9EE7AA8401B109CDD990E6EB7B5C232_”
   - Now onto editing <code>.elys/config/config.toml</code>, in the [statesync] section:

     <code>enable = true</code><br>
     <code>rpc_servers = “http://RPC_ip:RPC_port, http://RPC_ip:RPC_port”</code>

   - ⚠ You need to include “http://” (or “https://” if relevant). You also must specify 2 RPC nodes — but it can be the same one twice. You also must specify the port.

     <code>trust_hash = “D56FF01D27CD36D24EA97F069BCA5F81E9EE7AA8401B109CDD990E6EB7B5C232” 
      trust_period = “168h0m0s”
      trust_height = 15426000</code>

   - It is also recommended to add the RPC peer to the “persistent_peers” item in config.toml. This peer is in the form “151192fd3ac51c0a9b1fae3a64ae7cbd3df161c@rpc_IP:rpc_P2P_PORT”.<br>
    While there is no real rule here, without it the node can take a very long time to discover an available snapshot, or even fail doing so.<br>
    Save and exit the file, then start the node as **root** with:<br>

      <code># systemctl start elys.service</code>

   - You can monitor its activity with:

      <code># journalctl -fu elys</code>

   - which should output a lot of text. If everything works fine you will see something like:

      <code>level=info msg=”Discovered new snapshot” format=1 hash=”[144 138 2 186 11 177 28 240 34 191 141 224 80 158 55 79 178 239 112 159 36 167 66 90 204 243</code>

   - Followed by:

      <code>level=info msg=”Applied snapshot chunk to ABCI app” chunk=0 format=1 height=32600000 module=statesync total=27</code>

   - And once the node has started syncing:

      <code>level=info msg=”committed state” app_hash=8BD1ED39636380DC4D97A27218F10EA9CB2E4AE80D242956F6816517B2F8E644 height=32610006 module=state num_txs=2</code>

   - The process can end in error, for example:

  Something saying “_Tree must be empty_” → You may have attempted this earlier. Remove everything in <code>.gaia/data/</code> except _priv_validator_state.json_.

  “_level=info msg=”failed to fetch and verify app hash” err=”failed to obtain the header at height #34136796: post failed: Post \”http://xx.xx.xx.xx:2
6657\”: context deadline exceeded” module=statesync_” → restart the statesync configuration, selecting an earlier trusted height.

   - You can check the status of the node by running:

      <code>$ curl localhost:26657/status | jq</code>

   - Which should output something like:

         {
          "jsonrpc": "2.0",
          "id": -1,
          "result": {
            "node_info": {
              "protocol_version": {
                "p2p": "8",
                "block": "11",
                "app": "0"
              },
              "id": "cdf9ae8529aa00e6e6703b28f3dcfdd37e07b27c",
              "listen_addr": "tcp://0.0.0.0:26656",
              "network": "elystestnet-1",
              "version": "0.37.2",
              "channels": "40202122233038606100",
              "moniker": "moniker",
              "other": {
                "tx_index": "on",
                "rpc_address": "tcp://0.0.0.0:26657"
              }
            },
            "sync_info": {
              "latest_block_hash": "5EFF2CE212738437B0065020226271FFBB9E069F354B74993FF8FAB187345515",
              "latest_app_hash": "E7B6925B166EBDE40C335AA38713FD6A89C2A297C5269F73BC26814306925D5D",
              "latest_block_height": "2150913",
              "latest_block_time": "2023-08-13T11:09:14.176943617Z",
              "earliest_block_hash": "F90D393BBD53F3044C26406221DD4BC1DEB6EA97F836129E7F5688315EBB94C6",
              "earliest_app_hash": "E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855",
              "earliest_block_height": "1",
              "earliest_block_time": "2023-04-10T18:42:34Z",
              "catching_up": false 
            },
            "validator_info": {
              "address": "0FA26CE059A87FFF07EE750E057460C83BE52E48",
              "pub_key": {
                "type": "tendermint/PubKeyEd25519",
                "value": "fobiFP2mhB3xoB4uZkte5mo6yegTka7+o55tidSiNeY="
              },
              "voting_power": "1299999"
              }
            }
          }
     
 <code>"catching_up": false</code> means that the node is synchronized. If you intend to run a non-validator node, then you are all set.

 Otherwise, you can move on to the next step to become a validator.
 
## **3.5 Running a validator node:**

In this case, there are specific items that are essential.

  - ⚠ priv_validator_key.json is the most important file here. It identifies your validator node and should be safely copied and stored elsewhere.
  - ⚠ If you lose it, you lose your validator. There is no coming back from this.
  - ⚠ Lastly, never, ever run two nodes with the same priv_validator_key.json simultaneously. This will cause you to double-sign and be tombstoned (= permanently jailed), in addition to being slashed of typically 5% of your total stake. Delegators will NOT be happy.

### **3.5.1 Creating the validator key**

Which is essentially creating a wallet that will be linked to the validator and will be used for all operations — voting, withdrawing the rewards, etc.
Let’s call this key *validator*.

<code>$ elysd keys add validator</code>

You will be prompted to set a password (that will be required each time the key is accessed through a vote, withdrawal, etc.), and the output will display the seed phrase (mnemonic) for this wallet.
This seed phrase will never appear again so **make sure you save it somewhere safe**.

Note: if you are running a testnet node, you can skip the password step by specifying *--keyring-backend test* instead. That way, no password will be asked for any operation.<br>
If you select that, you will need to append this flag to each subsequent commands, otherwise they will fail with <code>key not found</code>.<br>
⚠ Do not select this option on a mainnet node as anyone who gains access to the server could also access the wallet.

New files will appear, e.g.:

<code>.elys/ 
├── 6a6094033197d062d4de7bcb2c59dae4e28ee338.address 
├── keyhash 
└── validator.info</code>

You can view the addresses associated with the wallet with:

<code>elysd keys show validator</code> to view the wallet address itself;<br>
<code>elysd keys show validator --bech val</code> to get the *valoper* address, i.e. the validator address.

### **3.5.2 Registering as a validator**

***Funding the Wallet***

You will need some tokens in the wallet you created earlier to pay for the transaction fees and create a validator. You can either use a faucet, or purchase tokens and transfer them to the wallet.
You don’t need many for the next steps, 2–5 should be enough.

***Creating the validator***

Then you can create the validator, by passing the following command:
⚠ Some items are permanent, i.e. cannot be edited later (commission-max-rate, commission-max-change-rate, min-self-delegation, pubkey).

    $ elysd tx staking create-validator \
     --amount 1000000uelys \
     --chain-id=cosmoshub-4 \
     --pubkey=$(elysd tendermint show-validator) \
    --moniker=”Your custom moniker” \
    --website=”https://your_website.com” \
    --details=”A brief description of your team/project" \
     -- identity your_keybase_identity \
    --security-contact “your_contact_email” \
    --commission-rate=”0.05" \
    --commission-max-rate=”0.20"
    --commission-max-change-rate=”0.01"
    --min-self-delegation=”1" \
    --from validator-key \
    --fees 200uelys

⚠ If you created the key with _--keyring-backend test_, then make sure to add this flag to the above command.<br>
⚠ The _fees_ amount can vary. If they are too low, the command will fail with an error _insufficient fees_: check the expected amount and update the command before running it again.

You can set the commission items to whatever you like : 0.05 means 5%, “commission-max-rate” is self-explanatory, and commission-max-change-rate=”0.01" means that you can only increase the commission by 1 point every 24h.<br>
This is reassuring for the stakers as it means that you can’t bump the commission instantly to the maximum you defined (which can absolutely be configured to 100%!).<br>
The identity flag is optional, but this is usually how the explorers retrieve and display the team logo.<br>
Website, details and security contact are also optional.

If the command succeeded, you can run <code>$ curl localhost:26657/status | jq</code> again: the “voting power” part should now indicate “1”.

### **3.5.3 Joining the Active Set**

Your validator is now registered and operational, but you may not actually participate to the consensus yet as your voting power is too low — and therefore you do not earn rewards.

In order to join the active set, you should either find delegators or stake your own funds so that you have more voting power than the current last validator in the set.

This is actually the hardest part. You need to provide a stable quality of service, have a competitive commission, and do some marketing and communication to make the community aware of your presence and start having holders trust you with their stake.<br>
There are plenty of ways to try and stand out, but that is entirely up to you.

The quality of service is essential. If you miss blocks, then you accrue less rewards and so do your delegators; and if you miss too many, you end up being jailed and slashed.<br>
Delegators may also redelegate to another validator.

⚠ This is why implementing a monitoring solution is the next step in this journey. Such tool will constantly evaluate the performance of the node and alert you whenever a problem arises, giving you the opportunity to resolve it before it visibly affects your quality of service.
    


    
  
  


