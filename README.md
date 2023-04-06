# Utilities

- vimrc: configuração do vim para usar o tema [`molokai`](https://github.com/tomasr/molokai).
- HyperV_Conf: scripts usados no Windows e no Linux para automatizar a criação de um ambiente virtualizado para estudo.
  - `createVMFromSnap.ps1`: usado no **WINDOWS**: cria uma VM a partir de um snapshot tirado de uma VM modelo
  - `router.sh`: usado no **LINUX**: configura o servidor Linux para ser usado como roteador e DNS Server
  - `files`:
    - `nftables.conf`: a configuração do Firewall NFTables
    
## `createVMFromSnap.ps1`: How to use?
Execute, no PowerShell COMO ADMINISTRADOR, o comando abaixo e leia o **DESCRIPTION**:
```sh
Get-Help createVMFromSnap.ps1 -full
```
**OBSERVAÇÃO**: tenha certeza que a pasta destino, onde os arquivos da VM importada estarão, NÃO está compactada.

## `router.sh`: How to use?

**OBSERVAÇÃO**: o script DEPENDE do arquivo `files/nftables.conf`.

Uma VM chamada "Router" no Hyper-V deve ser gerada a partir do script `createVMFromSnap.ps1`. Dessa forma, ela virá apenas uma interface de rede (Default Switch).

1. Adicione uma interface de rede EXTERNA ao Hyper-V
2. Mude a interface padrão que veio na VM para a interface EXTERNA criada. Configure-a com MAC ESTÁTICO
3. Adicione uma interface de rede na Default Switch em uma vlan específica (ex.: vlan 2)
4. Na interface 1 (geralmente `eth0`), configure um IP FIXO que se comunique com a internet (uma ideia para saber qual IP FIXO usar, seria configurar a interface primeiro para DHCP, obter seus dados e, depois, alterar a interface para IP FIXO colocando os dados obtidos anteriormente).
5. Coloque o script nessa VM e execute-o: ele habilitará o roteamento, configurará o NFT Router e também, configurará um IP (172.16.1.1/16) para a interface `eth1`.

## `nftables.conf`: How to use?

Para aplicar as configurações:
```sh
# nft -f /etc/nftables.conf
```

Para ver as regras do firewall em uso no momento:
```sh
# nft list ruleset
```
