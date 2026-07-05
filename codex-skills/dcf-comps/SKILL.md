---
name: dcf-comps
description: "AI Berkshire skill: DCF/Comps：绝对估值与相对估值双轨建模. Source: skills/dcf-comps.md."
---

## Codex adapter note

This skill is generated from `skills/dcf-comps.md` so Claude Code and Codex users share one canonical workflow.

- Treat `$ARGUMENTS` as the user's request in the current Codex thread.
- When the source mentions Claude-only surfaces such as Task, Agent, WebSearch, Bash, Read, or Write, use the closest Codex capability available in this session: subagents when available, web search when needed, shell commands for local tools, and normal file edits for workspace files.
- Use shared project tools from `tools/` in this repository. Prefer running commands from the repository root with paths like `python3 tools/financial_rigor.py ...`; if the current thread starts outside the repo, locate the actual checkout path first instead of assuming a fixed home-directory path.
- Before starting research, run the `date` command to confirm today's date; treat it as the baseline for "latest" data and state the data cutoff date in the report header. Never assume the current date from training data.
- Preserve the research quality rules from `AGENTS.md`: cross-check financial data, use exact arithmetic tools for valuation/math, and clearly label uncertainty and source gaps.

# DCF/Comps：绝对估值与相对估值双轨建模

对 $ARGUMENTS 执行 DCF（折现现金流）与 Comps（可比公司相对估值）双轨建模，交叉验证给出估值区间。

**支持输入格式**：`公司名`，例如：`腾讯`、`拼多多 现价100`（可附带当前股价用于计算安全边际，不附带则只给估值区间不给结论）

> "沃伦经常说要做DCF计算，但我从没见他真正算过。" —— 芒格
>
> "价格是你付出的，价值是你得到的。" —— 巴菲特
>
> "估值不是精确科学，是给你的判断上一个数字的枷锁，逼你诚实面对自己的假设。" —— 李录（意译）

## 设计理念

芒格对DCF的怀疑不是反对估值，而是反对**把假设的精确性误当成结果的精确性**。折现率、增长率、永续增长率任何一个假设变动1个百分点，算出来的"内在价值"就能差出30%+。

本Skill的应对方式：
1. **不给单一数字，给区间**：DCF乐观/中性/悲观三情景 + comps同业倍数区间，两种方法互相校验
2. **强制敏感度呈现**：每次DCF都附带折现率×永续增长率敏感度表，让使用者看到假设有多脆弱
3. **正反两面**：DCF算出来"低估"不等于买入理由，必须同时呈现支持和质疑这个估值的证据
4. **不适用就不用**：负FCF、商业模式未稳定、周期性行业底部，DCF直接不可靠，如实说明

## 适用性前置判断

| 情况 | DCF适用性 | 建议 |
|------|---------|------|
| 成熟现金牛（FCF稳定为正、增速可预测） | 适用 | 正常执行DCF |
| 高成长但FCF为负（如早期SaaS、扩张期平台） | 不适用 | 只做comps，或用远期FCF估算+更长显性期 |
| 周期性行业（大宗商品、航运、半导体设备） | 谨慎 | 用穿越周期的中枢FCF，而非最近一期 |
| 同业高度标准化、可比公司多（银行、消费品） | comps优先 | DCF作为交叉验证 |
| 同业稀少或不可比（独特商业模式） | DCF优先 | comps仅供参考，标注"可比性弱" |

如果标的不适合DCF（如持续负FCF且无清晰转正路径），**明确说明并只执行comps部分**，不要强行套用DCF制造虚假精确感。

## 执行流程

### 第一步：资料获取

按 `skills/financial-data.md` 规范获取，关键数据至少2个独立来源交叉验证：

- **DCF所需**：近5年经营现金流、资本开支（算FCF）、总股本、净负债（现金-有息负债）、无风险利率（10年期国债）、beta、行业风险溢价
- **Comps所需**：5-8家同业公司名单（业务模式/成长阶段/地域尽量可比），各自的PE、EV/EBITDA、PS、PB（视行业特性选1-2个最相关倍数，银行用PB、SaaS用PS或EV/Revenue、成熟消费品用PE）

### 第二步：估算折现率（WACC）

用CAPM框架估算股权成本：

```
股权成本 = 无风险利率 + beta × 股权风险溢价
```

若有有息负债，按资本结构加权：

```
WACC = 股权成本 × (股权/总资本) + 税后债务成本 × (负债/总资本)
```

用 `tools/financial_rigor.py calc` 做精确计算：

```bash
python3 tools/financial_rigor.py calc --expr "0.045 + 1.2 * 0.055"
```

无法可靠估算beta/资本结构时，可用行业惯用折现率区间代替（如成熟大盘股8-10%，高成长/高风险12-15%），但必须标注"简化假设，非CAPM精算"。

### 第三步：FCF三情景预测

基于历史FCF增速、管理层指引、行业展望，建立**乐观/中性/悲观**三组显性期增速假设（建议5年），以及对应的永续增长率（通常不超过长期GDP增速，2-3%）。

三组假设必须分别列出依据（不能凭空给数字）：

| 情景 | 显性期增速假设 | 依据 | 永续增长率 |
|------|--------------|------|-----------|
| 乐观 | ... | ... | ... |
| 中性 | ... | ... | ... |
| 悲观 | ... | ... | ... |

### 第四步：DCF精确计算

对三个情景分别调用工具计算（净负债：现金净额记为负值）：

```bash
python3 tools/financial_rigor.py dcf \
  --fcf <基期FCF> \
  --growth <显性期各年增速，如 0.15 0.15 0.12 0.10 0.08> \
  --discount-rate <WACC> \
  --terminal-growth <永续增长率> \
  --shares <总股本> \
  --net-debt <净负债> \
  --current-price <当前股价，若有> \
  --currency <币种>
```

记录三个情景各自的每股内在价值，以及工具自动输出的敏感度表格（原样纳入报告，不要省略）。

### 第五步：Comps相对估值

为每个选定倍数（如PE、EV/EBITDA）调用工具：

```bash
python3 tools/financial_rigor.py comps \
  --metric-name PE \
  --target-value <标的对应指标，如EPS> \
  --peer-values '{"同业A":18.2,"同业B":22.1,"同业C":15.5}' \
  --basis per-share \
  --currency <币种>
```

企业层面倍数（EV/EBITDA、EV/Revenue）用 `--basis enterprise`，需附 `--shares` 和 `--net-debt`。

**同业选取原则**：优先选业务模式、成长阶段、地域接近的公司；成长阶段差异大的同业（如成熟龙头 vs 早期挑战者）不应简单平均，需在报告中说明取舍。

### 第六步：三角验证

将 DCF 三情景区间、comps 各倍数隐含区间、当前市价放入同一张表比较：

| 方法 | 悲观/低 | 中性/中位数 | 乐观/高 |
|------|--------|------------|--------|
| DCF | ... | ... | ... |
| Comps(倍数1) | ... | ... | ... |
| Comps(倍数2) | ... | ... | ... |
| **当前股价** | colspan 说明相对位置 |||

**如果DCF和comps结论方向一致**（都显示低估或都显示高估）：可信度较高，说明原因。

**如果DCF和comps结论矛盾**：不要选择性采用对自己有利的一个，必须呈现两者矛盾的原因（如市场用更低的永续增长预期定价 vs DCF假设过于乐观），并明确标注"结论存在分歧，需进一步判断"。

### 第七步：输出报告

```
一、结论速览（估值区间 + 相对当前股价的位置，一段话，不超过150字）
二、适用性判断（为什么DCF/comps适合或不适合这家公司）
三、DCF建模
   - 三情景假设及依据
   - 每股内在价值（三情景）
   - 敏感度表格
四、Comps相对估值
   - 同业选取及理由
   - 隐含估值区间
五、三角验证表
六、正反两面论据
   - 支持当前估值区间的证据
   - 质疑/削弱这个估值区间的证据（估值方法本身的局限、假设的脆弱点）
七、局限性声明
```

结论必须明确回答：
1. **估值区间是多少，当前股价落在区间的什么位置？**（不能只说"合理"）
2. **这个结论对哪个假设最敏感？**（如"结论高度依赖永续增长率是否能维持2.5%"）
3. **DCF和comps是否互相印证？**

### 第八步：保存报告

写入 `reports/{公司名}/{公司名}-valuation-{YYYYMMDD}.md`

### 第九步：数据抽检（准出流程）

```bash
python3 ~/ai-berkshire/tools/report_audit.py extract \
  --report reports/{公司名}/{公司名}-valuation-{YYYYMMDD}.md

# 对清单每项从可靠信源取数（参见 skills/financial-data.md）

python3 ~/ai-berkshire/tools/report_audit.py verdict \
  --results '<填好的JSON>' \
  --report {报告文件名}
```

**【准出】** 全部通过 → 发布；**【打回】** 有不通过 → 修正后重审。

## 关键原则

- **给区间，不给单一数字**：任何"目标价"都应该是一个范围加概率判断，不是一个精确到小数点后两位的数字
- **假设先行，结论在后**：先明确写出所有假设，再看结论，不能倒着凑数字支持想要的结论
- **DCF的价值在于逼你想清楚假设，不在于算出的数字本身准确**——芒格的怀疑针对的是"假装精确"，不是估值这件事本身
- **不是投资建议**：本Skill输出仅供研究参考，不构成买卖依据，最终判断需结合定性分析（管理层、护城河、行业周期位置）
