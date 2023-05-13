## 安装和配置

Git 官网：[Git (git-scm.com)](https://git-scm.com/)
这是我在 github 线上修改的内容

安装完之后可以使用 `git -v` 命令查看当前 git 版本，测试是否安装成功。

>配置

git 的配置命令使用 git config 开头，配置的域可分为三个：
- --global              use global config file（推荐）
- --system            use system config file
- --local                use repository config file

三个域对应着三个配置文件

> 配置用户名和邮箱

- 配置用户名（如果输入的用户名没有空格，不用加 " "，邮箱同理）
`git config --global user.name "Donkey Qian"`
- 配置邮箱
`git config --global user.email DonkeyQian@gmail.com`

> 查看配置信息

`git config --global --list`

## 创建仓库

一般来说，创建仓库有两种方式：
1. `git init [name]`
2. 使用 `git clone [url]` 将一个已经存在的仓库从 Github 等线上平台克隆到本地

仓库被创建后，会默认生成一个名称为 .git 的隐藏目录，里面存放的都是 git 相关的文件，建议不要修改里面的东西。

## 工作区域和文件状态

>工作区域

git 的本地数据管理分为三个区域：
1. 工作区（Working Directory）
	1. 就是我们电脑上的目录
2. 暂存区（Staging Area）
	1. 是一个临时存储区域，用于存放即将提交到 Git 仓库的修改内容
3. 本地仓库（Local Repository）
	1. 包含了完整的项目历史和元数据

当你修改完工作区的文件之后，要将他们添加到暂存区，然后再将暂存区的内容添加到本地仓库。

![](../../image/Pasted%20image%2020230512002708.png)

在这个过程中，我们可以使用 git 提供的命令查看、修改、撤销 修改的内容。

打个比喻，工作区就是我们生产货物的工厂，本地仓库就是存放货物的仓库，我们在工厂中生产出货物之后，并不会直接将货物存进仓库里，因为在提交货物的过程中，我们可能还要对货物进行检查和修改。因此我们会将生产出来的代码先放到货车，也就是暂存区上。当凑够一批货物之后，再将货车上的货物一次性运到仓库里（本地仓库）。

***
>文件状态

git 管理的文件共有四种状态：
1. 未跟踪（untrack）
	1. 新创建的，还没有被 git 管理起来的文件
2. 未修改（unmodified）
	1. 已经被 git 管理，但是文件内容没有发生变化的文件
3. 已修改（modified）
	1. 已经被 git 管理且被修改，但是修改没有放到暂存区里
4. 已暂存（staged）
	1. 已经被 git 管理且修改之后被放到暂存区里的文件

![](../../image/Pasted%20image%2020230512004109.png)

## 添加和提交文件

>查看当前仓库的文件状态

`git status` 命令可以查看当前仓库所有**非已暂存**状态的文件，回显出来的内容会标识文件的状态。

![](../../image/Pasted%20image%2020230512005016.png)

>将文件添加到暂存区

`git add [files]` 命令可以将未保存的修改保存到暂存区。

- `git add .` 保存所有未保存的修改
- `git add [filename]` 保存指定的文件，文件名带空格的需要使用 " " 包裹
- `git add *.txt` 使用通配符保存修改

>查看暂存区中的文件

`git ls-files` 命令可以查看暂存区中的所有文件

>提交暂存区中的修改

`git commit` 命令可以提交暂存区中的修改，如果不加上 `-m` ，会进入一个 vim 界面，要求你添加提交信息。

- `git commit -m "commit message"` 指定提交信息

>查看历史提交记录

`git log` 命令可以查看历史提交记录，显示的内容有提交的 版本ID、日期、作者和邮箱

![](../../image/Pasted%20image%2020230512010710.png)

如果觉得显示的内容太多了，可以使用 `git log --oneline` 命令查看缩略信息。

![](../../image/Pasted%20image%2020230512011125.png)

## 回退版本

`git reset [version id]` 命令可以将 git 回退到某一个版本，这个命令共有三种模式：
- git reset --soft 
	- 回退到某一个版本，且保存工作区和暂存区的修改
- git reset --hard
	- 回退到某一个版本，但丢弃工作区和暂存区的修改
- git reset --mixed
	- 回退到某一个版本，丢弃工作区的修改，但保存暂存区的修改

![](../../image/Pasted%20image%2020230512011758.png)

如果回退到了某一个意料之外的版本也不用担心，git 中的所有操作都是可回溯的。`git reflog` 命令可以查看所有操作的历史记录，选择回退之前的 ID，再次 reset 即可。

![](../../image/Pasted%20image%2020230512012753.png)

## 查看差异

`git diff` 命令可以查看工作区、暂存区和本地仓库之间的差异，也可以查看不同版本，不同分支之间的差异。

- `git diff` 命令如果不带参数默认会比较工作区和暂存区之间的差异
- `git diff HEAD` 命令可以比较工作区和版本库之间的差异（HEAD 指向版本库）
- `git diff --cached` 命令可以比较暂存区和版本库之间的差异
- `git diff [版本ID] [版本ID]` 可以比较两个版本之间的差异
	- 也可以将某个版本和 HEAD 直接进行比较
- `git diff HEAD HEAD~` 命令可以比较当前版本和上一版本之间的差异
	- `git diff HEAD HEAD~n` 命令可以比较当前版本和前 n 个版本之间的差异
- `git diff HEAD HEAD~ [fileName]` 命令可以比较指定文件的两个版本之间的差异
- `git diff [branch_name] [branch_name]` 比较两个分支之间的差异

## 删除文件

一般来说有两种做法：
1. 使用操作系统提供的命令在工作区中删除，然后将修改提交到暂存区
2. 使用 `git rm [fileName]` 命令直接在工作区和暂存区中删除指定文件

上述的两种方式都能删除由 git 管理的文件，但要注意，版本库中依然保存着文件信息，删除之后要记得 commit 提交修改。

> 删除暂存区和版本库中的文件

`git rm --cached [fileName]` 命令可以将文件从暂存区中删除，但保留在当前工作区。

## .gitignore

>应该忽略哪些文件？

![](../../image/Pasted%20image%2020230512015505.png)

>匹配规则

![](../../image/Pasted%20image%2020230512020236.png)
