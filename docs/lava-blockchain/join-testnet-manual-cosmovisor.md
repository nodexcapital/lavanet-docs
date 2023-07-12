---
sidebar_position: 2
slug: /testnet-manual-cosmovisor
title: Option A - With Cosmovisor
---
import RoadmapItem from '@site/src/components/RoadmapItem';

# Join testnet - Manual setup with Cosmovisor
## Prerequisites

1. Verify [hardware requirements](reqs) are met
2. Install package dependencies
    - Note: You may need to run as `sudo`
    - Required packages installation
        
        ```bash
        ### Packages installations
        sudo apt -q update
        sudo apt -qy install curl git jq lz4 build-essential
        sudo apt -qy upgrade
        ```
        
    - Go installation 
        
        ```bash
        ### Go configuration
        sudo rm -rf /usr/local/go
        curl -Ls https://go.dev/dl/go1.20.5.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
        eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
        eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
        ```
        
    - Installation verifications
        
        1. You can verify the installed go version by running: `go version`
        2. PATH should include `$HOME/go/bin`
        To verify PATH, run `echo $PATH`        

## 1. Set up a local node

### Download lava binaries
    
- Download the latest lava binaries files needed for the setup local node
    
    ```bash
    # Clone lava binaries
    git clone https://github.com/lavanet/lava.git
    cd lava
    git checkout v0.16.0
    ```
- Build lava binaries
    ```bash
    make build
    ```
    -  lava binaries should be located at `$HOME/lava/build/lavad`
    

### Set the genesis and addrbook file

- Download the genesis.json file to the Lava config folder
    
    ```bash
    curl -Ls https://snapshots.kjnodes.com/lava-testnet/genesis.json > $HOME/.lava/config/genesis.json
    ```
- Download the addrbook.json file to the Lava config folder
    ```bash
    curl -Ls https://snapshots.kjnodes.com/lava-testnet/addrbook.json > $HOME/.lava/config/addrbook.json
    ```
   

## 2. Join the Lava Testnet

The following sections will describe how to install Cosmovisor for automating the upgrades process.


### Set up Cosmovisor {#cosmovisor}

- Set up cosmovisor to ensure any future upgrades happen flawlessly. To install Cosmovisor:
    
    ```bash
    go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.4.0
    ```
- Prepare binaries for cosmovisor
    ```bash
    mkdir -p $HOME/.lava/cosmovisor/genesis/bin
    mv build/lavad $HOME/.lava/cosmovisor/genesis/bin/
    rm -rf build
    ```
- Create application symlinks for cosmovisor
    ```bash
    sudo ln -s $HOME/.lava/cosmovisor/genesis $HOME/.lava/cosmovisor/current -f
    sudo ln -s $HOME/.lava/cosmovisor/current/bin/lavad /usr/local/bin/lavad -f
    ```
    
-  Create lava systemd unit file
    ```bash
    sudo tee /etc/systemd/system/lavad.service > /dev/null << EOF
    [Unit]
    Description=lava-testnet node service
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
    ```
- Enbale systemd service for lava
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable lavad
    ```
### Configure lava node {#initialize}
   - Set node configuration
     ```bash
     lavad config chain-id lava-testnet-1
     lavad config keyring-backend test
     lavad config node tcp://localhost:26657
     ```
   - Initialize the node
     ```bash
     lavad init <your-node-name> --chain-id lava-testnet-1
     ```
   - Configure seed peers
     ```bash
     sed -i -e "s|^seeds *=.*|seeds = \"ade4d8bc8cbe014af6ebdf3cb7b1e9ad36f412c0@testnet-seeds.polkachu.com:19956,3f472746f46493309650e5a033076689996c8881@lava- 
     testnet.rpc.kjnodes.com:14459\"|" $HOME/.lava/config/config.toml
     ```
   - Configure lava gas-prices
     ```bash
     sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0ulava\"|" $HOME/.lava/config/app.toml
     ```
   - Configure custom port for lava node (_optional_)
     ```bash
       PORT=210
       sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${PORT}58\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = 
       \"tcp://127.0.0.1:${PORT}57\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${PORT}60\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = 
       \"tcp://0.0.0.0:${PORT}56\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${PORT}66\"%" $HOME/.lava/config/config.toml
       sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:${PORT}17\"%; s%^address = \":8080\"%address = \":${PORT}80\"%; s%^address = 
       \"0.0.0.0:9090\"%address = \"0.0.0.0:${PORT}90\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:${PORT}91\"%" $HOME/.lava/config/app.toml
     ```
   - Update chain-specific configuration
     ```bash
     sed -i \
     -e 's/create_empty_blocks = .*/create_empty_blocks = true/g' \
     -e 's/create_empty_blocks_interval = ".*s"/create_empty_blocks_interval = "60s"/g' \
     -e 's/timeout_propose = ".*s"/timeout_propose = "60s"/g' \
     -e 's/timeout_commit = ".*s"/timeout_commit = "60s"/g' \
     -e 's/timeout_broadcast_tx_commit = ".*s"/timeout_broadcast_tx_commit = "601s"/g' \
     $HOME/.lava/config/config.toml
     ```
   - Update pruning setting (_optional_)
     ```bash
     sed -i \
     -e 's|^pruning *=.*|pruning = "custom"|' \
     -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
     -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
     -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
     $HOME/.lava/config/app.toml
     ```


### Download the latest Lava data snapshot (_recomended_) {#snapshots}
```bash
curl -L https://snap.nodexcapital.com/lava/lava-latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.lava
[[ -f $HOME/.lava/data/upgrade-info.json ]] && cp $HOME/.lava/data/upgrade-info.json $HOME/.lava/cosmovisor/genesis/upgrade-info.json
```

    

### Start service and check the logs
```bash
sudo systemctl start lavad
```
    

## 3. Verify

### Verify `lava node` setup

Make sure `cosmovisor` is running by checking the state of the cosmovisor service:

- Check the status of the service
    ```bash
    sudo systemctl status lavad
    ```
- To view the service logs - to escape, hit CTRL+C

    ```bash
    sudo journalctl -fu lavad -o cat
    ```

### Verify node status

```bash
# Check if the node is currently in the process of catching up
lavad status | jq .SyncInfo
```

## Welcome to Lava Testnet 🌋

:::tip Joined Testnet? Be a validator!
You are now running a Node in the Lava network 🎉🥳! 

Congrats, happy to have you here 😻 Celebrate it with us on Discord.

When you're ready, start putting the node to use **as a validator**:
[<RoadmapItem icon="🧑‍⚖️" title="Power as a Validator" description="Validate blocks, secure the network, earn rewards"/>](/validator-manual#account)

:::
