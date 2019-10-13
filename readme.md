# Hyperledger Fabric Study

멘토: 이명철 (한국 IBM)
멘티: 조상연, 민체화


## Installation Guide

1. Prerequisites
```
sudo apt-get install curl
sudo apt-get install -y docker docker-compose
sudo groupadd docker

curl -O https://dl.google.com/go/go1.12.3.linux-amd64.tar.gz
tar xvf go1.12.3.linux-amd64.tar.gz
```

```
vi .bashrc (만약 vi 없을 시 sudo apt-get install vi)
```

vi에서 shift+g (맨아래로 이동) / o (아래 줄 insert 모드)
아래 두 줄 복사 붙여넣기

```
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin
```

esc키 (모드 전환)
:w (저장)
Enter키 (저장 완료)
:q (나가기)

```
source ~/.bashrc
echo $GOPATH
echo $PATH
sudo apt-get install -y npm
sudo apt-get install -y git
```

1. Fabric Docker Image & First Netwrok
```
curl -sSL http://bit.ly/2ysbOFE | bash -s
ls (fabric-samples 확인)
cd fabric-samples/first-network/
./byfn.sh generate -c [채널이름]
./byfn.sh up -c [채널이름]
```

Reference
1.	https://hyperledger-fabric.readthedocs.io/en/release-1.4/prereqs.html 
2.	https://hyperledger-fabric.readthedocs.io/en/release-1.4/install.html
3.	https://hyperledger-fabric.readthedocs.io/en/release-1.4/build_network.html 


## Chaincode 관련 파일들

1. first-network/byfn.sh - 처음 설치
2. first-network/docker-compose-cli.yml 
3. first-network/scripts/script.sh - 체인코드 설치 실행 코드
4. first-network/scripts/utils.sh - 체인코드 설치 (peer chaincode)


## Chaincode Install (GO)

체인코드는 Cli에서 설치나 실행을 모두 할 수 있다. 
cli는 docker container로 구동 중이기 때문에 이 컨테이너 안으로 들어가서 "peer chaincode" 명령어를 실행하면 된다.

### Exec CLI
Cli환경으로 들어가는 코드
```
docker exec -it cli bash
```

### Chain Code Install
체인코드가 설치된 폴더 경로를 입력하면 체인코드를 앵커피어에 설치한다.
```
peer chaincode install -n mycc -v 1.3 -p github.com/chaincode/chaincode_example02/go/
```

### Chaincode Instantiate
체인코드를 처음 컴파일하고 시작하는 과정이다.
이때 꼭 tls를 사용해야하면 ca file 경로는 utils.sh의 상단의 변수 설정에서 ORDERER_CA를 확인하면 된다. 

링크: https://github.com/hyperledger/fabric-samples/blob/8271a472f8651283fb6a9f40b0f5ef631701d64f/first-network/scripts/utils.sh#L9

원래 -P "AND ('Org1MSP.peer','Org2MSP.peer')" 이지만 편의상 AND를 OR로 바꾸어 둘 중 하나의 허가만 받아도 되도록 하였다.

```
peer chaincode  -o orderer.example.com:7050 --tls "true" --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc -v 1.4 -c '{"Args":["init","a","90","b","210","c","0"]}' -P "OR ('Org1MSP.peer','Org2MSP.peer')"
```


### (After Instantiate) Chain Code Upgrade
이미 Instantiate를 한 체인코드를 업그레이드 하는 경우 다시 install를 수행하고 그다음 upgrade를 수행하면 된다. 코드는 Instantiate와 크게 다르지 않다. (upgrade 명령어와 버젼명만 다름)
```
peer chaincode upgrade -o orderer.example.com:7050 -–tls "true" --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc -v 1.4 -c '{"Args":["init","a","90","b","210","c","0"]}' -P "OR ('Org1MSP.peer','Org2MSP.peer')"
```

### Chain Code Invoke
체인코드를 실행하는 명령어로 체인코드에 명시된 input값을 순서대로 넣어주어야한다. 마찬가지로 tls로 실행한다.
```
peer chaincode invoke -o orderer.example.com:7050 --tls "true" --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc "" -c '{"Args":["invoke","a","b","c","10"]}'
```

### Chain Code Query
체인코드에서 값을 불러오는 명령어로 tls를 필요로 하지 않으며 마찬가지로 input을 잘 넣어주어야한다.
```
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","c"]}'
```
