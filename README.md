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

Uma VM chamada "Router" no Hyper-V deve ser gerada a partir do script `createVMFromSnap.ps1`. Dessa forma, ela virá apenas uma interface de rede (Default Switch).

1. [Adicione uma interface de rede EXTERNA ao Hyper-V](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/get-started/create-a-virtual-switch-for-hyper-v-virtual-machines).
```sh
New-VMSwitch -Name "External - Virtual Switch" -SwitchType External
```
2. Inicie a VM e desligue-a (será apenas para gerar um MAC aleatório, que será configurado no próximo passo).
3. Mude a interface padrão que veio na VM para a interface EXTERNA criada; configure-a com MAC ESTÁTICO e clique em APLICAR.
4. Adicione uma interface de rede na Default Switch em uma vlan específica (ex.: vlan 2)
5. Na interface 1 (geralmente `eth0`), configure um IP FIXO que se comunique com a internet, alterando, adequadamente, o arquivo `/etc/systemd/network/eth0.network`. Uma ideia para saber qual IP FIXO usar, é configurar a interface primeiro para DHCP, obter seus dados e, depois, alterar a interface para IP FIXO colocando os dados obtidos anteriormente. Você pode saber os dados obtidos pelo serviço DHCP executando:
```sh
interface=$(networkctl list --no-legend | awk 'NR==2{print $2}')
echo -e "[Match]\nName="$interface"\n\n[Network]\nDHCP=ipv4" > /etc/systemd/network/eth0.network
systemctl restart systemd-networkd.service
systemctl restart systemd-resolved.service
# ip addr # DHCP IP and subnet mask
# ip route # DHCP gateway
# resolvectl status # DHCP DNS Server
# echo "Configure IP and MASK"
ipAdd=$(ip addr | grep $interface | grep inet | awk '{print $2}')
# echo "Configure Gateway"
ipRoute=$(ip route | grep $interface | grep default | awk '{print $3}')
# echo "Configure DNS Server"
dnsServ=$(resolvectl status $interface | grep "DNS Servers" | awk '{print $3}')
echo -e "[Match]\nName="$interface"\n\n[Network]\nAddress=$ipAdd\nGateway=$ipRoute\nDNS=$dnsServ" > /etc/systemd/network/eth0.network
systemctl restart systemd-networkd.service
systemctl restart systemd-resolved.service
# ip addr # FIXED IP and subnet mask
# ip route # FIXED gateway
# resolvectl status # FIXED DNS Server
```
6. Depois de colocar o arquivo `nftables.conf` no diretório `files` (ler abaixo), coloque o script `router.sh` nessa VM e execute-o: ele habilitará o roteamento, configurará o NFTables Router e também, configurará um IP (172.16.1.1/16) para a interface `eth1`.

## `nftables.conf`: How to use?

Altere o valor da variável `IP_HOST` para o IP de seu sistema operacional (o HOST, que tem o Hyper-V instalado e que gerencia as VMs Linux).
Para aplicar as configurações:
```sh
# nft -f /etc/nftables.conf
```

Para ver as regras do firewall em uso no momento:
```sh
# nft list ruleset
```
