# Configuração do iSCSI no Ubuntu Server

Guia completo para configurar iSCSI no Ubuntu Server — tanto o lado **target (servidor de storage)** quanto o lado **iniciador (cliente)** que consome o disco remoto.

## Visão Geral

O iSCSI (Internet Small Computer System Interface) é uma tecnologia que permite acessar dispositivos de armazenamento remotos através da rede TCP/IP, fazendo com que um disco remoto apareça como um disco local no sistema operacional.

```
┌─────────────────┐         TCP/IP         ┌──────────────────────┐
│   Iniciador     │ ◄───────────────────── │  Target (Storage)    │
│  (cliente)      │       porta 3260       │  (servidor de disco) │
│  open-iscsi     │                        │  targetcli / tgt     │
└─────────────────┘                        └──────────────────────┘
```

## Requisitos

- Ubuntu Server 20.04 ou superior
- Permissões de administrador (sudo)
- Conectividade de rede entre iniciador e target

## Índice

1. [Configurar o Target (servidor de storage)](#1-configurar-o-target-servidor-de-storage)
2. [Instalar o Open-iSCSI no iniciador](#2-instalar-o-open-iscsi-no-iniciador)
3. [Descobrir os targets disponíveis](#3-descobrir-os-targets-disponíveis)
4. [Atualizar o IQN do iniciador](#4-atualizar-o-iqn-do-iniciador)
5. [Configurar autenticação CHAP](#5-configurar-autenticação-chap)
6. [Ativar login automático e estabelecer sessão](#6-ativar-login-automático-e-estabelecer-sessão)
7. [Identificar o disco no sistema](#7-identificar-o-disco-no-sistema)
8. [Criar partição e sistema de arquivos (novo disco)](#8-criar-partição-e-sistema-de-arquivos-novo-disco)
9. [Montar o disco](#9-montar-o-disco)
10. [Automatizar a montagem com fstab](#10-automatizar-a-montagem-com-fstab)

---

## 1. Configurar o Target (servidor de storage)

> Pule esta etapa se você já possui um NAS ou storage com iSCSI configurado.

### Instalação do targetcli

```bash
sudo apt update
sudo apt install targetcli-fb
```

### Configuração via targetcli

```bash
sudo targetcli
```

Dentro do shell interativo do targetcli:

```
# 1. Criar um backstorage (arquivo ou dispositivo de bloco)
/backstores/fileio> create nome_disco /caminho/para/arquivo.img 10G

# Ou usando um dispositivo de bloco real:
/backstores/block> create nome_disco /dev/sdb

# 2. Criar um IQN para o target
/iscsi> create iqn.2024-01.com.empresa:storage

# 3. Criar um LUN vinculando ao backstore
/iscsi/iqn.2024-01.com.empresa:storage/tpg1/luns> create /backstores/fileio/nome_disco

# 4. Configurar ACL — definir qual iniciador pode acessar
/iscsi/iqn.2024-01.com.empresa:storage/tpg1/acls> create iqn.2024-02.com.empresa:iniciador

# 5. (Opcional) Configurar usuário e senha CHAP
/iscsi/iqn.2024-01.com.empresa:storage/tpg1/acls/iqn.2024-02.com.empresa:iniciador> set auth userid=usuario password=senha

# 6. Salvar e sair
/> saveconfig
/> exit
```

### Habilitar o serviço

```bash
sudo systemctl enable --now targetclid
sudo systemctl enable --now rtslib-fb-targetctl
```

### Abrir a porta no firewall

```bash
sudo ufw allow 3260/tcp
```

---

## 2. Instalar o Open-iSCSI no iniciador

Execute no servidor que vai **consumir** o disco remoto:

```bash
sudo apt update
sudo apt install open-iscsi
```

A instalação cria dois arquivos de configuração:

- `/etc/iscsi/iscsid.conf` — configurações do daemon
- `/etc/iscsi/initiatorname.iscsi` — IQN único deste iniciador

---

## 3. Descobrir os targets disponíveis

Use `iscsiadm` para listar os targets que o storage expõe:

```bash
sudo iscsiadm -m discovery -t sendtargets -p <IP-do-Storage>
```

A saída retorna os IQNs disponíveis, por exemplo:
```
192.168.1.100:3260,1 iqn.2024-01.com.empresa:storage
```

---

## 4. Atualizar o IQN do iniciador

Edite o arquivo com o IQN que você configurou na ACL do target:

```bash
sudo vim /etc/iscsi/initiatorname.iscsi
```

```
InitiatorName=iqn.2024-02.com.empresa:iniciador
```

---

## 5. Configurar autenticação CHAP

Se o target exige autenticação, edite `/etc/iscsi/iscsid.conf`:

```bash
sudo vim /etc/iscsi/iscsid.conf
```

Localize e edite as seguintes linhas (use maiúscula para os nomes de CHAP):

```
node.session.auth.authmethod = CHAP
node.session.auth.username = usuario
node.session.auth.password = senha

discovery.sendtargets.auth.authmethod = CHAP
discovery.sendtargets.auth.username = usuario
discovery.sendtargets.auth.password = senha
```

Reinicie o daemon para aplicar:

```bash
sudo systemctl restart iscsid
```

---

## 6. Ativar login automático e estabelecer sessão

Configure o login automático ao iniciar:

```bash
sudo iscsiadm -m node --op=update -n node.conn[0].startup -v automatic
sudo iscsiadm -m node --op=update -n node.startup -v automatic
```

Habilite e inicie os serviços:

```bash
sudo systemctl enable open-iscsi
sudo systemctl enable iscsid
sudo systemctl restart iscsid
```

Faça login no target:

```bash
sudo iscsiadm -m node --loginall=automatic
```

Valide se a sessão foi estabelecida:

```bash
iscsiadm -m session -o show
```

---

## 7. Identificar o disco no sistema

Após o login, o disco remoto aparece como um dispositivo local. Para identificá-lo:

```bash
lsblk
```

```bash
sudo fdisk -l
```

O disco iSCSI normalmente aparece como `/dev/sdb`, `/dev/sdc`, etc.

---

## 8. Criar partição e sistema de arquivos (novo disco)

> Pule esta etapa se o disco já está formatado.

Crie uma partição (use o disco, não uma partição existente):

```bash
sudo fdisk /dev/sdb
```

Dentro do fdisk: `n` (nova partição) → `p` (primária) → `1` → Enter → Enter → `w` (gravar).

Formate a partição criada:

```bash
sudo mkfs.xfs /dev/sdb1
```

Ou com ext4, se preferir:

```bash
sudo mkfs.ext4 /dev/sdb1
```

---

## 9. Montar o disco

Crie o ponto de montagem:

```bash
sudo mkdir /mnt/iscsi
```

Monte o dispositivo:

```bash
sudo mount /dev/sdb1 /mnt/iscsi
```

Verifique a montagem:

```bash
df -h
```

---

## 10. Automatizar a montagem com fstab

Obtenha o UUID do dispositivo (mais confiável que o nome `/dev/sdX`):

```bash
sudo blkid /dev/sdb1
```

Abra o fstab:

```bash
sudo vim /etc/fstab
```

Adicione a linha ao final:

```
UUID=<UUID-do-dispositivo>   /mnt/iscsi   xfs   defaults,_netdev   0   0
```

> **`_netdev` é obrigatório para discos iSCSI.**
> Sem essa opção, o sistema tentará montar o disco antes da rede subir e ficará
> travado no boot caso o storage não esteja acessível.

Teste a entrada sem reiniciar:

```bash
sudo mount -a
```

---

## Conclusão

Com estes passos o disco iSCSI está disponível localmente e será remontado automaticamente após cada reinicialização, incluindo cenários onde o storage demora para ficar disponível na rede.

## Contribuição

Se você tiver sugestões de melhorias ou correções, sinta-se à vontade para enviar uma pull request.

## Referências

- [Documentação oficial do Ubuntu: iSCSI](https://ubuntu.com/server/docs/service-iscsi)
- [Documentação IBM Cloud: Montagem de volumes Block Storage no Ubuntu 20.04](https://cloud.ibm.com/docs/BlockStorage?topic=BlockStorage-mountingUbu20&locale=pt-BR&interface=ui)
- [Bacula Latam: Montagem de Discos Storage via iSCSI](https://www.bacula.lat/montar-discos-storage-nas-via-iscsi/)

## Licença

Este projeto está licenciado sob a [Licença MIT](LICENSE).
