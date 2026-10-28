# worrell-testnet-2 Validator Guide
<img width="400" height="400" alt="image" src="https://github.com/user-attachments/assets/389798f7-c990-4fb7-8399-53c27339a202" />


> Unofficial community guide to joining the worrell-testnet-2 genesis ceremony by submitting a gentx. It covers the server setup, running next to other nodes on the same machine, creating the gentx and opening the pull request. The official guide lives in the `worrell-testnet-2` folder of https://github.com/worrellchain/networks and always takes precedence. Commands were tested on an Ubuntu server. Use at your own risk and double-check everything against the official guide.

## Contents

- [Key dates](#key-dates)
- [Network parameters](#network-parameters)
- [Setup approach](#setup-approach)
- [Before you start: set your variables](#before-you-start-set-your-variables)
- [Step 1: Check which ports are free](#step-1-check-which-ports-are-free)
- [Step 2: Download the v0.2.0 binary](#step-2-download-the-v020-binary-into-its-own-folder)
- [Step 3: Alias, init and pre-genesis](#step-3-create-the-alias-init-the-node-and-add-the-pre-genesis-file)
- [Step 4: Move the node to its own ports](#step-4-move-the-node-to-its-own-ports)
- [Step 5: Operator account](#step-5-operator-account)
- [Step 6: Create the gentx](#step-6-create-the-gentx)
- [Step 7: Back up the consensus key](#step-7-back-up-the-consensus-key)
- [Step 8: Submit the gentx with a pull request](#step-8-submit-the-gentx-with-a-pull-request)
- [Step 9: Open the P2P port in the firewall](#step-9-open-the-p2p-port-in-the-firewall)
- [Step 10: Point client.toml to the right node](#step-10-point-clienttoml-to-the-right-node)
- [After the genesis is published](#after-the-genesis-is-published)
- [Troubleshooting](#troubleshooting)
- [Final notes](#final-notes)

## Key dates

| Step | Date (UTC) |
|---|---|
| Gentx window opens | October 27, 2026, 10:00 |
| Gentx window closes | November 6, 2026, 10:00 |
| Final genesis published (with sha256 and peer list) | November 7, 2026 |
| Network start | November 10, 2026, 08:00 |
| worrell-testnet-1 closes | November 24, 2026 |

## Network parameters

| Parameter | Value |
|---|---|
| Chain ID | worrell-testnet-2 |
| Denom | uworrell (1 WORRELL = 1,000,000 uworrell) |
| Binary | v0.2.0 |
| Amount credited to each gentx account | 500 WORRELL |
| Self-delegation range | 1 to 500 WORRELL (495 suggested) |
| Unbonding period | 3 days |
| Minimum commission | 5% |
| Minimum gas price | 0.025uworrell |
| Maximum validators | 100 |

## Setup approach

If your server already runs other nodes, including a worrell-testnet-1 node, keep testnet-2 completely separate so nothing existing breaks:

- Separate binary: `~/worrell2/bin/worrelld` (v0.2.0). Never overwrite the binary your other nodes use.
- Separate home: `~/.worrell-testnet-2` (a testnet-1 node uses `~/.worrell` by default).
- Separate ports: this guide uses the 45xxx range, but any free range works.
- Separate systemd service (after genesis).

The commands below use a `w2` alias that is created in Step 3. It is short for the v0.2.0 binary plus the testnet-2 home.

## Before you start: set your variables

Replace the placeholders with your own values:

```bash
export MONIKER="<your-moniker>"
export KEY_NAME="<your-key-name>"
export SERVER_IP="<your-server-public-ip>"
```

Find your public IP with `curl -4 ifconfig.me`. The variables only live in the current shell session, so set them again if you open a new one.

## Step 1: Check which ports are free

List every listening port and the process behind it:

```bash
ss -tlnp | awk 'NR>1{n=split($4,a,":"); print a[n], $NF}' | sort -n | uniq
```

Pick a range nobody uses. This guide uses 45xxx. Confirm the ports are free:

```bash
ss -tlnp | grep -E ':(45656|45657|45658|45060|45317|45090|45091)\b'
```

The output must be empty. If another node already uses part of this range, choose a different one and adjust Step 4 accordingly.

## Step 2: Download the v0.2.0 binary into its own folder

```bash
mkdir -p ~/worrell2/bin && cd ~/worrell2
curl -L -o v0.2.0_linux_amd64.tar.gz https://github.com/worrellchain/worrell/releases/download/v0.2.0/v0.2.0_linux_amd64.tar.gz
sha256sum v0.2.0_linux_amd64.tar.gz
tar -xzf v0.2.0_linux_amd64.tar.gz -C ~/worrell2/bin
~/worrell2/bin/worrelld version
```

Expected sha256 for linux amd64: `1ccccd39d57467236a2f64ee4809fee4de3cc3f1bc02c47e3e22aa9c6b414c76`. The `version` output must be `v0.2.0`. For other platforms, use the tarball and checksum listed in the official guide.

## Step 3: Create the alias, init the node and add the pre-genesis file

We use an alias instead of adding `~/worrell2/bin` to `PATH`. A PATH change would send every `worrelld` command on the server to v0.2.0 and mix it up with other nodes. The alias only affects testnet-2:

```bash
cat >> ~/.bashrc << 'EOF'

# worrell testnet-2
export T2_HOME=$HOME/.worrell-testnet-2
alias w2='$HOME/worrell2/bin/worrelld --home $T2_HOME'
EOF
source ~/.bashrc
w2 version
```

Now init the node and download the published pre-genesis file:

```bash
w2 init "$MONIKER" --chain-id worrell-testnet-2 --default-denom uworrell
curl -s https://raw.githubusercontent.com/worrellchain/networks/main/worrell-testnet-2/pre-genesis.json -o $T2_HOME/config/genesis.json
sha256sum $T2_HOME/config/genesis.json
```

Expected sha256: `f5231dc924c762bdbe5aa0605ed6472288c9b71f4b175015e0559edccdd96b94`.

## Step 4: Move the node to its own ports

Run this once and before creating the gentx, because the P2P port is written into the gentx.

```bash
sed -i 's|26657|45657|; s|26656|45656|; s|26658|45658|; s|6060|45060|' $T2_HOME/config/config.toml
sed -i 's|1317|45317|; s|:9090|:45090|; s|:9091|:45091|' $T2_HOME/config/app.toml
grep -n -E "45[0-9]{3}" $T2_HOME/config/config.toml $T2_HOME/config/app.toml
```

| Purpose | Port |
|---|---|
| P2P | 45656 |
| RPC | 45657 |
| ABCI (proxy_app) | 45658 |
| pprof | 45060 |
| API | 45317 |
| gRPC | 45090 |

The grep output should show these six values. A match in a comment line is just an example address and can be ignored.

## Step 5: Operator account

Restore your existing wallet with its mnemonic, or create a new one by removing `--recover` and writing down the mnemonic it prints. Never share your mnemonic and only type it into your own terminal.

```bash
w2 keys add $KEY_NAME --recover
w2 keys show $KEY_NAME -a
w2 genesis add-genesis-account $(w2 keys show $KEY_NAME -a) 500000000uworrell
```

`add-genesis-account` only changes your local genesis and is needed to build the gentx. Without it the gentx command fails with an "account does not exist" error.

## Step 6: Create the gentx

```bash
w2 genesis gentx $KEY_NAME 495000000uworrell --chain-id worrell-testnet-2 --moniker "$MONIKER" --commission-rate 0.05 --commission-max-rate 0.20 --commission-max-change-rate 0.01 --min-self-delegation 1000000 --ip $SERVER_IP --p2p-port 45656
```

Things to watch:

- Self-delegation must be between 1 and 500 WORRELL. 495 WORRELL leaves a few WORRELL liquid for fees.
- `--chain-id` must be exactly `worrell-testnet-2`. A gentx signed for another chain ID prevents the network from starting and gets removed from the genesis.
- The commission rate must be at least 0.05.
- `--ip` and `--p2p-port` end up in the gentx memo and are published as your peer address. Use the address other nodes can reach and the P2P port you really use (45656).

Check the result:

```bash
cat $T2_HOME/config/gentx/gentx-*.json | jq '.body.messages[0] | {moniker: .description.moniker, validator_address, value}'
jq -r '.body.memo' $T2_HOME/config/gentx/gentx-*.json
```

The `value` must show `uworrell` and `495000000`. The memo must look like `<node id>@<your-server-ip>:45656`.

## Step 7: Back up the consensus key

The consensus key inside your gentx comes from `priv_validator_key.json`. If you lose this file or delete the home folder, the gentx becomes useless.

```bash
cp $T2_HOME/config/priv_validator_key.json ~/priv_validator_key_testnet2.json.bak
```

Also copy the backup somewhere safe outside the server, and never share it.

## Step 8: Submit the gentx with a pull request

1. Fork https://github.com/worrellchain/networks on GitHub.
2. In your fork, choose **Add file, Create new file** and name it `worrell-testnet-2/gentx/<your-moniker>.json`. The folders are created automatically.
3. Run `cat $T2_HOME/config/gentx/gentx-*.json` on the server, copy the single-line JSON and paste it into the file.
4. The file is signed, so a single wrong character makes it invalid. To verify the paste before committing, run `cat > /tmp/check.json` on the server, paste the same JSON, press Enter and then Ctrl+D, and run:

   ```bash
   cmp /tmp/check.json $T2_HOME/config/gentx/gentx-*.json && echo SAME
   ```

5. Use `gentx: <your-moniker>` as the commit message. You can commit to a new branch (GitHub names it in the `patch-1` style automatically) or directly to your fork's `main`. Both give the same result, the branch name is not checked.
6. Important: the compare page opens inside your own fork by default. A pull request opened there goes to your fork and the team never sees it. Click **compare across forks** and set base repository to `worrellchain/networks`, base to `main`, head repository to your fork and compare to the branch with your gentx.
7. You should see "Able to merge", 1 commit and 1 changed file. Set the title to `gentx: <your-moniker>` and click **Create pull request**.

Automated checks may not run on the repository, and gentx files are likely processed after the window closes. According to the official guide, a pull request that fails validation is closed with a reason and you can open a new one before the window closes. Only one gentx per account is accepted, so do not open a second pull request for the same account.

## Step 9: Open the P2P port in the firewall

```bash
ufw allow 45656/tcp
ufw status | grep 45656
```

You should see one IPv4 and one IPv6 line. If your hosting provider has a separate firewall panel, open 45656/tcp there as well.

## Step 10: Point client.toml to the right node

The `client.toml` of the new home points to `localhost:26657` by default. If another node on the same machine uses the default ports, `w2 status` will return that node's data. Reading is harmless, but you do not want to send a transaction to the wrong network. Fix it:

```bash
sed -i 's|^node = .*|node = "tcp://localhost:45657"|' $T2_HOME/config/client.toml
sed -i 's|^chain-id = .*|chain-id = "worrell-testnet-2"|' $T2_HOME/config/client.toml
grep -n "^node\|^chain-id" $T2_HOME/config/client.toml
```

Then run `w2 status`. Your testnet-2 node is not running yet, so you should get `connection refused` on port `45657`. If you see data from another network, the setting is wrong.

## After the genesis is published

1. **November 7:** download the final genesis and compare its hash with the one announced on Telegram and in GitHub Discussions:

   ```bash
   curl -s https://raw.githubusercontent.com/worrellchain/networks/main/worrell-testnet-2/genesis.json -o $T2_HOME/config/genesis.json
   sha256sum $T2_HOME/config/genesis.json
   ```

2. In `config.toml`, add the founder peer from the official guide and the peer list published with the genesis to `persistent_peers`. At the time of writing the founder peer is `4ae3601a733c88a1b8f377774ff197dbd3e377c6@144.91.82.147:26656`. In `app.toml`, set `minimum-gas-prices = "0.025uworrell"`.
3. Do not touch `priv_validator_key.json`. It must be the key your gentx was created with.
4. Create a separate systemd service. Aliases do not work in services, so call the binary by its full path. Template, adjust it to your setup:

   ```ini
   [Unit]
   Description=Worrell testnet-2 node
   After=network-online.target

   [Service]
   User=<your-user>
   ExecStart=/home/<your-user>/worrell2/bin/worrelld start --home /home/<your-user>/.worrell-testnet-2
   Restart=on-failure
   RestartSec=5
   LimitNOFILE=65535

   [Install]
   WantedBy=multi-user.target
   ```

5. Start the node a few minutes before **November 10, 08:00 UTC**. A node started before genesis time simply waits and then produces the first block together with the other genesis validators.
6. After the start, check:

   ```bash
   w2 status | jq '.sync_info | {latest_block_height, catching_up}'
   w2 q staking validator $(w2 keys show $KEY_NAME --bech val -a)
   ```

   The validator should be `BOND_STATUS_BONDED`.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `signature verification failed` at start, chain does not move | A gentx in the genesis was signed for another chain ID or account. The coordinator removes it and republishes. Do not start with an unpublished genesis. |
| Genesis sha256 does not match | Download the file again from the repository and do not edit it. |
| `account ... does not exist` while creating the gentx | Run `add-genesis-account` first, with the same home. |
| `the provided denom ... does not match` | Run `init` with `--default-denom uworrell`. |
| `Address already in use` | Another node uses the same port. Check the port list from Step 1. |
| `w2 status` shows another network's data | `client.toml` still points to port 26657. Apply Step 10. |
| `w2 status` returns `connection refused` on 45657 | Your testnet-2 node is not running yet, which is expected before genesis. |

## Final notes

- If you also run a worrell-testnet-1 node, keep it running until November 24. Its activity is recorded until then. Do not touch its home folder or binary.
- Never share your mnemonic or `priv_validator_key.json`.
- Follow the Telegram group t.me/worrellvalidators and GitHub Discussions for dates and announcements. Only the times in the official guide and announcements are authoritative.
