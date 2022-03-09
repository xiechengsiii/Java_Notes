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
