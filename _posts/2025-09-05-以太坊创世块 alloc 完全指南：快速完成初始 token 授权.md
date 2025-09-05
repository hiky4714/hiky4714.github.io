---
layout:     post
title:      以太坊创世块 alloc 完全指南：快速完成初始 token 授权
date:       2025-09-05
header-img: img/post-bg-desk.jpg
catalog: true
---

> 关键词：以太坊私链、Geth 创世块、alloc 配置、genesis.json、initial balance、余额预分配

在以太坊私链开发中，开发者通常需要在 **挖矿启动前** 就把一部分 token（或说余额）“预先分发”到指定地址：测试账户、项目方账户、空投合约地址等。最直接、最高效的做法就是修改 **genesis.json** 里的 **`alloc`** 字段。本文将_step by step_ 演示如何从无到有地完成 **初始化账户 → 写入创世块 → 立即拥有余额** 的全过程，并穿插常见问题解答，帮你彻底踩完所有的坑。

---

## 准备工作：清理旧链并创建数据目录

1. 删除可能存在的历史数据（如果之前跑过测试链）  
   ```sh
   rm -rf datadir && mkdir datadir
   ```
2. 选择一个目录存放密码文件 `password.txt`，内容只需一行：  
   ```
   mySecretPassword
   ```
   **提示**  
   如果你不打算用文件方式，也可以手动输入密码，但脚本模式更省心。

---

## 生成钥匙：创建新账户或导入已有私钥

### 方案 A：创建两条临时测试账户
```sh
# 创建第一条
geth --datadir=./datadir --password ./password.txt account new > account1.txt
# 创建第二条
geth --datadir=./datadir --password ./password.txt account new > account2.txt
```
命令执行时会生成形如 `0x7ce4…abC2` 的地址，写入各自的 txt 文件。

### 方案 B：导入已有私钥
如果你已在 **MetaMask / Truffle Ganache / Hardhat** 等地方拥有私钥 `keyfile.json`，可以立刻导入：
```sh
geth account import ./keyfile --datadir=./datadir --password ./password.txt >> account3.txt
```

---

## 关键步骤：把地址写进 `alloc`

打开 `genesis.json` 或新建一份，重点观察 `alloc` 字段：

```json
{
  "config": {
    "chainId": 20250501,
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip155Block": 0,
    "eip158Block": 0,
    "byzantiumBlock": 0
  },
  "difficulty": "0x2000",
  "gasLimit": "0x8000000",
  "alloc": {
    "0x7ce4a9b876afac0e9CbFf3Ef875aFD6fABabC2fE": { "balance": "0x3635c9adc5dea00000" },
    "0xb79734D4c86A48f6bf99632aAccFCfC9DcEe8f37": { "balance": "0x56bc75e2d63100000" },
    "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045": { "balance": "0x1bc16d674ec800000" }
  }
}
```

### 余额换算小技巧
- `wei` → 以太坊最小单位，10¹⁸ wei = 1 ether  
- 工具：搜索“eth to wei converter”，输入 `10000` ether 会得到十六进制写法 `0x21e19e0c9bab2400000`
👉 [区块链开发常见工具合集，一行命令秒转单位！](https://okxdog.com/)

---

## 初始化链并验证余额

1. 执行初始化  
   ```sh
   geth init genesis.json --datadir ./datadir
   ```
2. 启动节点  
   ```sh
   geth --datadir ./datadir --networkid 20250501 --http --http.api eth,net,web3 --allow-insecure-unlock --nodiscover console
   ```
3. 在 Geth 控制台查看余额  
   ```javascript
   eth.getBalance("0x7ce4a9b876afac0e9CbFf3Ef875aFD6fABabC2fE")
   // 10000 ether 等价的 wei 字符串
   ```
   若返回大于 0，表示 **创世块 alloc 生效**，无需挖矿即可到账。

---

## FAQ：新手最常问到的 5 个问题

**Q1：为何我看了教程后，余额仍然为 0？**  
A：最常见的原因是把地址放在 `alloc` 里却忘记加 **0x 前缀**。请务必写成 `"0x7ce4a9...` 而不是直接写 `7ce4a9...`。

**Q2：我可以一次性放几百上千条地址吗？**  
A：可以，但要留意 JSON 文件大小>数 MB 时，第一次启动 **Geth init** 会明显变慢；3–5 秒算正常。

**Q3：想发放 1 万 token，但小数是 6 位，难道也要写 10¹⁸？**  
A：`genesis.json` 默认 **以 wei 为单位**，即使将来要开 6 位小数合约，也先把 1 万以 10¹⁸ 写进去，后续合约内部自行缩放即可。

**Q4：必须在挖矿/出块之前设置好 alloc 吗？**  
A：是的。创世块一旦挖出，后续区块动用余额必须通过 **正常交易** Flow，新地址不能再“空投”，想改动只能重启一条链。

**Q5：如何批量生成几千个账户并一步到位写入 alloc？**  
A：推荐使用 **脚本化私钥生成器** 结合 JSON 模板渲染。👉 [零门槛批量生成分发 token 的私链脚本仓库](https://okxdog.com/)  
源码思路：  
```
for i in {1..1000}; do
  geth --datadir=./data account new --password /dev/null <<< $i
done | jq -R -s 'reduce .[] as $a ({}; . + {($a):{"balance":"0x1bc16d674ec800000"}})'
```

---

## 进阶场景：与智能合约 initial mint 搭配

在部分场景中，代币不由 `alloc` 直接放 ETH 余额，而是配置 **合约地址** 的初始状态。Python 工具 [eth-abi](https://pypi.org/project/eth-abi/) 能够把合约构造参数据流沙化写入 `alloc`。核心思路：

1. 编写字节码与构造参数  
   ```
   {"0x0000000000000000000000000000000000001337": {
       "code": "0x608...",  # 生成的字节码
       "storage": {"0x0...": "0x..."},
       "balance": "0"
   }}
   ```
2. contract constructor 里写 `_mint(0xabc..., 1000000 * 10**6)` 即可同样达到 **“零交易” 空投百万代币**，避免で发送方的 gas 消耗。

---

## 总结

通过 **genesis.json 的 alloc 字段**，你能在 **整条链诞生第一秒** 就为任意地址分配余额，这是以太坊私链调试、测试网 sample wallet 体验乃至主网 fork 节点最优雅的“预置资产”方式。记住三步：

1. 取消旧链数据（`rm -rf datadir`）  
2. 生成 / 导入账户 → 记录完整 `0x` 地址  
3. 写入 alloc → `geth init` → 夺秒到账

祝你一键起飞，开发不迷路！