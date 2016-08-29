### github操作
- [ ] 仓库关系

- A作为主仓库；B和C作为从仓库，B和C从主仓库同步代码到自己本地

- [ ] 操作
 
- 登录自己的github主页，搜索主库的链接，打开需要fork的仓库，选择fork。
- 选中自github仓库的链接克隆到本地,会在本地生成一个跟代码库相同的目录。
```
git clone git@github.com:baizipo/notebook.git
```
- 进入到克隆的目录,新建一个分支,bzp-first-commit是分支名,切换到这个分支下

```
git checkout -b bzp-first-commit
```
- 编辑文件，如vim Readme.md,编辑后保存退出，并使用git diff查看修改

```
vim Readme.md
git diff
```
- 使用git commit 确认提交代码，并在打开的文件中添加有意义的信息，比如增加x功能
```
git commit -a
```
- 查看提交日志

```
git log
```
- 确认无误，提交代码

```
git push origin bzp-first-commit:bzp-first-commit
注:第一个bzp-first-commit是本地的分支，第二个是github仓库的分支
```
- 在github页面上发起pull request 请求，页面操作略

---

- [ ] 添加远端仓库到本地，如下:
- 添加远端仓库

```
git remote add chinadevops https://github.com/chinadevops/fextend.git
注:chinadevops是远端仓库的名字，可以自己定义

```
- 查看仓库信息
```
git remote -v 显示如下
chinadevops	https://github.com/chinadevops/notebook.git (fetch)
chinadevops	https://github.com/chinadevops/notebook.git (push)
origin	git@github.com:baizipo/notebook.git (fetch)
origin	git@github.com:baizipo/notebook.git (push)
```
- 从远程chinadevops获取分支到本地(git fetch)

```
git fetch chinadevops

注:从远程的chinadevops的master（只有这一个分支）主分支下载最新的版本到本地chinadevops/master分支上
```
- 列出远程分支

```
git branch -r

  chinadevops/master
  origin/HEAD -> origin/master
  origin/master

```
- 本地切换到chinadevops/master分支上，并创建一个chinadevops-master的分支
```
git checkout chinadevops/master -b chinadevops-master
```
- 在chinadevops-master这个分支上更新代码
```
git pull
```
- 在chinadevops-master这个分支上新建一个分支，准备提交最新代码
```
git checkout -b second-commit 
```
- ==提交代码跟上面操作章节一致==

- 提交代码后删除本地分支
```
git branch -D bzp-first-commit
```
- 提交代码后删除github端分支
```
git push origin :bzp-firsh-commit
```
---
- [ ] 保持自己github页面跟主仓库代码一致
- 保持本地github为最新代码,切换到chinadevops-master分支上,使用git pull 获取最新代码
```
git checkout chinadevops-master
git pull
```
- 切换到本地master分支，rebase 下chinadevops-master分支
```
git checkout master
git rebase chinadevops-master
```
- 本地push到本地的github分支上，保持页面最新
```
git push origin master:master
```
---
版本回退

- 先使用git reflog 查看需要回退到的点
```
git reflog

69e01f5 HEAD@{3}: commit: [图片]
c52b4b6 HEAD@{4}: checkout: moving from master to myself
```
- 如回退到69e01f5
```
git reset --hard 69e01f5
```
