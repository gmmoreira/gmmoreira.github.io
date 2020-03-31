---
title:      Adaptador TP-Link Archer T4U v3 no Ubuntu 19.10
date:       2020-03-31 11:50:00 -0300
lang:       pt
categories: linux driver ubuntu 19.10 tp-link archer t4u
description: Como instalar driver para o adaptador TP-Link Archer T4U v3
---

Recentemente tive a necessidade de instalar um adaptador wi-fi no meu desktop uma vez que o roteador está na sala e o PC em um escritório no andar de cima. O roteador utilizado fornece redes 2.4 GHz e 5 GHz. Primeiramente utilizei um adaptador wi-fi que já tinha, compatível apenas com a rede 2.4 GHz e padrão 802.11n de até 150 Mbps. Utilizando este adaptador observei que não seria possível utilizar toda a velocidade contratada, ficando com uma velocidade cerca de 80% menor. Em seguida realizei outro teste, desta vez com o meu celular conectado na rede de 5 GHz e conectado ao computador usando o Tethering USB. Utilizando desta forma vi que era possível atingir a velocidade máxima contratada de Internet e ter latências bem baixas. Sendo assim comecei a busca por um novo adaptador, compatível com o 5 GHz e padrão 802.11ac (também conhecido como Wi-Fi 5).

Na faixa de preço que estava procurando o que melhor atendeu as minhas necessidades foi o TP-Link Archer T4U v3. Cheguei a encontrar alguns com preços menores como o Intelbras A1200 e D-Link DWA-182, mas os maiores diferenciais para o TP-Link foram a antena com mais ajustes e driver linux fornecido no próprio site. O chip utilizado é da Realtek, que também fornece drivers linux ainda mais atualizados do que a próprio TP-Link.

Procurando no Google não encontrei facil um tutorial, então segue um passo-a-passo do que eu fiz para que o adaptador funcionasse corretamente no Ubuntu 19.10. Não garanto, mas é bem possível que esses passos também funcionem para outros linux com kernel 5.3 ou similar.

Encontrei o seguinte repositório no Github: <https://github.com/cilynx/rtl88x2bu>. O usuário tem mantido atualizado com os drivers mais recentes por parte da Realtek. Sendo assim, o primeiro passo é clonar o projeto:

```bash
cd ~/.local/src
git clone https://github.com/cilynx/rtl88x2bu.git
```

Gosto de clonar código-fonte de terceiros no meu diretório `.local/src`.

Uma vez que foi feito o clone do projeto, não tem muito segredo, apenas segui os passos do próprio README:

```bash
cd rtl88x2bu
VER=$(sed -n 's/\PACKAGE_VERSION="\(.*\)"/\1/p' dkms.conf)
sudo rsync -rvhP ./ /usr/src/rtl88x2bu-${VER}
sudo dkms add -m rtl88x2bu -v ${VER}
sudo dkms build -m rtl88x2bu -v ${VER}
sudo dkms install -m rtl88x2bu -v ${VER}
sudo modprobe 88x2bu
```

Uma vez feito isso já era possível listar e conectar nas redes utilizando as configurações Wi-Fi do GNOME.

## Resolvendo o problema de conexão que falha
Após alguns testes de conectar e desconectar das redes, observei que em algum momento não era mais possível conectar nas redes, o gerenciador tentava mas falhava. Após alguma pesquisa descobri que a partir do Ubuntu 17.04 o Network-Manager possui um recurso habilitado por padrão que randomiza o MAC Address dos dispositivos Wi-Fi a cada conexão. É uma medida de privacidade, voltada a quem utiliza redes publicas, mas que gera conflitos com alguns dispositivos (que é o caso).

Para resolver isso, criei o arquivo `/etc/NetworkManager/conf.d/100-disable-wifi-mac-randomization.conf`com o seguinte conteúdo:

```
[device-wlxd037456e906f]
match-device=wlxd037456e906f
wifi.scan-rand-mac-address=no
wifi.cloned-mac-address=preserve
ethernet.cloned-mac-address=preserve
```

O nome do dispositivo pode ser obtido através do comando:
```bash
nmcli -f GENERAL.DEVICE device show
```

A saída vai ser algo parecido com isso:
```bash
GENERAL.DEVICE:                         wlxd037456e906f

GENERAL.DEVICE:                         eno1

GENERAL.DEVICE:                         lo
```

Fontes e links:

<https://github.com/cilynx/rtl88x2bu>

<http://www.wolfteck.com/2018/02/22/wsky_1200mbps_wireless_usb_wifi_adapter/>

<http://en.techinfodepot.shoutwiki.com/wiki/TP-LINK_Archer_T4U_v3>

<https://blog.muench-johannes.de/networkmanager-disable-mac-randomization-314>
