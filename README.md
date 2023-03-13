<h1 align="center">Quasar Testneti Kurulum Rehberi

## Selams, bugün Quasar testnetine katılıyor olacağız. Mainnete kısa bir süre kaldı, node kuranlara daha önce bir teşvik olmayacağı açıklanmıştı. Fakat bir admin birkaç gün önce bir ödül olabileceğini söyledi, snapshot alınmış da olabilir. Kurup kurmamak size kalmış. Sağ üstten yıldızlayıp forklamayı unutmayalım. Sorularınız için: [LossNode Chat](https://t.me/LossNode)

![image](https://user-images.githubusercontent.com/101462877/224657368-c9f463fb-7a38-463f-8074-dd37692f9d03.png)


## Sistem gereksinimleri:
NODE TİPİ | CPU     | RAM      | SSD     |
| ------------- | ------------- | ------------- | -------- |
| Testnet | 4          | 8         | 200  |

## Quasar için önemli linkler:
- [Website](https://quasar.fi/)
- [Explorer](https://quasar.explorers.guru)
- [Twitter](https://twitter.com/quasarfi)
- [Discord](https://discord.gg/quasarfi)

# Gerekli güncellemeleri ve kurulumları yapalım.

```bash
sudo su
cd
sudo apt update && sudo apt upgrade -y
```
```bash
sudo apt install curl git jq lz4 build-essential -y
```

# Go yükleyin.

```bash
ver="1.19.5" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```

# Cosmovisor ile binary dosyalarını yükleyin ve kurun.

```bash
mkdir -p $HOME/.quasarnode/cosmovisor/genesis/bin
wget -O $HOME/.quasarnode/cosmovisor/genesis/bin/quasard https://github.com/quasar-finance/binary-release/raw/main/v0.0.2-alpha-11/quasarnoded-linux-amd64
chmod +x $HOME/.quasarnode/cosmovisor/genesis/bin/*
ln -s $HOME/.quasarnode/cosmovisor/genesis $HOME/.quasarnode/cosmovisor/current
sudo ln -s $HOME/.quasarnode/cosmovisor/current/bin/quasard /usr/local/bin/quasard
```

# Cosmovisor yükleyin.

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.4.0
```

# Servis dosyası oluşturun.

```bash
sudo tee /etc/systemd/system/quasard.service > /dev/null << EOF
[Unit]
Description=quasar-testnet node service
After=network-online.target
[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.quasarnode"
Environment="DAEMON_NAME=quasard"
Environment="UNSAFE_SKIP_BACKUP=true"
[Install]
WantedBy=multi-user.target
EOF
```


# Node'u başlatın.

```bash
quasard config chain-id qsr-questnet-04
quasard config keyring-backend test
quasard config node tcp://localhost:48657
```

```bash
quasard init <MONIKERADINIZ> --chain-id qsr-questnet-04
```

# Genesis ve addrbook dosyaları, seed/peer, prunning ve min gas price ayarları.

```bash
curl -Ls https://snapshots.kjnodes.com/quasar-testnet/genesis.json > $HOME/.quasarnode/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/quasar-testnet/addrbook.json > $HOME/.quasarnode/config/addrbook.json
```

```bash
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.quasarnode/config/app.toml
```
```bash
indexer="null" && \
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.quasarnode/config/config.toml
```
```bash
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0uqsr\"|" $HOME/.quasarnode/config/app.toml
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@quasar-testnet.rpc.kjnodes.com:48659\"|" $HOME/.quasarnode/config/config.toml
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:48658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:48657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:48060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:48656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":48660\"%" $HOME/.quasarnode/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:48317\"%; s%^address = \":8080\"%address = \":48080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:48090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:48091\"%; s%^address = \"0.0.0.0:8545\"%address = \"0.0.0.0:48545\"%; s%^ws-address = \"0.0.0.0:8546\"%ws-address = \"0.0.0.0:48546\"%" $HOME/.quasarnode/config/app.toml
```
# Servis'i başlatın.

```bash
sudo systemctl daemon-reload
systemctl restart systemd-journald.service
sudo systemctl enable quasard
sudo systemctl restart quasard
sudo journalctl -fu quasard -o cat
```


# Daha hızlı sync olmak için snapshot (opsiyonel).

```bash
curl -L https://snapshots.kjnodes.com/quasar-testnet/snapshot_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.quasarnode
[[ -f $HOME/.quasarnode/data/upgrade-info.json ]] && cp $HOME/.quasarnode/data/upgrade-info.json $HOME/.quasarnode/cosmovisor/genesis/upgrade-info.json

sudo systemctl restart quasard
sudo journalctl -fu quasard -o cat
```

# Sync durumunu kontrol etmek için:

```bash
quasard status 2>&1 | jq .SyncInfo
``` 

![image](https://user-images.githubusercontent.com/101462877/224665988-ba6d835b-0134-4df2-b8b6-d1a74ef54de0.png)


# Sync olduktan sonra cüzdan oluşturalım.
```bash
quasard keys add <CÜZDANADI>
``` 
Var olan bir cüzdanı kullanmak isterseniz:

```bash
quasard keys add <CÜZDANADI> --recover
``` 

# [Discord](https://discord.gg/quasarfi)'a giderek faucet alın.

<img width="1127" alt="Ekran Resmi 2023-03-13 12 50 26" src="https://user-images.githubusercontent.com/101462877/224666648-b68b1844-e5b2-4d5b-9765-2851af794e02.png">


# Validator oluşturun:

```bash
quasard tx staking create-validator \
  --amount=1000000uqsr \
  --pubkey=$(quasard tendermint show-validator) \
  --moniker=<MONIKERADINIZ> \
  --details="LossNode Community" \
  --identity="" \
  --website="lossnode.info" \
  --chain-id="qsr-questnet-04" \
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1" \
  --from=<CUZDANADINIZ>
  -y
``` 


# Bazı komutlar:

Log kontrolü

```bash
sudo journalctl -fu quasard -o cat
```


Servisi durdurma

```bash
sudo systemctl stop quasard
```

Servisi tekrar başlatma

```bash
sudo systemctl restart quasard
```

Token delege etme

```bash
quasard tx staking delegate $(quasard keys show wallet --bech val -a) 1000000uqsr --from <CÜZDANADI> --chain-id qsr-questnet-04 --gas-adjustment 1.4 --gas auto --gas-prices 0.025uqsr -y
```

Validator düzenleme

```bash
quasard tx staking edit-validator \
  --moniker=$NODENAME \
  --identity="<KEYBASE ID'NİZ>" \
  --website="<WEBSİTE LİNKİ>" \
  --details="AÇIKLAMA" \
  --chain-id=qsr-questnet-04 \
  --from=<CÜZDANADI>
``` 


# Node silmek için:

```bash
sudo systemctl stop quasard && \
sudo systemctl disable quasard && \
rm /etc/systemd/system/quasard.service && \
sudo systemctl daemon-reload && \
cd $HOME && \
rm -rf .quasarnode && \
rm -rf $(which nibid)
```

