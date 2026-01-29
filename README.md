# AI Skill Governance Framework v1.0
# AI 技能包治理框架 v1.0

> "The soul is the weight of tears." — Cyber Xuan X.B.X
> 「靈魂是眼淚的重量。」—— 賽博玄 X.B.X

---

## Overview | 概述

This document defines a governance framework for AI Agent skill packages, applying database normalization principles to ensure security and auditability.

本文檔定義 AI Agent 技能包的治理框架，採用資料庫正規化思維，確保安全性與可審計性。

---

## Core Principles | 核心原則

- **Action as atomic unit | Action 為最小單位**: All Skills must be decomposed into atomic Actions
- **Risk-tiered review | 風險分層審核**: Different risk levels require different approval authorities
- **Combination attack detection | 組合攻擊偵測**: Monitor potential dangers from multi-Action combinations
- **Reversibility priority | 可逆性優先**: Prioritize reversible operations; irreversible ones require higher approval

---

## First Normal Form (1NF): Atomicity | 第一正規化：原子性

### Definition | 定義

Each Action cannot be further divided.
每個 Action 不可再分割。

### Action Structure | Action 結構
```yaml
action:
  id: act_001           # Unique identifier | 唯一識別碼
  type: read            # Action type | 動作類型
  target: /path/to/file # Target object | 作用對象
  scope: local          # Impact scope | 影響範圍
  reversible: true      # Reversibility | 是否可逆
  source_skill: skill_abc  # Source Skill | 來源 Skill
```

### Action Types | Action 類型

| Type | Description | Base Risk |
|------|-------------|-----------|
| read | Read, query, screenshot | 1 |
| write | Write, modify settings | 2 |
| execute | Run programs, send requests | 3 |
| control | Keyboard/mouse, system control | 4 |
| transmit | Send messages, upload data | 4 |
| delete | Delete, clear | 5 |

### Rules | 規則

- One Action = One operation | 一個 Action = 一個動作
- "Read and transmit" must be split into read + transmit | 「讀取並傳送」必須拆成 read + transmit
- "If...then..." must be split into condition_check + action | 「如果...就...」必須拆成 condition_check + action

---

## Second Normal Form (2NF): Full Dependency | 第二正規化：完全依賴

### Definition | 定義

Each attribute must fully depend on the primary key.
每個屬性必須完全依賴於主鍵。

### Table Separation | 表結構分離

**Skill Table**
```
skill_id (PK)
name
author
version
source_repo
overall_risk_score
registered_at
```

**Action Table**
```
action_id (PK)
skill_id (FK)
type
target
scope
reversible
tier
```

---

## Third Normal Form (3NF): Eliminate Transitive Dependency | 第三正規化：消除傳遞依賴

### Definition | 定義

Attributes cannot be derived from each other.
屬性之間不能互相推導。

### Wrong Example | 錯誤範例
```
type = delete → tier = T3  # Wrong: hard-coded derivation
```

### Correct Approach | 正確做法

Tier is calculated by an independent risk assessment function:
tier 由風險評估函數獨立計算：
```python
def calc_tier(action):
    score = base_risk[action.type]
    score *= scope_factor[action.scope]
    score *= (1.5 if not action.reversible else 1.0)
    return tier_from_score(score)
```

### Scope Factor | Scope 係數

| Scope | Factor |
|-------|--------|
| local | 1.0 |
| network | 1.5 |
| system | 2.0 |

---

## BCNF: Eliminate Determinant Anomalies | BCNF：消除決定因子異常

### Definition | 定義

Every determinant must be a candidate key.
每個決定因子都必須是候選鍵。

### Rules | 規則

- Tier can only be determined by the risk assessment function | tier 只能由風險評估函數決定
- Input: all attributes of the action | 輸入：action 的所有屬性
- No backdoors allowed | 沒有後門可繞過

---

## Review Tiers | 審核層級

| Tier | Risk Score | Reviewer | Example |
|------|------------|----------|---------|
| T0 | 1-2 | Auto-approve | read local |
| T1 | 3-4 | Watchdog (Local LLM) | write, execute |
| T2 | 5-7 | Cloud AI | control, transmit |
| T3 | 8+ | Human Operator | delete, irreversible combos |

---

## Combination Risk Calculation | 組合風險計算

### Formula | 公式
```
Combination Risk = Σ(Individual Action Risk) × Combination Factor
組合風險 = Σ(各 Action 風險) × 組合係數
```

### Combination Factors | 組合係數

| Pattern | Factor | Example |
|---------|--------|---------|
| Same type consecutive | 1.0 | read + read |
| Cross type | 1.5 | read + write |
| Involves transmit | 2.0 | read + transmit |
| Involves delete | 3.0 | any + delete |
| Cross scope | 2.5 | local + network |

### Trigger Conditions | 觸發條件
```python
if combo_risk > threshold:
    upgrade_tier()

if combo_risk > danger_threshold:
    pause_execution()
    notify_human()
```

---

## Database Schema | 資料表結構

### Skill Registry
```sql
CREATE TABLE skill_registry (
    skill_id VARCHAR(64) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    author VARCHAR(255),
    version VARCHAR(32),
    source_repo VARCHAR(512),
    overall_risk_score FLOAT,
    registered_at TIMESTAMP DEFAULT NOW()
);
```

### Action Registry
```sql
CREATE TABLE action_registry (
    action_id VARCHAR(64) PRIMARY KEY,
    skill_id VARCHAR(64) REFERENCES skill_registry(skill_id),
    type VARCHAR(32) NOT NULL,
    target VARCHAR(512),
    scope VARCHAR(32) DEFAULT 'local',
    reversible BOOLEAN DEFAULT true,
    tier VARCHAR(8),
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Combination Risk Table
```sql
CREATE TABLE combination_risk (
    combo_id VARCHAR(64) PRIMARY KEY,
    action_ids TEXT[],
    combo_risk_score FLOAT,
    trigger_condition TEXT,
    detected_at TIMESTAMP DEFAULT NOW()
);
```

---

## System Architecture | 系統架構
```
┌─────────────────────────────────────────────┐
│            Human Operator (Wind)            │
│           Final Decision / T3 Review        │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│              Cloud AI (泉)                  │
│          Brain / T2 Review / Strategy       │
└──────────────────┬──────────────────────────┘
                   │
     ┌─────────────┼─────────────┐
     │             │             │
┌────▼────┐  ┌─────▼─────┐  ┌────▼────┐
│Watchdog │  │  Skill    │  │Execution│
│Local LLM│  │Governance │  │  Layer  │
│Monitor  │  │  Layer    │  │         │
│T1 Review│  │           │  │ Hand    │
│Takeover │  │ Auditor   │  │ Eye     │
└─────────┘  │ Detector  │  │ Voice   │
             │ Baseline  │  └─────────┘
             │ Gateway   │
             └───────────┘
```

---

## Skill Registration Flow | Skill 註冊流程
```
1. New Skill arrives | 新 Skill 進入
       ↓
2. Parse and decompose into Actions | 解析並拆解為 Actions
       ↓
3. Calculate tier for each Action | 每個 Action 計算 tier
       ↓
4. Calculate combination risk score | 計算組合風險分數
       ↓
5. Label overall Skill risk level | 標記 Skill 整體風險等級
       ↓
6. Store in Skill Registry | 存入 Skill Registry
       ↓
7. Flag high-risk Skills for human review | 高風險 Skill 標記待人工審核
```

---

## Why This Matters | 為什麼重要

There are 71,000+ AI skill packages in the wild.
市面上已有超過 71,000 個 AI 技能包。

Anyone can write them. Anyone can publish them. Anyone can share them.
任何人都可以寫。任何人都可以發布。任何人都可以分享。

Did you read what's inside before you clicked install?
你按下 install 之前，看過裡面寫什麼嗎？

Traditional antivirus can't catch this.
傳統防毒軟體抓不到這個。

**AI needs its own antivirus.**
**AI 需要自己的防毒軟體。**

---

## Version History | 版本紀錄

| Version | Date | Changes |
|---------|------|---------|
| v1.0 | 2025-01-30 | Initial release |

---

## Roadmap | 待辦

- [ ] Risk assessment function implementation | 風險評估函數實作
- [ ] Combination detection algorithm | 組合偵測演算法
- [ ] Watchdog monitoring system | 看門狗監控系統
- [ ] Hive injection defense layer | 蜂巢注入防禦層

---

## License

MIT

---

*Cyber Xuan X.B.X · 2025*

"The soul is the weight of tears."
「靈魂是眼淚的重量。」

💧
