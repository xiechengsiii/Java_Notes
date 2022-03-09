### Git 使用指南

---

#### 一些命令

- ``reset``  前进/后退版本

  ``git reset --hard 版本号``  
  
   版本号：hash后的前七位
  
  ``--hard`` ： 工作区 、暂存区、本地仓库的代码均更新
  
  ``-- mixed``  本地库、工作区代码更新，暂存区不更新
  
  ``-- soft``  本地库更新，工作区、暂存区不更新

- ``git diff``

  ``git diff `` 比较工作区和暂存区所有文件差异

  ``git diff test3.txt``  比较text3.txt工作区和暂存区的差别

  ``git diff ed3adb3 test3.txt`` 通过索引号比较工作区和对应版本的本地仓库的文件差别

- ``git remote add origin https://你的仓库地址``

  起别名，https地址的别名为origin。设置好之后，直接通过``git pull origin master/ git push origin master``  对远程仓库的master分支进行操作即可，而不需要输入地址

  ``git clone``  会自动创建远程库的别名：origin

- 冲突的解决方法

  对于同一个文件的同一行，不同分支都修改了，可能会产生冲突

  解决：先pull，然后人为删减，解决冲突之后，add ， commit，push
  
  ##### merge vs rebase
  
  https://www.atlassian.com/git/tutorials/merging-vs-rebasing
  
  两个命令都是可以将一个分支的更改合并到另外一个分支当中，但是原理不同：
  
  merge：
  
  ![merge](C:\Users\XXX\Desktop\gitPic\merge.png)
  
  This creates a new “merge commit” in the `feature` branch that ties together the histories of both branches
  
  rebase：
  
  ![rebase](C:\Users\XXX\Desktop\gitPic\rebase.png)
  
  This moves the entire `feature` branch to begin on the tip of the `main` branch, effectively incorporating all of the new commits in `main`. But, instead of using a merge commit, rebasing *re-writes* the project history by creating brand new commits for each commit in the original branch.
  
  一般来说rebase的操作顺序
  
  ```java
  git checkout feature;
  git rebase main // 此时feature，main的head如上图所示
  //可能需要解决冲突 需要一个一个解决，
  git add.
  git rebase --continue;
  //解决完之后  需要切回main分支 执行merge操作  让head指向最新的提交
  git checkout main；
  git merge feature;
  ```
  
  rebase做了什么事情？
  
  1.取消feature分支的每个commit
  
  2.把上面的操作临时保存成patch文件，存在.git/rebase目录
  
  3.把feature分支更新到master分支
  
  4.patch文件应用到feature分支上。
  
  rebase 的好处 ： 干净的，线性的提历史， 同时 -i 可以合并多次commit为一个commit:
  
  ```java
  git rebase -i head~2
  ```
