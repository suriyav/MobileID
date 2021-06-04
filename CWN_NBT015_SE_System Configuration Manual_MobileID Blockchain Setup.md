# MobileID Blockchain Setup

### TOC
* [Setup Host](#setup-host)
* [Deploy TLS Root CA](#deploy-tls-root-ca)
* [Deploy Orderer CA](#deploy-orderer-ca)
* [Deploy Peer CA](#deploy-peer-ca)
* [Generate Channel Artifacts](#generate-channel-artifacts)
* [Deploy Orderer](#deploy-orderer)
* [Deploy Peer](#deploy-peer)
* [Deploy Chaincode](#deploy-chaincode)
* [Chaincode Integration Test](#chaincode-integration-test)
* [Add New Org](#add-new-org)
* [Upgrade Chaincode](#upgrade-chaincode)
* [Deploy Middleware (optional)](#deploy-middleware-optional)
* [การเปลี่ยน Password ในการ Enroll Certificate](#การเปลี่ยน-password-ในการ-enroll-certificate-ใน-ca)
* [Trouble shooting เบื้องต้น](#trouble-shooting-เบื้องต้น)
## Setup Host

ติดตั้ง tool ที่ใช้ในการ deploy และ run service ทุก host

### 1.Install Tools

Ref:  [Docker Community Edition Installation](https://docs.docker.com/engine/install/ubuntu/)



```bash
# Docker CE (latest release)
sudo apt-get update

sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common \
    tree \
    zip unzip \
    postgresql-client
    
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
   
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose

# require for Fabric v2 native admin (run without CLI container)
sudo snap install yq # recommend version 3.3.1
sudo snap install jq
```

### 2.Config Docker Permission

Ref: [Docker Post Installation](https://docs.docker.com/engine/install/linux-postinstall/)


```bash
sudo groupadd docker

sudo usermod -aG docker $USER

# exit
```

Restart (logout) terminal 1 ครั้ง เพื่อ update permission

### 3.Setup DNS 

ให้ทำการตั้ง DNS server และเพิ่ม DNS Record ลง server ให้ถูกต้อง

หรือทำการ [Routing Localhost](#กรณี-routing-localhost) $\underline{\textbf{และ}}$ [เพิ่ม extra_hosts](#การเพิ่ม-extra_hosts)

**\*\*verify แต่ละเครื่องว่าสามารถ resolve DNS-IP ได้ถูกต้อง\*\***

## Deploy TLS Root CA

setup TLS Root CA service ที่เครื่อง TLS Root CA

set parameter สำหรับ deploy


```bash
export ORG=${ORG:=nbtc}
export OU=${OU:=nbtc}
export AFFILIATION=${AFFILIATION:=${OU}.department1}
export CA_URL=${CA_URL:=ca.thaimobileid.com:7054}

# work at home directory
cd $HOME
```

ย้ายไฟล์ artifacts เข้า host ซึ่งจะมี file ดังนี้

```bash
root_ca_server.zip
```

หลังจากนั้นให้ unzip ด้วย


```bash
unzip -o root_ca_server.zip
```

### 1. Install Fabric CA Client

ที่ TLS Root CA server ให้ลง fabric-ca-client ด้วยคำสั่ง


```bash
bash -i ./root-ca/install_client.sh
```

### 2. Setup TLS for CA-PostgresQL connection

กรณีที่ต้องการ encrypt การเชื่อมต่อระหว่าง IPv4


```bash
bash -i ./root-ca/genTLSConnection.sh
```

deploy database และทำการ set database config

### 3. Deploy PostgresQL Service


```bash
docker-compose -f ./root-ca/docker-compose.yaml up --force-recreate -d postgres_root_ca
sleep 10

bash -i ./root-ca/set_permission.sh
bash -i ./root-ca/update_config.sh
sleep 10
```

ตรวจสอบการ deploy database


```bash
docker logs --tail 10 $(docker ps -q -f name=postgres_root_ca)
```

### 4. Deploy Fabric CA Service


```bash
docker-compose -f ./root-ca/docker-compose.yaml up --force-recreate -d mobileid_root_ca
```

ตรวจสอบการ deploy Fabric CA


```bash
docker logs $(docker ps -q -f name=ca.thaimobileid.com)
```

ถ้าติดเรื่อง file permission ให้เพิ่มสิทธิ์ด้วยคำสั่ง


```bash
sudo chown -R $USER ./crypto-config
```

### 5. Download and Distribute TLS Certificate to Peer CA(s)

download TLS ไว้ให้ client ใช้งาน


```bash
bash -i ./root-ca/download_tls.sh
zip -r crypto-tmp.zip ./crypto-tmp
```

ส่ง crypto-tmp.zip ให้ Peer CA ทุก organization

### 6. Register Organization Membership

**\*\* register consortium ตั้งต้น \*\***

***\*\*\*\* อย่าลืมแก้ไข Domain name ในไฟล์ `./root-ca/register_consortium.sh` \*\*\*\****

ให้ทำการ register org ให้ครบที่กำหนด (Consortium ตั้งต้น ประกอบด้วย NBTC, ais, bbl) ถึงจะสามารถ generate channel artifact ได้



```bash
ORG=nbtc OU=nbtc AFFILIATION=nbtc.department1 bash -i ./root-ca/register_consortium.sh

ORG=ais OU=ais AFFILIATION=ais.department1 bash -i ./root-ca/register_consortium.sh

ORG=bbl OU=bbl AFFILIATION=bbl.department1 bash -i ./root-ca/register_consortium.sh

tree ./crypto-config
```

ส่วน org อื่นที่เข้ามาเพิ่มเติม ให้ Root CA NBTC ทำการ register org ตามตัวอย่าง (สามารถ register หลังจาก deploy mobileid network ได้ แต่ต้อง register ก่อนที่ org ใหม่จะ ทำการ generate certificate)


```bash
NEW_ORG=test
NEW_OU=test
NEW_AFFILIATION=test.department1

ORG=${NEW_ORG} OU=${NEW_OU} AFFILIATION=${NEW_AFFILIATION} bash -i ./root-ca/register_consortium.sh
```

หลังจากนั้น แต่ละ org ถึงจะสามารถ enroll TLS cert ไปใช้ได้

ให้ทำการ download และส่งไฟล์ 
```
crypto-tmp.zip 
```
ให้ Orderer CA และ Peer CA ทุก organization
## Deploy Orderer CA

setup Orderer CA service ที่เครื่อง Orderer CA

set parameter สำหรับ deploy



```bash
export URL_TLSCA=${URL_TLSCA:=ca.thaimobileid.com:7054}
export CA_URL=${CA_URL:=ca.orderer.thaimobileid.com:7054}
export URL_CA_ORDERER=${URL_CA_ORDERER:=ca.orderer.thaimobileid.com:7054}

# work at home directory
cd $HOME
```

upload ไฟล์ 
```
crypto-tmp.zip 
orderer_ca_server.zip
```
เข้า Orderer CA Server

หลังจากนั้นให้ unzip ด้วย


```bash
unzip -o orderer_ca_server.zip
```

### 1. Deploy `crypto-tmp.zip`

upload + extract crypto-tmp.zip


```bash
unzip -o crypto-tmp.zip
sudo chown -R $USER:$USER ./crypto-tmp
sudo chown -R $USER:$USER ./crypto-config
```

### 2. Install Fabric CA Client

ที่ Orderer CA server ให้ลง fabric-ca-client ด้วยคำสั่ง


```bash
bash -i ./orderer-ca/install_client.sh
```

### 3. Setup TLS for CA-PostgresQL Connection


```bash
bash -i ./orderer-ca/genTLSConnection.sh
```

### 4. Deploy PostgresQL Service

deploy database และทำการ set database config


```bash
docker-compose -f ./orderer-ca/docker-compose.yaml up --force-recreate -d postgres_ca_orderer
sleep 10

bash ./orderer-ca/set_permission.sh
bash ./orderer-ca/update_config.sh
sleep 10
```

### 5. Deploy Fabric CA Service


```bash
docker-compose -f ./orderer-ca/docker-compose.yaml up --force-recreate -d ca_orderer
```

### 6. Download TLS เครื่อง CA ตัวเองไว้ให้ client ใช้

download TLS ไว้ให้ client ใช้งาน


```bash
sudo chown -R $USER:$USER ./crypto-tmp
sudo chown -R $USER:$USER ./crypto-config
bash -i ./orderer-ca/download_tls.sh
```

ถ้าติดเรื่อง file permission ให้เพิ่มสิทธิ์ด้วยคำสั่ง


```bash
sudo chown -R $USER ./crypto-config
```

### 7. Register Orderer

register orderer


```bash
sudo chown -R $USER:$USER ./crypto-tmp
sudo chown -R $USER:$USER ./crypto-config
CA_URL=${CA_URL} bash -i ./orderer-ca/register_node.sh

tree ./crypto-config
```

### 8. Generate "ordererOrganizations" Certs

Generate "ordererOrganizations" certs


```bash
sudo chown -R $USER:$USER ./crypto-tmp
sudo chown -R $USER:$USER ./crypto-config
URL_TLSCA=${URL_TLSCA} URL_CA_ORDERER=${URL_CA_ORDERER} bash -i ./orderer-ca/generate_certs.sh

tree ./crypto-config
```

zip ไหล์ที่จำเป็นสำหรับการ gen channel


```bash
zip -r crypto-config-orderer.thaimobileid.com-msp.zip ./crypto-config/ordererOrganizations/thaimobileid.com/msp

tree ./crypto-config/ordererOrganizations/thaimobileid.com/msp
```

zip ไฟล์ที่จำเป็นสำหรับการ deploy orderer


```bash
# ZIP orderer crypto
zip -r crypto-config-orderer.thaimobileid.com.zip ./crypto-config/ordererOrganizations/thaimobileid.com/orderers/orderer.thaimobileid.com/
zip -r crypto-config-orderer2.thaimobileid.com.zip ./crypto-config/ordererOrganizations/thaimobileid.com/orderers/orderer2.thaimobileid.com/
zip -r crypto-config-orderer3.thaimobileid.com.zip ./crypto-config/ordererOrganizations/thaimobileid.com/orderers/orderer3.thaimobileid.com/
```

ส่งไฟล์ zip แต่ละไฟล์ไปให้เจ้าของ orderer แต่ละเครื่อง

zip ไฟล์ที่จำเป็นสำหรับการ deploy peer


```bash
zip -r crypto-config-orderer-for-peer-nbtc.zip ./crypto-config/ordererOrganizations/thaimobileid.com/orderers/orderer.thaimobileid.com/msp/tlscacerts/tlsca.thaimobileid.com-cert.pem
zip -r crypto-config-orderer-for-peer-ais.zip ./crypto-config/ordererOrganizations/thaimobileid.com/orderers/orderer2.thaimobileid.com/msp/tlscacerts/tlsca.thaimobileid.com-cert.pem
zip -r crypto-config-orderer-for-peer-bbl.zip ./crypto-config/ordererOrganizations/thaimobileid.com/orderers/orderer3.thaimobileid.com/msp/tlscacerts/tlsca.thaimobileid.com-cert.pem
```

zip ไฟล์ที่จำเป็นสำหรับการ deploy middleware แต่ละ org


```bash
zip -r crypto-config-orderer-for-middleware-nbtc.zip ./crypto-config/ordererOrganizations/thaimobileid.com/orderers/orderer.thaimobileid.com/msp/tlscacerts/tlsca.thaimobileid.com-cert.pem
zip -r crypto-config-orderer-for-middleware-ais.zip ./crypto-config/ordererOrganizations/thaimobileid.com/orderers/orderer2.thaimobileid.com/msp/tlscacerts/tlsca.thaimobileid.com-cert.pem
zip -r crypto-config-orderer-for-middleware-bbl.zip ./crypto-config/ordererOrganizations/thaimobileid.com/orderers/orderer3.thaimobileid.com/msp/tlscacerts/tlsca.thaimobileid.com-cert.pem
```

สรุปไฟล์ที่จำเป็นต้องใช้

```bash
crypto-config-orderer.thaimobileid.com-msp.zip
crypto-config-orderer.thaimobileid.com.zip
crypto-config-orderer2.thaimobileid.com.zip
crypto-config-orderer3.thaimobileid.com.zip
crypto-config-orderer-for-peer-nbtc.zip
crypto-config-orderer-for-peer-ais.zip
crypto-config-orderer-for-peer-bbl.zip
crypto-config-orderer-for-middleware-nbtc.zip
crypto-config-orderer-for-middleware-ais.zip
crypto-config-orderer-for-middleware-bbl.zip
```

ส่งไฟล์ zip ให้ทุก organization
## Deploy Peer CA

setup peer service ที่เครื่อง Peer CA แต่ละ org

set parameter สำหรับ deploy


```bash
export ORG_NAME="${ORG_NAME:=nbtc}"
export URL_TLSCA=${URL_TLSCA:=ca.thaimobileid.com:7054}
export CA_URL=${CA_URL:=ca.${ORG_NAME}.thaimobileid.com:7054}
export URL_CA_ORDERER=${URL_CA_ORDERER:=ca.orderer.thaimobileid.com:7054}
export URL_ORG_CA=${URL_ORG_CA:=ca.${ORG_NAME}.thaimobileid.com:7054}

# work at home directory
cd $HOME
```

upload ไฟล์ 
```
crypto-tmp.zip 
${ORG_NAME}_peer_ca_server.zip
```
เข้า Peer CA Server ข้างในจะมี TLS cert ไว้ต่อกับ TLS ROOT CA

หลังจากนั้นให้ unzip ด้วย


```bash
unzip -o ${ORG_NAME}_peer_ca_server.zip
```

### 1. Deploy `crypto-tmp.zip`



```bash
unzip -o crypto-tmp.zip
mkdir crypto-config

sudo chown -R $USER:$USER ./crypto-tmp
sudo chown -R $USER:$USER ./crypto-config
```

### 2. Install Fabric CA Client

ที่ Peer CA server ให้ลง fabric-ca-client ด้วยคำสั่ง


```bash
bash -i ./peer-ca/install_client.sh
```

### 3. Setup TLS for CA-PostgresQL Connection


```bash
bash -i ./peer-ca/genTLSConnection.sh
```

### 4. Deploy CA Service


deploy database และทำการ set database config


```bash
docker-compose -f ./peer-ca/docker-compose.yaml up --force-recreate -d postgres_ca_$ORG_NAME
sleep 10

bash -i ./peer-ca/set_permission.sh
bash -i ./peer-ca/update_config.sh
sleep 10
```

deploy Fabric CA services


```bash
docker-compose -f ./peer-ca/docker-compose.yaml up --force-recreate -d ca_$ORG_NAME
```

### 5. Download TLS เครื่อง CA ตัวเองไว้ให้ client ใช้


```bash
sudo chown -R $USER:$USER ./crypto-tmp
sudo chown -R $USER:$USER ./crypto-config
bash -i ./peer-ca/download_tls.sh
```

### 6. Register Org's user + peer


```bash
sudo chown -R $USER:$USER ./crypto-tmp
sudo chown -R $USER:$USER ./crypto-config
CA_URL=${CA_URL} bash -i ./peer-ca/register_node.sh
    
tree ./crypto-config
```

จะได้ cert ของ peer และ user อยู่ใน database ของ CA

### 7. Generate "peerOrganizations" certs


```bash
sudo chown -R $USER:$USER ./crypto-tmp
sudo chown -R $USER:$USER ./crypto-config
URL_TLSCA=${URL_TLSCA} URL_ORG_CA=${URL_ORG_CA} bash -i ./peer-ca/generate_certs.sh
    
tree ./crypto-config
```

zip ไฟล์ที่จำเป็นสำหรับการ deploy peer แต่ละ org


```bash
# ZIP peer crypto
zip -r crypto-config-peer0.${ORG_NAME}.thaimobileid.com.zip ./crypto-config/peerOrganizations/${ORG_NAME}.thaimobileid.com/{users/,peers/peer0.${ORG_NAME}.thaimobileid.com/}
zip -r crypto-config-peer1.${ORG_NAME}.thaimobileid.com.zip ./crypto-config/peerOrganizations/${ORG_NAME}.thaimobileid.com/{users/,peers/peer1.${ORG_NAME}.thaimobileid.com/}
```

zip ไฟล์ที่จำเป็นสำหรับการ deploy middleware แต่ละ org


```bash
zip -r crypto-config-peers-for-middleware.zip ./crypto-config/peerOrganizations/${ORG_NAME}.thaimobileid.com/peers/peer0.${ORG_NAME}.thaimobileid.com/msp/tlscacerts/tlsca.${ORG_NAME}.thaimobileid.com-cert.pem
zip -r crypto-config-peers-for-middleware.zip ./crypto-config/peerOrganizations/${ORG_NAME}.thaimobileid.com/peers/peer1.${ORG_NAME}.thaimobileid.com/msp/tlscacerts/tlsca.${ORG_NAME}.thaimobileid.com-cert.pem

zip -r crypto-config-ca-for-middleware.zip ./crypto-config/peerOrganizations/${ORG_NAME}.thaimobileid.com/tlsca/tlsca.${ORG_NAME}.thaimobileid.com-cert.pem
```

### 8. นำ crypto ออกมาจากเครื่อง

โดยไฟล์จะอยู่ที่

```bash
~/${ORG_NAME}.thaimobileid.com-msp.zip
~/crypto-config-peer0.${ORG_NAME}.thaimobileid.com.zip
~/crypto-config-peer1.${ORG_NAME}.thaimobileid.com.zip
~/crypto-config-peers-for-middleware.zip
~/crypto-config-ca-for-middleware.zip
```
## Generate Channel Artifacts

ที่เครื่อง ROOT-CA

generate mobileid channel artifacts

set environment variable สำหรับการ execute


```bash
mkdir -p /home/$USER/scripts
export SCRIPT_DIR="${PROJECT_DIR:=/home/$USER/scripts}"

# work at project directory
cd $SCRIPT_DIR
```

ที่เครื่อง ROOT-CA Path `/home/$USER/scripts` รวบรวมไฟล์ดังต่อไปนี้

```bash
crypto-config-orderer.thaimobileid.com-msp.zip
crypto-config-orderer.thaimobileid.com.zip
crypto-config-orderer2.thaimobileid.com.zip
crypto-config-orderer3.thaimobileid.com.zip

nbtc.thaimobileid.com-msp.zip
ais.thaimobileid.com-msp.zip
bbl.thaimobileid.com-msp.zip
scripts.zip
```

### 1. Install Fabric CA Client

#### ONLY Regulator

ที่เครื่อง ROOT-CA
install CA client binary ด้วย


```bash
bash .././root-ca/install_client.sh
```

### 2. Extract MSP Crypto

ให้ upload และ unzip ไฟล `${ORG_NAME}.thaimobileid.com-msp.zip` ทุก org ให้อยู่ใน directory เดียวกับ project
ให้ upload และ unzip ไฟล `crypto-config-${ORDERER_NAME}.thaimobileid.com.zip` ทุกไฟล์ให้อยู่ใน directory เดียวกับ project
ก่อนทำการ generate channel artifacts ด้วย


```bash
cd $SCRIPT_DIR
ls -l *.zip

unzip -o crypto-config-orderer.thaimobileid.com-msp.zip
unzip -o crypto-config-orderer.thaimobileid.com.zip
unzip -o crypto-config-orderer2.thaimobileid.com.zip
unzip -o crypto-config-orderer3.thaimobileid.com.zip
unzip -o nbtc.thaimobileid.com-msp.zip
unzip -o ais.thaimobileid.com-msp.zip
unzip -o bbl.thaimobileid.com-msp.zip
unzip -o scripts.zip
```

### 3. Generate Channel Update Transactions

ให้ทำการแก้ configtx.yaml ตามที่ design ไว้

**\*\*ให้ทำการแก้ domain name ต่างๆ ที่อยู่ใน configtx.yaml ก่อนทำการ gen channel\*\***

ตรวจสอบชื่อ Organization, MSP name (Organization[*].Name, Organization[*].ID, Organization[*].MSPDir)

ตรวจสอบสมาชิก consortium ใน Profiles.MobileIDPeerChannel.Application.Organizations


```bash
# แก้ configtx.yaml ให้ตรงกับ Arch
mkdir -p channel-artifacts

cp ./configtx.yaml $HOME/configtx.yaml

PATH=$PATH:../bin bash ./genChannel.sh
```

### 4. Package Channel Artifacts

ทำการ zip ไฟล์เพื่อส่งให้ NBTC organization


```bash
zip -r channel-artifacts-orderer.zip ./channel-artifacts/genesis.block
zip -r channel-artifacts-nbtc.zip ./channel-artifacts/channel.tx
```

ทำการ zip ไฟล์เพื่อส่งให้ peer organization


```bash
zip -r channel-artifacts-orderer.zip ./channel-artifacts/genesis.block
touch ./channel-artifacts/.empty
zip -r channel-artifacts-ais.zip ./channel-artifacts/.empty
zip -r channel-artifacts-bbl.zip ./channel-artifacts/.empty
```

ขั้นตอนนี้จะได้ไฟล์

```bash
# genesis block
channel-artifacts-orderer.zip

# v1.4 anchor peer update tx, not require in new v2.1.1
channel-artifacts-nbtc.zip
channel-artifacts-ais.zip
channel-artifacts-bbl.zip
```

ให้ download ออกมาจากเครื่องและส่งให้ peer organization

## Deploy Orderer

setup orderer service **\*\* ที่เครื่อง orderer แต่ละ org ให้ครบ \*\***

**\*\* set environment variable สำหรับการ execute แก้ไข ORG_NAME และ ORDERER_NAME ตาม deployment diagram \*\***


```bash
export ORG_NAME="${ORG_NAME:=nbtc}"
export ORDERER_NAME="${ORDERER_NAME:=orderer}"

# work at home directory
cd $HOME
```

clean orderer ด้วย

```bash
docker-compose -f orderer/docker-compose.yaml down -v

sudo rm -rf ${HOME}/crypto-config/ordererOrganizations/
sudo rm -rf ${HOME}/channel-artifacts/
sudo rm -rf /var/hyperledger/production/
```

### 1. Deploy Orderer's Crypto-Materials and Artifacts

ย้ายไฟล์ orderer crypto และ artifacts เข้า orderer host ซึ่งจะมี 3 file ดังนี้
```bash
crypto-config-${ORDERER_NAME}.thaimobileid.com.zip
channel-artifacts-orderer.zip
${ORG_NAME}_orderer_server.zip
```

unzip ทั้ง 3 ไฟล์


```bash
# stop orderer synchronization
docker-compose -f orderer/docker-compose.yaml down

unzip -o -q crypto-config-${ORDERER_NAME}.thaimobileid.com.zip
unzip -o -q channel-artifacts-orderer.zip
unzip -o -q ${ORG_NAME}_orderer_server.zip
```

### 2. Deploy Orderer Service


```bash
docker-compose -f ./orderer/docker-compose.yaml up -d --force-recreate
```

### 3. Verify Orderer Service


```bash
docker logs --tail 50 ${ORDERER_NAME}.thaimobileid.com 2>&1 | grep "Raft leader changed: "
```

ตรวจสอบ log ว่ามีการเลือก leader สำเร็จ

```bash
... Raft leader changed: ${N1} -> ${N2} channel=byfn-sys-channel node=1
```

เมื่อ deploy orderer ครบแล้วและ orderer มี Raft leader แล้ว

หลังจากนั้นจึงค่อย [Deploy Peer](#deploy-peer)
## Deploy Peer

setup peer service ที่เครื่อง peer แต่ละเครื่อง

**\*\* set environment variable สำหรับการ execute \*\***


```bash
export ORG_NAME="${ORG_NAME:=nbtc}"
export PEER_NAME="${PEER_NAME:=peer0}"

# work at home directory
cd $HOME
```

clean peer ด้วย

```bash
docker-compose -f peer/docker-compose.yaml down -v
    
sudo rm -rf ${HOME}/crypto-config/peerOrganizations
sudo rm -rf ${HOME}/channel-artifacts/
```

### 1. Deploy Peer's Crypto-Materials and Artifacts

ย้ายไฟล์ peer crypto เข้า peer host ซึ่งจะมี 5 file ดังนี้

```bash
crypto-config-orderer-for-peer.zip
crypto-config-${PEER_NAME}.${ORG_NAME}.thaimobileid.com.zip
channel-artifacts-${ORG_NAME}.zip
${ORG_NAME}_${PEER_NAME}_server.zip
mobileidcc.tar.gz
```

unzip ไฟล์ดังนี้


```bash
docker-compose -f peer/docker-compose.yaml down

unzip -o -q crypto-config-orderer-for-peer-${ORG_NAME}.zip
unzip -o -q crypto-config-${PEER_NAME}.${ORG_NAME}.thaimobileid.com.zip
unzip -o -q channel-artifacts-${ORG_NAME}.zip
unzip -o -q ${ORG_NAME}_peer0_server.zip

```

### 2. Verify 'crypto-config', 'channel-artifacts' and Peer Directory Structure

โดยใน folder crypto-config จะประกอบด้วย


```bash
tree ./crypto-config
```

และ ใน home directory ของ peer host จะประกอบด้วย directory โดยประมาณ


```bash
tree -d -L 4
```

### 3. Deploy Peer Service

เมื่อตรวจสอบว่ามีไฟล์ครบถ้วนแล้ว deploy peer ได้เลยครับ


```bash
docker-compose -f ./peer/docker-compose.yaml up -d --force-recreate
```

### 4. Join Channel

ที่ NBTC peer0

ให้สร้าง channel mobileid ด้วยคำสั่ง


```bash
if [[ "${ORG_NAME}" = "nbtc" ]] && [[ "${PEER_NAME}" = "peer0" ]]
then
    docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) bash scripts/createChannel.sh
fi
```

หลังจากที่ NBTC peer0 สร้าง channel แล้ว

ให้แต่ละ peer แต่ละเครื่องทำการ join peer เข้า channel mobileid และ update anchor peer ด้วยคำสั่ง


```bash
# bash ./joining_channel.sh

echo "Joining channel ..."
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) bash scripts/joinChannel.sh
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) bash scripts/updateAnchorPeer.sh
echo "Finished channel joining ..."
```

ถ้ามี error บอกว่า ledger id already exist แสดงว่าเคยมีการส่งคำสั่งข้อ join channel ไปแล้ว

ตรวจสอบสถานะการ join channel ด้วยคำสั่ง


```bash
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) peer channel list
```

จะต้องพบชื่อ channel mobileid
## Deploy Chaincode

setup chaincode service ที่ peer ทุกเครื่อง

set environment variable สำหรับการ execute


```bash
export CCPACK="mobileidcc.1.3.0.tar.gz"
export ORG_NAME="${ORG_NAME:=nbtc}"
export PEER_NAME="${PEER_NAME:=peer0}"
export CC_VERSION="${CC_VERSION:=1.3.0}"
export CC_SEQUENCE="${CC_SEQUENCE:=1}"
export CC_POLICY="${CC_POLICY:="OR ('NBTCMSP.member', 'AISMSP.member', 'BBLMSP.member')"}"

# work at home directory
cd $HOME
```

### Copy chaincode package to container

```bash
sudo docker cp $HOME/$CCPACK $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}):/opt/gopath/src/github.com/hyperledger/fabric/peer/$CCPACK
```
### List Container working Directory

```bash
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) ls
```

### 1. Install Chaincode Package

```bash
echo "Installing chaincode ..."

docker exec -t  $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) peer lifecycle chaincode install $CCPACK

echo "Chaincode installed ..."
```

ตรวจสอบ และ synch chaincode id ด้วยคำสั่ง


```bash
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) peer lifecycle chaincode queryinstalled
```

ตรวจสอบ chaincode id ต้องตรงกันทุก peer / ทุก org 

### 2. Approve Chaincode in Channel

ให้แต่ละ org approve chaincode โดยระบุ
```
CC_SEQUENCE
CC_VERSION
CC_POLICY
```
ให้ตรงกัน

จากนั้นให้แก้ไข file `$HOME/mobilecc/scripts/approveChaincodeWithSignature.sh`
และ แก้ไขบรรทัด
```bash 
export PACKAGE_ID=$(sed -n "/${CC_NAME}_${CC_VERSION}/{s/^Package ID: //; s/, Label:.*$//; p;}" log.txt)
```
โดยแทนที่ `PACKAGE_ID` ให้แทนด้วย Package ID ที่ได้มา จากขั้นตอน Install Chaincode เช่น `mobileidcc_7:3a80893adc48b2dd54a00498b3a60baa4e0104d8acf11587e41f0a69ee05215f` ดังตัวอย่าง ด้านล่างนี้

```bash 
export PACKAGE_ID=mobileidcc_7:3a80893adc48b2dd54a00498b3a60baa4e0104d8acf11587e41f0a69ee05215f
```

จากนั้นให้ Approve Chaincode

```bash
echo "Approving chaincode ..."
docker exec -e CC_SEQUENCE="${CC_SEQUENCE}" -e CC_VERSION="${CC_VERSION}" -e CC_POLICY="${CC_POLICY}" -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) \
    bash scripts/approveChaincodeWithSignature.sh
echo "Chaincode approved ..."
```

### 3. Commit Chaincode in Channel

หลังจากนั้นจึงให้เฉพาะ NBTC peer0 commit chaincode ใน mobileid channel ด้วย


```bash
if [[ "${ORG_NAME}" = "nbtc" ]] && [[ "${PEER_NAME}" = "peer0" ]]
then
    echo "Committing chaincode ..."
    docker exec -e CC_SEQUENCE="${CC_SEQUENCE}" -e CC_VERSION="${CC_VERSION}" -e CC_POLICY="${CC_POLICY}" -t $(docker ps -q -f name=cli_peer0_nbtc) \
        bash scripts/commitChaincodeWithSignature.sh
    echo "Chaincode committed ..."
fi
```

ซึ่งจะรันได้ที่ peer0.nbtc.thaimobileid.com ครั้งเดียว

หลังจาก Commmit Chaincode แล้ว Chaincode Instance จะถูก Launch อัตโนมัติ ซึ่งจะต่างจาก Hyperledger Fabric version 1.4 ที่ต้องทำการ query หรือ invoke ก่อน 1 ครั้ง เพืื่อ build และ deploy chaincode ขั้นตอนนี้จะใช้เวลานานในการรันครั้งแรก (สามารลดเวลาในการ deploy chaincode ด้วยการ pull docker `hyperledger/fabric-ccenv:latest` มารอก่อนได้)

### 5. Initialize Ledger

หลังจากนั้นจึงให้ NBTC initialize ledger และ chaincode ใน mobileid channel ด้วย


```bash
if [[ "${ORG_NAME}" = "nbtc" ]] && [[ "${PEER_NAME}" = "peer0" ]]
then
    echo "Initializing chaincode ..."
    docker exec -e CC_SEQUENCE="${CC_SEQUENCE}" -e CC_VERSION="${CC_VERSION}" -e CC_POLICY="${CC_POLICY}" -t $(docker ps -q -f name=cli_peer0_nbtc) \
        bash scripts/initLedger.sh
    echo "Chaincode ready ..."
fi
```

### 4. Verify Chaincode Installation

ตรวจสอบว่ามี chaincode พร้อมใช้งานด้วยคำสั่ง


```bash
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) bash scripts/queryCommitted.sh
# พบ ชื่อ chaincode เลข version, sequence และรหัส chaincode id ตรงกัน และ สถานะ approval เป็น true

docker ps -f name=mobileidcc
# พบ container ที่มีชื่อประกอบด้วย ชื่อ chaincode เลข version และรหัส chaincode id ตรงกับข้างบน
```

ตรวจสอบ block height ด้วยคำสั่ง


```bash
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) peer channel getinfo -c mobileid
# block height ควรจะตรงกับ peer อื่น หรือ อย่างน้อย block height กำลังอัพเดท (กรณี ที่มีการ commit block จำนวนมากใน channel peer จะต้องใช้เวลาในการ synchronize ledger)
```

คำสั่งต่างๆที่มีผลทำให้ block height เพิ่มขึ้น (ใช้ตรวจสอบการทำงานของ channel)
* create channel (Regulator only)
* update anchor peer (ปัจจุบัน แต่ละ org จะ update ได้เพียงครั้งเดียว)
* invoke chaincode
* update channel config (Orderer only)
## Chaincode Integration Test

ตรวจสอบการทำงานของ chaincode แบบ end to end testing

set environment variable สำหรับการ execute


```bash
export ORG_NAME="${ORG_NAME:=nbtc}"
export PEER_NAME="${PEER_NAME:=peer0}"

# work at home directory
cd $HOME
```

### 1. Verify Current State

ตรวจสอบว่ามี chaincode พร้อมใช้งานด้วยคำสั่ง


```bash
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) bash scripts/queryCommitted.sh
# พบ ชื่อ chaincode เลข version และรหัส chaincode id

docker ps -f name=mobileidcc
# พบ container ที่มีชื่อประกอบด้วย ชื่อ chaincode เลข version และรหัส chaincode id ตรงกับข้างบน
```

ตรวจสอบ block height ก่อนทำการ invoke


```bash
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) peer channel getinfo -c mobileid
# block height ควรจะตรงกับ peer อื่น หรือ อย่างน้อย block height กำลังอัพเดท (กรณี ที่มีการ commit block จำนวนมากใน channel peer จะต้องใช้เวลาในการ synchronize ledger)
```

### 2. Invoke Chaincode 's "healthCheck" method

ทำการ invoke method healthCheck เพื่อ stamp เวลา


```bash
# bash ./invoke_chaincode.sh

echo "Invoking chaincode ..."
echo "Waiting for cli starting up ..."
while [ "$(docker inspect -f {{.State.Running}} $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}))" == false ]; do sleep 1; done

docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) bash scripts/invokeChaincode.sh
echo "Chaincode invoked ..."
```

### 3. Query "healthCheck" 's Current State

ทำการ query เพื่อตรวจสอบ time stamp ของ healthCheck transaction


```bash
# bash ./query_chaincode.sh

echo "Querying chaincode ..."
echo "Waiting for cli starting up ..."
while [ "$(docker inspect -f {{.State.Running}} $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}))" == false ]; do sleep 1; done

docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) bash scripts/queryChaincode.sh
echo "Chaincode queried ..."
```

### 4. Verify Current State

ตรวจสอบ block height ด้วยคำสั่ง


```bash
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) peer channel getinfo -c mobileid
# block height ควรจะตรงกับ peer อื่น หรือ อย่างน้อย block height กำลังอัพเดท (กรณี ที่มีการ commit block จำนวนมากใน channel peer จะต้องใช้เวลาในการ synchronize ledger)
```

คำสั่งต่างๆที่มีผลทำให้ block height เพิ่มขึ้น (ใช้ตรวจสอบการทำงานของ channel)
* create channel (Regulator only)
* update anchor peer (ปัจจุบัน แต่ละ org จะ update ได้เพียงครั้งเดียว)
* invoke chaincode
* update channel config (Admin only)

## Add New Org

เพิ่ม org ใหม่เข้าสู่ channel


### 0. Deploy Peer CA

ให้ NBTC [Register Node](#6-register-organization-membership) สำหรับ org ใหม่

ให้ org ใหม่ [Deploy Peer CA](#deploy-peer-ca) ตามปกติ

### 1. Deploy New Org MSP Crypto

ที่เครื่อง NBTC peer0 

ให้ทำการ upload `${NEW_ORG_NAME}.thaimobileid.com-msp.zip` และทำการ deploy MSP crypto


```bash
echo "Deploy new org MSP crypto materials"

unzip -o ${NEW_ORG_NAME}.thaimobileid.com-msp.zip

```
### 2.Setup CLI to Add New Org

Download ~/peer-ca/configtx.yaml จากเครื่อง CA Org ใหม่ (หรือ จากไฟล์ `$BUILD_DIR/$NEW_ORG_NAME/peer-ca/configtx.yaml`)

ตรวจสอบชื่อ org และ msp ใน configtx.yaml

แล้ว upload configtx.yaml config เข้า CLI NBTC peer0

ที่ NBTC peer0


```bash
export NEW_ORG_NAME="${NEW_ORG_NAME:=test}"
echo "Reconfiguration channel admin peer"
mkdir -p ~/temp

docker cp ~/configtx.yaml \
    $(docker ps -q -f name=cli_peer0_nbtc):/opt/gopath/src/github.com/hyperledger/fabric/peer/

```

Upload MSP crypto ของ org ใหม่ 

`${NEW_ORG_NAME}.thaimobileid.com-msp.zip`

แล้วทำการ unzip


```bash
unzip -o ${NEW_ORG_NAME}.thaimobileid.com-msp.zip
docker exec -t $(docker ps -q -f name=cli_peer0_nbtc) \
    ls -l crypto/peerOrganizations/${NEW_ORG_NAME}.thaimobileid.com/msp
```

ตรวจสอบว่ามีไฟล์ configtx.yaml ให้ใช้งาน


```bash
docker exec -t $(docker ps -q -f name=cli_peer0_nbtc) \
    cat configtx.yaml
```

### 3.Generate Add NewOrg transaction

ที่ NBTC peer0

ให้ทำการสร้าง `channel_update_in_envelope.pb` ซึ่งจะถูกใช้เพื่อส่งคำสั่งเพิ่ม org


```bash
echo "Create add new org transaction"
CAP_ORG_NAME="$(yq e '.Organizations[0].Name' ~/configtx.yaml)"
MSP_NAME="$(yq e '.Organizations[0].ID' ~/configtx.yaml  )"
echo "Adding $NEW_ORG_NAME org" 

docker exec -t $(docker ps -q -f name=cli_peer0_nbtc) \
    bash scripts/createAddNewOrgTX.sh ${CAP_ORG_NAME} ${MSP_NAME}
    
docker exec -t $(docker ps -q -f name=cli_peer0_nbtc) \
    cat config_update_in_envelope.json
    
docker exec -t $(docker ps -q -f name=cli_peer0_nbtc) \
    jq -r .channel_group.groups.Application.groups.${CAP_ORG_NAME} config.json
    
rm -f ~/temp/channel_update_in_envelope.pb
docker cp $(docker ps -q -f name=cli_peer0_nbtc):/opt/gopath/src/github.com/hyperledger/fabric/peer/channel_update_in_envelope.pb ~/temp/

```
### 4.Endorse Add New Org Transaction

สำหรับแต่ละ endorsing org (org ที่อยู่ใน channel และมีหน้าที่ approve การ join channel ประกอบด้วย NBTC, ais, bbl)

ให้ upload `channel_update_in_envelope.pb` เข้า endorsing peer

ให้ endrosing org รัน signconfigtx แล้วส่ง channel_update_in_envelope.pb ให้ org อื่น sign ต่อ


```bash
export PEER_NAME="peer0"
export ENDORSING_ORG="${ENDORSING_ORG:=nbtc}"

echo "Endorsing channel_update_in_envelope.pb with peer ${PEER_NAME}.${ENDORSING_ORG}"
# copying endorsed transaction from admin
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ENDORSING_ORG}) \
    rm channel_update_in_envelope.pb
docker cp ~/channel_update_in_envelope.pb \
    $(docker ps -q -f name=cli_${PEER_NAME}_${ENDORSING_ORG}):/opt/gopath/src/github.com/hyperledger/fabric/peer/
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ENDORSING_ORG}) \
    md5sum channel_update_in_envelope.pb

# endorse transaction
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ENDORSING_ORG}) \
    bash ./scripts/signconfigtx.sh

# copying endorsed transaction to admin
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ENDORSING_ORG}) \
    md5sum channel_update_in_envelope.pb
docker cp $(docker ps -q -f name=cli_${PEER_NAME}_${ENDORSING_ORG}):/opt/gopath/src/github.com/hyperledger/fabric/peer/channel_update_in_envelope.pb ~/


```

download `channel_update_in_envelope.pb` และส่งให้ endorsing org อื่นทำการ endorse ต่อ จนกระทั่งมีการ endorse พอแล้ว จึงส่งกลับไปที่ NBTC
### 5.Submit Add New Org Transaction

ที่ NBTC peer0 

ให้นำ channel_update_in_envelope.pb ที่ถูก endorse ตาม channel policy (majority vote) แล้ว มา submit เข้า blockchain


```bash
export ADMIN_PEER_NAME="${ADMIN_PEER_NAME:=peer0}"
export ADMIN_ORG_NAME="${ADMIN_ORG_NAME:=nbtc}"

echo "Submitting endorsed channel_update_in_envelope.pb."

# copying endorsed transaction from admin
md5sum ~/channel_update_in_envelope.pb
docker cp ~/channel_update_in_envelope.pb \
    $(docker ps -q -f name=cli_peer0_nbtc):/opt/gopath/src/github.com/hyperledger/fabric/peer/channel_update_in_envelope.pb 
docker exec -t $(docker ps -q -f name=cli_${ADMIN_PEER_NAME}_${ADMIN_ORG_NAME}) \
    md5sum channel_update_in_envelope.pb
    
# submit add new org transaction via admin org
docker exec -e FABRIC_LOGGING_SPEC=INFO -t $(docker ps -q -f name=cli_${ADMIN_PEER_NAME}_${ADMIN_ORG_NAME}) \
    bash ./scripts/addNewOrg.sh

# copying endorsed transaction to admin
docker cp $(docker ps -q -f name=cli_peer0_nbtc):/opt/gopath/src/github.com/hyperledger/fabric/peer/channel_update_in_envelope.pb ~/

```

### 6.Create New Mobileid Member

ที่ NBTC peer0 

ให้ทำการสร้าง member code ใน Chaincode ด้วยการ invoke function createMember 

***(เปลี่ยน member_code, member_name, member_role, serial_no_prefix, status ก่อนทำการ invoke)***


```bash
echo "Add member to mobileid chaincode."

CC_INVOCATION='{"function":"createMember","Args":["{\"member_code\": \"TES\",\"member_name\": \"TEST\",\"member_role\": \"ISSUER\",\"serial_no_prefix\": \"05\",\"status\": \"A\"}"]}'
docker exec -e CC_INVOCATION="${CC_INVOCATION}" -t $(docker ps -q -f name=cli_${ADMIN_PEER_NAME}_${ADMIN_ORG_NAME}) \
    bash ./scripts/invokeChaincode.sh
    
```
### 7.Deploy Orderer Certs to New Org

ที่เครื่อง `NBTC peer0`

ให้ทำการ copy file สำหรับ deploy peer ใหม่


```bash
echo "Duplicate existing crypto materials for new organization"

cp ./crypto-config-orderer-for-peer-nbtc.zip \
    ./crypto-config-orderer-for-peer-${ORG_NAME}.zip 

cp ./crypto-config-orderer-for-middleware-nbtc.zip \
    ./crypto-config-orderer-for-middleware-${ORG_NAME}.zip 

cp ./channel-artifact/channel-artifacts-ais.zip \
    ./channel-artifacts-test.zip 
```

upload file `crypto-config-orderer-for-peer-${ORG_NAME}.zip ` เข้าเครื่อง peer org ใหม่
### 8.Setup Peer

ให้ [Deploy Peer](#deploy-peer) ตามปกติ

หลังจากนั้นให้ทำการ [Deploy Chaincode](#deploy-chaincode) ตามปกติ ถ้าไม่มีการเปลี่ยนแปลง endorsment policy

หรือทำการ [Upgrade Chaincode](#upgrade-chaincode) ทุก peer ถ้ามีการเปลี่ยนแปลง endorsment policy
## Upgrade Chaincode

upgrade deployed chaincode ที่เครื่อง peer ทุกเครื่อง

set environment variable สำหรับการ execute


```bash
export CCPACK="mobileidcc.1.3.0.tar.gz" ##ชื่อไฟล์ Chaincode
export ORG_NAME="${ORG_NAME:=nbtc}"
export PEER_NAME="${PEER_NAME:=peer0}"
export CC_VERSION="${CC_VERSION:=1.3.0}"
export CC_POLICY="${CC_POLICY:="OR ('NBTCMSP.member', 'AISMSP.member', 'BBLMSP.member', 'TESTMSP.member')"}"

# work at home directory
cd $HOME
```

ให้ตรวจสอบ CC_SEQUENCE ปัจจุบันด้วย [Query Committed](#4-verify-chaincode-installation)
```bash
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) bash scripts/queryCommitted.sh
```

แล้วให้ทำการ set CC_SEQUENCE ใหม่ โดยเพิ่มจาก version ปัจจุบัน 1 หน่วย


```bash
export CC_SEQUENCE="${CC_SEQUENCE:=2}"
```


### Copy chaincode package to container

```bash
sudo docker cp $HOME/$CCPACK $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}):/opt/gopath/src/github.com/hyperledger/fabric/peer/$CCPACK
```
### List Container working Directory

```bash
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) ls
```



### 1. Install Chaincode Package

ถ้า peer เคย install chaincode version ปัจจุบันแล้วไม่จำเป็นต้อง install ใหม่

ให้ install ใหม่กรณีที่ต้องการเปลี่ยน source code


```bash
echo "Installing chaincode ..."

docker exec -e CC_SEQUENCE="${CC_SEQUENCE}" -e CC_VERSION="${CC_VERSION}" -e CC_POLICY="${CC_POLICY}" -e CC_PACK="${CCPACK}" -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) \
    peer lifecycle chaincode install ${CCPACK}

echo "Chaincode installed ..."
```

ตรวจสอบ และ synch chaincode id ด้วยคำสั่ง


```bash
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) peer lifecycle chaincode queryinstalled
```

ตรวจสอบ chaincode id ต้องตรงกันทุก peer / ทุก org 

### 2. Approve Chaincode in Channel

ให้แต่ละ org approve chaincode โดยระบุ
```
CC_SEQUENCE
CC_VERSION
CC_POLICY
```
ให้ตรงกัน

จากนั้นให้แก้ไข file `$HOME/mobilecc/scripts/approveChaincodeWithSignature.sh`
และ แก้ไขบรรทัด
```bash 
export PACKAGE_ID=$(sed -n "/${CC_NAME}_${CC_VERSION}/{s/^Package ID: //; s/, Label:.*$//; p;}" log.txt)
```
โดยแทนที่ `PACKAGE_ID` ให้แทนด้วย Package ID ที่ได้มา จากขั้นตอน Install Chaincode เช่น `mobileidcc_7:3a80893adc48b2dd54a00498b3a60baa4e0104d8acf11587e41f0a69ee05215f` ดังตัวอย่าง ด้านล่างนี้

```bash 
export PACKAGE_ID=mobileidcc_7:3a80893adc48b2dd54a00498b3a60baa4e0104d8acf11587e41f0a69ee05215f
```

จากนั้นให้ Approve Chaincode


```bash
echo "Approving chaincode ..."
docker exec -e CC_SEQUENCE="${CC_SEQUENCE}" -e CC_VERSION="${CC_VERSION}" -e CC_POLICY="${CC_POLICY}" -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) \
    bash scripts/approveChaincodeWithSignature.sh
echo "Chaincode approved ..."
```

### 3. Commit Chaincode in Channel

หลังจากนั้นจึงให้ NBTC commit chaincode ใน mobileid channel โดยระบุ
```
CC_SEQUENCE
CC_VERSION
CC_POLICY
```
ให้ตรงกันับที่ approve


```bash
if [[ "${ORG_NAME}" = "nbtc" ]] && [[ "${PEER_NAME}" = "peer0" ]]
then
    echo "Committing chaincode ..."
    docker exec -e CC_SEQUENCE="${CC_SEQUENCE}" -e CC_VERSION="${CC_VERSION}" -e CC_POLICY="${CC_POLICY}" -t $(docker ps -q -f name=cli_peer0_nbtc) \
        bash scripts/commitChaincodeWithSignature.sh
    echo "Chaincode committed ..."
fi
```

ซึ่งจะรันได้ที่ peer0.nbtc.thaimobileid.com 

หลังจาก Commmit Chaincode แล้ว Chaincode Instance จะถูก Launch อัตโนมัติ ซึ่งจะต่างจาก Hyperledger Fabric version 1.4 ที่ต้องทำการ query หรือ invoke ก่อน 1 ครั้ง เพืื่อ build และ deploy chaincode ขั้นตอนนี้จะใช้เวลานานในการรันครั้งแรก

### 4. Verify Chaincode Installation

ตรวจสอบว่ามี chaincode พร้อมใช้งานด้วยคำสั่ง


```bash
docker exec -t $(docker ps -q -f name=cli_${PEER_NAME}_${ORG_NAME}) bash scripts/queryCommitted.sh
# พบ ชื่อ chaincode เลข version และรหัส chaincode id

```

พบ ชื่อ chaincode เลข version, sequence และรหัส chaincode id ตรงกัน และ สถานะ approval เป็น true

## Deploy Middleware (optional)

middleware ที่ generate จาก project นี้จะใช้เพื่อทดสอบและแก้ไขการทำงานของ mobileid network เท่านั้น

ในการ deploy middleware mobileid ตัวจริง ให้ใช้ middleware จากทีม civil และตรวจสอบ `connection profile` (`connection.json`) ให้ตรงกันด้วย

ในการ deploy middleware ให้รวบรวมไฟล์ artifact และ crypto ดังนี้

```bash
crypto-config-ca-for-middleware.zip
crypto-config-orderer-for-middleware-${ORG_NAME}.zip
crypto-config-peers-for-middleware.zip
${ORG_NAME}_middleware_server.zip
```

ให้ upload เข้าเครื่องที่จะ deploy middleware

แล้ว set parameter ที่จะ deploy middleware


```bash
export ORG_NAME="${ORG_NAME:=nbtc}"

# work at home directory
cd $HOME
```

### 1. Deploy Artifacts

unzip file


```bash
unzip -o crypto-config-ca-for-middleware.zip
unzip -o crypto-config-orderer-for-middleware-${ORG_NAME}.zip
unzip -o crypto-config-peers-for-middleware.zip
unzip -o ${ORG_NAME}_middleware_server.zip
```

### 2. Reconfig `connection.json`

ให้ทำการแก้ไข config `connection.json` ให้ตรงกับ deployment spec เช่น กำหนด peer connection ให้ถูกต้อง

ทำการตรวจสอบว่ามีไฟล์ crypto ตามที่ระบุไว้ใน `connection.json` หรือไม่

แล้วทำการระบุ `connection.json` ที่ใช้ใน docker container ให้ถูกต้องจาก environment variable `CONNECTION_PROFILE`

### 3. Reconfig Expose Port

ให้ทำการกำหนด port ที่จะใช้ให้ถูกต้องจากใน Dockerfile, middleware/docker-compose.yaml

### 4. Deploy Middleware


```bash
docker-compose -f middleware/docker-compose.yaml up -d --force-recreate
```

### 5. Test HTTP Request

download [Postman](https://www.postman.com/)  แล้วทำการ import file

```bash
pocmobileid_20190814.postman_collection.json

MobileID_API_Server_RD_Test.postman_collection.json
```

ทำการ setup environment variable ที่ postman

```bash
IP_ADDRESS  
PORT
```

แล้วทดลอง invoke request ดังนี้

1. NodeJS Healthcheck

ตัวอย่าง response
```json
OK
```

ให้ enroll user ด้วย

2. manage-user/Enroll User

ตัวอย่าง response
```json
{
    "message": "Successfully enrolled user \"User1@nbtc.thaimobileid.com\" and imported it into the wallet"
}
```

ให้เปลี่ยน http header `network-user` เป็น user ที่ enroll มา

แล้ว test mobileid network ด้วย

3. mobileidcc-healthcheck/Invoke mobileidcc Healthcheck

ตัวอย่าง response
```json
{
    "member_code": "NBT",
    "timestamp": "2020-08-10T13:07:25"
}
```

4. mobileidcc-healthcheck/Query mobileidcc Healthcheck

ตัวอย่าง response
```json
{
    "member_code": "NBT",
    "timestamp": "2020-08-10T13:08:00"
}
```

## การเปลี่ยน Password ในการ Enroll Certificate ใน CA

หลังจากที่ run peer-ca/generate_certs.sh แล้ว และทดสอบว่า Hyperledger Network สามารถทำงานได้ครบ
ให้ทำการเปลี่ยน password ที่ใช้ enroll เพื่อขอ certificate ด้วย โดยต้องเปลี่ยน cert ของ
* TLS Root CA
* Orderer CA
* Peer CA แต่ละ org 

โดย user ที่ต้องเปลี่ยนมีดังนี้


| CA Server   | Identity                       | การใช้งาน                |
| :---------- | :----------------------------- | :---------------------- |
| TLS Root CA | admin                          | (CA admin)              |
| TLS Root CA | orderer1                       | (orderer tls certfile)  |
| TLS Root CA | orderer2                       | (orderer2 tls certfile) |
| TLS Root CA | orderer3                       | (orderer3 tls certfile) |
| TLS Root CA | Admin@nbtc.thaimobileid.com    | (nbtc MSP admin)        |
| TLS Root CA | Admin@ais.thaimobileid.com     | (ais MSP admin)         |
| TLS Root CA | Admin@bbl.thaimobileid.com     | (bbl MSP admin)         |
| TLS Root CA | User1@nbtc.thaimobileid.com    | (nbtc MSP user)         |
| TLS Root CA | User1@ais.thaimobileid.com     | (ais MSP user)          |
| TLS Root CA | User1@bbl.thaimobileid.com     | (bbl MSP user)          |
| TLS Root CA | peer0.nbtc.thaimobileid.com    | (nbtc peer0)            |
| TLS Root CA | peer1.nbtc.thaimobileid.com    | (nbtc peer1)            |
| TLS Root CA | peer0.ais.thaimobileid.com     | (ais peer0)             |
| TLS Root CA | peer1.ais.thaimobileid.com     | (ais peer1)             |
| TLS Root CA | peer0.bbl.thaimobileid.com     | (bbl peer0)             |
| TLS Root CA | peer1.bbl.thaimobileid.com     | (bbl peer1)             |
| Orderer CA  | admin                          | (CA admin)              |
| Orderer CA  | orderer.thaimobileid.com       | (orderer)               |
| Orderer CA  | orderer2.thaimobileid.com      | (orderer2)              |
| Orderer CA  | orderer3.thaimobileid.com      | (orderer3)              |
| Orderer CA  | Admin@thaimobileid.com         | (orderer MSP admin)     |
| Orderer CA  | User1@thaimobileid.com         | (orderer MSP user)      |
| Orderer CA  | admin-orderer                  |                         |
| Orderer CA  | orderer1-orderer               |                         |
| Orderer CA  | orderer2-orderer               |                         |
| Orderer CA  | orderer3-orderer               |                         |
| Peer CA     | admin                          | (CA admin)              |
| Peer CA     | peer0.nbtc.thaimobileid.com    | (peer0)                 |
| Peer CA     | peer1.nbtc.thaimobileid.com    | (peer1)                 |
| Peer CA     | Admin@nbtc.thaimobileid.com    | (MSP admin/Admin Peer)  |
| Peer CA     | User1@nbtc.thaimobileid.com    | (MSP user/User Peer)    |
| Peer CA     | user1                          | (middleware user)       |


ตัวอย่างการเปลี่ยน Password ที่ Peer CA


```bash
cd $HOME
bash ./peer-ca/download_tls.sh
bash ./peer-ca/enroll_admin.sh
# interactive input
```


```bash
bash ./peer-ca/change_password.sh
# interactive input
```

 
## Trouble shooting เบื้องต้น

### กรณีต้อง clean instance


```bash
bash ./terminate_services.sh
# ระวังคำสั่งนี้เนื่องจากคำสั่งนี้จะปิด blockchain service ทั้งหมดบน host
bash ./clean_instance.sh
```

หรือ clean instance แบบ mannual ด้วย


```bash
docker-compose -f $SERVICE_NAME/docker-compose.yaml down -v
docker system prune -fa
docker volume prune -f
```

ควรจะ clean instance แยกตาม service ที่ต้องการ clean


```bash
# เลือก run command ด้วยความระมัดระวัง
# @ Build Machine
# docker kill $(docker ps -q -f name=thaimobileid) $(docker ps -q -f name=cli_peer) $(docker ps -q -f name=rootca)
# docker system prune -fa
# docker volume prune -f
# sudo rm -rf $BUILD_DIR/nbtc
# sudo rm -rf $BUILD_DIR/ais
# sudo rm -rf $BUILD_DIR/bbl
# sudo rm -rf /var/hyperledger/production/
# sudo rm *.zip
# sudo rm $PROJECT_DIR/build/*
# sudo rm $PROJECT_DIR/crypto-config/*
# sudo rm $PROJECT_DIR/channel-artifacts/*

# @ CA server 
# docker-compose -f xxx-ca/docker-compose.yaml down -v
# sudo rm $HOME/*.zip
# sudo rm $HOME/crypto-tmp/*
# sudo rm $HOME/crypto-config/*
# sudo rm $HOME/channel-artifacts/*

# @ Peer server 
# docker-compose -f peer/docker-compose.yaml down -v
# sudo rm $HOME/*.zip
# sudo rm $HOME/crypto-config/*
# sudo rm $HOME/channel-artifacts/*

# @ Orderer server 
# docker-compose -f orderer/docker-compose.yaml down -v
# sudo rm $HOME/*.zip
# sudo rm $HOME/crypto-config/*
# sudo rm $HOME/channel-artifacts/*
# sudo rm -rf /var/hyperledger/production/

```

### กรณีต้องการอัพเดท file เข้า container เช่น วางไฟล์ใหม่ลง peer host และต้องการให้ cli update file ให้ recreate docker service ด้วย


```bash
docker-compose -f $SERVICE_NAME/docker-compose.yaml up -d --force-recreate
# สังเกตุ log ของ docker จะขึ้นว่า service ถูก recreate
```

### กรณีต้องการลบข้อมูลของ PostgresQL database 

ให้ลบ directory data ด้วย ยกตัวอย่างเช่น ลบข้อมูลใน Peer CA


```bash
rm -rf peer-ca/data
```

### กรณีพบ error ledger id already exist

แสดงว่า เคยมีการส่ง transaction นั้นไปแล้ว เช่น การ update anchor peer ซ้ำด้วย domain name เดิมจะไม่มีผลอะไร เพราะเกิดการ update ไปแล้วนั่นเอง

### กรณี chaincode id ไม่ตรงกับระบบ (หรือคนอื่นในระบบ) - Fabric v1.4

ให้ตรวจสอบ 3 อย่าง คือ
1. content ของ chaincode (source code) ว่าตรงกันหรือไม่ มี hidden character หรือไม่ เช่น \r\n vs \n ผิดไปหรือไม่  สิทธิ์ของไฟล์ตรงกันหรือไม่  กรณีนี้พยายามส่ง chaincode ด้วยการ zip จะลดปัญหาได้
2. chaincode version (CC_VERSION) ตรงกันหรือไม่ โดยดูจาก CC_VERSION ใน cli
3. PATH (CC_SRC_PATH) ที่ install chaincode (ใน container) ตรงกันหรือไม่

### กรณีเจอ `OCI runtime exec failed: exec failed: container_linux.go:345: starting container process caused "exec: \"fe64b0b0a061\": executable file not found in $PATH": unknown`

แสดงว่ามี container ชื่อซ้ำกันในเครื่องเดียวกัน ให้เปลี่ยน `name=xxx` ในคำสั่ง `$(docker ps -q -f name=xxx)` ที่อยู่ในสคริปต์ให้เป็นชื่อเดียวกับ container ที่ต้องการ

### กรณี Routing localhost

ให้ทำการ set network mode เป็น host mode ด้วย (Ref: https://stackoverflow.com/a/32491150)

แล้วแก้ไฟล์ /etc/hosts ตามตัวอย่าง


```bash
if grep -q "thaimobileid" /etc/hosts 
then
    echo "Already routed thaimobileid.com"
else
    sudo cat <<EOT >> /etc/hosts

# local thaimobileid hosts
127.0.0.1  orderer.thaimobileid.com
127.0.0.1  orderer2.thaimobileid.com
127.0.0.1  orderer3.thaimobileid.com
127.0.0.1  peer0.nbtc.thaimobileid.com
127.0.0.1  peer1.nbtc.thaimobileid.com
127.0.0.1  peer0.ais.thaimobileid.com
127.0.0.1  peer1.ais.thaimobileid.com
127.0.0.1  peer0.bbl.thaimobileid.com
127.0.0.1  peer1.bbl.thaimobileid.com
127.0.0.1  ca.thaimobileid.com
127.0.0.1  ca.orderer.thaimobileid.com
127.0.0.1  ca.nbtc.thaimobileid.com
127.0.0.1  ca.ais.thaimobileid.com
127.0.0.1  ca.bbl.thaimobileid.com
127.0.0.1  couchdb0.nbtc.thaimobileid.com
127.0.0.1  couchdb1.nbtc.thaimobileid.com
127.0.0.1  couchdb0.ais.thaimobileid.com
127.0.0.1  couchdb1.ais.thaimobileid.com
127.0.0.1  couchdb0.bbl.thaimobileid.com
127.0.0.1  couchdb1.bbl.thaimobileid.com
127.0.0.1  postgres.thaimobileid.com
127.0.0.1  postgres.orderer.thaimobileid.com
127.0.0.1  postgres.nbtc.thaimobileid.com
127.0.0.1  postgres.ais.thaimobileid.com
127.0.0.1  postgres.bbl.thaimobileid.com

EOT

fi

cat /etc/hosts
sudo systemd-resolve --flush-caches
```

ทดสอบ host ปลายทาง


```bash
sudo systemd-resolve --flush-caches
ping -c 3 ca.thaimobileid.com
```

### การเพิ่ม extra_hosts

ใช้ในกรณีที่ไม่ใช้ docker swarm network เพื่อ resolve name server และไม่สามารถใช้ network_mode: host ได้เนื่องจาก docker จะ resolve หา prod server

ให้ทำการพิ่ม extra_hosts เข้าไปที่ทุก service


```bash
cat <<EOT > extra_hosts.yaml
# remote thaimobileid hosts
extra_hosts:
  # require tcp 7050
  - "orderer.thaimobileid.com:${NBTC_REGULATOR_HOST}"
  - "orderer2.thaimobileid.com:${AIS_HOST}"
  - "orderer3.thaimobileid.com:${BBL_HOST}"
  # require tcp 7051-7052
  - "peer0.nbtc.thaimobileid.com:${NBTC_REGULATOR_HOST}"
  - "peer0.ais.thaimobileid.com:${AIS_HOST}"
  - "peer0.bbl.thaimobileid.com:${BBL_HOST}"
  - "peer0.test.thaimobileid.com:${NBTC_VERIFIER_HOST}"
  - "peer1.nbtc.thaimobileid.com:${NBTC_REGULATOR_HOST}"
  - "peer1.ais.thaimobileid.com:${AIS_HOST}"
  - "peer1.bbl.thaimobileid.com:${BBL_HOST}"
  - "peer1.test.thaimobileid.com:${NBTC_VERIFIER_HOST}"
  # require tcp 7054
  - "ca.thaimobileid.com:${MOBILEID_TLS_ROOT_CA_HOST}"
  - "ca.orderer.thaimobileid.com:${MOBILEID_TLS_ROOT_CA_HOST}"
  - "ca.nbtc.thaimobileid.com:${NBTC_REGULATOR_HOST}"
  - "ca.ais.thaimobileid.com:${AIS_HOST}"
  - "ca.bbl.thaimobileid.com:${BBL_HOST}"
  - "ca.test.thaimobileid.com:${NBTC_VERIFIER_HOST}"
  # require tcp 5984
  - "couchdb0.nbtc.thaimobileid.com:${NBTC_REGULATOR_HOST}"
  - "couchdb0.ais.thaimobileid.com:${AIS_HOST}"
  - "couchdb0.bbl.thaimobileid.com:${BBL_HOST}"
  - "couchdb0.test.thaimobileid.com:${NBTC_VERIFIER_HOST}"
  - "couchdb1.nbtc.thaimobileid.com:${NBTC_REGULATOR_HOST}"
  - "couchdb1.ais.thaimobileid.com:${AIS_HOST}"
  - "couchdb1.bbl.thaimobileid.com:${BBL_HOST}"
  - "couchdb1.test.thaimobileid.com:${NBTC_VERIFIER_HOST}"
  # require tcp 5432
  - "postgres.thaimobileid.com:${MOBILEID_TLS_ROOT_CA_HOST}"
  - "postgres.orderer.thaimobileid.com:${MOBILEID_TLS_ROOT_CA_HOST}"
  - "postgres.nbtc.thaimobileid.com:${NBTC_REGULATOR_HOST}"
  - "postgres.ais.thaimobileid.com:${AIS_HOST}"
  - "postgres.bbl.thaimobileid.com:${BBL_HOST}"
  - "postgres.test.thaimobileid.com:${NBTC_VERIFIER_HOST}"
EOT

yq r extra_hosts.yaml
```


```bash
EXTRA_HOST_COMPOSE=extra_hosts.yaml
for COMPOSE_FILE in $(ls $BUILD_DIR/*/*/*/docker-compose.yaml)
do
    echo ${COMPOSE_FILE}
    yq w -i ${COMPOSE_FILE} 'services[*].extra_hosts' "$(yq r ${EXTRA_HOST_COMPOSE} 'extra_hosts')"
    sed -i "s/ |-//g" ${COMPOSE_FILE}
done
```

### กรณี RAFT log corrupt

ทำการ synch ข้อมูล raft log snapshot หม่ ด้วยการ
1. ทำให้ orderer cluster เข้าสู่สถานะ no leader  พื่อบังคับให้ orderer synch ข้อมูล raft log ด้วยการ shutdown orderer service ห้เหลือเพียง 1 node

ปิด service ด้วยคำสั่ง


```bash
docker-compose -f orderer/docker-compose.yaml down
```

ลบข้อมูลใน /var/hyperledger/production/orderer


```bash
rm -rf /var/hyperledger/production/orderer
# หรือ docker volume prune -f
```

2. ทำการ start orderer service อีกครั้ง


```bash
docker-compose -f orderer/docker-compose.yaml up -d --force-recreate
```

รอจนกระทั้ง orderer synch snapshot และ block เสร็จ

### กรณีพบ `KeyMaterial not found in SigningIdentityInfo`

ตรวจสอบที่ service ที่ใช้งานว่าสามารถอ่านไฟล์ใน crypto-config/XXX หรือไม่
ให้ทำการแก้ไข ตรวจสอบว่ามีไฟล์ สามารถอ่านได้ และลองรันคำสั่งใหม่

ถ้าไม่สามารถแก้ได้ เช่น ไม่มีไฟล์ดังกล่าวอยู่ให้ทำการ generate cert ใหม่

ให้ทำการลบ certificates ใน CA Server และ Peer/Orderer ที่เกิดปัญหา แล้วทำการรัน
`generate_certs.sh` ใหม่

*** ตรวจสอบว่าใน folder */keystore/* จะต้องมี key เพียง 1 ไฟล์ต่อ directory ก่อนที่จะทำการ zip ***

### กรณี Orderer ไม่สามารถ vote leader ได้

ให้ตรวจสอบดังนี้

1. Orderer IP address

เปลี่ยน `FABRIC_LOG_SPEC` ใน docker-compose.yaml เป็น `DEBUG`

ตรวจสอบ log ถ้าพบ
```text
"transport: Error while dialing dial tcp IP_ADDRESS:PORT: connect: connection refused"
```
แล้ว IP_ADDRESS:PORT ไม่ถูกต้อง ให้ตรวจสอบ DNS, /etc/hosts
และตรวจสอบ `network_mode` ใน docker-compose.yaml ต้องเป็น host mode (กรณีใช้ internal DNS, /etc/hosts)

2. Orderer System Channel

ตรวจสอบว่าได้ใช้ `channel-artifacts/genesis.block` อันเดียวกันไหม

3. crypto-config

ตรวจสอบ certificates ใน `crypto-config` ว่าถูกต้องไหม

4. Firewall

allow port 7050 และตรวจสอบ connection ทั้ง inbound/outbound

### กรณี Orderer ไม่สามารถเริ่ม vote leader ได้เนื่องจาก `Failed pulling system channel`

orderer log 


```
Failed pulling system channel: 
failed obtaining the latest block for channel byfn-sys-channel panic: Failed pulling system channel: failed obtaining the latest block for channel byfn-sys-channel
```


ให้ทำการลบ orderer cached


```bash
sudo rm -rf /var/hyperledger/production/
```

### กรณีต้อง update orderer config

ให้ทำการ zip file ที่ Orderer CA


```bash
zip -r crypto-config-Admin@orderer.thaimobileid.com.zip ./crypto-config/ordererOrganizations/thaimobileid.com/users/Admin@thaimobileid.com/msp
```

แล้ว upload และ extract ไฟล์ `crypto-config-Admin@orderer.thaimobileid.com.zip` เข้าเครื่อง NBTC peer0

### กรณีต้อง install fabric client แบบ offline

กรณีที่ไม่สามารถต่อ internet เพื่อ download file ที่ github ได้ หรือ internet ช้า หรือไม่ต้องการ download client หลายรอบ

ให้ทำการ download binary file ด้วยคำสั่ง


```bash
cd $HOME
curl -sSL https://bit.ly/2ysbOFE | bash -s -- -d -s 2.1.1 1.4.7 0.4.20

```

แล้ว upload file เข้าเครื่องปลายทาง หลังจากนั้นให้ run install_client.sh อีกรอบหนึ่งเพื่อ set environment PATH


```bash
# scp $HOME/bin/* $REMOTE_USER@$REMOTE_IP:./bin/

bash xxx-ca/install_client.sh
```
