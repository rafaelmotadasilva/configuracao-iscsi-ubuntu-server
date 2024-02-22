# Configuração do iSCSI no Ubuntu Server

Este repositório fornece instruções passo a passo sobre como configurar o iSCSI no Ubuntu Server para acessar dispositivos de armazenamento remotos.

## Visão Geral

O iSCSI (Internet Small Computer System Interface) é uma tecnologia que permite acessar dispositivos de armazenamento remotos através da rede TCP/IP. No Ubuntu Server, configurar o iSCSI pode expandir suas capacidades de armazenamento de forma eficiente e confiável.

## Requisitos

- Ubuntu Server 20.04 instalado e configurado
- Acesso à internet
- Permissões de administrador (sudo)

## Instruções

1. [Instalação do Open-iSCSI](#instalação-do-open-iscsi)
2. [Atualização do arquivo /etc/iscsi/initiatorname.iscsi](#atualização-do-arquivo-etciscsiinitiatornameiscsi)
3. [Configurar credenciais](#configurar-credenciais)
4. [Descobrir o dispositivo de armazenamento e o login](#descobrir-o-dispositivo-de-armazenamento-e-o-login)
5. [Montagem do disco](#montagem-do-disco)
6. [Automatização da montagem (Opcional)](#automatização-da-montagem-opcional)
7. [Criação de uma partição e um sistema de arquivos (opcional)](#criação-de-uma-partição-e-um-sistema-de-arquivos-opcional)
8. [Conclusão](#conclusão)

## Instalação do Open-iSCSI

Para começar, precisamos instalar o Open-iSCSI. Abra o terminal e execute o seguinte comando:

```
sudo apt update
sudo apt install open-iscsi
```

Quando o pacote é instalado, ele cria os dois arquivos a seguir.

* `/etc/iscsi/iscsid.conf`
* `/etc/iscsi/initiatorname.iscsi`

## Atualização do arquivo `/etc/iscsi/initiatorname.iscsi`

Atualize o arquivo `/etc/iscsi/initiatorname.iscsi` com o IQN do storage. Insira o valor como lowercase.

```
InitiatorName=<value-from-the-Portal>
```

## Configurar credenciais

Edite as configurações a seguir no `/etc/iscsi/iscsid.conf` usando o nome do usuário e a senha do console do storage. Use maiúscula para nomes de CHAP.

```
node.session.auth.authmethod = CHAP
node.session.auth.username = <Username-value-from-Portal>
node.session.auth.password = <Password-value-from-Portal>
discovery.sendtargets.auth.authmethod = CHAP
discovery.sendtargets.auth.username = <Username-value-from-Portal>
discovery.sendtargets.auth.password = <Password-value-from-Portal>
```
Reinicie o serviço iscsi para que as mudanças entre em vigor.

```
systemctl restart iscsid.service
```

## Descobrir o dispositivo de armazenamento e o login

O utilitário iscsiadm é uma ferramenta usada para a descoberta e o login para destinos iSCSI. Use o comando `iscsiadm` para realizar essa descoberta, inserindo o endereço IP de destino obtido do storage:

```
sudo iscsiadm -m discovery -t sendtargets -p <ip-value-from-storage>
```

Configure o login automático.

```
sudo iscsiadm -m node --op=update -n node.conn[0].startup -v automatic
sudo iscsiadm -m node --op=update -n node.startup -v automatic
```

Ative os serviços necessários.

```
systemctl enable open-iscsi
systemctl enable iscsid
```

Reinicie o serviço iscsid.

```
systemctl restart iscsid.service
```

Faça login na matriz iSCSI.

```
sudo iscsiadm -m node --loginall=automatic
```

Valide se a sessão iSCSI foi estabelecida.

```
iscsiadm -m session -o show
```
## Montagem do disco

Aqui estão os passos para montar o disco iSCSI:

**Identificação do dispositivo:** Primeiro você precisa identificar o dispositivo iSCSI que deseja montar. Isso pode ser feito usando o comando `blk` ou `fdisk-l` para listar todos os dispositivos disponíveis no sistema.

**Criação do ponto de montagem:** Escolha um diretório adequado no sistema de arquivos onde deseja montar o disco iSCSI. Você pode optar por criar um novo diretório existente, como `/mnt/iscsi` ou `/media/iscsi`.

```
sudo mkdir /mnt/iscsi
```

**Montagem do disco:**  Use o comando `mount` para montar o dispositivo iSCSI no diretório de montagem especificado. Substitua `/dev/sdb1` pelo nome do dispositivo iSCSI que você identificou anteriormente.

```
sudo mount /dev/sdb1 /mnt/iscsi
```
**Verificação da montagem:** Após a mensagem, verifique se o disco iSCSI está corretamente montado usando o comando `df-h` ou `mount`. Isso mostrará uma lista de todos os sistemas de arquivos montados, incluindo o disco iSCSI e e seu ponto de montagem.

```
df -h
```
## Criação de uma partição e um sistema de arquivos (opcional)

Depois que o volume é montado e está acessível no host, é possível criar um sistema de arquivos. Siga essas etapas para criar um sistema de arquivos no volume recém-montado.

Crie uma partição.

```
sudo fdisk /dev/sdb1
```

Crie o sistema de arquivos.

```
sudo mkfs.xfs /dev/sdb1
```

## Automatização da montagem (Opcional)

Para garantir que o disco iSCSI seja montado automaticamente na inicialização do sistema, você pode adicionar uma entrada ao arquivo `/etc/fstab`. Primeiro, precisamos obter o UUID do dispositivo iSCSI usando o comando `blkid`.

Abra um terminal e execute o seguinte comando para obter o UUID do dispositivo iSCSI:

```
sudo blkid
```

Procure pelo dispositivo iSCSI na lista e anote o UUID associado a ele.

Agora, abra o arquivo `/etc/fstab` com um editor de texto:

```
vim /etc/fstab
```

Adicione uma nova linha ao final do arquivo usando o UUID do dispositivo iSCSI, o ponto de montagem (/mnt/iscsi, por exemplo), o tipo de sistema de arquivos (xfs, ext4, etc.) e as opções de montagem desejadas. Por exemplo:

```
UUID=<UUID-do-dispositivo>   /mnt/iscsi   xfs   defaults   0   0
```

Substitua <UUID-do-dispositivo> pelo UUID que você anotou anteriormente. Salve e feche o arquivo.

Isso garantirá que o disco iSCSI seja montado automaticamente na inicialização do sistema.

## Conclusão

Parabéns! Você configurou com sucesso o iSCSI no seu Ubuntu Server.

## Contribuição

Se você tiver sugestões de melhorias ou correções para este guia, sinta-se à vontade para enviar uma pull request.

## Referências

- [Documentação IBM Cloud: Montagem de volumes do IBM Cloud Block Storage no Ubuntu 20.04](https://cloud.ibm.com/docs/BlockStorage?topic=BlockStorage-mountingUbu20&locale=pt-BR&interface=ui)
- [Bacula Latam: Montagem de Discos Storage no Ubuntu via iSCSI](https://www.bacula.lat/montar-discos-storage-nas-via-iscsi/)
- [Documentação oficial do Ubuntu: iSCSI](https://ubuntu.com/server/docs/service-iscsi)

## Licença

Este projeto está licenciado sob a [Licença MIT](LICENSE).