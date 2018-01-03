# fabric学习笔记
## peer-chaincode
命令peer -> 子命令chaincode
### peer-chaincode-install
```
const installDesc = "Package the specified chaincode into a deployment spec and save it on the peer's path."
```
in fabric/peer/chaincode/install.go
entrance: chaincodeInstall

```
func chaincodeInstall(cmd *cobra.Command, ccpackfile string, cf *chaincodeCmdFactory) {
	if (ccpackfile=="") //不提供文件名的时候
		//生成默认chaincodeDeploymentSpec
		ccpackmsg = genChaincodeDeploymentSpec(cmd, chaincodeName, chaincodeVersion)
	else { //提供了文件名
		//cds 类型 *pb.ChaincodeDeploymentSpec
		ccpackmsg, cds = getPackageFromFile(ccpackfile)
	}
	install(ccpackmsg, cf)
}
```
getPackageFromFile
```
func getPackageFromFile(ccpackfile string) (proto.Message, *pb.ChaincodeDeploymentSpec, error) {
	...
	b = ioutil.ReadFile(ccpackfile)
	ccpack = ccprovider.GetCCPackage(b)
	o = ccpack.GetPackeageObject()
	cds, ok := o.(*pb.ChaincodeDeploymentSpec)
	...
	return o, cds, nil;
}
```
install
```
//install the depspec to "peer.address"
func install(msg proto.Message, cf *ChaincodeCmdFactory) error {
        creator, err := cf.Signer.Serialize()
        prop, _, err := utils.CreateInstallProposalFromCDS(msg, creator)
        var signedProp *pb.SignedProposal
        signedProp, err = utils.GetSignedProposal(prop, cf.Signer)
        proposalResponse, err := cf.EndorserClient.ProcessProposal(context.Background(), signedProp)
        return nil
}
```
type ChaincodeDeploymentSpec struct ----- in fabric/protos/peer/chaincode.pb.go:195


## leveldb
###  defination of kv-db in fabric/common/ledger/util/leveldbhelper
helper 设置db配置和基本操作 provider提供对外服务，包装基本操作
### kv_ledger_provider in fabric/core/ledger/kvledger
```
type Provider struct {
        idStore            *idStore
        blockStoreProvider blkstorage.BlockStoreProvider
        vdbProvider        statedb.VersionedDBProvider
        historydbProvider  historydb.HistoryDBProvider
}
```
### fabric/common/ledger/blkstorage/
定义了interface type BlockSoreProvider
in blockstorage.go
```
// BlockStoreProvider provides an handle to a BlockStore
type BlockStoreProvider interface {
        CreateBlockStore(ledgerid string) (BlockStore, error)
        OpenBlockStore(ledgerid string) (BlockStore, error)
        Exists(ledgerid string) (bool, error)
        List() ([]string, error)
        Close()
}
```
具体方法定义在：
fsBlockstoreProvider in fsblkstorage
```
ckstoreProvider struct {
        conf            *Conf
        indexConfig     *blkstorage.IndexConfig
        leveldbProvider *leveldbhelper.Provider
}
```
接入leveldb
```
// NewProvider constructs a filesystem based block store provider
func NewProvider(conf *Conf, indexConfig *blkstorage.IndexConfig) blkstorage.BlockStoreProvider {
        p := leveldbhelper.NewProvider(&leveldbhelper.Conf{DBPath: conf.getIndexDir()})
        return &FsBlockstoreProvider{conf, indexConfig, p}
}
```
### fabric/core/ledger/ledger_interface.go
账本接口

## chaincode support in peers
### start
in fabric/peer/node/start.go: line 136
```
ccSrv, ccEpFunc := createChaincodeServer(peerServer, listenAddr)
registerChaincodeSupport(ccSrv.Server(), ccEpFunc)
go ccSrv.Start()
```
### defination
in fabric/core/chaincode/chaincode_support.go
```
//this is basically the singleton that supports the
//entire chaincode framework. It does NOT know about
//chains. Chains are per-proposal entities that are
//setup as part of "join" and go through this object
//via calls to Execute and Deploy chaincodes.
var theChaincodeSupport *ChaincodeSupport
```
### handler.go
利用有穷自动机实现chaincode的核心部分
```
// Handler responsible for management of Peer's side of chaincode stream
type Handler struct {
        sync.RWMutex
        //peer to shim grpc serializer. User only in serialSend
        serialLock  sync.Mutex
        ChatStream  ccintf.ChaincodeStream
        FSM         *fsm.FSM
        ChaincodeID *pb.ChaincodeID
        ccInstance  *sysccprovider.ChaincodeInstance

        chaincodeSupport *ChaincodeSupport
        registered       bool
        readyNotify      chan bool
        // Map of tx txid to either invoke tx. Each tx will be
        // added prior to execute and remove when done execute
        txCtxs map[string]*transactionContext

        txidMap map[string]bool

        // used to do Send after making sure the state transition is complete
        nextState chan *nextStateInfo

        policyChecker policy.PolicyChecker
}
```
### FSM
in fabric/core/chaincode
事件状态
```
const (
        createdstate     = "created"     //start state
        establishedstate = "established" //in: CREATED, rcv:  REGISTER, send: REGISTERED, INIT
        readystate       = "ready"       //in:ESTABLISHED,TRANSACTION, rcv:COMPLETED
        endstate         = "end"         //in:INIT,ESTABLISHED, rcv: error, terminate container

)
```
in fabric/protos/peer/chaincode_shim.pb.go
状态列表
```
type ChaincodeMessage_Type int32

const (
        ChaincodeMessage_UNDEFINED           ChaincodeMessage_Type = 0
        ChaincodeMessage_REGISTER            ChaincodeMessage_Type = 1
        ChaincodeMessage_REGISTERED          ChaincodeMessage_Type = 2
        ChaincodeMessage_INIT                ChaincodeMessage_Type = 3
        ChaincodeMessage_READY               ChaincodeMessage_Type = 4
        ChaincodeMessage_TRANSACTION         ChaincodeMessage_Type = 5
        ChaincodeMessage_COMPLETED           ChaincodeMessage_Type = 6
        ChaincodeMessage_ERROR               ChaincodeMessage_Type = 7
        ChaincodeMessage_GET_STATE           ChaincodeMessage_Type = 8
        ChaincodeMessage_PUT_STATE           ChaincodeMessage_Type = 9
        ChaincodeMessage_DEL_STATE           ChaincodeMessage_Type = 10
        ChaincodeMessage_INVOKE_CHAINCODE    ChaincodeMessage_Type = 11
        ChaincodeMessage_RESPONSE            ChaincodeMessage_Type = 13
        ChaincodeMessage_GET_STATE_BY_RANGE  ChaincodeMessage_Type = 14
        ChaincodeMessage_GET_QUERY_RESULT    ChaincodeMessage_Type = 15
        ChaincodeMessage_QUERY_STATE_NEXT    ChaincodeMessage_Type = 16
        ChaincodeMessage_QUERY_STATE_CLOSE   ChaincodeMessage_Type = 17
        ChaincodeMessage_KEEPALIVE           ChaincodeMessage_Type = 18
        ChaincodeMessage_GET_HISTORY_FOR_KEY ChaincodeMessage_Type = 19
)
```
## how chaincode api works
主要接口在 fabeic/chaincode/shim/interfaces.go  
其中Chaincode接口的Init/Invoke方法是每个chaincode必须实现的  
in fabric/core/chaincode/shim/chaincode.go
```
// ChaincodeStub is an object passed to chaincode for shim side handling of
// APIs.
type ChaincodeStub struct {
        TxID           string
        chaincodeEvent *pb.ChaincodeEvent
        args           [][]byte
        handler        *Handler
        signedProposal *pb.SignedProposal
        proposal       *pb.Proposal

        // Additional fields extracted from the signedProposal
        creator   []byte
        transient map[string][]byte
        binding   []byte
}
```
编写chaincode时使用stub *ChaincodeStubInterface接口。主要的方法是ChaincodeStub实现的。   
调用chaincode时使用stub.handler的方法处理。  
Handler上文提到。我们关注数据库操作，以function PutState为例。  
定位同目录下 fabric/core/chaincode/shim/handler.go  
```
// Handler handler implementation for shim side of chaincode.
type Handler struct {
        sync.RWMutex
        //shim to peer grpc serializer. User only in serialSend
        serialLock sync.Mutex
        To         string
        ChatStream PeerChaincodeStream
        FSM        *fsm.FSM
        cc         Chaincode
        // Multiple queries (and one transaction) with different txids can be executing in parallel for this chaincode
        // responseChannel is the channel on which responses are communicated by the shim to the chaincodeStub.
        responseChannel map[string]chan pb.ChaincodeMessage
        nextState       chan *nextStateInfo
}
```
定位function handlePutState，关键调用
```
responseMsg, err = handler.sendReceive(msg, respChan);
```
该方法也在handler.go文件中，依次调用  
handler.serialSendAsync(msg, errc)  
handler.serialSend(msg)  
handler.ChatStream.Send(msg)
ChatStream成员的类型PeerChaincodeStream 定义在本文件
```
// PeerChaincodeStream interface for stream between Peer and chaincode instance.
type PeerChaincodeStream interface {
        Send(*pb.ChaincodeMessage) error
        Recv() (*pb.ChaincodeMessage, error)
        CloseSend() error
}
```
Send由inProcStream实现 in inprostream.go  
接下来交给peer处理。。。
