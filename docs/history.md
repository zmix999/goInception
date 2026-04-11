# goInception Fork 历史

```
GitHub 上的仓库们：
┌─────────────────────────────────┐
│  hanchuanchuan/goInception      │  ← 原作者（已停更）
│  最后更新: 2024-11              │
└─────────────────────────────────┘
         │ fork
         ▼
┌─────────────────────────────────┐
│  naughtyGitCat/goInception      │  ← 我们的仓库
│  分支:                          │
│    master     = 原作者的代码     │
│    production = zmix999的代码    │  ← 生产使用
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  zmix999/goInception            │  ← 最活跃的维护者（70个改进）
│  young-0/goInception            │  ← 改了1处（已被zmix999包含）
│  xqiljkxw/goInception           │  ← 改了1处（已被zmix999包含）
└─────────────────────────────────┘
```

## 各 Fork 改动说明

| 仓库 | 有价值的改动 | 是否已合入 production |
|------|------------|---------------------|
| zmix999 | OceanBase 支持、MySQL 8/9 binlog 兼容、Go 1.24、gorm v2 升级、存储过程/函数备份、外键备份、Parser 增强、DRDS 清理、gh-ost 1.1.8 适配 | 是（production 分支基底） |
| young-0 | MySQL 8.4+ `SHOW BINARY LOG STATUS` 兼容 | 是（已被 zmix999 包含） |
| xqiljkxw | ALTER TABLE ALGORITHM 审计规则 | 是（已被 zmix999 包含） |

## 本地 Remote 配置

| remote 名 | 地址 | 用途 |
|-----------|------|------|
| origin | naughtyGitCat/goInception | 推送代码 |
| upstream | hanchuanchuan/goInception | 原作者基线 |
| zmix999 | zmix999/goInception | 拉取最新改进 |
| young0 | young-0/goInception | 备查 |
| xqiljkxw | xqiljkxw/goInception | 备查 |
