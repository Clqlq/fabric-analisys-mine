## 代码结构
```
/identity
|	|-　identity.go
```
---

## identity.go
- `var usageThreshold = time.Hour` // identityUsageThreshold　设置了一个身份在被删除之前，不能用来验证一些签名的最长时间

- Mapper 接口（PS：感觉这个接口里的方法在很多地方都有）
```go
// Mapper 维护 pkiID 到 peers　的 
//　certificates（证书）（identities（身份））之间的映射
type Mapper interface {
	//　Put 将 identity 和它的给定的 pkiID 联系起来
	// 如果给定的 pkiID 与 identity　不匹配则返回 error
	Put(pkiID common.PKIidType, identity api.PeerIdentityType) error

	// Get 返回给定 pkiID 的 indentity
	// 如果该 indentity　不存在则返回 error
	Get(pkiID common.PKIidType) (api.PeerIdentityType, error)

	// Sign　对一个 message 进行签名
	// 如果成功则返回一个签过名的 message　失败返回一个 error
	Sign(msg []byte) ([]byte, error)

	// Verify verifies a signed message
	// 验证一个签过名的 message
	Verify(vkID, signature, message []byte) error

	//　GetPKIidOfCert　一个证书的PKI-ID
	GetPKIidOfCert(api.PeerIdentityType) common.PKIidType

	// SuspectPeers 重验证那些满足给定 predicate（过滤条件）的所有 peers
	SuspectPeers(isSuspected api.PeerSuspector)

	// Stop 停止 Mapper 的所有后台计算
	Stop()
}
```
- identityMapperImpl 结构体是对 Mapper 接口的实现
```go
type purgeTrigger func(pkiID common.PKIidType, identity api.PeerIdentityType)

type identityMapperImpl struct {
	onPurge    purgeTrigger
	mcs        api.MessageCryptoService	//api/crypto.go 文件下的结构体
	pkiID2Cert map[string]*storedIdentity
	sync.RWMutex
	stopChan chan struct{}
	sync.Once
	selfPKIID string
}
```
- `func NewIdentityMapper(mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType, onPurge purgeTrigger) Mapper` 创建一个新的 identityMapperImpl，所需要的只是一个对 MessageCryptoService 类型变量的引用
	- 在新建一个 identityMapperImpl 实例的同时，还开启了一个 goroutine　`go idMapper.periodicalPurgeUnusedIdentities()`来定期清除不使用的的证书（PS：这个方法里的具体操作还要好好看一下，和 Stop　方法的实现也有关系）

---

主要看一下 identityMapperImpl 对接口中几个方法的实现

- `func (is *identityMapperImpl) Put(pkiID common.PKIidType, identity api.PeerIdentityType) error`
	- 首先检查并得到证书（identity）的过期时间
	- 验证证书的有效性，证书有效则获取该证书的 ID
	- 比较获得的 ID 与所给 pkiID 是否一致，不一致则返回匹配错误
	- 给该实例上锁
	- 判断证书是否存在，存在则直接返回 nil，不存在则执行下面的操作
	- 检查上面得到的证书的过期时间，如果证书不过期（过期时间为0）不做操作；如果证书有具体过期时间则调用 time.AfterFunc() 函数，开启一个 goroutine，在证书过期后1毫秒之后，将证书从该实例的 pkiID2Cert 字段的字典中删除
	- 将证书存储进实例的 pkiID2Cert 字段的字典中
	- 返回 nil
	- 给该实例解锁

- `func (is *identityMapperImpl) Get(pkiID common.PKIidType) (api.PeerIdentityType, error)`　返回给定 pkiID 的证书

- `func (is *identityMapperImpl) Sign(msg []byte) ([]byte, error)` 对 message 签名，成功则返回签名后的 message，失败返回 error

- `func (is *identityMapperImpl) Stop()`　将一个空结构体发送到 is.stopChan 中（PS：没看懂是什么操作，是不是和前面的 periodicalPurgeUnusedIdentities 有关系？）

- `func (is *identityMapperImpl) Verify(vkID, signature, message []byte) error` 获得所给 pkiID（即 vkID 参数）的证书，验证该证书对该 message 的签名）

- `func (is *identityMapperImpl) GetPKIidOfCert(identity api.PeerIdentityType) common.PKIidType` 获得所给证书的 pkiID

- `func (is *identityMapperImpl) SuspectPeers(isSuspected api.PeerSuspector) ` 将已撤销/过期/长时间不使用的证书从该实例的 pkiID2Cert 字典中删除对应键对值

---

- `func SetIdentityUsageThreshold(duration time.Duration)` 设置证书最长不使用的时间 usageThreshold，若证书在给定时间内未曾使用一次，则将证书清楚

- `func GetIdentityUsageThreshold() time.Duration` 返回所设置的 usageThreshold


