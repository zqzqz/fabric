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
