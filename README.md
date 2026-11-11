<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/ed6c82c8-9635-4f3e-ae75-98bddd8dec19" />

* The Celestia Mocha testnet has been actively running for a long time, and since development is ongoing, it will keep running for the foreseeable future. The network is currently running as mocha-5 (validators from mocha-4 were not migrated and need to be created again on mocha-5).

* Follow this guide step by step to run a Validator, Fibre, Bridge and Light Node on the testnet and contribute to the network. The guide is based on the current releases (celestia-app v10.6.0-mocha, celestia-node v0.35.0-mocha).

* The Full Storage Node type has been removed from celestia-node. If you want to keep all data, run a Bridge Node with --archival.

* If you have any questions during or after setup, you can reach me via Telegram, Mail or Discord. Contributions on GitHub are also welcome.

## Validator Node Minimum System Requirements

 - Memory: 32 GB RAM
 - CPU: 32 Cores (with GFNI and SHA-NI support, must pass the CPU test)
 - Disk: 2 TB NVMe SSD Storage
 - Bandwidth: 1 Gbps
 - Operating System: Ubuntu 24.04 (celestia-app requires glibc 2.38, it does not run on Ubuntu 22.04)

NOTE: Mocha is now on the mocha-5 network. Validators from mocha-4 were not migrated to mocha-5, you need to create your validator again on mocha-5.

### System Updates
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential git ncdu lz4 aria2 -y
sudo apt install make -y
```

### Installing Go
```
ver="1.26.10"
cd $HOME
wget "https://go.dev/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```

### Installing celestia-app
```
cd $HOME
rm -rf celestia-app
git clone https://github.com/celestiaorg/celestia-app.git
cd celestia-app/
APP_VERSION=v10.6.0-mocha
git checkout tags/$APP_VERSION -b $APP_VERSION
make install
celestia-appd version
```

### Init Node
```
celestia-appd init "Node Name" --chain-id mocha-5
```

### Downloading Genesis
```
celestia-appd download-genesis mocha-5
```

### Adding Seeds and Peers
```
SEEDS=$(curl -sL https://raw.githubusercontent.com/celestiaorg/networks/main/mocha-5/seeds.txt | tr '\n' ',' | sed 's/,$//')
PEERS=$(curl -sL https://raw.githubusercontent.com/celestiaorg/networks/main/mocha-5/peers.txt | tr '\n' ',' | sed 's/,$//')
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.celestia-app/config/config.toml
```

### Enabling Pruning (Optional)
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.celestia-app/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.celestia-app/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.celestia-app/config/app.toml
```

### Setting Minimum Gas Price
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.002utia"|g' $HOME/.celestia-app/config/app.toml
```

### Enabling gRPC and the Signer for Fibre (Required for Validators)
- The Fibre server connects to the node through these two addresses. Both listen on 127.0.0.1 only and are never exposed to the internet.
```
sed -i -e '/^\[grpc\]/,/^\[/{s/^enable *=.*/enable = true/; s/^address *=.*/address = "127.0.0.1:9090"/}' $HOME/.celestia-app/config/app.toml
sed -i -e 's/^priv_validator_grpc_laddr *=.*/priv_validator_grpc_laddr = "127.0.0.1:26669"/' $HOME/.celestia-app/config/config.toml
```

### Creating a Wallet
- When you run the command below, it asks you to set a password and then shows your wallet address and the wallet's secret words (mnemonic). Be sure to save them, you will need these words to recover your wallet.

```
celestia-appd keys add wallet_name
```

### Getting Test Tokens
- Join the Discord at https://discord.com/invite/YsnTPcSfWQ and type `$request YOUR_WALLET_ADDRESS` in the #mocha-faucet channel to get test tokens. Each address gets 10 tokens per week.
- If you don't want to use Discord, there is a web faucet: https://mocha.celenium.io/faucet

### Creating the Service
```
sudo tee /etc/systemd/system/celestia-appd.service > /dev/null <<EOF
[Unit]
Description=Celestia node
After=network-online.target
[Service]
User=root
ExecStart=$(which celestia-appd) start --home $HOME/.celestia-app
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

### Starting the Node
```
sudo systemctl daemon-reload
sudo systemctl enable celestia-appd
sudo systemctl restart celestia-appd && sudo journalctl -u celestia-appd -f
```

### Checking Sync Status
- When the command returns false, your node is synced.

```
celestia-appd status 2>&1 | jq .sync_info.catching_up
```

### Creating the Validator
- Once synced, you can create your validator with the command below. It is set for 1 test token.
```
celestia-appd tx staking create-validator \
--amount 1000000utia \
--from wallet_name \
--moniker "MonikerName" \
--chain-id mocha-5 \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(celestia-appd tendermint show-validator) \
--gas 500000 \
--fees 3000utia \
--identity "" \
--website "" \
--details "" \
-y
```

NOTE: You can follow everything on the network and your tx results on [Explorer](https://testnet.celestia.explorers.guru/) or [Celenium](https://mocha.celenium.io).


## Fibre Server Setup (For Validators)

- Fibre is the new data availability protocol that came with v10. You need an active (bonded) validator.
- For now, it is recommended to run Fibre on the same server as your validator. Use separate disks for Fibre data and celestia-app data.
- You must have completed the "Enabling gRPC and the Signer for Fibre" step above and restarted the node.

### Installing Fibre
- Fibre is downloaded from the same release as celestia-app.
```
cd $HOME
wget https://github.com/celestiaorg/celestia-app/releases/download/v10.6.0-mocha/fibre_Linux_x86_64.tar.gz
wget https://github.com/celestiaorg/celestia-app/releases/download/v10.6.0-mocha/checksums.txt
sha256sum --ignore-missing --check checksums.txt
tar -xzf fibre_Linux_x86_64.tar.gz
sudo mv fibre /usr/local/bin/
rm fibre_Linux_x86_64.tar.gz checksums.txt
fibre version
```

### Checking That v10 Is Active
- The result should be app_version 10 or higher.
```
curl -s http://127.0.0.1:26657/abci_info | jq .result.response.app_version
```

### Creating the Service
- Choose a FIBRE_HOME folder on a different disk than celestia-app. /data/fibre is used below as an example.
```
sudo mkdir -p /data/fibre
sudo tee /etc/systemd/system/fibre.service > /dev/null <<EOF
[Unit]
Description=Celestia Fibre server
After=network-online.target celestia-appd.service
[Service]
User=root
ExecStart=/usr/local/bin/fibre start --home /data/fibre --app-grpc-address 127.0.0.1:9090 --signer-grpc-address 127.0.0.1:26669 --server-listen-address 0.0.0.0:7980
Restart=on-failure
RestartSec=5
TimeoutStopSec=120
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

### Starting Fibre
```
sudo systemctl daemon-reload
sudo systemctl enable fibre
sudo systemctl restart fibre && sudo journalctl -u fibre -f
```

### Opening the Fibre Port
- Port 7980 must be reachable from the internet. Never expose ports 9090 and 26669.
```
sudo ufw allow 7980/tcp
```

### Registering Your Fibre Address On-Chain
- Do this once Fibre is running and reachable from the internet. Replace SERVER_IP with your server's IP or domain.
```
celestia-appd tx valaddr set-host SERVER_IP:7980 \
--from wallet_name \
--chain-id mocha-5 \
--gas 200000 \
--fees 2000utia \
-y
```

### Checking the Registration
- The result should show found: true and your own address.
```
celestia-appd query valaddr provider $(celestia-appd comet show-address)
```

### Syncing Fibre Settings After an Upgrade
- Run this after every version upgrade to add new settings to server_config.toml.
```
celestia-appd config sync --fibre-home /data/fibre --dry-run
celestia-appd config sync --fibre-home /data/fibre
sudo systemctl restart fibre
```


## Upgrading (Validator and Fibre)

- This section upgrades a validator and Fibre server installed with this guide to a new release. Latest upgrade: **v10.6.0-mocha**. For future upgrades, you only need to change the APP_VERSION line.
- We build and download the new release first and restart the services after, so your node is down for only a few seconds.

### Upgrading celestia-app
```
APP_VERSION=v10.6.0-mocha
cd $HOME/celestia-app
git fetch --tags
git checkout -f tags/$APP_VERSION
make install
celestia-appd version
```

- If the version is correct, restart the node.
```
sudo systemctl restart celestia-appd && sudo journalctl -u celestia-appd -f
```

### Upgrading Fibre
- Fibre is downloaded from the same release as celestia-app. If you keep Fibre data somewhere other than /data/fibre, replace /data/fibre in the commands with your own folder.
```
APP_VERSION=v10.6.0-mocha
cd $HOME
wget https://github.com/celestiaorg/celestia-app/releases/download/$APP_VERSION/fibre_Linux_x86_64.tar.gz
wget https://github.com/celestiaorg/celestia-app/releases/download/$APP_VERSION/checksums.txt
sha256sum --ignore-missing --check checksums.txt
tar -xzf fibre_Linux_x86_64.tar.gz
rm fibre_Linux_x86_64.tar.gz checksums.txt
```

- If the checksum check says OK, stop Fibre, replace the binary and sync the settings.
```
sudo systemctl stop fibre
sudo mv fibre /usr/local/bin/
fibre version
celestia-appd config sync --fibre-home /data/fibre --dry-run
celestia-appd config sync --fibre-home /data/fibre
sudo systemctl start fibre && sudo journalctl -u fibre -f
```

### Checking the Upgrade
- The node should be synced (false) and your Fibre address should still be registered (found: true).
```
celestia-appd status 2>&1 | jq .sync_info.catching_up
celestia-appd query valaddr provider $(celestia-appd comet show-address)
```

- If you use celestiawatch, you can check everything at once with `sudo celestiawatch doctor`.

## Bridge Node Minimum System Requirements

 - Memory: 64 GB RAM
 - CPU: 32 Cores
 - Disk: 2 TB NVMe SSD Storage (12 TB for archival)
 - Bandwidth: 1 Gbps
 - Operating System: Ubuntu 24.04

### System Updates
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential git ncdu -y
sudo apt install make -y
```

### Installing Go
```
ver="1.26.10"
cd $HOME
wget "https://go.dev/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```

### Installing celestia-node
```
cd $HOME
rm -rf celestia-node
git clone https://github.com/celestiaorg/celestia-node.git
cd celestia-node/
git checkout tags/v0.35.0-mocha
make build
make install
make cel-key
```

- Version check ```celestia version``` >>> v0.35.0-mocha

### Init
- If you run your own consensus node, use its IP instead of rpc-mocha.pops.one (gRPC must be enabled on that node).
```
celestia bridge init --core.ip rpc-mocha.pops.one --core.port 9090 --p2p.network mocha
```

### Wallet
```
./cel-key list --node.type bridge --p2p.network mocha --keyring-backend test
```

### Getting Tokens From the Faucet
- You need to get test tokens to your wallet address from Discord >>> https://discord.com/invite/YsnTPcSfWQ (`$request YOUR_WALLET_ADDRESS` in the #mocha-faucet channel)

### Creating the Service
- For an archival bridge, add --archival to the end of the ExecStart line. A node started as pruned cannot be converted to archival later.
```
sudo tee /etc/systemd/system/celestia-bridge.service > /dev/null <<EOF
[Unit]
Description=celestia-bridge Cosmos daemon
After=network-online.target

[Service]
User=root
ExecStart=/usr/local/bin/celestia bridge start --core.ip rpc-mocha.pops.one --core.port 9090 --keyring.backend test --keyring.keyname my_celes_key --p2p.network mocha
Restart=on-failure
RestartSec=3
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
EOF
```

### Opening the Port
- Port 2121 must be open so other nodes can find you.
```
sudo ufw allow 2121
```

### Starting the Bridge Node
```
sudo systemctl daemon-reload
sudo systemctl enable celestia-bridge
sudo systemctl start celestia-bridge && sudo journalctl -u celestia-bridge -f
```

### Getting the Node ID
```
AUTH_TOKEN=$(celestia bridge auth admin --p2p.network mocha)
```
```
curl -X POST \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H 'Content-Type: application/json' \
     -d '{"jsonrpc":"2.0","id":0,"method":"p2p.Info","params":[]}' \
     http://localhost:26658
```

### Removing the Bridge Node
- If a validator runs on the same server, DO NOT delete the $HOME/.celestia-app folder, it holds your validator data.
```
sudo systemctl stop celestia-bridge
sudo systemctl disable celestia-bridge
sudo rm /etc/systemd/system/celestia-bridge.service
rm -rf $HOME/celestia-node
rm -rf $HOME/.celestia-bridge-mocha-5
```

## Full Storage Node

- The full storage node type has been removed from celestia-node. Only bridge and light nodes remain. If you want to keep all data, set up the bridge node above with --archival.
- If you still have an old full node, you can remove it with the commands below.
```
sudo systemctl stop celestia-full
sudo systemctl disable celestia-full
sudo rm /etc/systemd/system/celestia-full.service
rm -rf $HOME/.celestia-full-mocha-4
```

## Light Node System Requirements

 - Memory: 500 MB RAM (Minimum)
 - CPU: Single Core
 - Disk: 20 GB SSD Storage
 - Bandwidth: 56 Kbps

### System Updates
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential git ncdu -y
sudo apt install make -y
```

### Installing Go
```
ver="1.26.10"
cd $HOME
wget "https://go.dev/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```

### Installing celestia-node
```
cd $HOME
rm -rf celestia-node
git clone https://github.com/celestiaorg/celestia-node.git
cd celestia-node/
git checkout tags/v0.35.0-mocha
make build
make install
make cel-key
```

### Wallet
```
./cel-key list --node.type light --p2p.network mocha --keyring-backend test
```

### Init
```
celestia light init --p2p.network mocha
```

### Creating the Service
```
sudo tee /etc/systemd/system/celestia-light.service > /dev/null <<EOF
[Unit]
Description=celestia-light Cosmos daemon
After=network-online.target

[Service]
User=root
ExecStart=/usr/local/bin/celestia light start --core.ip rpc-mocha.pops.one --core.port 9090 --keyring.backend test --keyring.keyname my_celes_key --p2p.network mocha
Restart=on-failure
RestartSec=3
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
EOF
```

### Starting the Node
```
sudo systemctl daemon-reload
sudo systemctl enable celestia-light
sudo systemctl start celestia-light && sudo journalctl -u celestia-light -f
```

### Getting the Node ID
```
AUTH_TOKEN=$(celestia light auth admin --p2p.network mocha)
```
```
curl -X POST \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H 'Content-Type: application/json' \
     -d '{"jsonrpc":"2.0","id":0,"method":"p2p.Info","params":[]}' \
     http://localhost:26658
```

### Removing the Light Node
```
sudo systemctl stop celestia-light
sudo systemctl disable celestia-light
sudo rm /etc/systemd/system/celestia-light.service
rm -rf $HOME/celestia-node
rm -rf $HOME/.celestia-light-mocha-5
```

## Upgrading Bridge and Light Nodes

- This section upgrades a Bridge or Light Node installed with this guide to a new release. Latest upgrade: **v0.35.0-mocha**. For future upgrades, you only need to change the NODE_VERSION line.
- v0.35.0-mocha removes Bitswap in favor of shrex, adds JWT token revocation and includes security and stability fixes. No coordinated upgrade is required, you can upgrade at any time.

### Upgrading Go
- Building v0.35.0-mocha requires Go 1.26.10 or later. If `go version` already shows 1.26.10 or later, skip this step.
```
ver="1.26.10"
cd $HOME
wget "https://go.dev/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
source $HOME/.bash_profile
go version
```

### Upgrading celestia-node
```
NODE_VERSION=v0.35.0-mocha
cd $HOME/celestia-node
git fetch --tags
git checkout -f tags/$NODE_VERSION
make build
make install
make cel-key
celestia version
```

### For Bridge Nodes
- Stop the node, update its config to the new release, then start it again.
```
sudo systemctl stop celestia-bridge
celestia bridge config-update --p2p.network mocha
sudo systemctl start celestia-bridge && sudo journalctl -u celestia-bridge -f
```

### For Light Nodes
```
sudo systemctl stop celestia-light
celestia light config-update --p2p.network mocha
sudo systemctl start celestia-light && sudo journalctl -u celestia-light -f
```

### Enabling bbr (Required)
- You can enable it easily with the commands below.
```
cd $HOME/celestia-app
make bbr-enable
make bbr-check
```

- If you get an error with the commands above, you can enable it with the commands below.
```
sudo modprobe tcp_bbr
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
