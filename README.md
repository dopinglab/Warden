Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.22.5"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export WARDEN_CHAIN_ID="chiado_10010-1"" >> $HOME/.bash_profile
echo "export WARDEN_PORT="18"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
rm -rf bin
mkdir bin && cd bin
wget https://github.com/warden-protocol/wardenprotocol/releases/download/v0.5.3/wardend_Linux_x86_64.zip
unzip wardend_Linux_x86_64.zip
chmod +x wardend
mv $HOME/bin/wardend $HOME/go/bin
```

**config and init app**
```
wardend init $MONIKER
sed -i -e "s|^node *=.*|node = \"tcp://localhost:${WARDEN_PORT}657\"|" $HOME/.warden/config/client.toml
```

**download genesis and addrbook**
```
wget -O $HOME/.warden/config/genesis.json https://server-4.itrocket.net/testnet/warden/genesis.json
wget -O $HOME/.warden/config/addrbook.json  https://server-4.itrocket.net/testnet/warden/addrbook.json
```

**set seeds and peers**
```
SEEDS="8288657cb2ba075f600911685670517d18f54f3b@warden-testnet-seed.itrocket.net:18656"
PEERS="b14f35c07c1b2e58c4a1c1727c89a5933739eeea@warden-testnet-peer.itrocket.net:18656,0fb6439f5e2cfc8622501769bb071076bce9dfc1@116.202.150.231:18656,ade613e0a08bcf22d8ae6da3bed7b692992f0aeb@213.199.38.94:31656,eb2e7095f86b24e8d5d286360c34e060a8db6334@188.40.85.207:12756,3bc060a19fa237fc632273414dd1e9551ad0b312@65.109.88.159:56656,29dfeed0f7933111c5452a1af4ca67b2fe4346f5@198.27.80.53:26656,d8d46d0175fe948529aa52d696e2fcf50691c492@162.55.97.180:15656,252cfe526707952ba2fb557b460161208901a973@82.208.22.200:26656,8f50605a5cd64c735f86ffc22f2dd0177bcc58b3@168.119.120.156:26656,1c1451d7cb1d6cb556343848aa2001fc96953346@62.171.181.179:18656,968d83090c18ef4ad87aedc803fa4edd82ea4b6d@5.75.160.247:18656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.warden/config/config.toml
```

**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${WARDEN_PORT}317%g;
s%:8080%:${WARDEN_PORT}080%g;
s%:9090%:${WARDEN_PORT}090%g;
s%:9091%:${WARDEN_PORT}091%g;
s%:8545%:${WARDEN_PORT}545%g;
s%:8546%:${WARDEN_PORT}546%g;
s%:6065%:${WARDEN_PORT}065%g" $HOME/.warden/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${WARDEN_PORT}658%g;
s%:26657%:${WARDEN_PORT}657%g;
s%:6060%:${WARDEN_PORT}060%g;
s%:26656%:${WARDEN_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${WARDEN_PORT}656\"%;
s%:26660%:${WARDEN_PORT}660%g" $HOME/.warden/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.warden/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.warden/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.warden/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "25000000award"|g' $HOME/.warden/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.warden/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.warden/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/wardend.service > /dev/null <<EOF
[Unit]
Description=Warden node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.warden
ExecStart=$(which wardend) start --home $HOME/.warden
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
wardend tendermint unsafe-reset-all --home $HOME/.warden
if curl -s --head curl https://server-4.itrocket.net/testnet/warden/warden_2024-11-04_202058_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-4.itrocket.net/testnet/warden/warden_2024-11-04_202058_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.warden
    else
  echo "no snapshot found"
fi
```
**enable and start service**
```
sudo systemctl daemon-reload
sudo systemctl enable wardend
sudo systemctl restart wardend && sudo journalctl -u wardend -f
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/warden/autoinstall/)
```

Create wallet
**to create a new wallet, use the following command. don’t forget to save the mnemonic**
```
wardend keys add $WALLET
```

**to restore exexuting wallet, use the following command**
```
wardend keys add $WALLET --recover
```

**save wallet and validator address**
```
WALLET_ADDRESS=$(wardend keys show $WALLET -a)
VALOPER_ADDRESS=$(wardend keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**check sync status, once your node is fully synced, the output from above will print "false"**
```
wardend status 2>&1 | jq 
```

# before creating a validator, you need to fund your wallet and check balance
wardend query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.warden/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://warden-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  if [ "$blocks_left" -lt 0 ]; then
    blocks_left=0
  fi

  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"

  sleep 5
done
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, award
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
cd $HOME
# Create validator.json file
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(wardend comet show-validator | grep -Po '\"key\":\s*\"\K[^"]*')\"},
    \"amount\": \"1000000award\",
    \"moniker\": \"test\",
    \"identity\": \"\",
    \"website\": \"\",
    \"security\": \"\",
    \"details\": \"I love blockchain ❤️\",
    \"commission-rate\": \"0.1\",
    \"commission-max-rate\": \"0.2\",
    \"commission-max-change-rate\": \"0.01\",
    \"min-self-delegation\": \"1\"
}" > validator.json
# Create a validator using the JSON configuration
wardend tx staking create-validator validator.json \
    --from $WALLET \
    --chain-id chiado_10010-1 \
	--gas auto --gas-adjustment 1.6 --fees 250000000000000award
	
Monitoring
If you want to have set up a monitoring and alert system use our cosmos nodes monitoring guide with tenderduty

Security
To protect you keys please don`t share your privkey, mnemonic and follow basic security rules

Set up ssh keys for authentication
You can use this guide to configure ssh authentication and disable password authentication on your server

Firewall security
Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port

sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${WARDEN_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop wardend
sudo systemctl disable wardend
sudo rm -rf /etc/systemd/system/wardend.service
sudo rm $(which wardend)
sudo rm -rf $HOME/.warden
sed -i "/WARDEN_/d" $HOME/.bash_profile
