# fabric_PutState_GetState_API_Part1

■ 참조 사이트 : https://medium.com/@kctheservant/putstate-and-getstate-the-api-in-chaincode-dealing-with-the-state-in-the-ledger-part-1-2008d6a86f98

< PutState and GetState: The API in Chaincode Dealing with the State in the Ledger (Part 1) >
-  체인 코드가 원장에서 쓰고, 상태를 읽는 방법을 PutState 및 GetState API를 통해 수행되는 과정을 습득합니다.
- fabric samples의 fabcar 예제를 이용합니다.
- 두가지 파트로 구분해서 설명 (fabcar chaincode 참조)
- 1 파트 : PutState , GetState, GetStateByRange, GetStateByRangeWithPagination (the chaincode를 pagination and modify)
- 2 파트 : GetStateByCompositeKey (record indexing)

1. Fabric Network Setup

1.1 Bring up the Basic Network

$ cd fabric-samples/basic-network
$./start.sh

1.2 Install and Instantiate fabcar chaincode

$ docker-compose up -d cli
$ docker exec cli peer chaincode install -n mycc -p github.com/fabcar/go -v 0
$ docker exec cli peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -n mycc github.com/fabcar/go -v 0 -c '{"Args": []}' -P "OR('Org1MSP.member')"

1.3 Invoke initLedger() (10개의 car record)

$ docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["initLedger"]}'
- Result : Retrieved channel (mychannel) orderer endpoint: orderer.example.com:7050 / Chaincode invoke successful. result: status:200

2. Fabcar 체인코드(5개의 함수제공) 내용

- initLedger() : 체인 코드가 인스턴스화 된 후 한 번 호출 된 10 개의 사전 정의 된 자동차 레코드를 원장에 로드하는 함수
- queryAllCars() : 모든 자동차의 레코드를 반환
- queryCar(carid) : carid에 해당하는 자동차 레코드를 반환
- createCar (carid, make, model, colour, owner) : 새로운 차를 등록
- changeCarOwner(carid, newOwner): 기존 자동차 소유자(carid)를 newOwner로 변경
- 원장과 연관된 API 함수 : PutState, GetState and GetStateByRange.

3. PutState (그림참조)

$ docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["initLedger"]}'
- Result : http://localhost:5984/_utils 확인

3.1 Code example : initLedger()

	i := 0
	for i < len(cars) {
		fmt.Println("i is ", i)
		carAsBytes, _ := json.Marshal(cars[i])
		APIstub.PutState("CAR"+strconv.Itoa(i), carAsBytes)
		fmt.Println("Added", cars[i])
		i = i + 1
	}

- key : CARi (ex: CAR0, CAR1, CAR2, …)
- value : JSON에서 처리된 바이트 배열
- createCar() and changeCarOwner()에서도 PutState 사용함.

4. GetState

$ docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["queryCar", "CAR0"]}'
- Result : {"colour":"blue","make":"Toyota","model":"Prius","owner":"Tomoko"}

4.1 Code example : queryCar()

	carAsBytes, _ := APIstub.GetState(args[0])
	return shim.Success(carAsBytes)

- returns the value (as byte array) by a given key (as string)

5. GetState and PutState

- changeCarOwner()는 GetState 를 사용 하여 지정된 carid의 레코드를 가져오고 지정된 새 소유자로 소유자를 업데이트 후 PutState 를 사용하여 원장에 기록

$ docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["changeCarOwner", "CAR0", "KC"]}'
- Result : Retrieved channel (mychannel) orderer endpoint: orderer.example.com:7050 / Chaincode invoke successful. result: status:200

$ docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["queryCar", "CAR0"]}'
- Result : {"colour":"blue","make":"Toyota","model":"Prius","owner":"KC"}

5.1 Code example : changeCarOwner()

	carAsBytes, _ := APIstub.GetState(args[0])
	car := Car{}

	json.Unmarshal(carAsBytes, &car)
	car.Owner = args[1]

	carAsBytes, _ = json.Marshal(car)
	APIstub.PutState(args[0], carAsBytes)
	
6. GetStateByRange

- GetStateByRange API는 지정된 범위의 레코드를 반환.
- GetStateByRange requires two strings: startKey and endKey

6.1 Code example : queryAllCars()-GetStateByRange

	startKey := "CAR0"
	endKey := "CAR999"

	resultsIterator, err := APIstub.GetStateByRange(startKey, endKey)
	if err != nil {
		return shim.Error(err.Error())
	}
	defer resultsIterator.Close()
	
- GetState에서는 주어진 키에 대한 값을 직접 가져옵니다
- GetStateByRange에서는 iterator return 값을 이용하여 여러 데이터들을 추출해내는 코드는 아래와 같음

6.2 Code example : queryAllCars()-Iterator

	// buffer is a JSON array containing QueryResults
	var buffer bytes.Buffer
	buffer.WriteString("[")

	bArrayMemberAlreadyWritten := false
	for resultsIterator.HasNext() {
		queryResponse, err := resultsIterator.Next()
		if err != nil {
			return shim.Error(err.Error())
		}
		// Add a comma before array members, suppress it for the first array member
		if bArrayMemberAlreadyWritten == true {
			buffer.WriteString(",")
		}
		buffer.WriteString("{\"Key\":")
		buffer.WriteString("\"")
		buffer.WriteString(queryResponse.Key)
		buffer.WriteString("\"")

		buffer.WriteString(", \"Record\":")
		// Record is a JSON object, so we write as-is
		buffer.WriteString(string(queryResponse.Value))
		buffer.WriteString("}")
		bArrayMemberAlreadyWritten = true
	}
	buffer.WriteString("]")

	fmt.Printf("- queryAllCars:\n%s\n", buffer.String())

	return shim.Success(buffer.Bytes())
	
- GetStateByRange 는 범위 내에 정의 된 모든 항목을 보여줌.

$ docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["queryAllCars"]}'
- Result : [{"Key":"CAR0", "Record":{"colour":"blue","make":"Toyota","model":"Prius","owner":"KC"}},{"Key":"CAR1", "Record":{"colour":"red","make":"Ford","model":"Mustang","owner":"Brad"}},{"Key":"CAR2", "Record":{"colour":"green","make":"Hyundai","model":"Tucson","owner":"Jin Soo"}},{"Key":"CAR3", "Record":{"colour":"yellow","make":"Volkswagen","model":"Passat","owner":"Max"}},{"Key":"CAR4", "Record":{"colour":"black","make":"Tesla","model":"S","owner":"Adriana"}},{"Key":"CAR5", "Record":{"colour":"purple","make":"Peugeot","model":"205","owner":"Michel"}},{"Key":"CAR6", "Record":{"colour":"white","make":"Chery","model":"S22L","owner":"Aarav"}},{"Key":"CAR7", "Record":{"colour":"violet","make":"Fiat","model":"Punto","owner":"Pari"}},{"Key":"CAR8", "Record":{"colour":"indigo","make":"Tata","model":"Nano","owner":"Valeria"}},{"Key":"CAR9", "Record":{"colour":"brown","make":"Holden","model":"Barina","owner":"Shotaro"}}]

7. GetStateByRangeWithPagination

- 수천건의 데이터 레코드가 있을 경우 특정 범위의 데이터 레코드만 추출하며 함수
- 레코드 시작과 원하는 레코드 수를 지정하여 범위를 선택함
- some arguments :  startKey and endKey, integer pageSize(how many items to be returned), string bookmark(the key to begin with)

7.1 New Coding 작업필요 : queryAllCarsWithPagination()

- two arguments : pageSize and bookmark
- fabcar chaincode 재사용

$ cd fabric-samples/chaincode
$ cp -r fabcar/go/ testrangepage/
$ cd testrangepage
$ mv fabcar.go testrangepage.go

7.2 chaincode 파일 작업
- queryAllCarsWithPagination 함수 추가

7.2.1 Code example : testrangepage.go - Invoke()에 queryAllCarsWithPagination() add
	
	func (s *SmartContract) Invoke(APIstub shim.ChaincodeStubInterface) sc.Response {

	// Retrieve the requested Smart Contract function and arguments
	function, args := APIstub.GetFunctionAndParameters()
	// Route to the appropriate handler function to interact with the ledger appropriately
	if function == "queryCar" {
		return s.queryCar(APIstub, args)
	} else if function == "initLedger" {
		return s.initLedger(APIstub)
	} else if function == "createCar" {
		return s.createCar(APIstub, args)
	} else if function == "queryAllCars" {
		return s.queryAllCars(APIstub)
	} else if function == "changeCarOwner" {
		return s.changeCarOwner(APIstub, args)
	} else if function == "queryAllCarsWithPagination" {
		return s.queryAllCarsWithPagination(APIstub, args)
	}
	
		return shim.Error("Invalid Smart Contract function name.")
	}
	
7.2.2 Code example : testrangepage.go - queryAllCarsWithPagination() : marbles0을 참조해서 queryAllCars()를 수정한 코드를 신규 추가.
	
	func (s *SmartContract) queryAllCarsWithPagination(APIstub shim.ChaincodeStubInterface, args []string) sc.Response {

		if len(args) < 2 {
			return shim.Error("Incorrect number of arguments. Expecting 2")
		}

		startKey := "CAR0"
		endKey := "CAR999"

		pageSize, err := strconv.ParseInt(args[0], 10, 32)
		if err != nil {
			return shim.Error(err.Error())
		}
		bookmark := args[1]

		resultsIterator, _, err := APIstub.GetStateByRangeWithPagination(startKey, endKey, int32(pageSize), bookmark)
		if err != nil {
			return shim.Error(err.Error())
		}
		defer resultsIterator.Close()

		// buffer is a JSON array containing QueryResults
		var buffer bytes.Buffer
		buffer.WriteString("[")

		bArrayMemberAlreadyWritten := false
		for resultsIterator.HasNext() {
			queryResponse, err := resultsIterator.Next()
			if err != nil {
				return shim.Error(err.Error())
			}
			// Add a comma before array members, suppress it for the first array member
			if bArrayMemberAlreadyWritten == true {
				buffer.WriteString(",")
			}
			buffer.WriteString("{\"Key\":")
			buffer.WriteString("\"")
			buffer.WriteString(queryResponse.Key)
			buffer.WriteString("\"")

			buffer.WriteString(", \"Record\":")
			// Record is a JSON object, so we write as-is
			buffer.WriteString(string(queryResponse.Value))
			buffer.WriteString("}")
			bArrayMemberAlreadyWritten = true
		}
			buffer.WriteString("]")

			fmt.Printf("- queryAllCars:\n%s\n", buffer.String())

			return shim.Success(buffer.Bytes())
	}
	
8. Bring up a new Basic Network with the new chaincode.

$ cd fabric-samples/basic-network
$ ./teardown.sh
$ ./start.sh

$ docker-compose up -d cli
$ docker exec cli peer chaincode install -n mycc -p github.com/testrangepage -v 0
$ docker exec cli peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -n mycc github.com/testrangepage -v 0 -c '{"Args": []}' -P "OR('Org1MSP.member')"
$ docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["initLedger"]}'

8.1 queryAllCars() 실행 (10개 레코드)

$ docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["queryAllCars"]}'
- Result : [{"Key":"CAR0", "Record":{"colour":"blue","make":"Toyota","model":"Prius","owner":"Tomoko"}},{"Key":"CAR1", "Record":{"colour":"red","make":"Ford","model":"Mustang","owner":"Brad"}},{"Key":"CAR2", "Record":{"colour":"green","make":"Hyundai","model":"Tucson","owner":"Jin Soo"}},{"Key":"CAR3", "Record":{"colour":"yellow","make":"Volkswagen","model":"Passat","owner":"Max"}},{"Key":"CAR4", "Record":{"colour":"black","make":"Tesla","model":"S","owner":"Adriana"}},{"Key":"CAR5", "Record":{"colour":"purple","make":"Peugeot","model":"205","owner":"Michel"}},{"Key":"CAR6", "Record":{"colour":"white","make":"Chery","model":"S22L","owner":"Aarav"}},{"Key":"CAR7", "Record":{"colour":"violet","make":"Fiat","model":"Punto","owner":"Pari"}},{"Key":"CAR8", "Record":{"colour":"indigo","make":"Tata","model":"Nano","owner":"Valeria"}},{"Key":"CAR9", "Record":{"colour":"brown","make":"Holden","model":"Barina","owner":"Shotaro"}}]

8.2 queryAllCarsWithPagination() 실행

8.2.1 CAR3으로 시작하는 5개의 레코드 쿼리

$ docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["queryAllCarsWithPagination", "5", "CAR3"]}'
- Result : [{"Key":"CAR3", "Record":{"colour":"yellow","make":"Volkswagen","model":"Passat","owner":"Max"}},{"Key":"CAR4", "Record":{"colour":"black","make":"Tesla","model":"S","owner":"Adriana"}},{"Key":"CAR5", "Record":{"colour":"purple","make":"Peugeot","model":"205","owner":"Michel"}},{"Key":"CAR6", "Record":{"colour":"white","make":"Chery","model":"S22L","owner":"Aarav"}},{"Key":"CAR7", "Record":{"colour":"violet","make":"Fiat","model":"Punto","owner":"Pari"}}]

8.2.2 CAR8으로 시작하는 5개의 레코드 쿼리

$ docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["queryAllCarsWithPagination", "5", "CAR8"]}'
- Result : [{"Key":"CAR8", "Record":{"colour":"indigo","make":"Tata","model":"Nano","owner":"Valeria"}},{"Key":"CAR9", "Record":{"colour":"brown","make":"Holden","model":"Barina","owner":"Shotaro"}}]

