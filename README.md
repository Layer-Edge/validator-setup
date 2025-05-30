# ✅ TODO: Validator Setup

## Pre-Requisites

- go 1.21+
- build-essential, pkg-config, libssl-dev & make


## 1. Set Up a New Instance

- **Install Base Packages and Dependencies**
  ```sh
  sudo apt update
  sudo apt install -y build-essential
  sudo apt install -y pkg-config libssl-dev
  sudo apt install -y make

  wget https://go.dev/dl/go1.24.3.linux-amd64.tar.gz
  sudo tar -C /usr/local -xzf go1.24.3.linux-amd64.tar.gz

  echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.profile
  echo 'export GOPATH=$HOME/go' >> ~/.profile
  echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.profile
  source ~/.profile
  ```

- **Clone Chain Repo and initialize the node**
  ```sh
  git clone https://github.com/Layer-Edge/cosmos-evm.git
  cd cosmos-evm

  # this will create evmd in ~/go/bin/evmd and ~/.evmd of chain configuration
  ./init_node.sh

  echo 'alias evmd="$HOME/go/bin/evmd"'

  source ~/.bashrc
  ```

- **Update Genesis File with Main Node's genesis file**
  ```sh
  nano ~/.evmd/config/genesis.json
  ```

- **Configure Persistent Peers**
  ```sh
  nano ~/.your-chain-name/config/config.toml
  # Set:
  persistent_peers = "d638950f8f8abe07261a10be23fef3c95b3b0626@63.178.182.38:26656,b4f6f7ac3d453c56d5cb7e644830c65803f87755@3.124.18.213:26656"
  ```

- **Enable State Sync (Optional, for faster sync)**
  ```toml
  enable = true
  rpc_servers = "http://main-node-ip:26657,http://main-node-ip:26657"
  trust_height = <latest-block-height>
  trust_hash = "<block-hash>"
  ```

  Get trust data:
  ```sh
  curl http://main-node-ip:26657/block | jq .result.block.header
  ```

- **Start the Node**
  ```sh
  evmd start
  ```

- **Verify Syncing**
  ```sh
  evmd status | jq .SyncInfo
  ```

## 2. Set Up Validator on the New Node

- **Create Validator Key**
  ```sh
  evmd keys add validator_key
  ```

- **Fund Validator Address**
  ```sh
  evmd tx bank send <wallet-address> <validator-address> <amount>
  ```

- **Get Validator PubKey**
  ```sh
  evmd tendermint show-validator
  ```

- **Create validator.json**
  ```sh
  echo '{
    "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"oWg2ISpLF405Jcm2vXV+2v4fnjodh6aafuIdeoW+rUw="},
    "amount": "1000000aedgen",
    "moniker": "myvalidator",
    "identity": "optional identity signature (ex. UPort or Keybase)",
    "website": "validator's (optional) website",
    "security": "validator's (optional) security contact email",
    "details": "validator's (optional) details",
    "commission-rate": "0.1",
    "commission-max-rate": "0.2",
    "commission-max-change-rate": "0.01",
    "min-self-delegation": "1"
  }' > ~/validator.json
  ```

- **Create the Validator**
  ```sh
  evmd tx staking create-validator ~/validator.json \
    --chain-id 4207 \
    --gas 300000 \
    --from=validator_key \
    --gas-prices 80000000aedgen
  ```

- **Check Validator Status**
  ```sh
  evmd query staking validator $(evmd keys show validator --bech val -a)
  ```

- **Enable Logging**
  ```sh
  journalctl -u evmd -f
  ```

## 3. Auto-Delegate Staking Rewards

- **Create `auto_delegate.sh`**
  ```bash
  #!/bin/bash
  VALIDATOR_ADDRESS="<your-validator-address>"
  WALLET_NAME="<your-wallet-name>"
  CHAIN_ID="<your-chain-id>"
  DENOM="aedgen"
  evmd="/home/ubuntu/go/bin/evmd"

  # Claim rewards
  $evmd tx distribution withdraw-rewards $VALIDATOR_ADDRESS --commission --from $WALLET_NAME --chain-id $CHAIN_ID --gas 400000 -y
  echo "reward withdrawn"

  sleep 5

  # Get balance
  BALANCE=$($evmd query bank balances $($evmd keys show $WALLET_NAME -a) --output json | jq -r '.balances[0].amount')
  echo "BALANCE: $BALANCE"

  # Set minimum threshold to keep for gas
  MIN_BALANCE="5000000000000000000000"
  THRESHOLD="100000000000000000000000"

  # Delegate remaining balance (leave some for fees)
  if [ "$(echo "$BALANCE > $THRESHOLD" | bc)" -eq 1 ]; then
      AMOUNT=$(echo "$BALANCE - $MIN_BALANCE" | bc)

      $evmd tx staking delegate $VALIDATOR_ADDRESS ${AMOUNT}${DENOM} --from $WALLET_NAME --chain-id $CHAIN_ID --gas 300000 --gas-prices 10000000aedgen -y

      echo "staking delegation done"
  else
      echo "Not enough balance to delegate"
  fi
  ```

- **Make Script Executable**
  ```sh
  chmod +x auto_delegate.sh
  ```

- **Schedule via Cron**
  ```sh
  crontab -e
  # Add:
  0 0 * * * /path/to/auto_delegate.sh >> /path/to/delegate.log 2>&1
  ```

## 4. Set Up Failover Validator Node

- **Set Up Secondary Node**
  > Follow Section 1 to sync. Don’t create validator on it yet.

- **Monitor Main Validator**
  ```sh
  evmd query staking validator <validator-address>
  ```

- **Manual Failover**
  ```sh
  evmd tx slashing unjail --from <wallet-name> --chain-id <chain-id>
  ```

## 5. Automate Failover & Monitoring

- **Deploy Sentry Nodes**
  > Full nodes for redundancy. Connect them to the validator.

- **Create `monitor_validator.sh`**
  ```bash
  #!/bin/bash
  VALIDATOR="<validator-address>"
  THRESHOLD=3
  evmd="/home/ubuntu/go/bin/evmd"

  MISSED_BLOCKS=$($evmd query slashing signing-info $($evmd tendermint show-validator) --output json | jq -r '.missed_blocks_counter')

  if [ "$MISSED_BLOCKS" -ge "$THRESHOLD" ]; then
      echo "Validator down, switching to backup..."
      ssh user@backup-validator "evmd tx slashing unjail --from <wallet-name> --chain-id <chain-id>"
      curl -s "https://api.telegram.org/bot<BOT_TOKEN>/sendMessage?chat_id=<CHAT_ID>&text=Validator%20Down!"
  fi
  ```

- **Make Script Executable**
  ```sh
  chmod +x monitor_validator.sh
  ```

- **Schedule Monitoring (Every 5 mins)**
  ```sh
  crontab -e
  # Add:
  */5 * * * * /path/to/monitor_validator.sh >> /path/to/failover.log 2>&1
  ```

## 6. Real-Time Monitoring with Prometheus & Grafana (Optional)

- ✅ Install Prometheus & Grafana
- ✅ Set up Cosmos SDK exporter
- ✅ Connect Grafana dashboards to metrics
- ✅ Set alerts for downtime/missed blocks

## ✅ Final Setup Summary

You now have:
- A fully running Cosmos validator.
- Automated delegation of staking rewards.
- A secondary failover node.
- Monitoring with alerting (optional Telegram + Grafana).
