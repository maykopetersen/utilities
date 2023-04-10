# Utilities

- vimrc: configuração do vim para usar o tema [`molokai`](https://github.com/tomasr/molokai).
- HyperV_Conf: scripts usados no Windows e no Linux para automatizar a criação de um ambiente virtualizado para estudo.
  - `createVMFromSnap.ps1`: usado no **WINDOWS**: cria uma VM a partir de um snapshot tirado de uma VM modelo
  - `router.sh`: usado no **LINUX**: configura o servidor Linux para ser usado como roteador e DNS Server
  - `files`:
    - `nftables.conf`: a configuração do Firewall NFTables
    
    
## `createVMFromSnap.ps1`: How to use?
**Detalhes**: Basicamente, o script cria um snapshot de uma VM determinada pela variável `$vmExpName` e usa-o para criar uma nova VM determinada pela variável `$vmImpName`.
Execute, no PowerShell COMO ADMINISTRADOR, o comando abaixo e leia o **DESCRIPTION**:
```sh
Get-Help .\createVMFromSnap.ps1 -full
```
Depois de ler a ajuda do programa e alterar as variáveis desejadas, execute o script:
```sh
.\createVMFromSnap.ps1
```
**OBSERVAÇÃO**: tenha certeza que a pasta destino, onde os arquivos da VM importada estarão, NÃO está compactada.


## `router.sh`: How to use?

**OBSERVAÇÃO**: o script DEPENDE do arquivo `files/nftables.conf`.

Uma VM chamada "Router" no Hyper-V deve ser gerada a partir do script `createVMFromSnap.ps1`. Dessa forma, ela virá com apenas uma interface de rede (Default Switch).

1. [Adicione uma interface de rede EXTERNA ao Hyper-V](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/get-started/create-a-virtual-switch-for-hyper-v-virtual-machines).
```sh
# Utilizando o PowerShell: 
New-VMSwitch -Name "External - Virtual Switch" -SwitchType External
```
2. Inicie a VM e desligue-a (será apenas para gerar um MAC aleatório, que será configurado no próximo passo).
3. Mude a interface padrão que veio na VM para a interface EXTERNA criada; configure-a com MAC ESTÁTICO e clique em APLICAR.
4. Adicione uma interface de rede na Default Switch em uma vlan específica (ex.: vlan 2). Assim, a VM terá duas interfaces de rede (uma externa e outra interna).
5. Na interface 1 (geralmente `eth0`), configure um IP FIXO que se comunique com a internet, alterando, adequadamente, o arquivo `/etc/systemd/network/eth0.network`. Uma ideia para saber qual IP FIXO usar, é configurar a interface primeiro para DHCP, obter seus dados e, depois, alterar a interface para IP FIXO colocando os dados obtidos anteriormente. Você pode fazer isso executando a sequência de comandos abaixo:
```sh
interface=$(networkctl list --no-legend | awk 'NR==2{print $2}')
echo -e "[Match]\nName="$interface"\n\n[Network]\nDHCP=ipv4" > /etc/systemd/network/eth0.network
systemctl restart systemd-networkd.service
systemctl restart systemd-resolved.service
# DHCP IP and subnet mask
ip addr
# DHCP gateway
ip route
# DHCP DNS Server
resolvectl status
# "Configure IP and MASK"
ipAdd=$(ip addr | grep $interface | grep inet | awk '{print $2}')
# "Configure Gateway"
ipRoute=$(ip route | grep $interface | grep default | awk '{print $3}')
# "Configure DNS Server"
dnsServ=$(resolvectl status $interface | grep "DNS Servers" | awk '{print $3}')
# "Configure Interface"
echo -e "[Match]\nName="$interface"\n\n[Network]\nAddress=$ipAdd\nGateway=$ipRoute\nDNS=$dnsServ" > /etc/systemd/network/eth0.network
systemctl restart systemd-networkd.service
systemctl restart systemd-resolved.service
# FIXED IP and subnet mask
ip addr 
# FIXED gateway
ip route
# FIXED DNS Server
resolvectl status
```
6. Depois de colocar o arquivo `nftables.conf` no diretório `files` (ler abaixo), coloque o script `router.sh` nessa VM e execute-o: ele habilitará o roteamento, configurará o NFTables Router e também, configurará um IP (172.16.1.1/16) para a interface `eth1`.


## `nftables.conf`: How to use?
 - O Firewall é BEM restrito. Cada VM adicionada ao seu ambiente precisa estar cadastrada no *named set* `allowed_hosts`.
 - Se uma porta acima da **1024** precisar ser liberada, pode ser incluída no *named set* `allowed_ports` ou adicionada na *chain* `inbound_private`. Ex.:
```sh
ip saddr ip.da.nova.maquina tcp dport 8080 accept
```
 - Altere o valor da variável `IP_HOST` para o IP de seu sistema operacional (o HOST, que tem o Hyper-V instalado e que gerencia as VMs Linux).
   - Por padrão, somente o SSH está aberto na máquina **router** para o *source address* do Host. Se você desejar que teu host tenha acesso irrestrito à VM **router**, edite a *chain* `inbound_world` da seguinte forma:
```sh
ip saddr $IP_HOST accept
```
 - Para aplicar as configurações execute:
```sh
# nft -f /etc/nftables.conf
```
 - Para ver as regras do firewall em uso no momento:
```sh
# nft list ruleset
```


## `pihole.sh`: How to use?

**OBSERVAÇÃO**: o script DEPENDE do arquivo `dockerInstall.sh` e do arquivo `files/pihole.yaml`.

O `pihole.sh` usará o arquivo `dockerInstall.sh` para instalar o [docker](https://docs.docker.com/engine/install/debian/) e o [docker compose](https://docs.docker.com/compose/install/linux/). Depois, instalará o `pihole.sh` como container no diretório definido pela variável `$piholeDir`. O IP de publicação será o presente na interface `eth1` (ou equivalente) que é definido pela variável `$ip`.

Ao executar o script será perguntado qual a senha de administração do Pi-hole você deseja usar.

Será adicionada uma regra de firewall no NFTables permitindo a porta 80 para o *source address* definido em `$IP_HOST` do `nftables.conf`.

O Pi-hole usará o arquivo de variáveis definido em `env_file` para, basicamente, configurar o IP e senha do serviço.
