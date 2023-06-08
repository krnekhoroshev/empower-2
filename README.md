![242064548-f5e9add1-5b55-40d2-83dd-539cbf64c266](https://github.com/molla202/empower-2/assets/91562185/1191c65e-441f-4d71-9acc-5ea06391b7ed)

<h1 align="center"> Empower Chain </h1>

>  - [Empower Discord](https://discord.gg/Zs3GMUhg)

<h1 align="center"> Установка ноды </h1>










# системные требования
```
4 CPU
8 RAM
200 SSD
```
## обновляем репозитории, устанавливаем необходимые утилиты
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```



## Устанавливаем GO
```
cd $HOME
! [ -x "$(command -v go)" ] && {
VER="1.20.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
}
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

## Устанавливаем бинарники
```
cd $HOME
rm -rf empowerchain
git clone https://github.com/EmpowerPlastic/empowerchain
cd empowerchain
git checkout v1.0.0-rc1
cd chain
make install
```
## Инициализируем ноду (<name_moniker> свое значение)
```
empowerd config keyring-backend os
empowerd config chain-id circulus-1
empowerd init <name_moniker> --chain-id circulus-1
```
## Скачиваем genesis и addrbook
```
wget -O $HOME/.empowerchain/config/genesis.json https://testnet-files.itrocket.net/empower/genesis.json
wget -O $HOME/.empowerchain/config/addrbook.json https://testnet-files.itrocket.net/empower/addrbook.json
```
## добавляем seeds и peers
```
SEEDS="c597ec01e412d6e0f62c6f5501224b7fb8393912@empower-testnet-seed.itrocket.net:16656"
PEERS="c413d3d16e250ddbd8f8d495204b2de46ef36b63@empower-testnet-peer.itrocket.net:16656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.empowerchain/config/config.toml
```
## Смена портов для нескольких нод
```
sed -i.bak -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:56658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:56657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:6065\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:56656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":56660\"%" $HOME/.empowerchain/config/config.toml
sed -i.bak -e "s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:9590\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:9591\"%; s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:5327\"%" $HOME/.empowerchain/config/app.toml
sed -i.bak -e "s%^node = \"tcp://localhost:26657\"%node = \"tcp://localhost:56657\"%" $HOME/.empowerchain/config/client.toml
external_address=$(wget -qO- eth0.me)
sed -i.bak -e "s/^external_address *=.*/external_address = \"$external_address:56656\"/" $HOME/.empowerchain/config/config.toml

```
## настраиваем прунинг
```
sed -i -e "s/^pruning *=.*/pruning = \"nothing\"/" $HOME/.empowerchain/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.empowerchain/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.empowerchain/config/app.toml
```
## настраиваем параметры
```
sed -i 's/minimum-gas-prices =.*/minimum-gas-prices = "0.0umpwr"/g' $HOME/.empowerchain/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.empowerchain/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.empowerchain/config/config.toml
```
## создаем сервисный файл
```
sudo tee /etc/systemd/system/empowerd.service > /dev/null <<EOF
[Unit]
Description=Empower node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.empowerchain
ExecStart=$(which empowerd) start --home $HOME/.empowerchain
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```
## Снэпшот
```
empowerd tendermint unsafe-reset-all --home $HOME/.empowerchain --keep-addr-book

SNAP_NAME=$(curl -s https://snapshots2-testnet.nodejumper.io/empower-testnet/info.json | jq -r .fileName)
curl "https://snapshots2-testnet.nodejumper.io/empower-testnet/${SNAP_NAME}" | lz4 -dc - | tar -xf - -C "$HOME/.empowerchain"

sudo systemctl daemon-reload
sudo systemctl enable empowerd
sudo systemctl restart empowerd && sudo journalctl -u empowerd -f -o cat

```
## проверить синхронизацию, если вывод "FALSE" нода поднялась можно создавать валидатора
```
empowerd status 2>&1 | jq .SyncInfo.catching_up
```
## создать кошелек
```
empowerd keys add wallet
```
## восстановить старый кошелек
```
empowerd keys add wallet --recover
```
## список кошельков
```
empowerd keys list
```


## запрашиваем токены в дискорде, ждем сонхронизацию ноды и создаем валидатора (<NODE_MONIKER> меняем на свое)
```
empowerd tx staking create-validator \
--amount=9000000umpwr \
--pubkey=$(empowerd tendermint show-validator) \
--moniker="<NODE_MONIKER>" \
--chain-id=circulus-1 \
--commission-rate=0.1 \
--commission-max-rate=0.2 \
--commission-max-change-rate=0.05 \
--min-self-delegation=1 \
--fees=10000umpwr \
--from=wallet \
-y

```



# Транзакции
### собрать реварды со всех валтдаторов
```
empowerd tx distribution withdraw-all-rewards --from wallet --chain-id circulus-1 --gas auto --gas-adjustment 1.5
```
### собрать реварды с конкретного валидатора
```
empowerd tx distribution withdraw-rewards $VALOPER_ADDRESS --from wallet --commission --chain-id circulus-1 --gas auto --gas-adjustment 1.5 -y
```
### проверить баланс
```
empowerd query bank balances <WALLET_ADDRESS>
```
### заделегировать себе
```
empowerd tx staking delegate $(empowerd keys show wallet --bech val -a) 1000000umpwr --from wallet --chain-id circulus-1 --gas auto --gas-adjustment 1.5 -y
```
### заделегировать в другой валидатор
```
empowerd tx staking delegate <TO_VALOPER_ADDRESS> 1000000umpwr --from wallet --chain-id circulus-1 --gas auto --gas-adjustment 1.5 -y
```
### ределигирование с одного на другой валидатор 
```
empowerd tx staking redelegate <src-validator-addr> <dst-validator-addr> 1000000umpwr --from wallet --chain-id circulus-1 --gas auto --gas-adjustment 1.5 -y
```
### Unbond (учитывайте время анбонд лока токенов)
```
empowerd tx staking unbond $(empowerd keys show wallet --bech val -a) 1000000umpwr --from wallet --chain-id circulus-1 --gas auto --gas-adjustment 1.5 -y
```
### отправить токены
```
empowerd tx bank send wallet <TO_WALLET_ADDRESS> 1000000umpwr --gas auto --gas-adjustment 1.5 -y
```



# перезагрузить сервис
```
sudo systemctl restart empowerd
```
# просмотр логов
```
journalctl -u empowerd -f -o cat
```

# удалить ноду
```
sudo systemctl stop empowerd
sudo systemctl disable empowerd
sudo rm -rf /etc/systemd/system/empowerd.service
sudo rm $(which empowerd)
sudo rm -rf $HOME/.empowerchain
sed -i "/EMPOWER_/d" $HOME/.bash_profile
```


