---
layout: post
title: Block chain
categories: [distributed]
description: block chain
keywords: blockchain
---

**Update: 2017年02月13日 星期一**

**第一个钱是从哪来的**

2009年比特币诞生的时候，每笔赏金是50个比特币。诞生10分钟后，第一批50个比特币生成了，而此时的货币总量就是50。随后比特币就以约每10分钟50个的速度增长。当总量达到1050万时(2100万的50%)，赏金减半为25个。当总量达到1575万(新产出525万，即1050的50%)时，赏金再减半为12.5个。

**一次交易的数据结构是什么样子的**



```
public class Transaction {
	private byte[] hash;
    private ArrayList<Input> inputs;

    // outputs 的序号就在这里了
    private ArrayList<Output> outputs;
    private boolean coinbase;

    public class Input {
        /**
         * hash of the Transaction whose output is being used
         */
        public byte[] prevTxHash;
        /**
         * used output's index in the previous transaction
         */
        public int outputIndex;
        /**
         * the signature produced to check validity
         */
        public byte[] signature;

        public Input(byte[] prevHash, int index) {
            if (prevHash == null)
                prevTxHash = null;
            else
                prevTxHash = Arrays.copyOf(prevHash, prevHash.length);
            outputIndex = index;
        }

        public void addSignature(byte[] sig) {
            if (sig == null)
                signature = null;
            else
                signature = Arrays.copyOf(sig, sig.length);
        }

        public boolean equals(Object other) {
            if (other == null) {
                return false;
            }
            if (getClass() != other.getClass()) {
                return false;
            }

            Input in = (Input) other;

            if (prevTxHash.length != in.prevTxHash.length)
                return false;
            for (int i = 0; i < prevTxHash.length; i++) {
                if (prevTxHash[i] != in.prevTxHash[i])
                    return false;
            }
            if (outputIndex != in.outputIndex)
                return false;
            if (signature.length != in.signature.length)
                return false;
            for (int i = 0; i < signature.length; i++) {
                if (signature[i] != in.signature[i])
                    return false;
            }
            return true;
        }

        public int hashCode() {
            int hash = 1;
            hash = hash * 17 + Arrays.hashCode(prevTxHash);
            hash = hash * 31 + outputIndex;
            hash = hash * 31 + Arrays.hashCode(signature);
            return hash;
        }
    }


    public class Output {
        /**
         * value in bitcoins of the output
         */
        public double value;
        /**
         * the address or public key of the recipient
         */
        public PublicKey address;

        public Output(double v, PublicKey addr) {
            value = v;
            address = addr;
        }

        public boolean equals(Object other) {
            if (other == null) {
                return false;
            }
            if (getClass() != other.getClass()) {
                return false;
            }

            Output op = (Output) other;

            if (value != op.value)
                return false;
            if (!((RSAPublicKey) address).getPublicExponent().equals(
                    ((RSAPublicKey) op.address).getPublicExponent()))
                return false;
            if (!((RSAPublicKey) address).getModulus().equals(
                    ((RSAPublicKey) op.address).getModulus()))
                return false;
            return true;
        }

        public int hashCode() {
            int hash = 1;
            hash = hash * 17 + (int) value * 10000;
            hash = hash * 31 + ((RSAPublicKey) address).getPublicExponent().hashCode();
            hash = hash * 31 + ((RSAPublicKey) address).getModulus().hashCode();
            return hash;
        }
    }

}
```

![](/images/posts/bigdata/blockchain.png)

**矿机，旷工是干什么的**

一个交易完成后，客户端会把 Transaction 信息发送到网络中，矿机收到这些 TX 后会做两件事情，第一件事是验证 Transaction 的有效性，第二件事是把多个 TX 打成包，成为一个 Block，并为这个 Block 生成一个 nonce，使得 Block ID + nonce 进行 SHA-256 后得到的值前 I 位是 0. 这就是 Proof of work 要做的事。因为计算的过程是很复杂的，Bitcoin 会调整难度，大约需要 10 分钟生成一个 nonce，计算效率提高以后，生成的 0 的个数也会提高。

Proof of work 也就是说运行做坏事，可以造假，但是每次造假都会付出一些代价，当你付出的代价足够高时你就不会考虑做一件坏事了。你在生成 Block 别人也在干活，如果你只干一次就结束，那么耗费你 10 分钟的时间而已，没什么大不了的

**signature 的位置和作用**

```java
        for (int i = 0; i < tx.getInputs().size(); i++) {
            Transaction.Input input = tx.getInput(i);
            UTXO utxo = new UTXO(input.prevTxHash, input.outputIndex);
            Transaction.Output txOutput = utxoPool.getTxOutput(utxo);
            if(!Crypto.verifySignature(txOutput.address, tx.getRawDataToSign(i), input.signature)) {
                return false;
            }
        }
```

这段代码是 普林斯顿大学比特币课程的第一个作业，它用来验证一个 Transaction 的签名是否有效。其中 input 是某个 TX 的第 i 个 input, signature 是生成的签名，getRawDataToSign(i) 被用来签名的原始内容。txOutput.address 是公约，它也是接受者的 public 地址。也就是说，对于一个 TX，找到 input 的来源，就是上一个 TX 的 output -> txOutput, 它的 address 就是本次 TX 的来源。

```
    public byte[] getRawDataToSign(int index) {
        // ith input and all outputs
        ArrayList<Byte> sigData = new ArrayList<Byte>();

        if (index > inputs.size())
            return null;

        Input in = inputs.get(index);
        byte[] prevTxHash = in.prevTxHash;
        ByteBuffer b = ByteBuffer.allocate(Integer.SIZE / 8);
        b.putInt(in.outputIndex);
        byte[] outputIndex = b.array();

        // input prevTxHash add to sigData
        if (prevTxHash != null)
            for (int i = 0; i < prevTxHash.length; i++)
                sigData.add(prevTxHash[i]);

        // input index add to sigData
        for (int i = 0; i < outputIndex.length; i++)
            sigData.add(outputIndex[i]);

        for (Output op : outputs) {
            ByteBuffer bo = ByteBuffer.allocate(Double.SIZE / 8);
            bo.putDouble(op.value); // money number
            byte[] value = bo.array();
            byte[] addressBytes = op.address.getEncoded();
            for (int i = 0; i < value.length; i++)
                sigData.add(value[i]);

            for (int i = 0; i < addressBytes.length; i++)
                sigData.add(addressBytes[i]);
        }

        byte[] sigD = new byte[sigData.size()];

        int i = 0;

        for (Byte sb : sigData)
            sigD[i++] = sb;

        return sigD;
    }

```

对第 i 个 input 签名，需要第 i 个 input 信息和所有的 output 信息。而被签好名的信息就保存在 Input.signature 属性中。用它来作为签名。而一个 TX 的 ID 也就是 hash 属性，是这么算的：

```
    @Override
    public void finalize() {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            md.update(getRawTx());
            hash = md.digest();
        } catch (NoSuchAlgorithmException x) {
            x.printStackTrace(System.err);
        }
    }

```

getRawTx 会把 input 的所有 signature 和 output 的所有信息都包含进去，生成一个 256 位的 ID。

**分叉问题**

任何成功计算出那个比目标阈值低的区块头哈希值的矿工，都可以将这整个区块加到区块链中（假设这个区块是有效的）。这些区块通常是通过区块高度来定位的，所谓区块高度是指某个区块与比特币第一个区块（及区块 0，众所周知的创世区块）之间的区块个数。比如，区块 2016 就是第一次难度调整的位置。+

[link]https://0dayzh.gitbooks.io/bitcoin_developer_guide/content/block_height_and_forking.html 

多个区块可以具有同样的区块高度，这在两个或更多的矿工同时创建一个区块时是很常见的。这导致了如上图所示的明显的分叉。+

最终，矿工会产出一个新的区块，它附加在且仅附加在同时被挖出的区块中的一个之后。这样使得某一个分叉比其他分叉更为健壮。假设一个分叉只包括有效的区块，普通的节点总是跟随更长的链，而放弃陈旧的短的分叉。（陈旧的区块常常被叫做孤链，但是该术语也被用于描述确实没有父区块的区块。）
如果一些矿工有其他的意图，长期的分叉是有可能出现的，比如一些矿工在延续区块链工作，而同时另一些矿工试图通过发起 51％ 攻击篡改交易纪录。+

因为多个区块可以在区块链分叉时具有同样的区块高度，所以区块高度不应该用来做全局的唯一标示符。通常，区块使用区块头的哈希值来做标示（大多数时候是字节倒序的十六进制表示）。

**交易数据**

每一个区块必须包含一笔或者多笔交易。这些交易的第一笔必须是一个 coinbase 交易，也被称作 generation 交易，它包含了这个区块所有花费和奖励（由一个区块的补贴和这个区块任何交易产生的交易费用构成）。旷工把钱给自己成为 coinbase

一个 coinbase 交易的 UTXO 有一个特殊的条件，那就是在之后的 100 个区块内都不能被花费（被用作输入）。这临时性的杜绝了一个矿工花费掉因为分叉而可能被判定为陈旧区块（这个 coinbase 交易因此被销毁掉）中所包含的补贴和交易费。+

例如，如果交易只是连接（没有做哈希运算）在一起，那么具有 5 个交易的 merkle 树应该如下图所示：

```
       ABCDEEEE .......Merkle 根节点
      /        \
   ABCD        EEEE
  /    \      /
 AB    CD    EE .......E 与它自身配对
/  \  /  \  /
A  B  C  D  E .........交易
```

merkle 树允许客户端通过一个完整节点从一个区块头中获取其 merkle 根节点和中间哈希值列表来验证一个交易被包含在这个区块中。这个完整节点并不需要是可信的，因为伪造区块头十分困难而中间哈希值是不可能被伪造的，否则验证就会失败。也就说查询的快

例如，要验证交易 D 被加到了区块中，一个 SPV 客户端只需要一份 C，AB 和 EEEE 的副本进而做哈希运算得到 merkle 根节点，客户端并不需要知道任何其他交易的信息。如果这 5 个交易的大小都是一个区块的最大上限，那么下载整个区块需要下载 500,000 个字节，但下载树的哈希值和区块头仅仅需要 140 个字节。+

注意：如果在同一个区块中找到了相同的 txids，那么 merkle 树可能会出现与另一个区块的碰撞，归因于 merkle 树对非平衡的实现（对孤立的哈希值做复制）会将一个区块中一些或全部的重复内容移除掉。从对于单独的交易具有相同的 txid 是不现实的角度来看，merkle 树的实现不会对诚实的软件增加负担，但如果一个区块的无效状态要被缓存那么就必须做检查，否则，一个移除了重复交易的有效区块会因为具有相同的 merkle 根节点和区块哈希而被缓存的无效状态拒绝，导致了编号为 CVE-2012-2459 的安全问题。+


**难度值**

难度值（difficulty）是矿工们在挖矿时候的重要参考指标，它决定了矿工大约需要经过多少次哈希运算才能产生一个合法的区块。比特币的区块大约每10分钟生成一个，如果要在不同的全网算力条件下，新区块的产生保持都基本这个速率，难度值必须根据全网算力的变化进行调整。简单地说，难度值被设定在无论挖矿能力如何，新区块产生速率都保持在10分钟一个。

难度的调整是在每个完整节点中独立自动发生的。每2016个区块，所有节点都会按统一的公式自动调整难度，这个公式是由最新2016个区块的花费时长与期望时长（期望时长为20160分钟即两周，是按每10分钟一个区块的产生速率计算出的总时长）比较得出的，根据实际时长与期望时长的比值，进行相应调整（或变难或变易）。也就是说，如果区块产生的速率比10分钟快则增加难度，比10分钟慢则降低难度。


**double spending**

下面一节有讨论到 double spending, 这也是最需要也是最难解决的问题。考虑这么一个场景，假如一个矿机作弊，在他的 block chain 中添加了一条假交易，这个交易传给了所有的节点，假消息生效了，系统就被摧毁了，道理哪里不对呢？想象区块链的要求，只要破坏者的算力不超过集群的 51%即可，而该破坏者做的结果发给了整个集群，就被接受了，说明他的算力是集群的 100%，所以不符合实际情况。实际情况是，破坏者在算的时候别的机器也在算，当出现冲突的时候较长的区块链获胜，所以破坏者要不停地算，才能保证自己不被别人比下去，如果它小于所有算力的 50%，那么它永远失败，这样考虑问题时，它实际在于全世界竞争，导致自己最终还是会失败。

**每个 Transaction 必须一次用完**

即便 50 块钱只发出去 30， 剩下 20 也要再发给自己，也就解决了 double spending 的一个计算难的问题（这一块需要扩展）


## Block chain 区块链

区块链由比特币火起来，如今比特币已经超过 5000 人民币一枚了，区块链也被应用到了各个领域。虽然这是一篇对区块链的文章，但是我目前对区块链的理解还是不够，只是对这两天区块链学习的总结。

区块链技术的目的是，在一个没有中心的 P2P 系统中，任意两个节点（人）能够完成交易，这个交易最终会被整个集群认可。这个是最终一致性的分布式系统，
根据 CAP 原理，它放弃了 C 则能够获得 A 和 P, 即可用性和分区容错性。从下面的分析中可以看出，CAP holds on block chain.

从学习的过程中，我一直在想，block chain 真的有别人讲起来的这么神奇么，假如有这么神奇的话，那么 Paxos, Raft 岂不是很尴尬？这两天学习的结论是，它是个很奇妙的技术，但是距离真正的使用还有很长的距离。它的缺点有 （1）不适合用于高交易量的系统，目前的每秒交易量是个位数。达到几万的级别应该会很难 （2）不适合低延迟交易，目前每个区块的打包时间在 10min 左右，打包一次并不能确保消息真的被系统所接受，目前 6 个更新的 block chain 出现后，才能确保数据真的有效，一次交易等待 1 小时，这个一般人还真等不起。这两个问题是在低频度的交易下已经暴露出的问题，等到交易的频度高起来后，可能出现的问题就更多了

下面讲解我对 block chain 的理解

### P2P system with only two users

1: 先假设有一个最简单的 P2P 系统，只有两个人 alice and bob. Alice 向 box 转了一笔账，这里有一个问题，box 收到钱后怎么知道是 alice 转的？单纯一个 From: alice 是不够的，因为这可以伪造，这个问题的解决方法很简单，账单上只需要加上 alice 的签名即可，alice 用私钥对账单进行签名，bob 拿着 alice 的公钥，就能确认这个账单是从 alice 发出的，并能解析出账单的内容

2: 钱从哪里来？系统中出现了第一张账单，但是与其说是账单，这更像是一个欠条，但是一个不可信的 P2P 系统是不允许欠条出现的。在区块链系统中，有一个创世块，创世块提供第一笔资金，这里可以简单的假设，alice 拥有创世块，所以她有第一笔钱。后面还会说到其他得到钱的方式

3: box 怎么知道 alice 有钱呢？每个账单上都会有钱的来源，账单会标注别人发给他的钱的所有账单的引用（账单号），bob 会验证账单的有效性，验证的过程就是检查所有的一直到初始块的账单。当一个新的用户安装客户端时，可能就需要验证世界上所有发生过的交易，听起来好像是一笔很大的工作量。感觉这种方式并不 scale

### P2P system with only three users

1: 当第三个人 chris 加入到系统来了，这时候 P2P 系统就变得复杂很多了，第一个问题就是如何识别 double spending. double spending 是说，alice 转账给 bob 和 chris, 但是钱的来源是相同的。因为是一个 P2P 系统，bob 和 chris 并不能问系统中的所有节点，因为这个系统是全球，这是一个很大的系统，另一个原因是 P2P 系统中的节点总是会加入或者离开，他并不是稳定的，也不知道 majority of P2P system. 那该怎么办呢？block chain 规定，每个节点在作出和收到一个交易时，会把这次交易的信息扩散到集群中的其他节点，最终这次交易会被集群中的所有节点所知晓，集群中有些节点会对这次交易进行验证，验证成功后会对结果进行存储和转发。

2: 紧接着第一条，如果某些节点同流合污，把虚假的信息发送到集群，该怎么办呢？因为存储和转发过程太过简单，代价太低，block chain 本身无法限制节点做坏事。block chain 是这么做的，如果某个节点要把某个或某些交易转发给其他节点，让其他节点认可，他除了转发交易信息本身外，还要再提供一个信息，nonce，其中 nonce + transactionId -> hash(_) -> ^0{n}。也就说说，节点要求出一个 nonce, 这个 nonce 加上 transaction id 后取 hash 的值，前 n 个数都是 0. 这就好像是说，你要投票？可以，先做道数学题。这个数学题倒不是难，只是做出来要花很长时间，假如某些节点一意孤行，宁愿花很长时间，也要把一个错误的结果发出去，蒙蔽他人，这个时候看整个系统作何反应，假设整个系统只有他一个人验证转发，那么他成功了的蒙蔽了所有的人，就好像是他伪造了一个 alice 到 box 的交易，这个交易并不存在，某一天，alice 说我怎么少了些钱，这样系统就被这个节点给毁了。假如还有一个人，这个人也验证并转发了这条交易信息，这个人无论是好人还是骗子，都会出现不一致的现象，因为这个骗子可能验证的是另一个假信息。假如有个好人验证了两条交易信息，那么两条交易信息的会取代一条，骗子花费的心血白费了。这也是 proof of work 的过程。只有当系统中骗子的计算力大于集群的一半时，系统才会不可靠。并且，这些骗子必须是同一阵营的，所有骗子产生相同的虚假信息。而好人，因为是诚实的，所以产生的消息天生是一致的。

3: 为什么有些节点愿意劳动，愿意验证信息。因为他们有奖赏，目前每次验证会得到 12.5 个比特币的收入，所以他们愿意花时间搞，这也是挖矿做的事，找到那个 nonce. 此外，比特币做了一个优化，不会对每个消息都单独验证，而是会把消息 group 起来，批量处理，降低每条消息的平均工作量，这样矿工有更多的利润空间，一般每 10 分钟 batch 一次，batch 出的一个东西就叫做 block. 每个 block  会有指向前一个 block ，这是为了让所有的节点都拥有相同的 block chain。

4: 在一个 block 出现后，用户的交易就算完成了，但是会出现冲突，即别的机器也做了一个 block, 接收的节点看到这两个 block 后不知道该选哪一个，解决冲突的办法是，让节点总是接收更长的那个。假如一个节点发现一个更长的，且与自己冲突的 block chain, 那么他会放弃自己的 block 转而接收更长的那一个，那么被抛弃的 block 上的交易该怎么办？呃，他们也被抛弃了。block never final. 不过 block 越长，以前的 block 被抛弃的概率越低，每个 block 大约 10 分钟，1小时后，基本上可以认为不会发生变化了

### 其他不理解的地方

1: 如果中国境内的机器和外国的分开了，假设分开了长达一天，那么中国境内的所有交易不就全部作废了么？

2: 如果参与比特币交易的人变得多了起来，全球各处每秒几万次交易，岂不是会有很多的冲突？ 1 小时后基本不变的准则岂不是要发生变化？

3: 假如某山寨币的参与用户比较少，那么计算 nonce 的标准就会比较低，参与的人多， nonce 的标准就比较高



