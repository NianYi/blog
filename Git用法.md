Git用法
基础用法
git remote  -v
git branch -a
git checkout -b develop
git push origin develop:develop
git push origin :develop
git push origin develop


1. 合并另一个分支上单个commit:
git cherry-pick 62ecb3（不用指明分支）

2. 合并另一个分支上多个commit:
以指定commit为结尾（包含该commit），创建新分支
git checkout -b newbranch 62ecb3

3. 将当前分支指定commit以后的commit（不包含），加到指定分支master上去
git rebase --onto master 76cada
可能需要修改冲突并git add xx
git rebase –continue继续合并
4. 拉取远程分支并自动切换
git checkout -b 本地分支名x origin/远程分支名x

5. 合并同分支上多个commit
Git rebase –i HEAD~commit

git pull <远程主机名> <远程分支名>:<本地分支名>
6. 修改最近一次commit的作者和邮箱
git commit --amend --author="NewAuthor <NewEmail@address.com>"
7. 修改最近一次commit的内容
git commit –-amend
如果需要修改文件，只需git add之后git commit –-amend，然后push –f 强制推送到远程仓库
8. 配置提交信息
git config user.name “firstname lastname”
git config user.email xxx@yy.com
9. git commit libavformat/movenc.c -s [-s 主要是指明这个代码是你改的]

10. 删除远程分支
git branch -r -d origin/branch_name 删除本地的远程分支
git push origin :branch_name 删除远程分支
