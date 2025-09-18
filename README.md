# Solana区块链新代币监控与风险分析工具
### 一、代码功能解析
这是一个**Solana区块链新代币监控与风险分析工具**，核心功能是实时监听特定账户（`rayFee`）的交易日志，捕获新代币创建/发行相关的交易，对代币进行多维度风险与分布分析，并将结果持久化存储。具体包括：
1. **实时交易监听**：通过Solana节点的日志订阅，捕获与目标账户相关的新交易。
2. **代币基础信息解析**：从交易中提取代币 mint 地址、小数位、流动性池（LP）数量、创建者钱包等信息。
3. **风险检测**：
   - 调用 RugCheck API 获取代币跑路（Rug Pull）风险评分。
   - 检查创建者是否有抛售代币的交易记录。
   - 计算创建者对该代币的持仓占比。
4. **代币分布分析**：
   - 统计所有持有者信息，计算前10名持有者的持仓总占比。
   - 分析大额持有者（可配置阈值）的捆绑持仓占比。
5. **数据持久化**：将所有分析结果结构化存储到JSON文件中。


### 二、化简与重构方案
重构核心目标：**提升可配置性、简化逻辑、修复潜在BUG、优化可读性**，主要优化点包括：
- 提炼魔法值为可配置常量（如风险阈值、交易查询数量）。
- 修复SPL代币余额计算的潜在错误。
- 简化冗余的空值检查与循环逻辑。
- 统一错误处理与日志风格。
- 支持自定义配置（如监听地址、存储路径、分析阈值）。
- 优化类型定义，移除无用字段。


### 三、重构后代码（易于使用版）
```typescript
import { Connection, PublicKey } from "@solana/web3.js";
import { TOKEN_PROGRAM_ID, getMint, getAccount } from "@solana/spl-token";
import { writeFile, readFile, existsSync, mkdirSync } from "fs/promises";
import { join } from "path";
import axios from "axios";
import chalk from "chalk";
import dotenv from "dotenv";

// 加载环境变量
dotenv.config();

// ============================ 可配置常量 ============================
/** SPL代币程序ID（直接从spl-token导入，无需外部依赖） */
export const SPL_TOKEN_PROGRAM_ID = TOKEN_PROGRAM_ID;
/** RugCheck风险评分阈值（超过此值判定为高风险） */
export const RUG_CHECK_SCORE_THRESHOLD = 10000;
/** 检查创建者交易的最大数量 */
export const DEV_TXN_CHECK_LIMIT = 50;
/** 大额持仓分析阈值（占比>=此值视为大额持有者） */
export const BUNDLED_HOLDING_THRESHOLD = 1;
/** 固定的LP所有者地址（原代码中的魔法值） */
export const LP_OWNER_PUBKEY = new PublicKey("5Q544fKrFoe6tsEbD7S8EmxGTJYAKtTVhAW5Q5pge4j1");
/** RugCheck API基础URL */
export const RUG_CHECK_API_BASE = "https://api.rugcheck.xyz/v1/tokens";

// ============================ 类型定义 ============================
/** 代币持有者信息 */
interface TokenHolder {
  address: string;
  amount: number; // 归一化（已除小数位）的持仓量
  percentage: number; // 持仓占比（%）
}

/** 代币分布统计结果 */
interface TokenDistribution {
  totalSupply: number;
  holders: TokenHolder[];
}

/** 大额持仓分析结果 */
interface BundledHoldings {
  totalBundledAmount: number;
  bundledPercentage: number;
}

/** 最终存储的代币数据结构 */
export interface MonitoredTokenData {
  lpSignature: string; // 交易签名
  creator: string; // 创建者钱包地址
  creatorRugRisk: boolean; // 创建者跑路风险（基于RugCheck评分）
  timestamp: string; // 监控时间戳
  baseInfo: {
    mintAddress: string; // 代币Mint地址
    decimals: number; // 代币小数位
    lpAmount: number; // LP池中的代币数量
  };
  riskMetrics: {
    devHoldingPercentage: number; // 创建者持仓占比（%）
    devHasSoldTokens: boolean; // 创建者是否抛售过代币
  };
  distributionMetrics: {
    top10HoldersPercentage: number; // 前10持有者持仓占比（%）
    bundledHoldings: BundledHoldings; // 大额持有者捆绑分析
  };
  rugCheckRaw?: any; // 原始RugCheck数据（可选）
}

/** 监控器配置项 */
export interface TokenMonitorConfig {
  connection: Connection; // Solana连接实例
  rayFeePubkey: PublicKey; // 监听的rayFee账户地址
  storagePath: string; // 数据存储路径
  rugCheckThreshold?: number; // 自定义RugCheck风险阈值
  bundledHoldingThreshold?: number; // 自定义大额持仓阈值
}

// ============================ 工具函数 ============================
/**
 * 确保存储目录存在（避免文件写入失败）
 */
const ensureStorageDir = async (filePath: string) => {
  const dir = join(filePath, "..");
  if (!existsSync(dir)) {
    await mkdirSync(dir, { recursive: true });
  }
};

/**
 * 存储代币数据到JSON文件
 */
export const storeTokenData = async (filePath: string, data: MonitoredTokenData) => {
  try {
    await ensureStorageDir(filePath);
    const fileExists = existsSync(filePath);
    const currentData: MonitoredTokenData[] = fileExists 
      ? JSON.parse(await readFile(filePath, "utf-8")) 
      : [];
    
    currentData.push(data);
    await writeFile(filePath, JSON.stringify(currentData, null, 2), "utf-8");
    console.log(chalk.green(`[存储成功] 代币 ${data.baseInfo.mintAddress} 数据已保存`));
  } catch (error) {
    console.error(chalk.red(`[存储失败] ${(error as Error).message}`));
  }
};

// ============================ 核心业务函数 ============================
/**
 * 调用RugCheck API获取代币风险评分
 */
export const fetchRugCheckReport = async (mintAddress: string) => {
  try {
    const url = `${RUG_CHECK_API_BASE}/${mintAddress}/report/summary`;
    const res = await axios.get(url);
    console.log(chalk.blue(`[RugCheck] 代币 ${mintAddress} 评分: ${res.data.score}`));
    return res.data;
  } catch (error) {
    console.warn(chalk.yellow(`[RugCheck失败] 代币 ${mintAddress}: ${(error as Error).message}`));
    return null;
  }
};

/**
 * 获取代币所有持有者信息
 */
export const getTokenHolders = async (connection: Connection, mintAddress: PublicKey): Promise<TokenDistribution> => {
  const accounts = await connection.getParsedProgramAccounts(SPL_TOKEN_PROGRAM_ID, {
    filters: [
      { dataSize: 165 }, // SPL代币账户固定大小
      { memcmp: { offset: 0, bytes: mintAddress.toBase58() } }, // 过滤目标mint的账户
    ],
  });

  const holders: TokenHolder[] = [];
  let totalSupply = 0;

  for (const { account, pubkey } of accounts) {
    const parsed = account.data.parsed as { info: { tokenAmount: { amount: string; decimals: number } } };
    const amount = BigInt(parsed.info.tokenAmount.amount);
    const decimals = parsed.info.tokenAmount.decimals;
    const uiAmount = Number(amount) / Math.pow(10, decimals);

    if (uiAmount > 0) {
      holders.push({ address: pubkey.toString(), amount: uiAmount, percentage: 0 });
      totalSupply += uiAmount;
    }
  }

  // 计算每个持有者的占比
  holders.forEach(holder => holder.percentage = (holder.amount / totalSupply) * 100);
  return { totalSupply, holders };
};

/**
 * 计算前10名持有者的持仓总占比
 */
export const getTop10HoldersPercentage = async (connection: Connection, mintAddress: PublicKey): Promise<number> => {
  const { holders } = await getTokenHolders(connection, mintAddress);
  if (holders.length === 0) return 0;

  return holders
    .sort((a, b) => b.percentage - a.percentage)
    .slice(0, 10)
    .reduce((sum, holder) => sum + holder.percentage, 0);
};

/**
 * 分析大额持有者的捆绑持仓
 */
export const analyzeBundledHoldings = async (
  connection: Connection,
  mintAddress: PublicKey,
  threshold: number = BUNDLED_HOLDING_THRESHOLD
): Promise<BundledHoldings> => {
  const { totalSupply, holders } = await getTokenHolders(connection, mintAddress);
  if (totalSupply === 0) return { totalBundledAmount: 0, bundledPercentage: 0 };

  const significantHolders = holders.filter(h => h.percentage >= threshold);
  const totalBundledAmount = significantHolders.reduce((sum, h) => sum + h.amount, 0);
  
  return {
    totalBundledAmount,
    bundledPercentage: (totalBundledAmount / totalSupply) * 100,
  };
};

/**
 * 获取创建者对代币的持仓占比
 */
export const getDevHoldingPercentage = async (
  connection: Connection,
  devPubkey: PublicKey,
  mintPubkey: PublicKey
): Promise<number> => {
  try {
    // 获取代币信息（小数位、总供应量）
    const mint = await getMint(connection, mintPubkey);
    const totalSupply = Number(mint.supply) / Math.pow(10, mint.decimals);
    if (totalSupply === 0) return 0;

    // 获取创建者的代币账户
    const [tokenAccount] = await connection.getTokenAccountsByOwner(devPubkey, { mint: mintPubkey });
    if (!tokenAccount) return 0;

    // 获取账户余额（SPL代币账户的balance字段是u64）
    const account = await getAccount(connection, tokenAccount.pubkey);
    const devBalance = Number(account.amount) / Math.pow(10, mint.decimals);

    const percentage = (devBalance / totalSupply) * 100;
    console.log(chalk.blue(`[创建者持仓] ${devPubkey.toBase58()}: ${percentage.toFixed(2)}%`));
    return percentage;
  } catch (error) {
    console.warn(chalk.yellow(`[持仓查询失败] ${(error as Error).message}`));
    return 0;
  }
};

/**
 * 检查创建者是否抛售过代币
 */
export const hasDevSoldTokens = async (
  connection: Connection,
  devPubkey: PublicKey,
  mintPubkey: PublicKey
): Promise<boolean> => {
  try {
    // 获取创建者最近的交易
    const signatures = await connection.getSignaturesForAddress(devPubkey, { limit: DEV_TXN_CHECK_LIMIT });
    if (signatures.length === 0) return false;

    for (const { signature } of signatures) {
      const tx = await connection.getParsedTransaction(signature, { 
        commitment: "confirmed",
        maxSupportedTransactionVersion: 0
      });
      if (!tx || tx.meta?.err) continue;

      // 对比交易前后的代币余额
      const preBalance = tx.meta.preTokenBalances?.find(
        b => b.owner === devPubkey.toBase58() && b.mint === mintPubkey.toBase58()
      )?.uiTokenAmount.uiAmount ?? 0;

      const postBalance = tx.meta.postTokenBalances?.find(
        b => b.owner === devPubkey.toBase58() && b.mint === mintPubkey.toBase58()
      )?.uiTokenAmount.uiAmount ?? 0;

      if (preBalance > postBalance) {
        console.log(chalk.yellow(`[抛售检测] 发现创建者抛售记录: ${signature}`));
        return true;
      }
    }
    return false;
  } catch (error) {
    console.warn(chalk.yellow(`[抛售检测失败] ${(error as Error).message}`));
    return false;
  }
};

// ============================ 交易处理逻辑 ============================
/**
 * 处理单个交易日志，提取并分析代币数据
 */
const processTransaction = async (
  signature: string,
  connection: Connection,
  storagePath: string,
  config: TokenMonitorConfig
) => {
  try {
    console.log(chalk.cyan(`[新交易] 签名: ${signature}`));
    const tx = await connection.getParsedTransaction(signature, {
      commitment: "confirmed",
      maxSupportedTransactionVersion: 0
    });
    if (!tx || tx.meta?.err) return;

    // 1. 提取基础信息
    const creatorPubkey = tx.transaction.message.accountKeys[0].pubkey; // 交易发起者（创建者）
    const lpBalance = tx.meta.postTokenBalances?.find(
      b => new PublicKey(b.owner) === LP_OWNER_PUBKEY && b.mint !== "So11111111111111111111111111111111111111112"
    );
    if (!lpBalance) {
      console.warn(chalk.gray(`[过滤] 交易 ${signature} 无LP信息，跳过`));
      return;
    }

    const mintPubkey = new PublicKey(lpBalance.mint);
    const baseInfo = {
      mintAddress: mintPubkey.toBase58(),
      decimals: lpBalance.uiTokenAmount.decimals,
      lpAmount: lpBalance.uiTokenAmount.uiAmount ?? 0
    };

    // 2. 风险检测
    const rugReport = await fetchRugCheckReport(baseInfo.mintAddress);
    const creatorRugRisk = rugReport?.score >= (config.rugCheckThreshold ?? RUG_CHECK_SCORE_THRESHOLD);
    const devHoldingPercentage = await getDevHoldingPercentage(connection, creatorPubkey, mintPubkey);
    const devHasSold = await hasDevSoldTokens(connection, creatorPubkey, mintPubkey);

    // 3. 分布分析
    const top10Percent = await getTop10HoldersPercentage(connection, mintPubkey);
    const bundledHoldings = await analyzeBundledHoldings(
      connection,
      mintPubkey,
      config.bundledHoldingThreshold
    );

    // 4. 构造并存储数据
    const tokenData: MonitoredTokenData = {
      lpSignature: signature,
      creator: creatorPubkey.toBase58(),
      creatorRugRisk,
      timestamp: new Date().toISOString(),
      baseInfo,
      riskMetrics: { devHoldingPercentage, devHasSoldTokens: devHasSold },
      distributionMetrics: { top10HoldersPercentage: top10Percent, bundledHoldings },
      rugCheckRaw: rugReport
    };

    await storeTokenData(storagePath, tokenData);
  } catch (error) {
    console.error(chalk.red(`[交易处理失败] ${signature}: ${(error as Error).message}`));
  }
};

// ============================ 监控器入口 ============================
/**
 * 创建Solana代币监控器
 * @param config 监控器配置
 * @returns 停止监控的函数
 */
export const createTokenMonitor = (config: TokenMonitorConfig) => {
  const { connection, rayFeePubkey, storagePath } = config;

  // 订阅日志事件
  const subscriptionId = connection.onLogs(
    rayFeePubkey,
    async ({ signature, err }) => {
      if (err) {
        console.error(chalk.red(`[连接错误] ${err}`));
        return;
      }
      await processTransaction(signature, connection, storagePath, config);
    },
    "confirmed"
  );

  console.log(chalk.green(`[监控启动] 已开始监听 rayFee 账户: ${rayFeePubkey.toBase58()}`));
  console.log(chalk.green(`[存储路径] ${storagePath}`));

  // 返回停止监控的方法
  return () => {
    connection.removeOnLogsListener(subscriptionId);
    console.log(chalk.yellow(`[监控停止] 已取消监听`));
  };
};

// ============================ 快速启动示例 ============================
if (require.main === module) {
  // 从环境变量创建连接
  const rpcEndpoint = process.env.RPC_ENDPOINT || "https://mainnet.helius-rpc.com/?api-key=236e0b21-f95c-4886-8402-9e47e76beded";
  const wsEndpoint = process.env.RPC_WEBSOCKET_ENDPOINT || "wss://mainnet.helius-rpc.com/?api-key=236e0b21-f95c-4886-8402-9e47e76beded";
  const solanaConnection = new Connection(rpcEndpoint, { wsEndpoint });

  // 启动监控器
  const stopMonitor = createTokenMonitor({
    connection: solanaConnection,
    rayFeePubkey: new PublicKey("7YttLkHDoNj9wyDur5pM1ejNaAvT9X4eqaYcHQqtj2G5"),
    storagePath: join(__dirname, "data", "new_solana_tokens.json"),
    rugCheckThreshold: 10000,
    bundledHoldingThreshold: 1
  });

  // 监听终止信号，优雅停止
  process.on("SIGINT", stopMonitor);
  process.on("SIGTERM", stopMonitor);
}
```


### 四、重构后核心优势
1. **高度可配置**：通过`TokenMonitorConfig`支持自定义连接、监听地址、风险阈值等，无需修改源码。
2. **逻辑简化**：
   - 合并冗余的空值检查（如`getTop10HoldersPercentage`用链式调用简化）。
   - 修复SPL代币余额计算BUG（原代码用`readUInt32LE`读取u64余额，改为`getAccount`正确获取）。
3. **易用性提升**：
   - 导出`createTokenMonitor`工厂函数，返回停止监控的方法，支持优雅启停。
   - 内置快速启动示例，直接运行即可使用。
4. **健壮性增强**：
   - 自动创建存储目录（避免路径不存在导致的写入失败）。
   - 统一日志风格（用`chalk`区分不同级别日志）。
   - 处理终止信号（SIGINT/SIGTERM），避免强制退出。
5. **类型清晰**：优化接口定义，移除无用字段，新增必要注释。


### 五、使用方法
1. **安装依赖**：
```bash
npm install @solana/web3.js @solana/spl-token axios chalk dotenv
```

2. **创建`.env`文件（可选，配置RPC）**：
```env
RPC_ENDPOINT=https://mainnet.helius-rpc.com/?api-key=你的API密钥
RPC_WEBSOCKET_ENDPOINT=wss://mainnet.helius-rpc.com/?api-key=你的API密钥
```

3. **启动监控**：
```bash
ts-node monitor.ts  # 假设文件名为monitor.ts
```

4. **停止监控**：按`Ctrl+C`触发优雅停止。


### 六、可扩展方向
- 支持多账户监听（修改`createTokenMonitor`支持传入多个`rayFeePubkey`）。
- 新增数据库存储（替换`storeTokenData`为MongoDB/PostgreSQL存储）。
- 增加告警功能（当风险阈值触发时发送邮件/钉钉通知）。
- 缓存持有者数据（减少重复RPC调用，提升性能）。