# ការបង្កើតនូវ Hyperledger Fabric Business Network 4 Organizations

យើងនឹងបង្ហាញពីការបង្កើតនូវ Business Network និង ការដំឡើងនូវ Chaincode ដោយប្រើប្រាស់នៅ [Hyperledger Fabric v1.0.5](https://github.com/hyperledger/fabric/tree/v1.0.5)។ ក្នុងការបង្ហាញនេះផងដែរ យើងបានប្រើប្រាស់នូវ Network មួយចំនួនដូចខាងក្រោម៖

* Orderer Node មួយ
* Peers Node ៤​ (៤ Organizations ដែលមួយ Organization មានតែ Peer Node មួយគត់)
* Channel មួយ
* Example chaincode

នៅពេលបញ្ចប់ការបង្ហាញនេះផងដែរ អ្នកទាំងអស់គ្នានឹងអាចធ្វើការដំឡើងនូវ Hyperledger Fabric Business Network មួយ ព្រមជាមួយនឹងការដំឡើង Chaincode បង្កើត Chaincode និង Execute Chaincode។ ជាងនេះទៅទៀត Network របស់អ្នកទាំងអស់គ្នានឹងភ្ជាប់មកជាមួយនៅ TLS(Transport Layer Secure)ផងដែរ។

## Generate Peer និង Orderer Certificates

### crypto-config.yaml

```
OrdererOrgs:
    - Name: Orderer
      Domain: healthcare.kr
      Specs:
          - Hostname: orderer

PeerOrgs:
    - Name: Org1
      Domain: org1.healthcare.kr
      Template:
          Count: 1
      Users:
          Count: 1

    - Name: Org2
      Domain: org2.healthcare.kr
      Template:
          Count: 1
      Users:
          Count: 1

    - Name: Org3
      Domain: org3.healthcare.kr
      Template:
          Count: 1
      Users:
          Count: 1

    - Name: Org4
      Domain: org4.healthcare.kr
      Template:
          Count: 1
      Users:
          Count: 1
```

ដើម្បី Generate Certificates យើងធ្វើការ Runនូវ Command ខាងក្រោម

```
cryptogen generate --config=./crypto-config.yaml
```

## Creating/Modifing configtx.yaml

The configtx.yaml file is broken into several sections:

Profiles: Profiles describe the organizational structure of your network

Organizations: The details regarding individual organizations

Orderers: The details regarding the Orderer parameters

Applications: Application defaults - not needed

### crypto-config.yaml

```
---

## Profiles Section
Profiles:

    FourOrgsOrdererGenesis:
        Orderer:
            <<: *OrdererDefaults
            Organizations:
                - *OrdererOrg
        Consortiums:
            SampleConsortium:
                Organizations:
                    - *Org1
                    - *Org2
                    - *Org3
                    - *Org4
    FourOrgsChannel:
        Consortium: SampleConsortium
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - *Org1
                - *Org2
                - *Org3
                - *Org4

## Organizations Section
Organizations:

    ## Create OrdererOrg and Alias Name with OrdereOrg
    - &OrdererOrg
        Name: OrdererOrg
        ID: OrdererMSP
        ## MSP Directory in the ordererOrganization
        MSPDir: crypto-config/ordererOrganizations/healthcare.kr/msp

    ## Create Org1 and Alias Name with Org1
    - &Org1
        Name: Org1MSP
        ID: Org1MSP
        MSPDir: crypto-config/peerOrganizations/org1.healthcare.kr/msp
        AnchorPeers:
            - Host: peer0.org1.healthcare.kr
              Port: 7051

    - &Org2
        Name: Org2MSP
        ID: Org2MSP
        MSPDir: crypto-config/peerOrganizations/org2.healthcare.kr/msp
        AnchorPeers:
            - Host: peer0.org2.healthcare.kr
              Port: 7051

    - &Org3
        Name: Org3MSP
        ID: Org3MSP
        MSPDir: crypto-config/peerOrganizations/org3.healthcare.kr/msp
        AnchorPeers:
            - Host: peer0.org3.healthcare.kr
              Port: 7051

    - &Org4
        Name: Org4MSP
        ID: Org4MSP
        MSPDir: crypto-config/peerOrganizations/org4.healthcare.kr/msp
        AnchorPeers:
            - Host: peer0.org4.healthcare.kr
              Port: 7051

## Alias Name with &OrdererDefault
Orderer: &OrdererDefaults
    OrdererType: solo
    Addresses:
        - orderer.healthcare.kr:7050
    BatchTimeout: 2s
    BatchSize:
        MaxMessageCount: 10
        AbsoluteMaxBytes: 99 MB
        PreferredMaxBytes: 512 KB
    Kafka:
        Brokers: 
            - 127.0.0.1:9092
    Organizations:

## Alias Name with &ApplicationDefault
Application: &ApplicationDefaults
    Organizations:
```

Executing the configtxgen tool

To create orderer genesis block, run the following commands

```
export FABRIC_CFG_PATH=$PWD
mkdir channel-artifacts
configtxgen -profile FourOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.
```

After we created the orderer genesis block it is a time to create channel configuration transaction.

```
export CHANNEL_NAME=mychannel
configtxgen -profile FourOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME
```

The last operation we are going to perform with configtxgen is the definition of anchor peers for our organizations. This is especially important if there are more peers belonging to a single organization.

Run the following three commands to define anchor peers for each organization. Note that the `asOrg` parameter refers to the MSP ID definitions in configtx.yaml

```
## Organization1
configtxgen -profile FourOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP

## Organization2
configtxgen -profile FourOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP

## Organization3
configtxgen -profile FourOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org3MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org3MSP

## Organization4
configtxgen -profile FourOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org4MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org4MSP
```

# Start the Hyperledger Fabric blockchain network

To start our network we will use `docker-compose` tool. Based on its configuration, we launch containers based on the Docker images we downloaded in the beginning.

The `base` directory contains the `peer-base.yaml` file which is extended by the `docker-compose-cli.yaml` file (serves a base configuration file for each peer). The only required modification to the `peer-base.yaml` file is to chainge the network id as illustrated by the following example:

```
version: '2'

services:
  peer-base:
    image: hyperledger/fabric-peer
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # the following setting starts chaincode containers on the same
      # bridge network as the peers
      # https://docs.docker.com/compose/networking/
      # ---CHANGED---
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=fabricfourorgs_net_basic
      #- CORE_LOGGING_LEVEL=ERROR
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_PROFILE_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
```

You use the `docker-compose-base.yaml` in `base` directory to define the overall topology of your network as illustrated by the following example:

```
version: '2'

services:

  # ---CHANGED--- The orderer name is taken from the name generated by the "cryptogen" certs – it indicates the orderer orgs one and only orderer
  orderer.healthcare.kr:
    # ---CHANGED--- The container name is a copy of the orderer name
    container_name: orderer.healthcare.kr
    image: hyperledger/fabric-orderer
    environment:
      - ORDERER_GENERAL_LOGLEVEL=debug
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp
      # enabled TLS
      - ORDERER_GENERAL_TLS_ENABLED=true
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer
    volumes:
    - ../channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block
    # ---CHANGED--- the path is different to reflect our company's domain
    - ../crypto-config/ordererOrganizations/healthcare.kr/orderers/orderer.healthcare.kr/msp:/var/hyperledger/orderer/msp
    # ---CHANGED--- the path is different to reflect our company's domain
    - ../crypto-config/ordererOrganizations/healthcare.kr/orderers/orderer.healthcare.kr/tls/:/var/hyperledger/orderer/tls
    ports:
      - 7050:7050

  # ---CHANGED--- The peer name is taken from the name generated by the "cryptogen" certs – it indicates the peer org 1 and one peer "peer0"
  peer0.org1.healthcare.kr:
    # ---CHANGED--- Container name – same as the peer name
    container_name: peer0.org1.healthcare.kr
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_ID=peer0.org1.healthcare.kr
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_ADDRESS=peer0.org1.healthcare.kr:7051
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.healthcare.kr:7051
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org1.healthcare.kr:7051      
      - CORE_PEER_LOCALMSPID=Org1MSP
    volumes:
        - /var/run/:/host/var/run/
        # ---CHANGED--- changed to reflect peer name, org name and our company's domain
        - ../crypto-config/peerOrganizations/org1.healthcare.kr/peers/peer0.org1.healthcare.kr/msp:/etc/hyperledger/fabric/msp
        # ---CHANGED--- changed to reflect peer name, org name and our company's domain
        - ../crypto-config/peerOrganizations/org1.healthcare.kr/peers/peer0.org1.healthcare.kr/tls:/etc/hyperledger/fabric/tls
    ports:
      - 7051:7051
      - 7053:7053

  # ---CHANGED--- The peer name is taken from the name generated by the "cryptogen" certs – it indicates the peer org 2 and one peer "peer0"
  peer0.org2.healthcare.kr:
    # ---CHANGED--- Container name – same as the peer name
    container_name: peer0.org2.healthcare.kr
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_ID=peer0.org2.healthcare.kr
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_ADDRESS=peer0.org2.healthcare.kr:7051
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org2.healthcare.kr:7051
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org2.healthcare.kr:7051
      # ---CHANGED--- ensure that the MSP ID is correctly set of Org2
      - CORE_PEER_LOCALMSPID=Org2MSP
    volumes:
        - /var/run/:/host/var/run/
        # ---CHANGED--- changed to reflect peer name, org name and our company's domain
        - ../crypto-config/peerOrganizations/org2.healthcare.kr/peers/peer0.org2.healthcare.kr/msp:/etc/hyperledger/fabric/msp
        # ---CHANGED--- changed to reflect peer name, org name and our company's domain
        - ../crypto-config/peerOrganizations/org2.healthcare.kr/peers/peer0.org2.healthcare.kr/tls:/etc/hyperledger/fabric/tls

    ports:
      - 8051:7051
      - 8053:7053

  # ---CHANGED--- The peer name is taken from the name generated by the "cryptogen" certs – it indicates the peer org 3 and one peer "peer0"
  peer0.org3.healthcare.kr:
    # ---CHANGED--- Container name – same as the peer name
    container_name: peer0.org3.healthcare.kr
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_ID=peer0.org3.healthcare.kr
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_ADDRESS=peer0.org3.healthcare.kr:7051
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org3.healthcare.kr:7051
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org3.healthcare.kr:7051
      # ---CHANGED--- ensure that the MSP ID is correctly set of Org3
      - CORE_PEER_LOCALMSPID=Org3MSP
    volumes:
        - /var/run/:/host/var/run/
        # ---CHANGED--- changed to reflect peer name, org name and our company's domain
        - ../crypto-config/peerOrganizations/org3.healthcare.kr/peers/peer0.org3.healthcare.kr/msp:/etc/hyperledger/fabric/msp
        # ---CHANGED--- changed to reflect peer name, org name and our company's domain
        - ../crypto-config/peerOrganizations/org3.healthcare.kr/peers/peer0.org3.healthcare.kr/tls:/etc/hyperledger/fabric/tls
    ports:
      - 9051:7051
      - 9053:7053

  # ---CHANGED--- The peer name is taken from the name generated by the "cryptogen" certs – it indicates the peer org 3 and one peer "peer0"
  peer0.org4.healthcare.kr:
    # ---CHANGED--- Container name – same as the peer name
    container_name: peer0.org4.healthcare.kr
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_ID=peer0.org4.healthcare.kr
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_ADDRESS=peer0.org4.healthcare.kr:7051
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org4.healthcare.kr:7051
      # ---CHANGED--- changed to reflect peer name, org name and our company's domain
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org4.healthcare.kr:7051
      # ---CHANGED--- ensure that the MSP ID is correctly set of Org3
      - CORE_PEER_LOCALMSPID=Org4MSP
    volumes:
        - /var/run/:/host/var/run/
        # ---CHANGED--- changed to reflect peer name, org name and our company's domain
        - ../crypto-config/peerOrganizations/org4.healthcare.kr/peers/peer0.org4.healthcare.kr/msp:/etc/hyperledger/fabric/msp
        # ---CHANGED--- changed to reflect peer name, org name and our company's domain
        - ../crypto-config/peerOrganizations/org4.healthcare.kr/peers/peer0.org4.healthcare.kr/tls:/etc/hyperledger/fabric/tls
    ports:
      - 10051:7051
      - 10053:7053
```

The last file we are going to modify is `docker-compose-cli.yaml`. It glues our orderer and peers together into one network and defines CLI container. The CLI container is where you enter to issue commands that interact with the peers (creating channels, deploying chaincode, and so on) and it represents "command line API" we will use for interaction with our peers.

```
version: '2'

# ---CHANGED--- our network is called "basic"
networks:
  net_basic:

services:

  # ---CHANGED--- The orderer name is taken from the name generated by the "cryptogen" certs – it indicates the orderer orgs one and only orderer
  orderer.healthcare.kr:
    extends:
      file:   base/docker-compose-base.yaml
      # ---CHANGED--- refers to orderer name
      service: orderer.healthcare.kr
    # ---CHANGED--- The container name is a copy of the orderer name
    container_name: orderer.healthcare.kr
    networks:
      - net_basic

  # ---CHANGED--- The peer name is taken from the name generated by the "cryptogen" certs – it indicates the peer org 1 and one peer "peer0"
  peer0.org1.healthcare.kr:
    # ---CHANGED--- Container name – same as the peer name
    container_name: peer0.org1.healthcare.kr
    extends:
      file:  base/docker-compose-base.yaml
      # ---CHANGED--- Refers to peer name
      service: peer0.org1.healthcare.kr
    networks:
      # ---CHANGED--- our network is called "basic"
      - net_basic

  # ---CHANGED--- The peer name is taken from the name generated by the "cryptogen" certs – it indicates the peer org 2 and one peer "peer0"
  peer0.org2.healthcare.kr:
    # ---CHANGED--- Container name – same as the peer name
    container_name: peer0.org2.healthcare.kr
    extends:
      file:  base/docker-compose-base.yaml
      # ---CHANGED--- Refers to peer name
      service: peer0.org2.healthcare.kr
    networks:
      # ---CHANGED--- our network is called "basic"
      - net_basic

  # ---CHANGED--- The peer name is taken from the name generated by the "cryptogen" certs – it indicates the peer org 3 and one peer "peer0"
  peer0.org3.healthcare.kr:
    # ---CHANGED--- Container name – same as the peer name
    container_name: peer0.org3.healthcare.kr
    extends:
      file:  base/docker-compose-base.yaml
      # ---CHANGED--- Refers to peer name
      service: peer0.org3.healthcare.kr
    networks:
      # ---CHANGED--- our network is called "basic"
      - net_basic

  # ---CHANGED--- The peer name is taken from the name generated by the "cryptogen" certs – it indicates the peer org 3 and one peer "peer0"
  peer0.org4.healthcare.kr:
    # ---CHANGED--- Container name – same as the peer name
    container_name: peer0.org4.healthcare.kr
    extends:
      file:  base/docker-compose-base.yaml
      # ---CHANGED--- Refers to peer name
      service: peer0.org4.healthcare.kr
    networks: 
      # ---CHANGED--- our network is called "basic"
      - net_basic


  cli:
    container_name: cli
    image: hyperledger/fabric-tools
    tty: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_ID=cli
      # ---CHANGED--- peer0 from Org1 is the default for this CLI container
      - CORE_PEER_ADDRESS=peer0.org1.healthcare.kr:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
      - CORE_PEER_TLS_ENABLED=true
      # ---CHANGED--- changed to reflect peer0 name, org1 name and our company's domain
      - CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/peers/peer0.org1.healthcare.kr/tls/server.crt
      # ---CHANGED--- changed to reflect peer0 name, org1 name and our company's domain
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/peers/peer0.org1.healthcare.kr/tls/server.key
      # ---CHANGED--- changed to reflect peer0 name, org1 name and our company's domain
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/peers/peer0.org1.healthcare.kr/tls/ca.crt
      # ---CHANGED--- changed to reflect peer0 name, org1 name and our company's domain
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/users/Admin@org1.healthcare.kr/msp
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    # ---CHANGED--- command needs to be connected out as we will be issuing commands explicitly, not using by any script
    # command: /bin/bash -c './scripts/script.sh ${CHANNEL_NAME}; sleep $TIMEOUT'
    volumes:
        - /var/run/:/host/var/run/
        # ---CHANGED--- chaincode path adjusted
        - ./chaincode/:/opt/gopath/src/github.com/chaincode
        - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
        - ./channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
    depends_on:
       # ---CHANGED--- reference to our orderer
      - orderer.healthcare.kr
       # ---CHANGED--- reference to peer0 of Org1
      - peer0.org1.healthcare.kr
       # ---CHANGED--- reference to peer0 of Org2
      - peer0.org2.healthcare.kr
       # ---CHANGED--- reference to peer0 of Org3
      - peer0.org3.healthcare.kr
       # ---CHANGED--- reference to peer0 of Org4
      - peer0.org4.healthcare.kr
    networks:
      # ---CHANGED--- our network is called "basic"
      - net_basic

```

## Start the Docker Containers

After we have generated the certificates, the genesis block, the channel transaction, and created or modified the appropriate yaml files, we are ready to start our network. use the following command to start the network.

```
docker-compose -f docker-compose-cli.yaml up -d
```

You can `docker ps` to list the executing containers. In our case, you should see the following six containers executing:
* Four peers(4)
* The ordere (1)
* The CLI (1)

After running `docker-compose` and `docker ps` you can expect output similar to the following:

#### Note: At any time, you can use the `docker logs <container ID|Name>` to view the logs

#### Note: You can shut down the container (i.e the whole Hyperledger Fabric blockchain network) using the `docker rm -f $(docker ps -aq)` command.


## The Channel

After the Docker containers start, you can enter the CLI container using the following command:

```
docker exec -it cli bash
```

After you enter the CLI container, you can interact with the other peers by prefixing our peer commands with the appropriate environment varaiables. Usually, this means pointing to the certificates for that peer. You don't need to do that when interacting with peer0 or Org1 which is set as the default one in CLI container.
* Peer0 in Organization1
```
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/users/Admin@org1.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org1.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org1MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/peers/peer0.org1.healthcare.kr/tls/ca.crt 
```
* Peer0 in Organization2
```
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.healthcare.kr/users/Admin@org2.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org2.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org2MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.healthcare.kr/peers/peer0.org2.healthcare.kr/tls/ca.crt 
```
* Peer0 in Organization3
```
ORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.healthcare.kr/users/Admin@org3.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org3.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org3MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.healthcare.kr/peers/peer0.org3.healthcare.kr/tls/ca.crt 
```
* Peer0 in Organization4
```
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org4.healthcare.kr/users/Admin@org4.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org4.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org4MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org4.healthcare.kr/peers/peer0.org4.healthcare.kr/tls/ca.crt 
```

### Create Channel
The first command that we issue is the `peer create channel` command. This command targes the orderer (where the channels must be created) and used the `channel.tx` and the channel name that is created using the `configtxgen` tool. As we are in the context of CLI container command line we define `CHANNEL_NAME` environment variable withour channel NAME.

#### Note: For Creating the Channel we use only the peer0 in the Organization1
```
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/users/Admin@org1.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org1.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org1MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/peers/peer0.org1.healthcare.kr/tls/ca.crt 

export CHANNEL_NAME=mychannel

peer channel create -o orderer.healthcare.kr:7050 -c $CHANNEL_NAME -f ./channel-artifacts/channel.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/healthcare.kr/orderers/orderer.healthcare.kr/msp/tlscacerts/tlsca.healthcare.kr-cert.pem
```

### Join Channel

After the orderer creates the channel, we can have the peers join the channel, again using the peer CLI:

```
## Peer0 in the Organization 1
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/users/Admin@org1.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org1.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org1MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/peers/peer0.org1.healthcare.kr/tls/ca.crt 
peer channel join -b mychannel.block

## Peer0 in the Organization 2
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.healthcare.kr/users/Admin@org2.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org2.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org2MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.healthcare.kr/peers/peer0.org2.healthcare.kr/tls/ca.crt 
peer channel join -b mychannel.block

## Peer0 in the Organization 3
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.healthcare.kr/users/Admin@org3.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org3.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org3MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.healthcare.kr/peers/peer0.org3.healthcare.kr/tls/ca.crt 
peer channel join -b mychannel.block

## Peer0 in the Organization 4
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org4.healthcare.kr/users/Admin@org4.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org4.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org4MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org4.healthcare.kr/peers/peer0.org4.healthcare.kr/tls/ca.crt 
peer channel join -b mychannel.block
```

When you review the logs of the peers, you can see the channel creation message and the system chaincode being deployed as illustrated by the following example. Run the following command from a separate terminal:

```
docker logs peer0.org1.healthcare.kr
```

### Update anchor peers

Now, going back to our main temrinal, as the last step before we will interact with our network is to update the anchor peers. The following three commands are channel updates and they will become a part of the definition of the channel:

```
export CHANNEL_NAME=mychannel

## Peer0 in the Organization 1
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/users/Admin@org1.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org1.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org1MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/peers/peer0.org1.healthcare.kr/tls/ca.crt 
peer channel update -o orderer.healthcare.kr:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/healthcare.kr/orderers/orderer.healthcare.kr/msp/tlscacerts/tlsca.healthcare.kr-cert.pem

## Peer0 in the Organization 2
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.healthcare.kr/users/Admin@org2.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org2.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org2MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.healthcare.kr/peers/peer0.org2.healthcare.kr/tls/ca.crt 
peer channel update -o orderer.healthcare.kr:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org2MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/healthcare.kr/orderers/orderer.healthcare.kr/msp/tlscacerts/tlsca.healthcare.kr-cert.pem

## Peer0 in the Organization 3
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.healthcare.kr/users/Admin@org3.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org3.healthcare.kr:7051 CORE_PEER_LOCALMSPID="Org3MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.healthcare.kr/peers/peer0.org3.healthcare.kr/tls/ca.crt 
peer channel update -o orderer.healthcare.kr:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org3MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/healthcare.kr/orderers/orderer.healthcare.kr/msp/tlscacerts/tlsca.healthcare.kr-cert.pem

## Peer0 in the Organization 4
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org4.healthcare.kr/users/Admin@org4.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org4.healthcare.kr:7051 CORE_PEER_LOCALMSPID="Org4MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org4.healthcare.kr/peers/peer0.org4.healthcare.kr/tls/ca.crt 
peer channel update -o orderer.healthcare.kr:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org4MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/healthcare.kr/orderers/orderer.healthcare.kr/msp/tlscacerts/tlsca.healthcare.kr-cert.pem
```

### Install chaincode
Right now, we have fully configured and running Hyperledger Fabric Network. However, the network does not contian any business logic - from this perspective, we can consider it as "empty" and as of now we have not way how to enter data there. Hyperledger Fabric blockchain applications interact with the ledger through the chaincode and its methods. So as a next step we need to install(deploy) chaincode on each peer.

```
## Peer0 in the Organization1
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/users/Admin@org1.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org1.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org1MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/peers/peer0.org1.healthcare.kr/tls/ca.crt 
peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/

## Peer0 in the Organization2
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.healthcare.kr/users/Admin@org2.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org2.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org2MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.healthcare.kr/peers/peer0.org2.healthcare.kr/tls/ca.crt 
peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/

## Peer0 in the Organization3
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.healthcare.kr/users/Admin@org3.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org3.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org3MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.healthcare.kr/peers/peer0.org3.healthcare.kr/tls/ca.crt 
peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/

## Peer0 in the Organization4
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org4.healthcare.kr/users/Admin@org4.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org4.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org4MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org4.healthcare.kr/peers/peer0.org4.healthcare.kr/tls/ca.crt 
peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/
```

### Instantiate Chaincode

```
peer chaincode instantiate -o orderer.healthcare.kr:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/healthcare.kr/orderers/orderer.healthcare.kr/msp/tlscacerts/tlsca.healthcare.kr-cert.pem -C mychannel -n mycc -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "OR ('Org1MSP.member','Org2MSP.member','Org3MSP.member','Org4MSP.member')"
```

### Query for initial state of the ledger

```
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
```

### Invoke Chaincode 

```
## Peer0 in Organization1
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/users/Admin@org1.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org1.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org1MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.healthcare.kr/peers/peer0.org1.healthcare.kr/tls/ca.crt 
peer chaincode invoke -o orderer.healthcare.kr:7050  --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/healthcare.kr/orderers/orderer.healthcare.kr/msp/tlscacerts/tlsca.healthcare.kr-cert.pem  -C $CHANNEL_NAME -n mycc -c '{"Args":["invoke","a","b","10"]}'

## Peer0 in Organization2
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.healthcare.kr/users/Admin@org2.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org2.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org2MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.healthcare.kr/peers/peer0.org2.healthcare.kr/tls/ca.crt 
peer chaincode invoke -o orderer.healthcare.kr:7050  --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/healthcare.kr/orderers/orderer.healthcare.kr/msp/tlscacerts/tlsca.healthcare.kr-cert.pem  -C $CHANNEL_NAME -n mycc -c '{"Args":["invoke","a","b","10"]}'

## Peer0 in Organization3
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.healthcare.kr/users/Admin@org3.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org3.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org3MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.healthcare.kr/peers/peer0.org3.healthcare.kr/tls/ca.crt 
peer chaincode invoke -o orderer.healthcare.kr:7050  --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/healthcare.kr/orderers/orderer.healthcare.kr/msp/tlscacerts/tlsca.healthcare.kr-cert.pem  -C $CHANNEL_NAME -n mycc -c '{"Args":["invoke","a","b","10"]}'

## Peer0 in Organization4
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org4.healthcare.kr/users/Admin@org4.healthcare.kr/msp 
CORE_PEER_ADDRESS=peer0.org4.healthcare.kr:7051 
CORE_PEER_LOCALMSPID="Org4MSP" 
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org4.healthcare.kr/peers/peer0.org4.healthcare.kr/tls/ca.crt 
peer chaincode invoke -o orderer.healthcare.kr:7050  --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/healthcare.kr/orderers/orderer.healthcare.kr/msp/tlscacerts/tlsca.healthcare.kr-cert.pem  -C $CHANNEL_NAME -n mycc -c '{"Args":["invoke","a","b","10"]}'
```

### Query for net values

We did some changes to the values of our assets `a` and `b`. If you recall, we instantiated `a` to be 100 and invoked the 4 times by transferring 10 from `a` to `b`. So we expect the queries return the value of 60 for `a` and 230 for `b`.

* Query for `a`:
```
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
```

# Summary
At this point, our network is fully operational and this concludes our tutorial.

Here is a summary of the tasks hata we completed:
* We installed all dependencies required to run Hyperledger Fabric v1.0.5 on Ubuntu 16.04 LTS
* We used the `cryptogen` tool to generate certificates and keys for the local MSP.
* we used the `configtxgen` tool to create our channel transaction and genesis block.
* We configured the `docker-compose-cli.yaml` files to describe our networks and peers.
* We used the CLI container to perform the following tasks:
    * Create the Channel
    * Deploy chaincode
    * Initialize chaincode
    * Invoke chaincode
    * Query chaincode

