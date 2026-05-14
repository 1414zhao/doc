git init
git add . 到缓存区
git commit -m name 到本地
git remote add origin http://   关联外部到origin
git push origin master    提交到外部

git branch -m newname  重命名分支
git branch -d name    删除本地分支
git push origin --delete name  删除远程分支


./qemu-arm-static -g 1234 ./bin/httpd
gdb-multiarch -q ./bin/httpd
(gdb) target remote :1234




查看远程仓库：$ git remote -v

添加远程仓库：$ git remote add [name] [url]

删除远程仓库：$ git remote rm [name]

拉取远程仓库：$ git pull [remoteName] [localBranchName]

推送远程仓库：$ git push [remoteName] [localBranchName]

$git push origin test:master         // 提交本地test分支作为远程的master分支

查看本地分支：$ git branch

查看远程分支：$ git branch -r

创建本地分支：$ git branch [name] ----注意新分支创建后不会自动切换为当前分支

切换分支    $ git checkout -b newbranch

合并分支：$ git merge [name] ----将名称为[name]的分支与当前分支合并

查看节点：git log --oneline

图形形式查看节点：git log --oneline --graph --all

存在缓存区： git add

提交： $ git commit


docker start NT00195
docker exec -it NT00195 /bin/bash
ssh -p 12345 root@localhost

git add 具体文件
git commit -m 提交名
git stash  保留更改
git pull 来更新分支
git stash pop 将更改出栈
git branch --set-upstream-to=远程分支名  本地分支关联指定的远程分支
git push 
git submodule foreach --recursive git clean -dfx 清除缓存

推送本地分支到远程
git push origin feature/test:feature/online
103.131.168.161:2000
101.36.160.161:2000 (home)

git reset id 本地户软回退
git stash
git push --force 强制push到远程分支，相当于远程分支回退
git stash pop
git commit -m "--"提交
git push 提交push

