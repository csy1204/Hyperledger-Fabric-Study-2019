# Hyperledger Fabric Study

멘토: 이명철 (한국 IBM 상무)
멘티: 조상연, 민체화


## Installation Guide

Ubuntu 18.04 LTS에서 시작

1. Prerequisites
```bash
sudo apt-get install curl
sudo apt-get install -y docker docker-compose
sudo groupadd docker

curl -O https://dl.google.com/go/go1.12.3.linux-amd64.tar.gz
tar xvf go1.12.3.linux-amd64.tar.gz
```

```bash
vi .bashrc #(만약 vi 없을 시 sudo apt-get install vi)
```

vi에서 
shift+g (맨아래로 이동)
o (아래 줄 insert 모드)
아래 두 줄 복사 붙여넣기

```
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin
```

esc키 (모드 전환)
:wq 입력
Enter키 (저장 완료 및 나가기)

```bash
source ~/.bashrc
echo $GOPATH
echo $PATH
sudo apt-get install -y npm
sudo apt-get install -y git
```

1. Fabric Docker Image & First Netwrok
```bash
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


---


## Chaincode 관련 파일들

1. first-network/byfn.sh: 처음 설치
2. first-network/docker-compose-cli.yml 
3. first-network/scripts/script.sh: 체인코드 설치 실행 코드
4. first-network/scripts/utils.sh: 체인코드 설치 (peer chaincode)
5. chaincode/chaincode_example02/go/haincode_example02.go


## Editting Chaincode

현재 Fabric 1.4기준으로 chaincode_example02.go는 
A, B를 초기에 상정하고 각 참여자에게 값을 부여한 다음
둘 간의 Transaction을 통해 값을 이동시키는 구성으로 이루어져있다.

여기서 C라는 중재자를 추가하고 중간에서 고정 수수료를 납부하는 코드를 추가하고자한다.

먼저 Init부분은 다음과 같이 바꾼다.
``` go
func (t *SimpleChaincode) Init(stub shim.ChaincodeStubInterface) pb.Response {
	fmt.Println("ex02 Init")
	_, args := stub.GetFunctionAndParameters()
	var A, B, C string    // Entities C 추가
	var Aval, Bval, Cval int // Asset holdings Cval 추가
	var err error

	if len(args) != 6 { // len가 6이 되도록한다.
		return shim.Error("Incorrect number of arguments. Expecting 4")
	}

	// Initialize the chaincode
	A = args[0]
	Aval, err = strconv.Atoi(args[1])
	if err != nil {
		return shim.Error("Expecting integer value for asset holding")
	}
	B = args[2]
	Bval, err = strconv.Atoi(args[3])
	if err != nil {
		return shim.Error("Expecting integer value for asset holding")
	}
	C = args[4] // C를 위한 초기값 설정
	Cval, err = strconv.Atoi(args[5])
	if err != nil {
		return shim.Error("SS")
	}
	fmt.Printf("Aval = %d, Bval = %d, Cval = %d \n", Aval, Bval, Cval)

	// Write the state to the ledger
	err = stub.PutState(A, []byte(strconv.Itoa(Aval)))
	if err != nil {
		return shim.Error(err.Error())
	}

	err = stub.PutState(B, []byte(strconv.Itoa(Bval)))
	if err != nil {
		return shim.Error(err.Error())
    }

    // C의 값을 블록체인에 저장
	err = stub.PutState(C, []byte(strconv.Itoa(Cval)))
	if err != nil {
		return shim.Error(err.Error())
	}

	return shim.Success(nil)
}

```


그 다음 Invoke function을 수정한다.
``` go
// Transaction makes payment of X units from A to B
func (t *SimpleChaincode) invoke(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	var A, B, C string    // Entities
	var Aval, Bval, Cval int // Asset holdings
	var X int          // Transaction value
	var Y int = 5 // Transaction Fee
	var err error

	if len(args) != 4 {
		return shim.Error("Incorrect number of arguments. Expecting 3")
	}

	A = args[0]
	B = args[1]
	C = args[2]
	// Get the state from the ledger
	// TODO: will be nice to have a GetAllState call to ledger
	Avalbytes, err := stub.GetState(A)
	if err != nil {
		return shim.Error("Failed to get state")
	}
	if Avalbytes == nil {
		return shim.Error("Entity not found")
	}
	Aval, _ = strconv.Atoi(string(Avalbytes))

	Bvalbytes, err := stub.GetState(B)
	if err != nil {
		return shim.Error("Failed to get state")
	}
	if Bvalbytes == nil {
		return shim.Error("Entity not found")
	}
    Bval, _ = strconv.Atoi(string(Bvalbytes))
    
    Cvalbytes, err := stub.GetState(C)
    if err != nil {
		return shim.Error("Failed to get state")
	}
	if Cvalbytes == nil {
		return shim.Error("Entity not found")
	}
    Cval, _ = strconv.Atoi(string(Cvalbytes))
    
	// Perform the execution
	X, err = strconv.Atoi(args[3])
	if err != nil {
		return shim.Error("Invalid transaction amount, expecting a integer value")
	}
	Aval = Aval - X - Y
	Bval = Bval + X
	Cval += Y
	fmt.Printf("Aval = %d, Bval = %d Cval=%d\n", Aval, Bval, Cval)

	// Write the state back to the ledger
	err = stub.PutState(A, []byte(strconv.Itoa(Aval)))
	if err != nil {
		return shim.Error(err.Error())
	}

	err = stub.PutState(B, []byte(strconv.Itoa(Bval)))
	if err != nil {
		return shim.Error(err.Error())
	}
	err = stub.PutState(C, []byte(strconv.Itoa(Cval)))
	if err != nil {
		return shim.Error(err.Error())
	}

	return shim.Success(nil)
}
```

---

## Chaincode Install (GO)

체인코드는 Cli에서 설치나 실행을 모두 할 수 있다. 
cli는 docker container로 구동 중이기 때문에 이 컨테이너 안으로 들어가서 "peer chaincode" 명령어를 실행하면 된다.

### Execute CLI Bash
Cli환경으로 들어가는 코드
```bash
docker exec -it cli bash
```

### Chaincode Install
체인코드가 설치된 폴더 경로를 입력하면 체인코드를 앵커피어에 설치한다.
이때 핵심은 아래와 같다.
 1. go언어의 경우 `-l [언어명]` 을 하지 않아도 됨
 2. 경로를 `$GOPATH/src` 가 앞에 있다고 상정한다.
 3. `$PEER`, `$ORG`로 설치할 피어와 Org를 설정할 수 있다.

```bash
peer chaincode install\
 -n mycc\
 -v 1.3\
 -p github.com/chaincode/chaincode_example02/go/
```

### Chaincode Instantiate
체인코드를 처음 컴파일하고 시작하는 과정이다.
이때 꼭 tls를 사용해야한다.
ca file 경로는 utils.sh의 상단의 변수 설정에서 ORDERER_CA를 확인하면 된다. 

링크: https://github.com/hyperledger/fabric-samples/blob/8271a472f8651283fb6a9f40b0f5ef631701d64f/first-network/scripts/utils.sh#L9

원래 -P "AND ('Org1MSP.peer','Org2MSP.peer')" 이지만 편의상 AND를 OR로 바꾸어 둘 중 하나의 허가만 받아도 되도록 하였다.

```bash
peer chaincode instantiate\
 -o orderer.example.com:7050\
 --tls "true"\
 --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem\
 -C mychannel\
 -n mycc\
 -v 1.0\
 -c '{"Args":["init","a","90","b","210","c","0"]}'\
 -P "OR ('Org1MSP.peer','Org2MSP.peer')"
```


### (Instantiate를 이미 한 경우) Chaincode Upgrade
이미 Instantiate를 한 체인코드를 업그레이드 하는 경우 **다시 install를 수행**하고 그다음 upgrade를 수행하면 된다. 코드는 Instantiate와 크게 다르지 않다. (upgrade 명령어와 버젼명만 다름)
```bash
peer chaincode upgrade\
 -o orderer.example.com:7050\
 -–tls "true"\
 --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem\
 -C mychannel\
 -n mycc\
 -v 1.4\
 -c '{"Args":["init","a","90","b","210","c","0"]}'\
 -P "OR ('Org1MSP.peer','Org2MSP.peer')"
```

### Chaincode Invoke
체인코드를 실행하는 명령어로 체인코드에 명시된 input값을 순서대로 넣어주어야한다. 마찬가지로 tls로 실행한다.
```bash
peer chaincode invoke\
 -o orderer.example.com:7050\
 --tls "true"\
 --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem\
 -C mychannel\
 -n mycc "" \
 -c '{"Args":["invoke","a","b","c","10"]}'
```

### Chain Code Query
체인코드에서 값을 불러오는 명령어로 tls를 필요로 하지 않으며 마찬가지로 input을 잘 넣어주어야한다.
```bash
peer chaincode query\
 -C mychannel\
 -n mycc\
 -c '{"Args":["query","c"]}'
```



