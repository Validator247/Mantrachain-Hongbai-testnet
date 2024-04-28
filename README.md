# Mantrachain-Hongbai-testnet

Update system & install prerequisites:

    sudo apt update
    sudo apt upgrade
    sudo apt install -y curl git jq lz4 build-essential unzip
    bash <(curl -s "https://raw.githubusercontent.com/MANTRA-Finance/public/main/go_install.sh")
    source .bash_profile

Install binary:

    mkdir bin
    cd bin
    source ~/.profile

    wget https://github.com/MANTRA-Finance/public/raw/main/mantrachain-hongbai/mantrachaind-linux-amd64.zip
    unzip mantrachaind-linux-amd64.zip
    rm -rf mantrachaind-linux-amd64.zip

Install cosmwasm library

    sudo wget -P /usr/lib https://github.com/CosmWasm/wasmvm/releases/download/v1.3.1/libwasmvm.x86_64.so

Initialise node

    mantrachaind init <your-moniker> --chain-id mantra-hongbai-1

Download genesis.json & Addrbook.json 

    wget https://raw.githubusercontent.com/Validator247/Mantrachain-Hongbai-testnet/main/genesis.json

Download Addrbook.json

    wget https://raw.githubusercontent.com/Validator247/Mantrachain-Hongbai-testnet/main/addrbook.json    

Update the config.toml & Seed & Peer

    CONFIG_TOML="$HOME/.mantrachain/config/config.toml"
    SEEDS="d6016af7cb20cf1905bd61468f6a61decb3fd7c0@34.72.142.50:26656"
    PEERS="da061f404690c5b6b19dd85d40fefde1fecf406c@34.68.19.19:26656,20db08acbcac9b7114839e63539da2802b848982@34.72.148.3:26656"
    sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $CONFIG_TOML
    sed -i.bak -e "s/^seeds =.*/seeds = \"$SEEDS\"/" $CONFIG_TOML
    external_address=$(wget -qO- eth0.me)
    sed -i.bak -e "s/^external_address *=.*/external_address = \"$external_address:26656\"/" $CONFIG_TOML
    sed -i 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.0002uom"|g' $CONFIG_TOML
    sed -i 's|^prometheus *=.*|prometheus = true|' $CONFIG_TOML
    sed -i -e "s/^filter_peers *=.*/filter_peers = \"true\"/" $CONFIG_TOML

Install Cosmovisor

    go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.4.0
    mkdir -p ~/.mantrachain/cosmovisor/genesis/bin
    mkdir -p ~/.mantrachain/cosmovisor/upgrades
    cp ~/bin/mantrachaind ~/.mantrachain/cosmovisor/genesis/bin

Create Service

    sudo tee /etc/systemd/system/mantrachaind.service > /dev/null << EOF
    [Unit]
    Description=Mantra Node
    After=network-online.target
    [Service]
    User=$USER
    ExecStart=$(which cosmovisor) run start
    Restart=on-failure
    RestartSec=3
    LimitNOFILE=10000
    Environment="DAEMON_NAME=mantrachaind"
    Environment="DAEMON_HOME=$HOME/.mantrachain"
    Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
    Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
    Environment="UNSAFE_SKIP_BACKUP=true"
    [Install]
    WantedBy=multi-user.target
    EOF

Starting, stoping and restarting service

    #reload, enable and start
    sudo systemctl daemon-reload
    sudo systemctl enable mantrachaind
    sudo systemctl start mantrachaind

    #stop
    sudo systemctl stop mantrachaind

    #restart
    sudo systemctl restart mantrachaind

    #logs
    journalctl -u mantrachaind -f -o cat

    #logs - filtered on block height lines
    sudo journalctl -xefu mantrachaind -g ".*txindex"

Snapshots

    sudo systemctl stop mantrachaind

    cp $HOME/.mantrachain/data/priv_validator_state.json $HOME/.mantrachain/priv_validator_state.json.backup

    rm -rf $HOME/.mantrachain/data $HOME/.mantrachain/wasmPath
    curl https://snapshot.validatorvn.com/mantra/data.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.mantrachain

    mv $HOME/.mantrachain/priv_validator_state.json.backup $HOME/.mantrachain/data/priv_validator_state.json

    sudo systemctl restart mantrachaind && sudo journalctl -u mantrachaind -f -o cat

State Sync

    sudo systemctl stop mantrachaind
    cp $HOME/.mantrachain/data/priv_validator_state.json $HOME/.mantrachain/priv_validator_state.json.backup
    mantrachaind tendermint unsafe-reset-all --home ~/.mantrachain/ --keep-addr-book
    SNAP_RPC="https://mantra-rpc-testnet.validatorvn.com:443"

    LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
    BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000)); \
    TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
    echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH

    sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
    s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
    s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
    s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" ~/.mantrachain/config/config.toml
    more ~/.mantrachain/config/config.toml | grep 'rpc_servers'
    more ~/.mantrachain/config/config.toml | grep 'trust_height'
    more ~/.mantrachain/config/config.toml | grep 'trust_hash'

    sudo mv $HOME/.mantrachain/priv_validator_state.json.backup $HOME/.mantrachain/data/priv_validator_state.json
    sudo systemctl restart mantrachaind && journalctl -u mantrachaind -f -o cat

# Create keys and Validator

Generate a new public key with mnemonic phrase

Create a key file.

    mantrachaind config keyring-backend file

Add New Wallet Key                

    mantrachaind keys add wallet

Import keys ( if using an existing wallet )

    mantrachaind keys add wallet --recover

Faucet:

    https://faucet.hongbai.mantrachain.io/

create-validator

    mantrachaind tx staking create-validator \
      --amount=1000000uom \
      --pubkey=$(mantrachaind tendermint show-validator) \
      --moniker=<MONIKER> \
      --chain-id=mantra-hongbai-1 \
      --commission-rate="0.10" \
      --commission-max-rate="0.20" \
      --commission-max-change-rate="0.01" \
      --min-self-delegation="1000000" \
      --gas="auto" \
      --gas-adjustment 2 \
      --gas-prices="0.0002uom" \
      --from=wallet -y

  edit-validator

      mantrachaind tx staking edit-validator \
      --new-moniker=<NEW-MONIKER> \
      --website="https://your-website.com" \
      --details="Some impressive info about your validator." \
      --chain-id=mantra-hongbai-1 \
      --commission-rate="0.10" \
      --gas="auto" \
      --gas-adjustment 2 \
      --gas-prices="0.0002uom" \
      --from=wallet -y

# DONE : Read carefully - before asking any questions
          
