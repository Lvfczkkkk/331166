# 玄学大狮 v2.0 技术架构文档

**架构师**: K哥 (产品经理 × 编程大师)  
**版本**: 2.0.0  
**日期**: 2026-03-12

---

## 📐 架构总览

### 分层架构

```
┌─────────────────────────────────────────────────────────────┐
│                    表示层 (Presentation)                      │
│  ├── CLI 命令接口                                            │
│  ├── 自然语言对话接口                                         │
│  └── Web API 接口 (RESTful)                                  │
├─────────────────────────────────────────────────────────────┤
│                    应用层 (Application)                       │
│  ├── 命令路由 (Command Router)                               │
│  ├── 会话管理 (Session Manager)                              │
│  ├── 结果格式化 (Response Formatter)                         │
│  └── 权限控制 (Access Control)                               │
├─────────────────────────────────────────────────────────────┤
│                    领域层 (Domain)                            │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                 核心引擎 (Core Engines)                │  │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │  │
│  │  │  历法引擎    │ │  八字引擎    │ │  易经引擎    │     │  │
│  │  │  Calendar   │ │    Bazi     │ │   Yijing    │     │  │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │  │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │  │
│  │  │  风水引擎    │ │  运势引擎    │ │  AI解读引擎  │     │  │
│  │  │  Fengshui   │ │   Fortune   │ │  AI Parser  │     │  │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │  │
│  └───────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                    数据层 (Data)                              │
│  ├── 用户档案存储 (User Repository)                          │
│  ├── 知识库 (Knowledge Base)                                 │
│  ├── 缓存层 (Cache Layer)                                    │
│  └── 搜索引擎 (Search Engine)                                │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 核心算法详解

### 1. 历法计算引擎 (Calendar Engine)

#### 1.1 公历转农历
```typescript
/**
 * 公历日期转农历日期
 * 算法：基于已知基准日（1900年1月31日 = 农历1900年正月初一）推算
 */
function solarToLunar(solarDate: Date): LunarDate {
  // 1. 计算与基准日的天数差
  const baseDate = new Date(1900, 0, 31);
  const diffDays = Math.floor((solarDate - baseDate) / (24 * 60 * 60 * 1000));
  
  // 2. 查农历年表，确定所在农历年
  const lunarYear = findLunarYear(diffDays);
  
  // 3. 计算月日
  const { month, day, isLeap } = calculateLunarMonthDay(
    diffDays - lunarYear.startDay
  );
  
  return { year: lunarYear.year, month, day, isLeap };
}
```

#### 1.2 计算四柱
```typescript
/**
 * 计算八字四柱
 * 关键：年柱以立春为界，月柱以节气为界
 */
function calculateBaziPillars(
  year: number, 
  month: number, 
  day: number, 
  hour: number
): FourPillars {
  // 年柱：以立春为界
  const yearPillar = calculateYearPillar(year, month, day);
  
  // 月柱：以节气为界（十二节令）
  const monthPillar = calculateMonthPillar(year, month, day);
  
  // 日柱：基于已知基准日推算
  const dayPillar = calculateDayPillar(year, month, day);
  
  // 时柱：根据日干和时辰
  const hourPillar = calculateHourPillar(dayPillar.stem, hour);
  
  return { year: yearPillar, month: monthPillar, day: dayPillar, hour: hourPillar };
}

/**
 * 计算年柱
 * 立春通常在2月3-5日之间
 */
function calculateYearPillar(year: number, month: number, day: number): Pillar {
  const liChun = getLiChunDate(year); // 获取当年立春日期
  
  // 如果已过立春，用当年；否则用上年
  const effectiveYear = (month > liChun.month || 
    (month === liChun.month && day >= liChun.day)) ? year : year - 1;
  
  const yearIndex = (effectiveYear - 4) % 60;
  return {
    stem: TIAN_GAN[yearIndex % 10],
    branch: DI_ZHI[yearIndex % 12]
  };
}

/**
 * 计算月柱
 * 十二节令划分十二个月
 */
function calculateMonthPillar(year: number, month: number, day: number): Pillar {
  const jieQi = getJieQiDate(year); // 获取当年所有节气
  
  // 找到当前日期所在的节气月
  const currentJie = findCurrentJieQi(jieQi, month, day);
  const monthIndex = JIE_QI_ORDER.indexOf(currentJie.name);
  
  // 月干根据年干确定（五虎遁）
  const yearStemIndex = TIAN_GAN.indexOf(yearPillar.stem);
  const monthStemIndex = (yearStemIndex * 2 + monthIndex) % 10;
  
  return {
    stem: TIAN_GAN[monthStemIndex],
    branch: DI_ZHI[monthIndex]
  };
}
```

### 2. 八字分析引擎 (Bazi Engine)

#### 2.1 日主强弱分析
```typescript
/**
 * 分析日主强弱
 * 考虑：得令、得地、得势
 */
function analyzeDayMasterStrength(bazi: FourPillars): StrengthResult {
  const dayMaster = bazi.day.stem;
  const dayElement = getElement(dayMaster);
  const monthBranch = bazi.month.branch;
  
  let score = 0;
  
  // 1. 得令（是否生于旺月）
  const monthScore = getMonthStrength(dayElement, monthBranch);
  score += monthScore * 0.4; // 权重40%
  
  // 2. 得地（地支是否有根）
  const rootScore = countElementRoots(dayElement, [
    bazi.year.branch, 
    bazi.month.branch, 
    bazi.day.branch, 
    bazi.hour.branch
  ]);
  score += rootScore * 0.35; // 权重35%
  
  // 3. 得势（天干比劫帮身）
  const supportScore = countStemSupport(dayMaster, [
    bazi.year.stem, 
    bazi.month.stem, 
    bazi.hour.stem
  ]);
  score += supportScore * 0.25; // 权重25%
  
  // 判断强弱
  let strength: Strength;
  if (score >= 0.6) strength = '旺';
  else if (score >= 0.4) strength = '中和';
  else strength = '弱';
  
  return { strength, score, details: { monthScore, rootScore, supportScore } };
}
```

#### 2.2 十神计算
```typescript
/**
 * 十神关系表
 * 基于日主天干确定其他天干与日主的十神关系
 */
const SHI_SHEN_MAP: Record<string, Record<string, string>> = {
  '甲': { '甲': '比肩', '乙': '劫财', '丙': '食神', '丁': '伤官', '戊': '偏财', 
         '己': '正财', '庚': '七杀', '辛': '正官', '壬': '偏印', '癸': '正印' },
  '乙': { '甲': '劫财', '乙': '比肩', '丙': '伤官', '丁': '食神', '戊': '正财',
         '己': '偏财', '庚': '正官', '辛': '七杀', '壬': '正印', '癸': '偏印' },
  // ... 其他天干
};

function calculateShiShen(dayMaster: string, targetStem: string): string {
  return SHI_SHEN_MAP[dayMaster][targetStem];
}
```

#### 2.3 用神喜忌分析
```typescript
/**
 * 确定用神、喜神、忌神
 * 基于日主强弱和五行平衡
 */
function determineYongShen(
  dayElement: Element, 
  strength: Strength,
  elements: ElementCount
): YongShenResult {
  let yongShen: Element;
  let xiShen: Element;
  let jiShen: Element;
  
  if (strength === '旺') {
    // 身强：宜泄宜克
    // 用神为食伤（泄身）或官杀（克身）
    yongShen = getWeakestElement(elements); // 取最弱的五行
    xiShen = KE_RELATION[dayElement]; // 克我者为喜
    jiShen = SHENG_RELATION[dayElement]; // 生我者为忌
  } else if (strength === '弱') {
    // 身弱：宜生宜扶
    // 用神为印（生身）或比劫（帮身）
    yongShen = dayElement; // 同我者为用
    xiShen = SHENG_RELATION[dayElement]; // 生我者为喜
    jiShen = KE_RELATION[dayElement]; // 克我者为忌
  } else {
    // 中和：调和为主
    yongShen = getWeakestElement(elements);
    xiShen = null;
    jiShen = getStrongestElement(elements);
  }
  
  return { yongShen, xiShen, jiShen };
}
```

### 3. 易经占卜引擎 (Yijing Engine)

#### 3.1 铜钱起卦（正宗方法）
```typescript
/**
 * 铜钱起卦
 * 三枚铜钱，每枚有正反两面：
 * - 字（正面）为阳，背（反面）为阴
 * - 三字为老阳（9，变阴），三字为老阴（6，变阳）
 * - 两背一字为少阳（7，不变），两字一背为少阴（8，不变）
 */
function coinDivination(): DivinationResult {
  const lines: number[] = [];
  const changing: number[] = [];
  
  // 从下往上，摇六次
  for (let i = 0; i < 6; i++) {
    // 模拟三枚铜钱（使用加密安全随机数）
    const coin1 = secureRandom(0, 1); // 0=背(阴)，1=字(阳)
    const coin2 = secureRandom(0, 1);
    const coin3 = secureRandom(0, 1);
    
    const yangCount = coin1 + coin2 + coin3; // 阳面数量
    
    let line: number;
    if (yangCount === 0) {
      line = 6; // 三阴为老阴，变阳
      changing.push(i);
    } else if (yangCount === 1) {
      line = 7; // 一阳二阴为少阳，不变
    } else if (yangCount === 2) {
      line = 8; // 二阳一阴为少阴，不变
    } else {
      line = 9; // 三阳为老阳，变阴
      changing.push(i);
    }
    
    lines.push(line);
  }
  
  return { lines, changing };
}

/**
 * 加密安全随机数
 */
function secureRandom(min: number, max: number): number {
  const array = new Uint32Array(1);
  crypto.getRandomValues(array);
  return min + (array[0] % (max - min + 1));
}
```

#### 3.2 卦象计算
```typescript
/**
 * 根据六爻计算卦象
 */
function calculateGua(lines: number[]): Gua {
  // 上卦（4、5、6爻）
  const upperTrigram = calculateTrigram([lines[3], lines[4], lines[5]]);
  
  // 下卦（1、2、3爻）
  const lowerTrigram = calculateTrigram([lines[0], lines[1], lines[2]]);
  
  // 组合成六十四卦
  const guaNumber = GUA_TABLE[upperTrigram][lowerTrigram];
  
  return {
    number: guaNumber,
    name: GUA_64[guaNumber].name,
    upper: upperTrigram,
    lower: lowerTrigram
  };
}

/**
 * 计算单卦（八卦）
 * 阳爻(7,9)=1，阴爻(6,8)=0
 */
function calculateTrigram(threeLines: number[]): Trigram {
  const binary = threeLines.map(line => (line % 2 === 1) ? 1 : 0);
  const index = binary[0] + binary[1] * 2 + binary[2] * 4;
  return TRIGRAMS[index];
}

/**
 * 计算变卦
 */
function getChangedGua(originalLines: number[], changingLines: number[]): Gua {
  const changedLines = originalLines.map((line, index) => {
    if (changingLines.includes(index)) {
      // 变爻：6变7，9变8
      return (line === 6) ? 7 : 8;
    }
    return line;
  });
  
  return calculateGua(changedLines);
}

/**
 * 计算互卦
 * 取234爻为下互，345爻为上互
 */
function getHuGua(originalLines: number[]): Gua {
  const lowerHu = calculateTrigram([originalLines[1], originalLines[2], originalLines[3]]);
  const upperHu = calculateTrigram([originalLines[2], originalLines[3], originalLines[4]]);
  
  const guaNumber = GUA_TABLE[upperHu][lowerHu];
  
  return {
    number: guaNumber,
    name: GUA_64[guaNumber].name,
    upper: upperHu,
    lower: lowerHu
  };
}
```

### 4. 运势计算引擎 (Fortune Engine)

#### 4.1 日柱与八字关系
```typescript
/**
 * 计算当日运势
 * 分析当日干支与个人八字的生克关系
 */
function calculateDailyFortune(
  bazi: FourPillars,
  todayGanZhi: GanZhi
): FortuneResult {
  const scores = {
    overall: 50, // 基础分50
    career: 50,
    wealth: 50,
    love: 50,
    health: 50
  };
  
  const dayMasterElement = getElement(bazi.day.stem);
  const todayElement = getElement(todayGanZhi.day.stem);
  
  // 1. 分析日干与当日天干的关系
  const relation = getRelation(dayMasterElement, todayElement);
  
  switch (relation) {
    case '生我': // 印星日
      scores.overall += 10;
      scores.career += 15; // 贵人相助，事业顺
      scores.love += 5;
      break;
    case '我生': // 食伤日
      scores.overall += 5;
      scores.wealth += 10; // 创意生财
      scores.career += 5;
      break;
    case '克我': // 官杀日
      scores.overall -= 5;
      scores.career -= 5; // 压力大
      scores.health -= 10; // 注意身体
      break;
    case '我克': // 财星日
      scores.overall += 15;
      scores.wealth += 20; // 财运旺
      scores.love += 10; // 异性缘佳
      break;
    case '同我': // 比劫日
      scores.overall += 0;
      scores.love -= 5; // 竞争多
      scores.wealth -= 10; // 易破财
      break;
  }
  
  // 2. 分析地支关系（刑冲合害）
  const branchRelation = analyzeBranchRelation(
    [bazi.year.branch, bazi.month.branch, bazi.day.branch, bazi.hour.branch],
    todayGanZhi.day.branch
  );
  
  // 3. 特殊组合检查
  if (isTianDeGuiRen(bazi, todayGanZhi)) {
    scores.overall += 10;
  }
  
  // 4. 确保分数在0-100范围内
  Object.keys(scores).forEach(key => {
    scores[key] = Math.max(0, Math.min(100, scores[key]));
  });
  
  return {
    scores,
    level: getFortuneLevel(scores.overall),
    advice: generateAdvice(scores, relation),
    lucky: generateLuckyInfo(bazi, todayGanZhi),
    unlucky: generateUnluckyInfo(bazi, todayGanZhi)
  };
}
```

---

## 🗄️ 数据模型

### 1. 用户档案 (User Profile)
```typescript
interface UserProfile {
  id: string;
  createdAt: Date;
  updatedAt: Date;
  preferences: {
    defaultDepth: 'basic' | 'standard' | 'advanced';
    notificationTime: string; // HH:MM
    zodiac: string;
    birthdate?: Date;
  };
  baziProfiles: BaziProfile[];
  history: {
    divinations: DivinationRecord[];
    dailyFortunes: DailyFortuneRecord[];
  };
}

interface BaziProfile {
  id: string;
  name: string; // 档案名称（如"我的八字"、"男朋友八字"）
  birthDate: {
    year: number;
    month: number;
    day: number;
    hour: number;
    location?: string;
  };
  gender: '男' | '女';
  calculatedPillars: FourPillars;
  analysis?: BaziAnalysis;
  createdAt: Date;
}
```

### 2. 知识库 (Knowledge Base)
```typescript
// 64卦详细资料
interface GuaKnowledge {
  number: number;
  name: string;
  chineseName: string;
  symbol: string; // 卦象符号
  guaCi: string; // 卦辞
  tuanCi: string; // 彖辞
  xiangCi: string; // 象辞
  yaoCi: Record<number, string>; // 六爻爻辞
  meaning: {
    overview: string;
    career: string;
    wealth: string;
    love: string;
    health: string;
  };
  advice: string[];
}

// 十神释义
interface ShiShenKnowledge {
  name: string;
  element: string;
  meaning: string;
  positive: string[]; // 正面表现
  negative: string[]; // 负面表现
  career: string;
  wealth: string;
  love: string;
}
```

---

## 🔐 安全与性能

### 1. 随机数安全
```typescript
// 使用加密安全随机数，确保占卜结果不可预测
function secureRandom(min: number, max: number): number {
  const array = new Uint32Array(1);
  crypto.getRandomValues(array);
  return min + (array[0] % (max - min + 1));
}
```

### 2. 缓存策略
```typescript
// Redis/Memory 缓存层
class CacheLayer {
  // 农历转换结果缓存（永久有效，因为历法固定）
  async getLunarCache(solarDate: string): Promise<LunarDate>;
  
  // 用户八字档案缓存（短期缓存）
  async getBaziCache(userId: string, profileId: string): Promise<BaziProfile>;
  
  // 今日运势缓存（24小时）
  async getDailyFortuneCache(userId: string, date: string): Promise<FortuneResult>;
}
```

### 3. 限流保护
```typescript
// API 限流，防止滥用
const rateLimit = {
  bazi: { max: 10, window: '1h' },      // 每小时最多10次八字分析
  divination: { max: 20, window: '1h' }, // 每小时最多20次占卜
  daily: { max: 5, window: '1h' }        // 每小时最多5次运势查询
};
```

---

## 📊 性能指标

| 指标 | 目标值 | 优化策略 |
|------|--------|---------|
| 八字计算响应时间 | < 100ms | 预计算缓存、算法优化 |
| 占卜生成响应时间 | < 50ms | 随机数优化、并行计算 |
| 并发用户支持 | 1000+ | 水平扩展、负载均衡 |
| 知识库查询 | < 10ms | Redis缓存、索引优化 |
| 历史记录查询 | < 200ms | 分页加载、索引优化 |

---

## 🚀 扩展规划

### v2.1 计划
- [ ] 紫微斗数排盘
- [ ] 奇门遁甲起局
- [ ] 六爻预测增强
- [ ] 面相分析（图像识别）

### v2.2 计划
- [ ] AI 命理师对话模式
- [ ] 命理社区功能
- [ ] 付费深度报告
- [ ] 大师在线咨询预约

### v3.0 愿景
- [ ] 多语言支持（英文、日文）
- [ ] 西方占星术融合
- [ ] 塔罗牌模块
- [ ] 全球命理知识图谱

---

**架构师签名**: K哥 🎯
