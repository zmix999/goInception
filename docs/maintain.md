# goInception 日常维护

## 1. 部署使用 production 分支

```bash
cd ~/github/goInception
git checkout production
# 编译、部署
```

## 2. 同步 zmix999 的新提交

```bash
cd ~/github/goInception
git fetch zmix999                  # 把他的新代码拉到本地
git checkout production            # 切到生产分支
git merge zmix999/master           # 合并他的更新
git push origin production         # 推到我们的 GitHub
```
