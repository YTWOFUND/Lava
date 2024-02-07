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
sudo apt upgrade -y
sudo apt install -y curl git jq lz4 build-essential
sudo apt install -y unzip logrotate git jq sed wget curl coreutils systemd
```

Go installation.
```
cd $HOME
curl https://dl.google.com/go/go1.20.5.linux-amd64.tar.gz | sudo tar -C/usr/local -zxvf -
cat <<'EOF' >>$HOME/.profile
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
source $HOME/.profile
go version
```

### Download and build binaries
```
MONIKER="YOUR MONIKER"
cd $HOME
rm -rf lava
git clone https://github.com/lavanet/lava.git
cd lava
git checkout v0.34.0
```

### Build binaries
```
export LAVA_BINARY=lavad
make build
```
### Prepare binaries for Cosmovisor
```
mkdir -p $HOME/.lava/cosmovisor/genesis/bin
mv build/lavad $HOME/.lava/cosmovisor/genesis/bin/
rm -rf build
```

### Create application symlinks
```
sudo ln -s $HOME/.lava/cosmovisor/genesis $HOME/.lava/cosmovisor/current -f
sudo ln -s $HOME/.lava/cosmovisor/current/bin/lavad /usr/local/bin/lavad -f
```

### Download and install Cosmovisor
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
```

### Create service
```
sudo tee /etc/systemd/system/lava.service > /dev/null << EOF
[Unit]
Description=lava node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.lava"
Environment="DAEMON_NAME=lavad"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable lava.service
```

# Set node configuration
lavad config chain-id lava-testnet-2
lavad config keyring-backend test
lavad config node tcp://localhost:14457

# Initialize the node
lavad init $MONIKER --chain-id lava-testnet-2

# Download genesis and addrbook
curl -Ls https://snapshots.kjnodes.com/lava-testnet/genesis.json > $HOME/.lava/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/lava-testnet/addrbook.json > $HOME/.lava/config/addrbook.json

# Add seeds
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@lava-testnet.rpc.kjnodes.com:14459\"|" $HOME/.lava/config/config.toml

# Set minimum gas price
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0ulava\"|" $HOME/.lava/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.lava/config/app.toml

# Set custom ports
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:14458\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:14457\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:14460\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:14456\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":14466\"%" $HOME/.lava/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:14417\"%; s%^address = \":8080\"%address = \":14480\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:14490\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:14491\"%; s%:8545%:14445%; s%:8546%:14446%; s%:6065%:14465%" $HOME/.lava/config/app.toml

### Update chain-specific configuration
sed -i \
  -e 's/timeout_commit = ".*"/timeout_commit = "30s"/g' \
  -e 's/timeout_propose = ".*"/timeout_propose = "1s"/g' \
  -e 's/timeout_precommit = ".*"/timeout_precommit = "1s"/g' \
  -e 's/timeout_precommit_delta = ".*"/timeout_precommit_delta = "500ms"/g' \
  -e 's/timeout_prevote = ".*"/timeout_prevote = "1s"/g' \
  -e 's/timeout_prevote_delta = ".*"/timeout_prevote_delta = "500ms"/g' \
  -e 's/timeout_propose_delta = ".*"/timeout_propose_delta = "500ms"/g' \
  -e 's/skip_timeout_commit = ".*"/skip_timeout_commit = false/g' \
  $HOME/.lava/config/config.toml

### Download latest chain snapshot
```
curl -L https://snapshots.kjnodes.com/lava-testnet/snapshot_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.lava
[[ -f $HOME/.lava/data/upgrade-info.json ]] && cp $HOME/.lava/data/upgrade-info.json $HOME/.lava/cosmovisor/genesis/upgrade-info.json
```

### Start service and check the logs
```
sudo systemctl start lava.service && sudo journalctl -u lava.service -f --no-hostname -o cat
```

### Becoming a Validator

# Create wallet key new
```
lavad keys add wallet
```

(OPTIONAL) RECOVER EXISTING KEY
```
lavad keys add wallet --recover
```

### We receive tokens from the tap in the [discord](https://discord.com/invite/5VcqgwMmkA)
```
then go to the #faucet branch and write $request your lava address(lava@....)
```

### Create the Validator

Before creating a validator, enter the command and check that you have false. This means that the Node has synchronized and you can create a validator:
```
lavad status 2>&1 | jq .SyncInfo
```

### Check the balance before creating for the presence of tokens
```
lavad q bank balances $(lavad keys show wallet -a)
```

Please enter your details below in quotation marks where required

```
lavad tx staking create-validator \
--amount 1000000ulava \
--pubkey $(lavad tendermint show-validator) \
--moniker "YOUR_MONIKER_NAME" \
--identity "YOUR_KEYBASE_ID" \
--details "YOUR_DETAILS" \
--website "YOUR_WEBSITE_URL" \
--chain-id lava-testnet-2 \
--commission-rate 0.05 \
--commission-max-rate 0.20 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--from wallet \
--gas-adjustment 1.4 \
--gas auto \
--gas-prices 0ulava \
-y
```

### Update
```
There have been no updates at the moment, as soon as they come out, we will immediately add them to this section.

Current network:lava-testnet-2
Current version:v0.34.0
```

### Useful commands

Check balance
```
lavad q bank balances $(lavad keys show wallet -a)
```

CHECK SERVICE LOGS
```
sudo journalctl -u lava.service -f --no-hostname -o cat
```

RESTART SERVICE
```
sudo systemctl restart lava.service
```

GET VALIDATOR INFO
```
lavad status 2>&1 | jq .ValidatorInfo
```

DELEGATE TOKENS TO YOURSELF
```
lavad tx staking delegate $(lavad keys show wallet --bech val -a) 1000000ulava --from wallet --chain-id lava-testnet-2 --gas-adjustment 1.4 --gas auto --gas-prices 0ulava -y
```

REMOVE NODE
Please, before proceeding with the next step! All chain data will be lost! Make sure you have backed up your priv_validator_key.json!
```
cd $HOME
sudo systemctl stop lava.service
sudo systemctl disable lava.service
sudo rm /etc/systemd/system/lava.service
sudo systemctl daemon-reload
rm -f $(which lavad)
rm -rf $HOME/.lava
rm -rf $HOME/lava
```

