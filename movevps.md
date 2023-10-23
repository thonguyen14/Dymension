# Backup roller keys and configs on the OLD SERVER
```
# stop services
sudo systemctl stop da-light-client && journalctl -u da-light-client --no-pager -n 10

sudo systemctl stop sequencer && journalctl -u sequencer --no-pager -n 10

sudo systemctl stop relayer && journalctl -u relayer --no-pager -n 10

sudo systemctl disable da-light-client
sudo systemctl disable sequencer
sudo systemctl disable relayer

# create backup archive 
cd $HOME
mkdir -p $HOME/_tar

tar cvf $HOME/_tar/roller_keys_config_backup.tar \
  .roller/config.toml \
  .roller/da-light-node/keys/ \
  .roller/da-light-node/config.toml \
  .roller/hub-keys/ \
  .roller/relayer/config/ \
  .roller/relayer/keys/ \
  .roller/rollapp/config/ \
  .roller/rollapp/keyring-test/

# share tar
cd _tar
$(which python3) -m http.server 8150
```
## Prepare NEW server
# install roller binaries
```
# export ROLLER_RELEASE_TAG="v0.1.17-beta"
curl -L https://dymensionxyz.github.io/roller/install.sh | bash
```
# init roller
```
# init roller
roller config init rollapp denom

# remove new created keys
rm -r $HOME/.roller/da-light-node/keys
rm -r $HOME/.roller/hub-keys
rm -r $HOME/.roller/relayer/keys
rm -r $HOME/.roller/rollapp/keyring-test
rm -r $HOME/.roller/rollapp/config
```
# download backup
```
# download backup
cd $HOME
wget <OLD_SERVER_IP>:8150/roller_keys_config_backup.tar

# untar
tar xvf roller_keys_config_backup.tar
```
# configure the node to run as non aggregator
```
# correct paths (in case USER has changed)
sed -i "s|^Home *=.*|Home = \"${HOME}/.roller\"|" $HOME/.roller/config.toml
sed -i "s|^keyring_home_dir *=.*|keyring_home_dir = \"${HOME}/.roller/hub-keys\"|" $HOME/.roller/rollapp/config/dymint.toml

# configure the node to run as non aggregator
sed -i 's/^aggregator *=.*/aggregator = "false"/' $HOME/.roller/rollapp/config/dymint.toml
```
# create services
```
tee $HOME/da-light-client.service > /dev/null <<EOF
[Unit]
Description=da-light-client
After=network-online.target
[Service]
User=$USER
ExecStart=/usr/local/bin/roller da-light-client start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

tee $HOME/sequencer.service > /dev/null <<EOF
[Unit]
Description=sequencer
After=network-online.target
[Service]
User=$USER
ExecStart=/usr/local/bin/roller sequencer start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

tee $HOME/relayer.service > /dev/null <<EOF
[Unit]
Description=relayer
After=network-online.target
[Service]
User=$USER
ExecStart=/usr/local/bin/roller relayer start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF


sudo mv $HOME/da-light-client.service /etc/systemd/system/
sudo mv $HOME/sequencer.service /etc/systemd/system/
sudo mv $HOME/relayer.service /etc/systemd/system/

sudo systemctl enable da-light-client
sudo systemctl enable sequencer
sudo systemctl enable relayer

sudo systemctl daemon-reload
```
# start
```
# start light node
sudo systemctl start da-light-client && journalctl -u da-light-client --no-pager -n 10
tail -n 100 -f $HOME/.roller/da-light-node/light_client.log

# wait until light node finish syncing - check logs (wait until new head = sampled header)

# 2023-09-20T17:06:25.712Z        INFO    header/store    store/store.go:353      new head        {"height": 473749, "hash": "3AEB9B72344D45FAE79A86B5687"}
# 2023-09-20T17:06:25.938Z        INFO    das     das/worker.go:148       sampled header  {"type": "recent", "height": 473749, "hash": "3AEB9B72344D46865106EC65687", "square width": 64, "data root": "F30...E0", "finished (s)": 0.226491431}


# start sequencer
sudo systemctl start sequencer && journalctl -u sequencer --no-pager -n 10
tail -n 100 -f $HOME/.roller/rollapp/rollapp.log

# start relayer
sudo systemctl start relayer && journalctl -u relayer --no-pager -n 10
tail -n 100 -f $HOME/.roller/relayer/relayer.log
```
* Check sequencer logs:

syncTarget is the last block

e.g.: level=info msg="Syncing until target[current height 1334 syncTarget 1436]" module=BlockManager

Wait for sync *
# after synced successfully stop the node and set aggregator = "true" to start producing blocks
```
# stop services
sudo systemctl stop da-light-client && journalctl -u da-light-client --no-pager -n 10
sudo systemctl stop sequencer && journalctl -u sequencer --no-pager -n 10
sudo systemctl stop relayer && journalctl -u relayer --no-pager -n 10

# configure the node to run as aggregator (produce blocks)
sed -i 's/^aggregator *=.*/aggregator = "true"/' $HOME/.roller/rollapp/config/dymint.toml
```
# start 
```
# start light node
sudo systemctl start da-light-client && journalctl -u da-light-client --no-pager -n 10
tail -n 100 -f $HOME/.roller/da-light-node/light_client.log

# start sequencer
sudo systemctl start sequencer && journalctl -u sequencer --no-pager -n 10
tail -n 100 -f $HOME/.roller/rollapp/rollapp.log

# start relayer
sudo systemctl start relayer && journalctl -u relayer --no-pager -n 10
tail -n 100 -f $HOME/.roller/relayer/relayer.log
```
# check new blocks produced
```
curl http://localhost:26657/status | jq
```
** Thank to @JohnTranT for sharing **

