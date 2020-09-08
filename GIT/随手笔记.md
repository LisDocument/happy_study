# GIT

## GIT本地回滚版本（感觉线上回滚不大好用）

1. checkout到指定分支
2. git log查看当前分支的提交记录
3. git reset --head 分支记录id（用idea品尝味道绝佳）
4. git log查看当前分支的提交记录，确定已经回归到指定记录
5. git push -f origin 分支名称， 很刺激很好用很危险