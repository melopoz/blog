##### 初始化成git仓库

`git init`

##### 绑定到远程仓库

`git remote add origin https://xxx.git`

##### 第一次push 的时候携带的参数

`git push -u origin master`



## revert

`git revert HEAD^`  `git revert HEAD^^`  `git revert HEAD~3`   一个^就是前1次commit 

`git revert <commitId>`  revert 到指定的commit版本

会再生成一次commit，并且当前代码和指定的那次commit的代码一致。



## reset

用` reset `回滚，参数：

> - git reset --soft						  保留工作目录，并把新的全部差异放进暂存区
>
> > 原节点和Reset节点之间的所有差异都会放到暂存区中
>
> - git reset (git reset --mixed)	保留工作目录，清空暂存区
>
> - git reset --hard						 清空工作目录，清空暂存区，全部和git log 中的上n次commit内容相同
>
>   > 然后push的话会报错，‘必须先pull’ 才能push， 但是pull，会把刚才回退的commit又带出来，所以得 push -f   （强制）

>  soft 和 mixed 区别应该就是 mixed 不需要再手动 add 而 soft 需要自己再手动 add 然后 commit。



## 修改commit的msg
`git commit --amend` 
会进入vim，修改保存就行了

### 如果commit已经被覆盖

`git rebase -i HEAD^` 

`git rebase -i HEAD~n` n是逆推几次

```
pick 07bca843 添加 广告线索来源构成分析、概览 接口 /source-ad/constitute、/source-ad/list
pick 67d1e638 移除/income/top
```
将开头的pick修改为edit就可以编辑了
然后git会提示

```
You can amend the commit now, with
  git commit --amend
Once you are satisfied with your changes, run
  git rebase --continue
```
`git commit --amend` 进入修改
完事continue rebase 即可
`git rebase --continue`

​             

## git diff

`git diff --cached` 查看本地分支和当前代码的区别



## 统计git log

```
git log --author="$(git config --get user.name)" --since=2021-01-01 --until=2021-03-30 --pretty=tformat: --numstat | gawk '{ add += $1 ; subs += $2 ; loc += $1 - $2 } END { printf "增加的行数:%s 删除的行数:%s 总行数: %s\n",add,subs,loc }'
```

