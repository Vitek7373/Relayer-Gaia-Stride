Обновляем и устанавливаем пакеты:

apt update && apt upgrade -y
apt install curl build-essential git wget jq make gcc tmux chrony -y

Устанавливаем go:

if ! [ -x "$(command -v go)" ]; then
  ver="1.18.2"
  cd $HOME
  wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
  rm -rf /usr/local/go
  tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
  rm "go$ver.linux-amd64.tar.gz"
  echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
  source ~/.bash_profile
fi

Устанавливаем релеер (актуальный релиз смотрим по ссылке):https://github.com/cosmos/relayer/releases

cd
git clone https://github.com/cosmos/relayer.git
cd relayer
git checkout v2.0.0
make install
cp $HOME/go/bin/rly /usr/local/bin


Скачиваем config

mkdir -p $HOME/.relayer/config
wget -O $HOME/.relayer/config/config.yaml https://raw.githubusercontent.com/maxbutenko/RLY/main/config.yaml

Конфиг в итоге в моем случае выглядит так:

global:
    api-listen-addr: :5183
    timeout: 10s
    memo: vitek#....
    light-cache-size: 20
chains:
    gaia:
        type: cosmos
        value:
            key: wallet
            chain-id: GAIA
            rpc-addr: http://127.0.0.1:26657
            account-prefix: cosmos
            keyring-backend: test
            gas-adjustment: 1.2
            gas-prices: 0.001uatom
            debug: true
            timeout: 10s
            output-format: json
            sign-mode: direct
    stride:
        type: cosmos
        value:
            key: wallet
            chain-id: STRIDE-TESTNET-2
            rpc-addr: http://127.0.0.1:26657
            account-prefix: stride
            keyring-backend: test
            gas-adjustment: 1.2
            gas-prices: 0.001ustrd
            debug: true
            timeout: 10s
            output-format: json
            sign-mode: direct
paths:
    gaia-stride:
        src:
            chain-id: GAIA
            client-id: 07-tendermint-0
            connection-id: connection-0
        dst:
            chain-id: STRIDE-TESTNET-2
            client-id: 07-tendermint-0
            connection-id: connection-0
        src-channel-filter:
            rule: allowlist
            channel-list:
                - channel-0
                - channel-1
                - channel-2
                - channel-3
                - channel-4

Открываем редактор:

nano /root/.relayer/config/config.yaml
Что нужно поменять на свои значения:

1) memo: vitek#.... - вписываем свое имя

2) rpc-addr: http://127.0.0.1:26657 адрес и порт ноды Gaia меняем на свой

3) rpc-addr: http://127.0.0.1:26657 адрес и порт ноды Stride меняем на свой

Настройка цены газа - на ваше усмотрение, я ставлю 0.001


Добавляем кошельки по мнемоник-фразе:

 rly keys restore stride wallet "мнемоник вашего кошелька Stride"
 rly keys restore gaia wallet "мнемоник кошелька Gaia"
Проверяем баланс на кошельках(должно быть больше 0, если пусто то запрашиваем в дискорде Stride):

 rly q balance stride
 rly q balance gaia
Проверяем правильность путей:

rly paths list
Ожидаем такой результат:

0: gaia-stride          -> chns(✔) clnts(✔) conn(✔) (GAIA<>STRIDE-TESTNET-2)


Создаем сервис:

tee /etc/systemd/system/rlyd.service > /dev/null <<EOF
[Unit]
Description=relayerGo
After=network-online.target
[Service]
User=$USER
ExecStart=/usr/local/bin/rly start gaia-stride -p events --home $HOME/.relayer
Restart=on-failure
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

Стартуем сервис:

systemctl daemon-reload
systemctl enable rlyd
systemctl restart rlyd
Смотрим логи:

journalctl -u rlyd -f -o cat
В логах кроме кучи warn и error (не обращаем внимания) должно быть такое:

lvl=info msg="Successful transaction" provider_type=cosmos chain_id=STRIDE-TESTNET-2 packet_src_channel=channel-0

Проверяем что пошли транзакции в эксплорере.

Полезные команды:

rly tx update-clients gaia-stride# Relayer-Gaia-Stride
