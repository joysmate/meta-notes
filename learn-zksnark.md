# 学习 zkSNARK

## 基础

[Create your first zero-knowledge snark circuit using circom and snarkjs](https://blog.iden3.io/first-zk-proof.html)

### 核心链路

Computation &rarr; Arithmetic Circuit &rarr; R1CS &rarr; QAP &rarr; zk-SNARK

总体上看，这条链路的分界点在R1CS。下面分R1CS前后两部分进行探讨。

### 如何达到R1CS

我们可以选择直接定义代数电路，利用circom工具编译成R1CS，例如定义如下乘法电路：

```
template Multiplier() {
   signal private input a;
   signal private input b;
   signal output c;
   c <== a*b;
}

component main = Multiplier();
```
命名为circuit.circom。可以看到这个电路有两个private input，a和b，意味着a和b是对验证者隐藏的，也就是说，证明者希望在不暴露a和b的情况下，告诉验证者，他知道两个数，并且它们的乘积为c。

然后我们可以用circom工具编译这个电路，得到R1CS：

```
circom circuit.circom --r1cs --wasm --sym
```

**注意**这里的R1CS是一个抽象的概念，不仅仅指约束（```.r1cs```文件），也包括用于生成witness的代码（```.wasm```文件）和用于debug的符号标识（```.sym```文件）。


当然我们也可以用更高阶的语言定义计算（PySnark），然后直接编译成R1CS。这里只是一个高低阶语言的问题，终归是要编译成R1CS。例如我们可以用python这样定义一个计算：
```
from pysnark.runtime import snark

@snark
def cube(x):
    return x*x*x
```

保存为cube.py，使用

```
python3 cube.py
```
也可以帮我们生成```.r1cs```文件和```.wasm```文件。

### 从R1CS出发

接下来我们如何使用```.r1cs```文件和```.wasm```文件呢？在回答这个问题之前，我们先跳到最后一步，看看最终验证者到底要如何验证证明者的证明呢？

以snarkjs工具为例，我们看到，最终的证明操作是这样的

```
snarkjs verify --verificationkey verification_key.json --proof proof.json --public public.json
```
可以看到，这里有三个文件：
- ```verification_key.json```：和证明者的```proving_key.json```同时基于```.r1cs```文件生成，这里的核心是生成这个key需要```.r1cs```文件，也就是这个key保证了约束的满足性
- ```proof.json```：由证明者生成，这个proof的核心，是证明了a和b的存在，但又不暴露a和b
- ```public.json```：公开的信息，比如输出c，验证者可以直接看到c的值

现在我们回头，从R1CS出发。首先，我们执行
```
snarkjs setup -r circuit.r1cs
```
生成一对```proving_key.json```和```verification_key.json```。这种setup方式，其实是默认了对执行这条setup命令人的信任。
实际生产过程中，我们需要一个公共可信的原始key，这里不展开。

> 简单理解，标准的初始化方法，需要生成一个初始公共key，并且要把这个key销毁，否则那这个key可以伪造证明。为了确保这个key不被利用，现在的方法是，全网共同生成这个key，这里有个重要的性质是，只要有一个参与者销毁了他的sub_key，那么整个key就安全。

同时，证明人用```.wasm```文件和他的隐私输入，例如```a=3,b=11```，来生成见证文件```witness.json```
```
snarkjs calculatewitness --wasm circuit.wasm --input input.json
```
这里的见证文件可以简单理解为整个计算过程（包含中间结果）的记录。

接着，使用```proving_key.json```和```witness.json```生成证明
```
snarkjs proof --witness witness.json --provingkey proving_key.json
```

如前所述，证明包含供验证者验证的```proof.json```和```public.json```。这里核心要点是，```proving_key.json```代表约束，所以证明过程保证了中间结果```witness.json```是满足约束的。

最后，snarkjs可以直接生成验证程序的智能合约

```
snarkjs generateverifier --verificationkey verification_key.json --verifier verifier.sol
```
可以看到，```verification_key.json```是被置入智能合约的。

关于这条链路以及每个环节的介绍，同时推荐阅读[神秘的零知识证明之zkSNARK](https://zhuanlan.zhihu.com/p/38205067)

## 应用

### ZK Poker
一个非常有意思的扑克游戏，可以帮助理解zkSNARK在实际场景中的应用，推荐阅读[ZK Poker — A Simple ZK-SNARK Circuit](https://medium.com/coinmonks/zk-poker-a-simple-zk-snark-circuit-8ec8d0c5ee52)

### ZCash
和```UTXO```类似，ZCash使用```Commitment```记账，定义如下：

*Commitment = HASH(recipient address, amount, rho, r)*

这里，```rho```是这个Commitment的唯一标识，当我们消费这个Commitment时，我们生成与之对应的```Nullifier```

*Nullifier = HASH(spending key, rho)*

zkSNARK在这里起的作用是，在不暴露```rho```的情况下，证明存在一个与之对应的```Commitment```并且不存在一个与之冲突的```Nullifier```。

这里主要用到的是zkSNARK的隐私能力。推荐阅读[Zcash](https://z.cash/technology/zksnarks/)

### Filecoin

对于每一个challenge，lotus要求证明者构造出从data node到merkel tree root的每一步计算过程，并且最终得到已经保存在链上的root值。这里data node就是证明者的private input，而链上的root值则是电路最后的output。而电路的构造保证了计算过程的正确性和真实性。

值得一提的是，lotus的电路规模已经达到了1亿，生成proof非常耗时，但是验证依然非常快。
推荐阅读[Filecoin — How storage replication is proved using zk-SNARK?](https://starli.medium.com/filecoin-how-storage-replication-is-proved-using-zk-snark-8a2a06b1c582)



