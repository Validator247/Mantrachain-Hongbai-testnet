# Mantrachain-Hongbai-testnet

Explorer : https://explorer.validator247.com/hongbai-testnet/staking

Update system & install prerequisites:

    sudo apt update && sudo apt upgrade -y
    sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y

install go / Var / 

    cd $HOME
    VER="1.21.3"
    wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
    sudo rm -rf /usr/local/go
    sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
    rm "go$VER.linux-amd64.tar.gz"
    [ ! -f ~/.bash_profile ] && touch ~/.bash_profile
    echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
    source $HOME/.bash_profile
    [ ! -d ~/go/bin ] && mkdir -p ~/go/bin

    echo "export WALLET="wallet"" >> $HOME/.bash_profile
    echo "export MONIKER="Your_Nodename"" >> $HOME/.bash_profile
    echo "export MANTRA_CHAIN_ID="mantra-hongbai-1"" >> $HOME/.bash_profile
    echo "export MANTRA_PORT="22"" >> $HOME/.bash_profile
    source $HOME/.bash_profile

download binary

    cd $HOME
    sudo wget -O /usr/lib/libwasmvm.x86_64.so https://github.com/CosmWasm/wasmvm/releases/download/v1.3.1/libwasmvm.x86_64.so
    wget https://github.com/MANTRA-Finance/public/raw/main/mantrachain-hongbai/mantrachaind-linux-amd64.zip
    unzip mantrachaind-linux-amd64.zip
    rm mantrachaind-linux-amd64.zip
    mv mantrachaind $HOME/go/bin

config & init

    mantrachaind config node tcp://localhost:${MANTRA_PORT}657
    mantrachaind config keyring-backend os
    mantrachaind config chain-id mantra-hongbai-1
    mantrachaind init "Your_Nodename" --chain-id mantra-hongbai-1

Download genesis.json

    wget https://raw.githubusercontent.com/Validator247/Mantrachain-Hongbai-testnet/main/genesis.json

Download Addrbook.json

    wget https://raw.githubusercontent.com/Validator247/Mantrachain-Hongbai-testnet/main/addrbook.json    

Update the config.toml & Seed & Peer

    SEEDS="a9a71700397ce950a9396421877196ac19e7cde0@mantra-testnet-seed.itrocket.net:22656"
    PEERS="1a46b1db53d1ff3dbec56ec93269f6a0d15faeb4@mantra-testnet-peer.itrocket.net:22656,6414bdede0cfe6c8d6d522db8ec2ff062b066e94@213.199.48.74:23656,33cf22311a510b01552fb1e323add74c641f01c5@65.109.62.39:18656,a209274985b1babe4eabaeb7d1dce17d233e9878@116.202.162.188:17656,005025067680ab6767e1b931306b0b83e526703d@65.109.30.147:23656,34d121663730da1bd09051263a519d6b9406a1bd@135.181.44.138:23656,28dcca0ba822cc7a99ec5390da81d2f1bc9746a8@81.31.197.120:16656,ca61127cfd694c06589642c8ef58c6fb31f492d6@65.109.30.35:23656,30235fa097d100a14d2b534fdbf67e34e8d5f6cf@139.45.205.60:21656,584c3b8f18f1227f02faea250e6f6e01bfb04271@81.0.220.178:11656,4ccf2fb06244e8b39e3cb28a04602f1c4c593344@37.60.245.125:16656,ccd9c19b78e4a4075bd228b6d6d534f8c4fd54da@167.235.14.117:26656,cba9c1a3e42430e491f371bc626067c4a85a2a54@65.109.133.245:26656,ede298514a846130ef0f6170eab78dccfd55e6d3@62.171.150.241:23656"
    sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.mantrachain/config/config.toml


app.toml

    sed -i.bak -e "s%:1317%:${MANTRA_PORT}317%g;
    s%:8080%:${MANTRA_PORT}080%g;
    s%:9090%:${MANTRA_PORT}090%g;
    s%:9091%:${MANTRA_PORT}091%g;
    s%:8545%:${MANTRA_PORT}545%g;
    s%:8546%:${MANTRA_PORT}546%g;
    s%:6065%:${MANTRA_PORT}065%g" $HOME/.mantrachain/config/app.toml

config.toml file

    sed -i.bak -e "s%:26658%:${MANTRA_PORT}658%g;
    s%:26657%:${MANTRA_PORT}657%g;
    s%:6060%:${MANTRA_PORT}060%g;
    s%:26656%:${MANTRA_PORT}656%g;
    s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${MANTRA_PORT}656\"%;
    s%:26660%:${MANTRA_PORT}660%g" $HOME/.mantrachain/config/config.toml    

pruning & Gas

    # config pruning
    sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.mantrachain/config/app.toml
    sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.mantrachain/config/app.toml
    sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.mantrachain/config/app.toml

    # set minimum gas price, enable prometheus and disable indexing
    sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0002uom"|g' $HOME/.mantrachain/config/app.toml
    sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.mantrachain/config/config.toml
    sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.mantrachain/config/config.toml    


Create Service

    sudo tee /etc/systemd/system/mantrachaind.service > /dev/null <<EOF
    [Unit]
    Description=Mantra node
    After=network-online.target
    [Service]
    User=$USER
    WorkingDirectory=$HOME/.mantrachain
    ExecStart=$(which mantrachaind) start --home $HOME/.mantrachain
    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65535
    [Install]
    WantedBy=multi-user.target
    EOF

Download snapshot   

    mantrachaind tendermint unsafe-reset-all --home $HOME/.mantrachain
    if curl -s --head curl https://testnet-files.itrocket.net/mantra/snap_mantra.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
    curl https://testnet-files.itrocket.net/mantra/snap_mantra.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.mantrachain
    else
    echo no have snap
    fi    

            
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
    

# Create keys and Validator

Generate a new public key with mnemonic phrase

Create a key file.

    mantrachaind config keyring-backend file

Add New Wallet Key                

    mantrachaind keys add $WALLET

Import keys ( if using an existing wallet )

    mantrachaind keys add $WALLET --recover

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
      --from=$WALLET -y

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
      --from=$WALLET -y

# DONE : Read carefully - before asking any questions
          
