# Git

## Delete all commits

适用于想要删除之前所有的提交，只保留当前版本。

<details>
<summary>删除所有 commit</summary>

```sh
# 创建只包含当前版本的分支
git checkout --orphan tmp

# 提交工作区中的内容
git add -A
git commit -m "clean project"

# 删除原有分支
git branch -D master

# 将当前分支重命名为原来分支的名字
git branch -m master

# 强制推送至远程
git push -f origin master
```

</details>

## 修改 commit 作者信息

### 批量修改

<details>
<summary>批量修改 commit 作者信息</summary>

```sh
#!/bin/sh
git filter-branch --env-filter '

OLD_EMAIL="l@q.com"
NEW_NAME="hello"
NEW_EMAIL="q@q.com"

if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ] ; then
    export GIT_COMMITTER_NAME=$NEW_NAME
    export GIT_COMMITTER_EMAIL=$NEW_EMAIL
fi

if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ] ; then
    export GIT_AUTHOR_NAME=$NEW_NAME
    export GIT_AUTHOR_EMAIL=$NEW_EMAIL
fi
' --tag-name-filter cat -- --branches --tags

# ! 强制推送
git push --force --tags origin 'refs/heads/*'
```

</details>

### 单次修改

<details>
<summary>修改某次 commit 的信息</summary>

```sh
#git rebase -i commit_hash^ # 父提交
git rebase commit_hash^ # 父提交

# 修改
git commit --amend --author="New Author Name <new@email.com>"

git rebase --continue

# ! 强制推送
git push --force
```

</details>
