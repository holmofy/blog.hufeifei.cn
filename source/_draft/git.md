工作区 -> index区 -> local repository -> remote repository

remote repository默认名字叫做origin

git使用hash记录每次修改

HEAD是暂存index区的指针名

每个分支都是一个指针，这个指针存储了某次修改的hash
默认主分支的指针名为master

标签tag也是指针，但是指针存储的hash值不能修改，我把它叫做“常指针”，有了tag就没必要记那次提交改变的hash值了

git push就是把改变的内容和相应的hash发到远程仓库
git push origin <tagname>
