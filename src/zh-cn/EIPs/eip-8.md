---
eip: 8
title: devp2p 向后兼容性要求（Homestead）
author: Felix Lange <felix@ethdev.com>
status: Final
type: Standards Track
category: Networking
created: 2015-12-18
---

### 摘要

本 EIP 引入了对 devp2p Wire 协议、RLPx 发现协议和 RLPx TCP 传输协议实现的新向后兼容性要求。实现 EIP-8 的客户端应遵循 Postel 法则：

> 在你所做的事情上要保守，在你接受他人时要宽容。

### 规范

**devp2p Wire 协议**的实现应忽略 hello 数据包的版本号。在发送 hello 数据包时，版本元素应设置为支持的最高 devp2p 版本。实现还应忽略 hello 数据包末尾的任何额外列表元素。

同样，**RLPx 发现协议**的实现不应验证 ping 数据包的版本号，忽略任何数据包中的额外列表元素，并忽略任何数据包中第一个 RLP 值后的数据。未知数据包类型的发现数据包应被静默丢弃。任何发现数据包的最大大小仍为 1280 字节。

最后，**RLPx TCP 传输协议**的实现应接受加密密钥建立握手数据包的新编码。如果接收到 EIP-8 风格的 RLPx `auth-packet`，则应根据以下规则发送相应的 `ack-packet`。

解码 `auth-body` 和 `ack-body` 中的 RLP 数据时，应忽略 `auth-vsn` 和 `ack-vsn` 的不匹配、任何额外的列表元素以及列表后的任何尾随数据。在过渡期间（即在旧格式被淘汰之前），实现应用至少 100 字节的垃圾数据填充 `auth-body`。建议添加一个范围在 [100, 300] 之间的随机量，以变化数据包的大小。

```text
auth-vsn         = 4
auth-size        = size of enc-auth-body, encoded as a big-endian 16-bit integer
auth-body        = rlp.list(sig, initiator-pubk, initiator-nonce, auth-vsn)
enc-auth-body    = ecies.encrypt(recipient-pubk, auth-body, auth-size)
auth-packet      = auth-size || enc-auth-body

ack-vsn          = 4
ack-size         = size of enc-ack-body, encoded as a big-endian 16-bit integer
ack-body         = rlp.list(recipient-ephemeral-pubk, recipient-nonce, ack-vsn)
enc-ack-body     = ecies.encrypt(initiator-pubk, ack-body, ack-size)
ack-packet       = ack-size || enc-ack-body

where

X || Y
    denotes concatenation of X and Y.
X[:N]
    denotes an N-byte prefix of X.
rlp.list(X, Y, Z, ...)
    denotes recursive encoding of [X, Y, Z, ...] as an RLP list.
sha3(MESSAGE)
    is the Keccak256 hash function as used by Ethereum.
ecies.encrypt(PUBKEY, MESSAGE, AUTHDATA)
    is the asymmetric authenticated encryption function as used by RLPx.
    AUTHDATA is authenticated data which is not part of the resulting ciphertext,
    but written to HMAC-256 before generating the message tag.
```

### 动机

由于运行旧版本的客户端在 hello（发现 ping，RLPx 握手）数据包的版本号或结构不符合本地期望时会拒绝通信，因此对 devp2p 协议的更改很难部署。

将向后兼容性要求作为 Homestead 共识升级的一部分引入，将确保以太坊网络上使用的所有客户端软件能够应对未来的网络协议升级（只要保持向后兼容性）。

### 理由

所提议的更改通过在整个协议栈中应用 Postel 法则（也称为鲁棒性原则）来解决向后兼容性。自 RFC 761 最初应用以来，这种方法的优点和适用性已被反复研究。有关最近的观点，请参见
["鲁棒性原则再考虑"（Eric Allman, 2011）](https://queue.acm.org/detail.cfm?id=1999945)。

#### 对 devp2p Wire 协议的更改

所有客户端当前都包含如下语句：

```python
# pydevp2p/p2p_protocol.py
if data['version'] != proto.version:
    log.debug('incompatible network protocols', peer=proto.peer,
        expected=proto.version, received=data['version'])
    return proto.send_disconnect(reason=reasons.incompatibel_p2p_version)
```

这些检查使得无法更改 hello 数据包的版本或结构。去掉它们可以切换到更新的协议版本：实现更新版本的客户端只需发送一个具有更高版本和可能额外列表元素的数据包。

* 如果这样的数据包被低版本的节点接收，它将盲目假设远程端是向后兼容的，并以旧的握手进行响应。
* 如果数据包被相同版本的节点接收，则可以使用协议的新特性。
* 如果数据包被高版本的节点接收，它可以启用向后兼容性逻辑或断开连接。

#### 对 RLPx 发现协议的更改

对发现数据包解码规则的放宽在很大程度上规范了当前的实践。大多数现有实现不关心列表元素的数量（go-ethereum 是一个例外），并且不拒绝版本不匹配的节点。然而，这种行为并未在规范中得到保证。

如果被采纳，该更改使得以类似于 devp2p hello 更改的方式部署协议更改成为可能：简单地提升版本并发送额外信息。旧客户端将忽略额外元素，即使网络的大多数部分已经转向更新的协议，它们仍然可以继续运行。

#### 对 RLPx TCP 握手的更改

关于 RLPx v5 更改（分块数据包、密钥派生的变化）的讨论部分停滞不前，因为 v4 握手编码仅提供了一种带内方式来添加版本号：缩短随机部分的 nonce。即使 RLPx v5 握手提案被接受，未来的升级也很困难，因为握手数据包是固定大小的 ECIES 密文，具有已知布局。

我提议对握手数据包进行以下更改：

* 将密文的长度添加为明文头部。
* 将握手的主体编码为 RLP。
* 在两个数据包中添加版本号以替代标记标志（未使用）。
* 移除临时公钥的哈希（它是冗余的）。

这些更改使得以与其他协议相同的方式升级 RLPx TCP 传输协议成为可能，即通过添加列表元素和提升版本。由于这是对 RLPx 握手数据包的第一次更改，我们可以抓住机会移除所有当前未使用的字段。

在 RLP 列表之后允许（实际上是要求）额外数据，因为握手数据包需要增长以便与旧格式区分开。客户端可以使用如下伪代码同时处理两种格式。

```go
packet = read(307, connection)
if decrypt(packet) {
    // process as old format
} else {
    size = unpack_16bit_big_endian(packet)
    packet += read(size - 307 + 2, connection)
    if !decrypt(packet) {
        // error
    }
    // process as new format
}
```

明文大小前缀可能是本文档中最具争议的方面。有人认为前缀有助于试图在网络层过滤和识别 RLPx 连接的对手。

这在很大程度上取决于对手愿意花费多少精力。如果遵循随机化长度的建议，纯模式识别的数据包识别不太可能成功。

* 对于典型的防火墙操作员，阻止所有前两个字节形成范围在 [300,600] 之间的整数的连接可能过于侵入。基于端口的阻止将是过滤大多数 RLPx 流量的更有效措施。
* 对于能够关联许多标准的攻击者，大小前缀将简化识别，因为它增加了指示器集。然而，这样的攻击者也可以被期望读取或参与 RLPx 发现流量，这将足以阻止 RLPx TCP 连接，无论其格式如何。

### 向后兼容性

本 EIP 是向后兼容的，所有有效的版本 4 数据包仍然被接受。

### 实现

[go-ethereum](https://github.com/ethereum/go-ethereum/pull/2091)
[libweb3core](https://github.com/ethereum/libweb3core/pull/46)
[pydevp2p](https://github.com/ethereum/pydevp2p/pull/32)

### 测试向量

#### devp2p 基础协议

devp2p hello 数据包广告版本 22 并包含一些额外的列表元素：

```text
f87137916b6e6574682f76302e39312f706c616e39cdc5836574683dc6846d6f726b1682270fb840
fda1cff674c90c9a197539fe3dfb53086ace64f83ed7c6eabec741f7f381cc803e52ab2cd55d5569
bce4347107a310dfd5f88a010cd2ffd1005ca406f1842877c883666f6f836261720304
```

#### RLPx 发现协议

实现应接受以下编码的发现数据包为有效。数据包使用 secp256k1 节点密钥签名

```text
b71c71a67e1177ad4e901695e1b4b9ee17ae16c6668d313eac2f96dbcda3f291
```

版本 4 的 ping 数据包，附加列表元素：
```text
e9614ccfd9fc3e74360018522d30e1419a143407ffcce748de3e22116b7e8dc92ff74788c0b6663a
aa3d67d641936511c8f8d6ad8698b820a7cf9e1be7155e9a241f556658c55428ec0563514365799a
4be2be5a685a80971ddcfa80cb422cdd0101ec04cb847f000001820cfa8215a8d790000000000000
000000000000000000018208ae820d058443b9a3550102
```

ping 数据包，版本为 555，附加列表元素和附加随机数据：

```text
577be4349c4dd26768081f58de4c6f375a7a22f3f7adda654d1428637412c3d7fe917cadc56d4e5e
7ffae1dbe3efffb9849feb71b262de37977e7c7a44e677295680e9e38ab26bee2fcbae207fba3ff3
d74069a50b902a82c9903ed37cc993c50001f83e82022bd79020010db83c4d001500000000abcdef
12820cfa8215a8d79020010db885a308d313198a2e037073488208ae82823a8443b9a355c5010203
040531b9019afde696e582a78fa8d95ea13ce3297d4afb8ba6433e4154caa5ac6431af1b80ba7602
3fa4090c408f6b4bc3701562c031041d4702971d102c9ab7fa5eed4cd6bab8f7af956f7d565ee191
7084a95398b6a21eac920fe3dd1345ec0a7ef39367ee69ddf092cbfe5b93e5e568ebc491983c09c7
6d922dc3
```

pong 数据包，附加列表元素和附加随机数据：

```text
09b2428d83348d27cdf7064ad9024f526cebc19e4958f0fdad87c15eb598dd61d08423e0bf66b206
9869e1724125f820d851c136684082774f870e614d95a2855d000f05d1648b2d5945470bc187c2d2
216fbe870f43ed0909009882e176a46b0102f846d79020010db885a308d313198a2e037073488208
ae82823aa0fbc914b16819237dcd8801d7e53f69e9719adecb3cc0e790c57e91ca4461c9548443b9
a355c6010203c2040506a0c969a58f6f9095004c0177a6b47f451530cab38966a25cca5cb58f0555
42124e
```

findnode 数据包，附加列表元素和附加随机数据：

```text
c7c44041b9f7c7e41934417ebac9a8e1a4c6298f74553f2fcfdcae6ed6fe53163eb3d2b52e39fe91
831b8a927bf4fc222c3902202027e5e9eb812195f95d20061ef5cd31d502e47ecb61183f74a504fe
04c51e73df81f25c4d506b26db4517490103f84eb840ca634cae0d49acb401d8a4c6b6fe8c55b70d
115bf400769cc1400f3258cd31387574077f301b421bc84df7266c44e9e6d569fc56be0081290476
7bf5ccd1fc7f8443b9a35582999983999999280dc62cc8255c73471e0a61da0c89acdc0e035e260a
dd7fc0c04ad9ebf3919644c91cb247affc82b69bd2ca235c71eab8e49737c937a2c396
```

neighbours 数据包，附加列表元素和附加随机数据：

```text
c679fc8fe0b8b12f06577f2e802d34f6fa257e6137a995f6f4cbfc9ee50ed3710faf6e66f932c4c8
d81d64343f429651328758b47d3dbc02c4042f0fff6946a50f4a49037a72bb550f3a7872363a83e1
b9ee6469856c24eb4ef80b7535bcf99c0004f9015bf90150f84d846321163782115c82115db84031
55e1427f85f10a5c9a7755877748041af1bcd8d474ec065eb33df57a97babf54bfd2103575fa8291
15d224c523596b401065a97f74010610fce76382c0bf32f84984010203040101b840312c55512422
cf9b8a4097e9a6ad79402e87a15ae909a4bfefa22398f03d20951933beea1e4dfa6f968212385e82
9f04c2d314fc2d4e255e0d3bc08792b069dbf8599020010db83c4d001500000000abcdef12820d05
820d05b84038643200b172dcfef857492156971f0e6aa2c538d8b74010f8e140811d53b98c765dd2
d96126051913f44582e8c199ad7c6d6819e9a56483f637feaac9448aacf8599020010db885a308d3
13198a2e037073488203e78203e8b8408dcab8618c3253b558d459da53bd8fa68935a719aff8b811
197101a4b2b47dd2d47295286fc00cc081bb542d760717d1bdd6bec2c37cd72eca367d6dd3b9df73
8443b9a355010203b525a138aa34383fec3d2719a0
```

#### RLPx 握手

在这些测试向量中，节点 A 向节点 B 发起连接。
所有数据包中包含的值如下：

```text
Static Key A:    49a7b37aa6f6645917e7b807e9d1c00d4fa71f18343b0d4122a4d2df64dd6fee
Static Key B:    b71c71a67e1177ad4e901695e1b4b9ee17ae16c6668d313eac2f96dbcda3f291
Ephemeral Key A: 869d6ecf5211f1cc60418a13b9d870b22959d0c16f02bec714c960dd2298a32d
Ephemeral Key B: e238eb8e04fee6511ab04c6dd3c89ce097b11f25d584863ac2b6d5b35b1847e4
Nonce A:         7e968bba13b6c50e2c4cd7f241cc0d64d1ac25c7f5952df231ac6a2bda8ee5d6
Nonce B:         559aead08264d5795d3909718cdd05abd49572e84fe55590eef31a88a08fdffd
```

(Auth₁)  RLPx v4 格式（从 A 发送到 B）：
```text
048ca79ad18e4b0659fab4853fe5bc58eb83992980f4c9cc147d2aa31532efd29a3d3dc6a3d89eaf
913150cfc777ce0ce4af2758bf4810235f6e6ceccfee1acc6b22c005e9e3a49d6448610a58e98744
ba3ac0399e82692d67c1f58849050b3024e21a52c9d3b01d871ff5f210817912773e610443a9ef14
2e91cdba0bd77b5fdf0769b05671fc35f83d83e4d3b0b000c6b2a1b1bba89e0fc51bf4e460df3105
c444f14be226458940d6061c296350937ffd5e3acaceeaaefd3c6f74be8e23e0f45163cc7ebd7622
0f0128410fd05250273156d548a414444ae2f7dea4dfca2d43c057adb701a715bf59f6fb66b2d1d2
0f2c703f851cbf5ac47396d9ca65b6260bd141ac4d53e2de585a73d1750780db4c9ee4cd4d225173
a4592ee77e2bd94d0be3691f3b406f9bba9b591fc63facc016bfa8
```

(Auth₂) EIP-8 格式，版本为 4，没有附加列表元素（从 A 发送到 B）：
```text
01b304ab7578555167be8154d5cc456f567d5ba302662433674222360f08d5f1534499d3678b513b
0fca474f3a514b18e75683032eb63fccb16c156dc6eb2c0b1593f0d84ac74f6e475f1b8d56116b84
9634a8c458705bf83a626ea0384d4d7341aae591fae42ce6bd5c850bfe0b999a694a49bbbaf3ef6c
da61110601d3b4c02ab6c30437257a6e0117792631a4b47c1d52fc0f8f89caadeb7d02770bf999cc
147d2df3b62e1ffb2c9d8c125a3984865356266bca11ce7d3a688663a51d82defaa8aad69da39ab6
d5470e81ec5f2a7a47fb865ff7cca21516f9299a07b1bc63ba56c7a1a892112841ca44b6e0034dee
70c9adabc15d76a54f443593fafdc3b27af8059703f88928e199cb122362a4b35f62386da7caad09
c001edaeb5f8a06d2b26fb6cb93c52a9fca51853b68193916982358fe1e5369e249875bb8d0d0ec3
6f917bc5e1eafd5896d46bd61ff23f1a863a8a8dcd54c7b109b771c8e61ec9c8908c733c0263440e
2aa067241aaa433f0bb053c7b31a838504b148f570c0ad62837129e547678c5190341e4f1693956c
3bf7678318e2d5b5340c9e488eefea198576344afbdf66db5f51204a6961a63ce072c8926c
```

(Auth₃) EIP-8 格式，版本为 56，包含 3 个附加列表元素（从 A 发送到 B）：
```text
01b8044c6c312173685d1edd268aa95e1d495474c6959bcdd10067ba4c9013df9e40ff45f5bfd6f7
2471f93a91b493f8e00abc4b80f682973de715d77ba3a005a242eb859f9a211d93a347fa64b597bf
280a6b88e26299cf263b01b8dfdb712278464fd1c25840b995e84d367d743f66c0e54a586725b7bb
f12acca27170ae3283c1073adda4b6d79f27656993aefccf16e0d0409fe07db2dc398a1b7e8ee93b
cd181485fd332f381d6a050fba4c7641a5112ac1b0b61168d20f01b479e19adf7fdbfa0905f63352
bfc7e23cf3357657455119d879c78d3cf8c8c06375f3f7d4861aa02a122467e069acaf513025ff19
6641f6d2810ce493f51bee9c966b15c5043505350392b57645385a18c78f14669cc4d960446c1757
1b7c5d725021babbcd786957f3d17089c084907bda22c2b2675b4378b114c601d858802a55345a15
116bc61da4193996187ed70d16730e9ae6b3bb8787ebcaea1871d850997ddc08b4f4ea668fbf3740
7ac044b55be0908ecb94d4ed172ece66fd31bfdadf2b97a8bc690163ee11f5b575a4b44e36e2bfb2
f0fce91676fd64c7773bac6a003f481fddd0bae0a1f31aa27504e2a533af4cef3b623f4791b2cca6
d490
```

(Ack₁) RLPx v4 格式（从 B 发送到 A）：
```text
049f8abcfa9c0dc65b982e98af921bc0ba6e4243169348a236abe9df5f93aa69d99cadddaa387662
b0ff2c08e9006d5a11a278b1b3331e5aaabf0a32f01281b6f4ede0e09a2d5f585b26513cb794d963
5a57563921c04a9090b4f14ee42be1a5461049af4ea7a7f49bf4c97a352d39c8d02ee4acc416388c
1c66cec761d2bc1c72da6ba143477f049c9d2dde846c252c111b904f630ac98e51609b3b1f58168d
dca6505b7196532e5f85b259a20c45e1979491683fee108e9660edbf38f3add489ae73e3dda2c71b
d1497113d5c755e942d1
```

(Ack₂) EIP-8 格式，版本为 4，没有附加列表元素（从 B 发送到 A）：
```text
01ea0451958701280a56482929d3b0757da8f7fbe5286784beead59d95089c217c9b917788989470
b0e330cc6e4fb383c0340ed85fab836ec9fb8a49672712aeabbdfd1e837c1ff4cace34311cd7f4de
05d59279e3524ab26ef753a0095637ac88f2b499b9914b5f64e143eae548a1066e14cd2f4bd7f814
c4652f11b254f8a2d0191e2f5546fae6055694aed14d906df79ad3b407d94692694e259191cde171
ad542fc588fa2b7333313d82a9f887332f1dfc36cea03f831cb9a23fea05b33deb999e85489e645f
6aab1872475d488d7bd6c7c120caf28dbfc5d6833888155ed69d34dbdc39c1f299be1057810f34fb
e754d021bfca14dc989753d61c413d261934e1a9c67ee060a25eefb54e81a4d14baff922180c395d
3f998d70f46f6b58306f969627ae364497e73fc27f6d17ae45a413d322cb8814276be6ddd13b885b
201b943213656cde498fa0e9ddc8e0b8f8a53824fbd82254f3e2c17e8eaea009c38b4aa0a3f306e8
797db43c25d68e86f262e564086f59a2fc60511c42abfb3057c247a8a8fe4fb3ccbadde17514b7ac
8000cdb6a912778426260c47f38919a91f25f4b5ffb455d6aaaf150f7e5529c100ce62d6d92826a7
1778d809bdf60232ae21ce8a437eca8223f45ac37f6487452ce626f549b3b5fdee26afd2072e4bc7
5833c2464c805246155289f4
```

(Ack₃) EIP-8 格式，版本为 57，包含 3 个附加列表元素（从 B 发送到 A）：
```text
01f004076e58aae772bb101ab1a8e64e01ee96e64857ce82b1113817c6cdd52c09d26f7b90981cd7
ae835aeac72e1573b8a0225dd56d157a010846d888dac7464baf53f2ad4e3d584531fa203658fab0
3a06c9fd5e35737e417bc28c1cbf5e5dfc666de7090f69c3b29754725f84f75382891c561040ea1d
dc0d8f381ed1b9d0d4ad2a0ec021421d847820d6fa0ba66eaf58175f1b235e851c7e2124069fbc20
2888ddb3ac4d56bcbd1b9b7eab59e78f2e2d400905050f4a92dec1c4bdf797b3fc9b2f8e84a482f3
d800386186712dae00d5c386ec9387a5e9c9a1aca5a573ca91082c7d68421f388e79127a5177d4f8
590237364fd348c9611fa39f78dcdceee3f390f07991b7b47e1daa3ebcb6ccc9607811cb17ce51f1
c8c2c5098dbdd28fca547b3f58c01a424ac05f869f49c6a34672ea2cbbc558428aa1fe48bbfd6115
8b1b735a65d99f21e70dbc020bfdface9f724a0d1fb5895db971cc81aa7608baa0920abb0a565c9c
436e2fd13323428296c86385f2384e408a31e104670df0791d93e743a3a5194ee6b076fb6323ca59
3011b7348c16cf58f66b9633906ba54a2ee803187344b394f75dd2e663a57b956cb830dd7a908d4f
39a2336a61ef9fda549180d4ccde21514d117b6c6fd07a9102b5efe710a32af4eeacae2cb3b1dec0
35b9593b48b9d3ca4c13d245d5f04169b0b1
```

节点 B 根据 (Auth₂, Ack₂) 派生连接密钥如下：

```text
aes-secret = 80e8632c05fed6fc2a13b0f8d31a3cf645366239170ea067065aba8e28bac487
mac-secret = 2ea74ec5dae199227dff1af715362700e989d889d7a493cb0639691efb8e5f98
```

运行 B 的 `ingress-mac` keccak 状态在字符串 "foo" 上产生的哈希为

```text
ingress-mac("foo") = 0c7ec6340062cc46f5e9f1e3cf86f8c8c403c5a0964f5df0ebd34a75ddc86db5
```

### 版权

版权及相关权利通过 [CC0](../LICENSE.md) 放弃。