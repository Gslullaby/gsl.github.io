---
title: BIP 32
date: 2018-09-27 17:29:35
tags:
- 区块链
- BIP
keywords:
- BIP
- 区块链
categories:
- 区块链
description: 此篇为BIP 32的译文，介绍分层确定性钱包
---
此篇为BIP 32的译文，介绍分层确定性钱包
[原文链接](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)

## 概述
本文介绍分层确定性钱包（HD Wallet）：一种可以在不同系统间部分或全部共享，有或没有花费的能力的钱包

该规范旨在为可在不同客户端之间通信的确定性钱包设定标准。虽然本文描述的钱包有很多特性，但对于钱包客户端来说并不是每个特性都需要实现    

本文分为两部分
* 第一部分主要介绍如何从单一种子派生出一颗秘钥树
* 第二部分主要介绍如何利用派生出来的秘钥树构建钱包

## 动机
#### 非确定性钱包的缺点
现有的一些Bitcoin客户端使用随机生成的秘钥，为了避免为每笔交易后都进行备份，这些客户端会创建一个（默认）包含100个秘钥的秘钥池。并且这种钱包很难跨系统共享，或者多端同时使用。虽然可以使用钱包的加密特性隐藏私钥并且不共享密码，但是这也导致钱包失去了生成公钥的能力。

#### 确定性钱包及非确定钱包   
确定性钱包不需要如此高频率的备份，而且由于椭圆曲线的数学特性，他可以在不暴露私钥的情况下计算出公钥。该特性就当于一个网络商店商家可以让他的服务器为每个顾客或每笔订单都生成一个新地址（即公钥hash），并且服务器不需要获取私钥。    
然而，确定性钱包只有一条密钥对链组成。这个事实意味着钱包只能全部共享或不共享两种选择。但是在某些场景下我们只希望共享部分密钥（如只共享公钥）。还是拿网络商店举例，网络服务器并不需要访问商户的所有公钥，他们只需要获取用于接受客户付款的地址对应的公钥，而不是商户消费时生成的找零地址。而分层确定性钱包支持从单一root多条密钥链的特性，使得选择性的分享成为了可能。    

## 密钥生成规范
#### 约定
在本文的其余部分，我们将假定比特币中使用的公钥加密算法，即为secp256k1定义的字段和曲线参数椭圆曲线加密算法。变量如下
* 整数模数曲线顺序（用n表示）
* 曲线上点的坐标
* 字节序列
另外两个坐标的加法（+）定义为EC组操作的应用
级联（||）是把一个字节序列附加到另一个字节序列的操作

作为标准的转换函数，我们假设
* point(t):返回secp256k1基点与EC点做乘法（重复应用EC组操作）得到的坐标对与整数p。
* ser32(i):把一个32位的无符号整数序列化为一个4字节序列，高位在前
* ser256(p):把一个整数序列化为一个32字节序列，高位在前
* serP(P):使用SEC1的压缩格式将坐标对p(x,y)序列化为字节序列：(0x02 or 0x03) || ser256(x),其中头部字节为y坐标的奇偶校验
* parse256(p):将32字节序列转化为一个256位数字，高位字节优先

#### 扩展密钥
接下来，我们顶一个一个函数，该函数可以从一个给定的父密钥派生出一定数量的子密钥。为了防止派生密钥仅依赖父密钥本身，我们会额外加入一个256位的熵对私钥与公钥进行扩展。熵称之为链码，链码对于相应的公私钥是相同的，都为32字节    
我们把一个扩展私钥表示为(k,c)，k为普通私钥，c为256位的熵。扩展公钥表示为(K,c)，其中K=point(k)，c为256位的熵    
每个扩展密钥有2<sup>31</sup>个普通子密钥，以及2<sup>31</sup>个强化子密钥，每个密钥都有一个下标。普通密钥的下标为0 - 2<sup>31</sup>-1，强化密钥的下标为2<sup>31</sup> - 2<sup>32</sup>-1，为了方便，我们使用i(H)来标记强化密钥，i<sub>H</sub> = i + 2<sup>31</sup>

#### 子密钥派生函数（CKD：child key derivation）
给定一个父扩展密钥及其下标i，可以计算出相应的子扩展密钥，算法取决于子扩展密钥是否是一个强化密钥（或即i是否>=2^31），以及我们是否在讨论私钥或公钥

###### 父私钥 -> 子私钥
函数CKDpriv((k<sub>par</sub>, c<sub>par</sub>), i) &rarr; (k<sub>i</sub>, c<sub>i</sub>)
1. 判断下标i是否大于等于2<sup>31</sup>，如果满足则为强化子密钥，计算I = HMAC-SHA512(Key = c<sub>par</sub>, Data = 0x00 || ser<sub>256</sub>(k<sub>par</sub>) || ser<sub>32</sub>(i))（注意：0x00为填充字节，保证私钥长度为33字节），如果不满足则为普通子密钥，计算I = HMAC-SHA512(Key = c<sub>par</sub>, Data = ser<sub>P</sub>(point(k<sub>par</sub>)) || ser<sub>32</sub>(i))
2. 将I分为两个32字节序列，I<sub>L</sub>和I<sub>R</sub>
3. 返回的子密钥k<sub>i</sub> = parse<sub>256</sub>(I<sub>L</sub>) + k<sub>par</sub> (mod n)
4. 链码(熵)c<sub>i</sub>为I<sub>R</sub>
5. 在parse<sub>256</sub>(I<sub>L</sub>) ≥ n 或 k<sub>i</sub> = 0的情况下，生成的密钥是无效的，这时应该继续计算下一个子密钥(发生的概率小于 1/2127)

HMAC-SHA512函数，可查看[RFC 4231](http://tools.ietf.org/html/rfc4231)

###### 父公钥 -> 子公钥
函数CKDpub((K<sub>par</sub>, c<sub>par</sub>), i) &rarr; (K<sub>i</sub>, c<sub>i</sub>)，仅适用于普通子密钥
1. 判断下标i是否大于等于2<sup>31</sup>，如果满足则返回失败，如果不满足则为普通子密钥，计算I = HMAC-SHA512(Key = c<sub>par</sub>, Data = ser<sub>P</sub>(K<sub>par</sub>) || ser<sub>32</sub>(i))
2. 将I分为两个32字节序列，I<sub>L</sub>和I<sub>R</sub>
3. 返回的子密钥k<sub>i</sub> = point(parse<sub>256</sub>(I<sub>L</sub>)) + K<sub>par</sub>
4. 链码(熵)c<sub>i</sub>为I<sub>R</sub>
5. 在parse<sub>256</sub>(I<sub>L</sub>) ≥ n 或 K<sub>i</sub>是无穷远的一个点时，这时应该继续计算下一个子密钥

###### 父私钥 -> 子公钥
函数N((k, c)) &rarr; (K, c)计算出与扩展私钥相对应的扩展公钥
1. 返回值K = point(k)
2. 返回链码(熵)c即为函数输入中传递过来的链码

两种通过父私钥计算子公钥的方法
1. N(CKDpriv((k<sub>par</sub>, c<sub>par</sub>), i))(适用于任何条件)
2. CKDpub(N(k<sub>par</sub>, c<sub>par</sub>), i)（仅适用于生成普通子公钥）

###### 父公钥 -> 子私钥
不可能

#### 密钥树
下一步我们使用若干个CKD结构构建一颗密钥树。从单一根节点开始，即主密钥m，对多个i计算CKDpriv(m,i)，进而得到1级派生子密钥。由于派生出来的密钥也都是扩展密钥，所以CKDpriv也可以应用于这些密钥。

为了方便书写，我们把CKDpriv(CKDpriv(CKDpriv(m,3<sub>H</sub>),2),5)简写为m/3<sub>H</sub>/2/5，同样的，对于公钥，我们把CKDpub(CKDpub(CKDpub(M,3),2),5)简写为M/3/2/5。因此有了下面的等式
* N(m/a/b/c) = N(m/a/b)/c = N(m/a)/b/c = N(m)/a/b/c = M/a/b/c.
* N(m/a<sub>H</sub>/b/c) = N(m/a<sub>H</sub>/b)/c = N(m/a<sub>H</sub>)/b/c.

但是，N(m/a<sub>H</sub>)不能写为N(m)/a<sub>H</sub>，因为我们无法通过父扩展公钥计算出子扩展私钥

每个叶子节点都对应于一个真正的密钥，叶子节点的链码会被忽略，只有节点中的公私钥是有用的。因为这种结构，我们有了父扩展私钥，就可以计算出所有的子扩展公钥及子扩展私钥，知道了父扩展公钥，就可以计算出所有的非强化子扩展公钥

#### 密钥id
可以使用序列化的ECDSA公钥的Hash160作为扩展密钥的id，忽略链码。id对应于比特币中地址。不建议使用Base58对id进行编码，因为id可能被作为地址（并且钱包软件不需要接受对链密钥本身的付款）。

密钥id的钱32位称为指纹

#### 扩展密钥序列化格式
扩展公钥及扩展私钥格式如下
1. 4byte：版本号(主网: 0x0488B21E public, 0x0488ADE4 private; 测试网: 0x043587CF public, 0x04358394 private)
2. 1byte：深度，根节点：0x00，一级节点：0x01 以此类推
3. 4byte：父密钥的指纹(主密钥为0x00000000)
4. 4byte：子节点个数(这里的英文看不懂 T T)
5. 32byte：链码
6. 33byte：原生公私钥数据(公钥部分为ser<sub>P</sub>(K)，私钥部分为0x00 || ser<sub>256</sub>(k))

这78byte的数据可以像比特币其他数据一样使用Base58进行编码，首先要添加32位的校验码(通过两次SHA-256生成)，然后转为Base58编码格式，编码后输出长度为112位的字符串。版本号部分经过Base58编码，对应的字符串为，主网：xprv、xpub，测试网：tprv、tpub

注意：上面格式中父密钥的指纹仅为了方便软件(钱包)检测父子节点使用，并且软件需要处理冲突，在内部，可以使用完整的160位id。

在导入序列化扩展公钥时，必须验证公钥数据中的X坐标是否存在于椭圆曲线上。 如果不是，则扩展公钥无效

#### 主密钥生成
可能的扩展密钥对总数几乎为2<sup>512</sup>，但是生成出来的密钥为256位，所以安全性会折半。主密钥并不是直接生成的，而是使用已有的种子代替。主密钥生成步骤如下：
1. 使用prng算法生成种子序列S，长度为128 - 512位之间。建议使用256位
2. 计算I = HMAC-SHA512(Key = "Bitcoin seed", Data = S)，直接使用比特币种子作为输入
3. 将I分为两个32字节序列，I<sub>L</sub> and I<sub>R</sub>
4. 使用parse<sub>256</sub>(I<sub>L</sub>)作为主私钥，I<sub>R</sub>作为主链码

在I<sub>L</sub> = 0 或 ≥n的情况下，生成的主密钥无效，需要重新生成
{% asset_img derivation.png 主密钥生成 %}

## 分层确定性钱包结构
前面的部分中描述了密钥树及其节点，下一步是在此基础上构建一个钱包。本文中介绍的仅为一种范例，但是建议开发者开发钱包时模仿范例的兼容性，但也不是所有的功能都需要支持

#### 范例钱包的结构
一个HDW可以看做一系列账户的集合，每个账户都有个编号，默认账户account("")的编码为0。钱包可以不支持多账户，如果不支持，则使用默认账户。

每个账户都是两条密钥链组成：内部链，外部链。外部链用来生成新的公共钱包地址，内部链可用于所有其他的操作(更改地址，生成地址，可以是任何不需要通信的操作)，支持双密钥链的钱包，应该使用外部链以支持所有的操作

* m/i<sub>H</sub>/0/k 为从主密钥m导出的第i个账号的外部链的第k个密钥对
* m/i<sub>H</sub>/1/k 为从主密钥m导出的第i个账号的内部链的第k个密钥对

## 用例
#### 全钱包共享
在两个系统需要访问同一个钱包，并且两个系统都需要有花费的功能，则需要共享主扩展私钥。每个网络节点需要为外部链维护一个前瞻密钥缓池，其中保存n个前瞻密钥，用于监听收到的付款。对内部内部链的展望可能非常小，因为不存在任何缺口。注意账户名无法通过区块链同步，仍然需要手动输入。

#### 审计
如果一名审计员需要获取所有交易(收支)记录，则可以将所有账户的扩展公钥共享给审计员。这样审计员就可以查看当前钱包的所有交易(收支)了

#### 公司多部门场景
当一家公司拥有数个独立的部门，则这些部门可以使用从同一主密钥派生出来的钱包。进而公司的领导通过维护一个超级钱包，就可以看到各个部门的收支情况了，甚至可以在各部门之间转移资金

#### 合作企业间交易
可以使用特定账户的外部链的扩展公钥作为“超级地址”，可以使得即使高频交易也不容易被别人追踪到，但无需为每笔付款都申请一个新地址。矿池操作员也可以使用这种机制作为可变支付地址

#### 不安全的收款人
当运行电子商务的服务器，存在安全风险时，他需要知道用于收款的公共地址。服务器只需要知道单个账户外部链扩展公钥。这意味着非法访问网络服务器的人最多只能查看交易记录，但是无法盗取资金，如果存在多台服务器，则他只攻破一台服务器，也无法访问到其它服务器的交易记录
