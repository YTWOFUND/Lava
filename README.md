# Lava
Lava Node Installation Instructions </br>
### [Official documentation](https://docs.lavanet.xyz/reqs)

### System requirements: </br>
CPU: 4 Core </br>
RAM: 8 Gb </br>
SSD: 100 Gb </br>
OS: Ubuntu 20.04 LTS </br>

### Network configurations: </br>
Outbound - allow all traffic </br>
Inbound - open the following ports :</br>
1317 - REST </br>
26657 - TENDERMINT_RP </br>
26656 - Cosmos </br>
Running as a Provider? Add these specific ports: </br>
22221 - provider port </br>
22231 - provider port </br>
22241 - provider port </br>
    
# Manual configuration of the Lava node with Cosmovisor
Required packages installation </br>
```
sudo apt update
sudo apt install -y unzip logrotate git jq sed wget curl coreutils systemd
```
Create the temp dir for the installation. </br>
```
temp_folder=$(mktemp -d) && cd $temp_folder
```
Go installation.
```
wget https://go.dev/dl/go1.20.linux-amd64.tar.gz
cp go1.20.linux-amd64.tar.gz /usr/local
tar -C /usr/local -xzf go1.20.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
export PATH=$PATH:$(go env GOPATH)/bin
go version
```
```
cat <<'EOF' >>$HOME/.profile
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
```
Download configuration files needed for the installation.
```
git clone https://github.com/lavanet/lava-config.git
cd lava-config/testnet-1
source setup_config/setup_config.sh
```
Set app configurations. Copy lavad default config files to config Lava config folder
```
echo "Lava config file path: $lava_config_folder"
mkdir -p $lavad_home_folder
mkdir -p $lava_config_folder
cp default_lavad_config_files/* $lava_config_folder
```
Set the genesis file in the configuration folder
```
cp genesis_json/genesis.json $lava_config_folder/genesis.json
```
Set up cosmovisor to ensure any future upgrades happen flawlessly. To install Cosmovisor:
```
go install github.com/cosmos/cosmos-sdk/cosmovisor/cmd/cosmovisor@v1.0.0
mkdir -p $lavad_home_folder/cosmovisor
wget https://lava-binary-upgrades.s3.amazonaws.com/testnet/cosmovisor-upgrades/cosmovisor-upgrades.zip
unzip cosmovisor-upgrades.zip
cp -r cosmovisor-upgrades/* $lavad_home_folder/cosmovisor
```
Set the environment variables
```
echo "# Setup Cosmovisor" >> ~/.profile
echo "export DAEMON_NAME=lavad" >> ~/.profile
echo "export CHAIN_ID=lava-testnet-1" >> ~/.profile
echo "export DAEMON_HOME=$HOME/.lava" >> ~/.profile
echo "export DAEMON_ALLOW_DOWNLOAD_BINARIES=true" >> ~/.profile
echo "export DAEMON_LOG_BUFFER_SIZE=512" >> ~/.profile
echo "export DAEMON_RESTART_AFTER_UPGRADE=true" >> ~/.profile
echo "export UNSAFE_SKIP_BACKUP=true" >> ~/.profile
source ~/.profile
```
Initialize the chain
```
$lavad_home_folder/cosmovisor/genesis/bin/lavad init \
my-node \
--chain-id lava-testnet-1 \
--home $lavad_home_folder \
--overwrite
cp genesis_json/genesis.json $lava_config_folder/genesis.json
```
Create the systemd unit file
```
echo "[Unit]
Description=Cosmovisor daemon
After=network-online.target
[Service]
Environment="DAEMON_NAME=lavad"
Environment="DAEMON_HOME=${HOME}/.lava"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
Environment="DAEMON_LOG_BUFFER_SIZE=512"
Environment="UNSAFE_SKIP_BACKUP=true"
User=$USER
ExecStart=${HOME}/go/bin/cosmovisor start --home=$lavad_home_folder --p2p.seeds $seed_node
Restart=always
RestartSec=3
LimitNOFILE=infinity
LimitNPROC=infinity
[Install]
WantedBy=multi-user.target
" >cosmovisor.service
sudo mv cosmovisor.service /lib/systemd/system/cosmovisor.service
```
Enable and start the Cosmovisor service
```
sudo systemctl daemon-reload
sudo systemctl enable cosmovisor.service
sudo systemctl restart systemd-journald
sudo systemctl start cosmovisor
```
# Run Validator
Create a cosmos network wallet, replace <name_here> with a name of your choice.
```
current_lavad_binary="$HOME/.lava/cosmovisor/current/bin/lavad"
ACCOUNT_NAME="name_here"
$current_lavad_binary keys add $ACCOUNT_NAME
```
_Ensure you write down the mnemonic as you can not recover the wallet without it._ </br>

Obtain your validator pubkey:
```
$current_lavad_binary tendermint show-validator
```
We replenish the account through the [faucet](https://discord.com/channels/963778337904427018/1024618316276441118) in the discord. </br>

Verify that your node has finished synching and it is caught up with the network
```
$current_lavad_binary status | jq .SyncInfo.catching_up
```
_Wait until you see the output: "false"_ </br>

Make sure you see your account has Lava tokens in it.
```
YOUR_ADDRESS=$($current_lavad_binary keys show -a $ACCOUNT_NAME)
$current_lavad_binary query \
    bank balances \
    $YOUR_ADDRESS \
    --denom ulava
```
Creating a Validator </br>
Replace <moniker_node> with the human-readable name you choose for your validator.
```
$current_lavad_binary tx staking create-validator \
    --amount="50000000ulava" \
    --pubkey=$($current_lavad_binary tendermint show-validator --home "$HOME/.lava/") \
    --moniker="<<moniker_node>>" \
    --chain-id=lava-testnet-1 \
    --commission-rate="0.10" \
    --commission-max-rate="0.20" \
    --commission-max-change-rate="0.01" \
    --min-self-delegation="10000" \
    --gas="auto" \
    --gas-adjustment "1.5" \
    --gas-prices="0.05ulava" \
    --home="$HOME/.lava/" \
    --from=$ACCOUNT_NAME
```
_Once you have finished running the command above, if you see code: 0 in the output, the command was successful_ </br>

# Useful Commands </br>
Check the status of the service
```
sudo systemctl status cosmovisor
```
To view the service logs - to escape, hit CTRL+C
```
sudo journalctl -u cosmovisor -f
```
Check balance
```
cosmovisor q bank balances $($current_lavad_binary keys show -a $ACCOUNT_NAME)
```
Delegation to yourself.
```
cosmovisor tx staking delegate <your_valoper_address> 100000ulava --from <account_name> --chain-id lava-testnet-1 --fees 10ulava -y
```
