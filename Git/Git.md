- [x] index文件，暂存区包含所以当前分支文件
- [x] push后**不会**删除无用的object
- [x] 切换分支后提交远程，新克隆后是master
- [x] reset不删除对象
- [ ] 移除大文件
- [x] refs目录，跟踪分支，上游分支
- [ ] `branch.<name>.remote`和`branch.<name>.merge`

------

# 基础概念

Git是分布式版本控制系统，客户端并不只提取最新版本的文件快照，而是把代码仓库完整地镜像下来。 这么一来，任何一处协同工作用的服务器发生故障，事后都可以用任何一个镜像出来的本地仓库恢复。 

Git直接记录快照，而非差异比较，近乎所有操作都是本地执行，使用哈希值保证完整性，一般只添加数据。

Git有三种状态：**已提交（committed）**、**已修改（modified）**和**已暂存（staged）**。**已提交**表示数据已经安全的保存在本地数据库中。 **已修改**表示修改了文件，但还没保存到数据库中。 **已暂存**表示对一个已修改文件的当前版本做了标记，使之包含在下次提交的快照中。

Git项目有三个工作区域：**Git仓库（Git directory/repository）**、**暂存区（staging area）**和**工作树/工作目录（working tree/working directory）**。**Git仓库**是Git用来保存项目的元数据和对象数据库的地方，一般是位于工作树根目录的`.git`目录。 **工作树**是对项目的某个版本从Git仓库的压缩数据库中独立提取出来的内容，放在磁盘上供你使用或修改。**暂存区**是一个文件，保存了下次将提交的文件列表信息，一般在Git仓库目录中，有时候也被称作“索引”。

## 名词解释

**裸仓库（bare repository）**

一个不包含当前工作目录的仓库，通常是一个以`.git`结尾的目录。

**commit-ish/commitish**

提交对象或递归指向提交对象的对象。

**tree-ish/treeish**

树对象或递归指向树对象的对象，commit-ish也属于tree-ish。

**悬空对象（dangling object）**

无法从任何分支、标签和引用访问的对象。

**游离的HEAD（detached HEAD）**

HEAD指向任意提交对象，而不是分支。

**特性分支（topic branch）**

一种短期分支，被用来实现单一特性或其相关工作。

**远程跟踪分支（remote-tracking branch）**

远程分支状态的引用，对应于远程引用，命名格式为`(remote)/(branch)`。

**跟踪分支（tracking branch）**

与远程分支有直接关系的本地分支，即从远程跟踪分支检出的本地分支，其跟踪的分支为上游分支。

**上游分支（upstream branch）**

被本地分支跟踪的远程分支，通过config配置文件的`branch.<name>.remote`和`branch.<name>.merge`配置。命令行中经常使用远程跟踪分支来指代上游分支。上游分支可以通过`@{upstream}`或`@{u}`来引用。

**合并源分支**

将A分支合并到B分支，A就是合并源分支。

**合并目标分支**

将A分支合并到B分支，B就是合并目标分支。

**快进合并（fast-forward merge）**

当试图合并两个分支时，如果顺着一个分支走下去能够到达另一个分支，那么Git在合并两者的时候，只会简单的将指针向前推进（指针右移），因为这种情况下的合并操作没有需要解决的分歧。

**拉取请求/合并请求（pull request）**

让任何能够看到这个项目的协作者在被管控的情况下对这个项目作出贡献。当一个分支开发完成之后，就可以创建一个合并请求，将该分支（**head分支（head branch）**）合并到另一个分支（**基本分支（base branch）**）。

**clean**

当前分支没有需要提交的修改。

**dirty**

当前分支包含未提交的修改。

**底层（plumbing）命令**

用于完成底层工作的命令，适合作为新命令和自定义脚本的组成部分。

**高层（porcelain）命令**

更友好的命令，提供对Git的高级访问。

**哑（dumb）协议**

该协议在传输过程中，服务端不需要有针对Git特有的代码，抓取过程是一系列HTTP的GET请求，这种情况下，客户端可以推断出服务端Git仓库的布局。使用哑协议的版本库很难保证安全性和私有化，且不能从客户端向服务端发送数据，所以已经很少使用。

**智能（smart）协议**

该协议需要在服务端运行一个进程，用于读取本地数据、理解客户端有什么和需要什么，并生成合适的包文件，支持通过SSH和HTTP(S)传输数据。

**引用规格（refspec）**

抓取（fetch）和推送（push）使用引用规格来描述远程分支和本地引用的映射，通过config配置文件的`remote.<name>.fetch`和`remote.<name>.push`配置。

引用规格的格式由一个可选的`+`号和紧随其后的`<src>:<dst>`组成，当`<dst>`为空时，可以省略冒号。`+`号表示即使在不能快进（fast-forward）的情况下也要（强制）更新引用。`<src>:<dst>`表示根据源引用`<src>`更新目标引用`<dst>`，引用不能使用部分通配符（如`refs/remotes/origin/qa*`），只能是目录`/`后面加`*`，（如`refs/remotes/origin/qa/*`）。对于抓取操作，`<src>`表示远程仓库自己的引用，`<dst>`表示本地仓库的远程引用。对于推送操作，`<src>`表示本地仓库的引用，`<dst>`表示远程仓库自己的引用，`<src>`为空`<dst>`不为空时，表示从远程仓库中删除引用`<dst>`。

**路径规格（pathspec）**

限制路径的匹配模式，语法为：

- 以`/`开头相对于系统根目录，不以`/`开头相对于当前目录。
- 可以使用`*`、`?`、`[list]`、`[!list]`、`[^list]`等通配符，而且`*`和`?`可以匹配目录分隔符`/`。如`Documentation/*.jpg`匹配Documentation/figure.jpg，也匹配Documentation/chapter_1/figure_1.jpg。

以`:`开头的路径规格具有特殊含义，有两种格式。短格式为`:`后跟零个或多个*魔术签名*，可选的一个`:`终止，其余部分与路径匹配的模式。长格式为`:`后跟一个`(`，以`,`分隔的零个或多个*魔术词*，以及紧接着一个`)`，其余部分与路径匹配的模式。*魔术词*有如下这些：

- **top**

  对应*魔术签名*为`/`，使匹配模式相对于工作树根目录。

- **literal**

  使通配符当作普通字符处理，如`*.c`只会匹配文件名是*.c的文件。

- **icase**

  不区分大小写。

- **glob**

  `*`表示匹配除`/`以外的零个或多个任意字符，`?`表示只匹配除`/`以外的一个任意字符，`[abc]`表示匹配任何一个列在方括号中的字符，`[0-9]`表示匹配所有在这两个字符范围内的字符，`[!abc]`或`[^a-c]`表示匹配不在方括号范围内的字符，`**`表示匹配任意中间目录。如`Documentation/*.html`匹配Documentation/git.html，但不匹配Documentation/ppc/ppc.html或tools/perf/Documentation/perf.html。

- **attr**

  匹配以空格` `分隔的零个或多个属性要求，必须满足所有的属性要求才能将路径视为匹配，其写法为`:(attr:binary -text)src/*`。属性要求有：`ATTR`表示ATTR属性被设置为true；`-ATTR`表示ATTR属性被设置为false；`ATTR=VALUE`表示ATTR属性被设置为VALUE；`!ATTR`表示未声明ATTR属性。

- **exclude**

  对应*魔术签名*为`！`或`^`，排除匹配的路径。

## Git仓库结构

**config**

此存储库的配置文件。

**description**

记录此存储库的描述信息，仅供GitWeb程序使用。

**hooks/**

包含客户端或服务端的钩子脚本，能在特定的重要动作发生时触发。示例钩子脚步删除`.sample`后缀即可启用。

**index**

保存暂存区信息。

**info/**

记录有关此存储库其他信息的目录。

**info/attributes**

全局性路径属性文件，用以放置那些不希望被记录在`.gitattributes`文件中的路径属性模式。

**info/exclude**

全局性排除文件，用以放置那些不希望被记录在`.gitignore`文件中的忽略模式。

**logs/**

记录本地对引用所做更改的日志目录，目录结构与`refs/`目录类似。

**objects/**

与此存储库关联的对象库目录。

**objects/\[0-9a-f][0-9a-f]/**

存储**松散（loose）**对象的目录。其将头部信息（header）和待存储的数据一起做SHA-1校验运算得到一个40个字符的哈希值，哈希值的前2个字符用于命名子目录，余下的38个字符则用作文件名。文件内容为头部信息（header）和待存储的数据通过zlib压缩后的值。

对象类型有：**数据对象（blob object）**、**树对象（tree object）**、**提交对象（commit object）**和**标签对象（tag object）**。**数据对象**包含文件的内容。**树对象**包含一个或多个数据对象或子树对象的哈希值，以及相应的模式、类型、文件名信息，树对象相当于目录。**提交对象**包含作者、提交者、日期、注释、父提交对象和代表当前项目快照顶层树对象。**标签对象**包含创建者、日期、注释和指向任意类型对象的哈希值，可对任意类型的Git对象打标签，通常指向提交对象。

**objects/info/**

记录有关此对象库其他信息的目录。

**objects/pack/**

存储**包文件（packfile）**和相应的**索引文件**的目录。**包文件**包含许多以压缩形式打包的对象。 **索引文件**包含了包文件的偏移信息，可以快速定位任意一个指定对象。

**packed-refs**

以更友好和高效的方式记录了与`refs/`目录相同的引用信息，一般为一行表示一个引用，对于附注标签引用，其下一行以`^`开头，`^`所在的那一行是附注标签指向的那个对象。

如果更新一个引用，Git并不会修改这个文件，而是在`refs/`目录下修改或创建对应的引用文件。查找引用时，首先在`refs/`目录中查找指定的引用，然后再到`packed-refs`文件中查找。

**refs/**

存储**引用（references）**的目录。**引用**文件一般包含一个哈希值，指向提交对象的指针。另一种方式为**符号引用（symbolic reference）**，它是一个指向其他引用的指针，格式为`ref: refs/some/thing`。

**refs/heads/**

存储本地引用的目录。**本地引用**指向本地分支提交之首，对于本地分支。

**refs/remotes/**

存储远程引用的目录。**远程引用**记录最近一次与服务器通信时远程版本库分支所对应的哈希值，对应于远程跟踪分支。永远不能通过`commit`命令来更新远程引用。

**refs/tags/**

存储标签引用的目录。**标签引用**永远指向一个固定的对象，可看作给这个对象加上一个更友好的名字。标签类型有：**轻量标签（lightweight tag）**和**附注标签（annotated tag）**。**轻量标签**简单的指向一个非标签对象，**附注标签**指向一个标签对象。

**HEAD**

指向目前所在分支的引用文件。通常是符号引用，指向`refs/heads/`目录下的分支引用文件。分离的HEAD则直接指向提交对象。

**ORIG_HEAD**

当使用命令（如`git reset`）强制移动HEAD时创建，记录操作之前HEAD指向的位置，方便回退。

**FETCH_HEAD**

记录上次调用`git fetch`时，从远程仓库中获取的分支信息。

**MERGE_HEAD**

合并操作时，记录合并源分支的指向的位置。该文件存在一般表示处于合并状态。

# 配置文件

## config

包含控制Git外观和行为的配置变量。文件可位于以下位置：

1. `/etc/gitconfig` 

   包含系统上每一个用户及他们仓库的通用配置。可以使用`git config`的`--system`选项读写此文件配置变量。

2. `~/.config/git/config`

   只针对当前用户。可以使用`git config`的`--global`选项读写此文件配置变量。

3. `~/.gitconfig`

   同上，但优先级更高。

4. `.git/config`

   只针对该Git仓库。

每一个级别覆盖上一级别的配置，所以`.git/config`的配置变量会覆盖`/etc/gitconfig`中的配置变量。

**remote**

```ini
[remote "origin"]
	url = https://github.com/schacon/simplegit-progit
	fetch = +refs/heads/*:refs/remotes/origin/*
	push = refs/heads/master:refs/heads/qa/master
```

`origin`表示远程仓库的别名。`url`表示远程仓库的地址。`fetch`表示抓取的引用规格，可参考[名词解释](#名词解释)中的*引用规格（refspec）*。`push`表示推送的引用规格。

## gitignore

指定要忽略的文件，使其不被Git跟踪，一般使用存储库中的`.gitignore`文件进行配置，配置文件每一行都表示一个匹配模式。应用匹配模式时，会读取当前目录和其父目录的`.gitignore`文件，父目录的配置会被子目录的配置覆盖。格式规范如下：

- 所有空行或者以`＃`开头的行都不会匹配任何文件或目录。
- 结尾的空格不会识别，除非使用`\ `。
- 以`!`开头排除该模式，即重新包含匹配的的文件。如果父目录被忽略，则无法重新包含该目录下的文件。
- 以`/`开头只匹配以当前配置文件所在目录开始的路径。如`/*.c`匹配cat-file.c，但不匹配mozilla-sha1/sha1.c。
- 以`/`结尾只匹配目录，而不匹配文件或符号链接。
- 除结尾之外没有`/`会递归查找子目录。如`*.c`匹配cat-file.c，也匹配mozilla-sha1/sha1.c。
- 其余使用glob模式匹配，可参考[名词解释](#名词解释)中*路径规格（pathspec）*的glob模式。

# Git命令

当参数与选项出现混淆时，可以在参数前加`--`进行区分。

## 获取与创建

### init

```shell
git init [<options>…] [directory]
```

将一个目录初始化成一个Git仓库。该命令将创建`.git`子目录，这个子目录含有Git仓库中所有的必须文件。

### clone

```shell
git clone [<options>…] [--] <url> [<directory>]
```

从服务器克隆一个现有的Git仓库。该命令将创建了一个新目录`<directory>`（默认与远程仓库同名），切换到新的目录，然后执行`git init`来初始化一个空的Git仓库， 执行`git remote add`来使用指定的`<url>`添加一个远程仓库（默认别名为`origin`），再针对远程仓库执行`git fetch`抓取所有信息，最后通过`git checkout`创建一个与远程仓库默认分支同名的跟踪分支，将跟踪分支最新提交检出到本地的工作目录。

`--bare`

通过克隆来创建一个裸仓库，即一个不包含当前工作目录的仓库。

## 基本快照

### add

```shell
git add [<options>…] [--] [<pathspec>…]
```

用来开始跟踪新文件，或者把已跟踪的文件放到暂存区，或者合并时把有冲突的文件标记为已解决状态等。理解为“添加内容到下一次提交中”而不是“将一个文件添加到项目中”。

### status

```shell
git status [<options>…] [--] [<pathspec>…]
```

显示工作树状态。

`-s`, `--short`

紧凑格式输出。每个文件前面有两个字母标记，对于没有合并冲突，左边字母表示该文件在暂存区中的状态，右边字母表示该文件在工作树中的状态；对于存在合并冲突，左边字母表示该文件在本地的状态，右边字母表示该文件在对方的状态。` `表示未经修改，`M`表示已修改，`A`表示已添加，`D`表示已删除，`R`表示重命名，`C`表示已复制，`U`表示已更新未合并，`??`表示未跟踪，`!!`表示忽略。

### commit

```shell
git commit [<options>…] [--] [<file>…]
```

提交更改到Git仓库。默认会启动文本编辑器以便输入本次提交的说明。

`-m <msg>`, `—message=<msg>`

指定提交信息。如果有多个`-m`将会作为单独段落拼接。

`-a`, `--all`

自动把所有已经跟踪过的文件暂存起来一并提交，从而跳过`git add`步骤。

`--amend`

通过创建新提交替换当前分支的最近一次提交。

### rm

```shell
git rm [<options>…] [--] <file>…
```

删除已跟踪到文件。默认从暂存区和工作树中都删除文件。

`-f`, `--force`

强制删除，针对删除之前修改过并且已经放到暂存区的文件。

`--cached`

仅从暂存区删除文件，不删除工作树中的文件。

### mv

```shell
git mv [<options>…] <source> <destination>
git mv [<options>…] <source> ... <destination directory>
```

移动或重命名文件。等价于`git rm`之后`git add`。

## 检查与比较

### show

```shell
git show [<options>…] [<object>…]
```

显示各种类型的对象。对于提交对象，显示日志消息和文本差异；对于标签对象，显示标签消息和引用的对象；对于树对象，显示包含的文件和目录名。对于数据对象，显示具体内容。

### log

```shell
git log [<options>…] [<revision range>] [[--] <path>…]
```

显示提交日志。默认不用任何参数的话，会按提交时间由近到远列出每个提交的提交对象哈希值、作者的名字和电子邮箱地址、提交时间以及提交说明。`<revision range>`特殊情况可以使用`A..B`限制从B可以访问，但不能从A访问的所有提交。

`-p`, `-u`, `--patch`

按补丁格式显示每个提交之间的差异。

`--stat`

显示每次提交的文件修改统计信息，包括所有被修改过的文件、有多少文件被修改了以及被修改过的文件的哪些行被移除或是添加了等。

`--shortstat`

只显示`--stat`中最后的行数修改添加移除统计，包含已修改文件的总数以及已添加和已删除行数。

`--name-only`

仅在提交信息后显示已修改的文件名称。

`--name-status`

仅在提交信息后显示已修改的文件名称和状态。

`--abbrev-commit`

仅显示哈希值的前几个字符，而非所有的 40 个字符。

`--relative-date`

使用较短的相对时间显示（比如“2 weeks ago”）。

`--graph`

显示ASCII图形表示的分支合并历史。

`--pretty`

使用其他格式显示历史提交信息。可用的选项包括`oneline`、`short`、`full`、`fuller`和`format`。其中`format`可定制显示格式，如`--pretty=format:"%h - %an, %ar : %s"`，`format`常用的选项有：`%H`显示提交对象的完整哈希值，`%h`显示提交对象的简短哈希值，`%T`显示树对象的完整哈希值，`%t`显示树对象的简短哈希值，`%P`显示父对象的完整哈希值，`%p`显示父对象的简短哈希值，`%an`显示作者的名字，`%ae`显示作者的电子邮箱地址，`%ad`显示作者修订日期（可以用`--date=`选项定制格式），`%ar`按多久以前的方式显示作者修订日期，`%cn`显示提交者的名字，`%ce`显示提交者的电子邮件地址，`%cd`显示提交日期，`%cr`按多久以前的方式显示提交日期，`%s`显示提交说明。

`--oneline`

等价于`--pretty=oneline --abbrev-commit`。

`--decorate[=short|full|auto]`

显示指向每个提交的引用对象名称。`short`不显示`refs/heads/`、`refs/remotes/`和`refs/tags/`前缀。`full`显示完整的名称。`auto`输出到终端时使用`short`配置，否则不输出引用对象名称。默认选项是`short`。

`--all`

显示`refs/`目录下所有引用能访问到的提交对象。

`-<number>`, `-n <number>`, `--max-count=<number>`

仅显示最近的`<number>`条提交。

`--since=<date>`, `--after=<date>`

仅显示指定时间之后的提交。

`--until=<date>`, `--before=<date>`

仅显示指定时间之前的提交。

`--author=<pattern>`

仅显示指定作者相关的提交。

`--committer=<pattern>`

仅显示指定提交者相关的提交。

`--grep=<pattern>`

仅显示提交说明中含指定关键字的提交。当有多个`--grep=<pattern>`时，默认满足任何一个条件就匹配，使用`--all-match`则要求满足所有条件才匹配。

`-S<string>`

仅显示文件中添加或移除了某个关键字的提交。

`-g`, `--walk-reflogs`

显示引用日志。

### diff

```shell
git diff [<options>…] [<commit>] [--] [<path>…]
git diff [<options>…] <commit> <commit> [--] [<path>…]
git diff [<options>…] <blob> <blob>
```

显示文件发生的变化。第一种格式默认比较的是工作树中当前文件和Git仓库指定提交对象`<commit>`之间的差异。如果省略`<commit>`，则比较工作树和暂存区域快照之间的差异， 也就是修改之后还没有暂存起来的变化内容。第二种格式用于比较Git仓库中任意两个提交对象`<commit>`之间的差异，右边的`<commit>`相对于左边的`<commit>`的差异。两个`<commit>`之间插入两点`<commit>..<commit>`与空格同作用，省略一侧的`<commit>`与使用`HEAD`具有相同的效果。两个`<commit>`之间插入三点`<commit>...<commit>`用于比较两个`<commit>`最近的共同祖先和第二个`<commit>`之间的差异，如`git diff A...B`相当于`git diff $(git merge-base A B) B`。第三种格式用于比较Git仓库中任意两个数据对象`<blob>`之间的差异。

`--staged`, `--cached`

只针对第一种格式，比较的是暂存区域快照和Git仓库指定提交`<commit>`（默认是`HEAD`，即最新提交）之间的差异，已暂存的将要添加到下次提交里的内容。

`--check`

只针对第一种格式，检查提交中是否包含冲突标记或空白错误。空白错误是指行尾的空格、Tab制表符，行首空格后跟 Tab制表符的行为。

### shortlog

```
git shortlog [<options>…] [<revision range>] [[--] <path>…]
```

归纳`git log`的输出，显示一个根据作者分组的提交记录的概括性信息，很多选项与`git log`相同。

### describe

```shell
git describe [<options>…] [<commit-ish>…]
```

生成一个人类可读的字符串，由最近的标签名、自该标签之后的提交数目和`<commit-ish>`的部分哈希值构成，如`v1.6.2-20-g8c5b85c`。如果`<commit-ish>`自身就有一个标签，那么将只会输出标签名，没有后面两项信息。默认情况下，只显示最近附注标签，没有附注标签将报错。

## 分支与合并

### branch

```shell
git branch [<options>…] [<pattern>…]
git branch [<options>…] <branchname> [<start-point>]
git branch [<options>…] [<branchname>]
git branch [<options>…] <branchname>…
```

管理分支。第一种格式用于列出已有的分支，默认只列出本地分支。第二种格式用于创建一个指向`HEAD`或提交对象`<start_point>`的本地分支`<branchname>`，未指定`<start-point>`时默认是`HEAD`。`<start-point>`特殊情况可以使用`A...B`作为A和B最近的共同祖先。第三种格式用于修改分支信息，默认是当前分支。第四种格式用于删除本地分支。

`-l [<pattern>…]`, `--list [<pattern>…]`

只针对第一种格式，显示匹配指定模式的分支。模式可以使用shell通配符。

`-v`

只针对第一种格式，显示每个分支的最后一次提交信息，以及本地分支相对与上游分支是否领先（ahead）或落后（behind）。

`-vv`

只针对第一种格式，在`-v`的基础上显示每个分支对应的上游分支。

`--merged [<commit>]`

只针对第一种格式，仅显示分支的最新提交对象可从指定提交对象`<commit>`（默认是`HEAD`）访问到的分支，即已经合并到提交对象`<commit>`所在分支的分支。

`--no-merged [<commit>]`

只针对第一种格式，与`--merged [<commit>]`相反。

`-u <upstream>`, `--set-upstream-to=<upstream>`

设置分支的上游分支。

`-d`, `--delete`

只针对第四种格式，删除指定的本地分支。要求该分支已经全部合并到对应的上游分支，如果没有上游分支则已经全部合并到`HEAD`。

`-D`

只针对第四种格式，强制删除指定的本地分支，相当于`--delete --force`。

### checkout

```shell
git checkout [<options>…] [<branch>]
git checkout [<options>…] [<start_point>]
```

切换分支或检出内容到工作目录。第一种格式用于切换分支，如果本地分支没有`<branch>`，但远程跟踪分支有，则等价于`git checkout -b <branch> --track <remote>/<branch>`。第二种格式用于创建一个指向提交对象`<start_point>`的本地分支并切换到该分支。

`-t`, `--track`

只针对第二种格式，创建一个上游分支是`<start_point>`的跟踪分支。如果没有`-b <branchname>`选项，则跟踪分支的名称与上游分支同名。

`-b <branchname>`

只针对第二种格式，相当于使用`git branch`创建一个名称为`<branchname>`本地分支，然后切换到该分支。如果`<start_point>`是远程跟踪分支，则设置新创建的本地分支跟踪`<start_point>`。

### merge

```shell
git merge [<options>…] [<commit>…]
```

合并一个或多个分支到当前的分支中。

`--no-commit`

执行合并，并在创建合并提交之前停止。此时暂存区是合并状态。

`--squash`

接受合并源分支上的所有提交，更新工作树和暂存区，但不会记录`MERGE_HEAD`，也不会进行合并提交。暂存区是合并状态这意味着可以做更多的改动，并将合并作为一个新的普通提交，该提交将只有一个父提交。

### tag

```shell
git tag [<options>…]
git tag [<options>…] <tagname> [<commit> | <object>]
git tag [<options>…] <tagname>…
```

管理标签对象。第一种格式用于列出已有的标签。第二种格式用于给Git仓库历史记录中的某个点指定一个永久的标签，未指定`<commit>`或`<object>`时默认是`HEAD`。第三种格式用于删除或校验标签。

`-l [<pattern>…]`, `--list [<pattern>…]`

只针对第一种格式，显示匹配指定模式的标签。模式可以使用shell通配符。

`-a`, `--annotate`

只针对第二种格式，创建一个附注标签。

`-m <msg>`, `--message=<msg>`

只针对第二种格式，指定标签信息。如果有多个`-m`将会作为单独段落拼接。

`-d`, `--delete`

只针对第三种格式，删除本地仓库上的标签。

## 分享与更新

### fetch

```shell
git fetch [<options>…] [<repository> [<refspec>…]]
```

将远程仓库中有但是在本地仓库的没有的所有信息抓取下来，存储在本地仓库中，然后更新远程仓库引用。该命令并不会自动合并或修改当前的分支。

### pull

```shell
git pull [<options>…] [<repository> [<refspec>…]]
```

从远程仓库抓取数据并自动尝试合并到当前所在的分支，大多数情况相当于`git fetch`和`git merge`命令的组合体。

### push

```
git push [<options>…] [<repository> [<refspec>…]]
```

计算本地仓库与远程仓库的差异，然后将差异推送到远程仓库。只有当你有远程仓库的写入权限，并且之前没有人推送过时，该命令才能生效。当有人已经推送过时，必须先将他们的工作抓取下来并将其合并进你的工作后才能推送。

`-u`, `--set-upstream`

为成功推送的分支设置上游分支。一般用于本地新分支推送到远程仓库的时候。

`--tags`

把所有的本地标签推送到远程仓库。默认情况下，`git push`命令并不会传送标签到远程仓库服务器上。

`-d`, `--delete`

从远程仓库中删除指定的引用。

### remote

```shell
git remote [<options>…]
git remote add [<options>…] <name> <url>
git remote rename <old> <new>
git remote (remove | rm) <name>
git remote show [<options>…] <name>…
git remote set-head <name> (-a | --auto | -d | --delete | <branch>)
git remote set-branches [<options>…] <name> <branch>…
git remote get-url [<options>…] <name>
git remote set-url [<options>…] <name> <newurl> [<oldurl>]
git remote prune [<options>…] <name>…
git remote update [<options>…] [(<group> | <remote>)…]
```

管理远程仓库。默认命令列出每一个远程仓库的别名。`add`子命令添加一个别名为`<name>`，位于`<url>`的远程仓库。`rename`子命令修改远程仓库`<old>`的别名为`<new>`，并修改所有远程引用。`remote`或`rm`子命令从本地移除远程仓库`<name>`，并移除所有远程引用。`show`子命令显示远程仓库`<name>`的信息，包括读写远程仓库的地址、当前所处的分支、远程仓库的所有分支、`git pull`会拉取的所有远程分支、`git push`会推送的所有远程分支。

`-v`, `--verbose`

针对默认命令显示需要读写远程仓库的别名与其对应的地址。

## 管理

### reflog

```shell
git reflog [show] [<options>…] [<ref>]
```

管理引用日志。show子命令也是没有任何子命令时的默认命令，显示本地对引用`<ref>`的头指针所做更改的日志，默认是`HEAD`。

# 常用操作

**生成SSH公钥**

```shell
# 邮箱地址只是作为注释，没有实际意义
ssh-keygen -t rsa -b 4096 -m PEM -C "your_email@example.com"
# 确认密钥的存储位置（默认是~/.ssh/id_rsa）
# 输入两次密钥口令，不想设置密钥口令则将其留空
# .ssh目录权限是700，id_rsa文件权限是600，id_rsa.pub文件权限是644
```

**取消暂存的文件**

```shell
git reset HEAD file.txt
```

**撤消对文件的修改**

```shell
git checkout -- file.txt
```

**创建并切换一个跟踪分支**

```shell
# 本地分支与远程分支同名
git checkout --track origin/serverfix
```

或

```shell
# 本地分支与远程分支不同名
git checkout -b sf origin/serverfix
```

