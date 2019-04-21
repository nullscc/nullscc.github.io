---
layout: post
title: 'vim打造python ide，只看这一篇就够'
tags: vim python
---
由于日常用macbook开发，也不用鼠标，工作内容是全python环境，用pycharm一天下来手酸的很，刚好同事提到了说之前有个家伙用vim看linux内核，才猛然想起自己大学时候也玩过这玩意，无奈大学时也没编多少码所以不了了之。

想了一下发现vim还挺适合我现在的工作模式，简直一拍即合，花了几天业余时间看了些教程和必要的快捷键，磕磕碰碰的把环境搭起来了，然后在公司以龟速的工作效率适应了两三天之后pycharm已经被我放角落里了。一天下来除了看资料也用不到触控板，怎一个爽字了得。

网上的绝大部分资料都不能开箱即用，不是太老就是不适合python，要不就是只有零零散散的，这篇文章的宗旨就是看完这篇配置就能完全搭起来一个自己日常使用的的python开发ide，而不是一个装逼货。当然个人用户习惯不一样，你可以根据自己的情况来配置。

本篇只是教你怎么配置(理论上debian系可以直接跟着命令敲)，而不是教你怎么使用，如果有需要使用教程的可以email给我，我尽快写一个使用教程(如果你是一个使用vim的人，估计也不需要使用教程，前段时间垃圾留言太多了，也没空理它，索性把评论功能去掉了～)

## 环境
全新安装的linux mint操作系统
```bash
	# lsb_release -a

	Distributor ID: LinuxMint
	Description:    Linux Mint 18.3 Sylvia
	Release:        18.3
	Codename:       sylvia
```

开始这些操作之前请确保你有一定的linux、vim、tmux使用基础，或者你有学习欲望，后面可能会再开一篇贴上必要掌握的的tmux和vim快捷键(也是我自己现在每天工作都在用的)，基本上你只需要掌握贴出的这些指令就可以很愉快的使用vim作为python ide进行日常开发了。

## 安装vim

### 准备工作
首先看你的系统中是否有安装首先看系统自带的vim是否满足要求，如果不满足要求，则需要卸载掉重新安装。怎么看是否满足要求？
```bash
	vim --version | grep python
	vim --version | grep clipboard
```
确保查到的对应的模块前面有`+`号，例如：
```bash
	+python
	+clipboard
```
如果没有，则不满足要求，`+python`是用来支持python的，`+clipboard`是用来支持系统剪切板的，相信我，不支持系统剪切板的vim是很难用的，由于它需要一些其他的依赖，所以通过包管理工具(例如yum、apt等)安装的vim一般是没有`+clipboard`的支持的

先看看机器上有没有vim：
```bash
	dpkg -l | grep vim
```
如果你的机器上已经有vim了，但是不满足要求，linux mint(ubuntu也适用)下卸载掉它的命令:
```bash
	sudo apt-get remove vim
	sudo apt-get purge --auto-remove vim-*
```
卸载掉后再通过`dpkg -l | grep vim`命令查看，就没有了

准备装软件咯，全新安装的linux发行版嘛，第一件事就是换软件源或者更新源，这里没必要换源(如果你源嫌速度慢，那也可以换，或者开代理吧～)，所以更新源吧～
```bash
	sudo apt-get update
```
由于需要git下载源码，所以先把git安装了
```bash
	sudo apt-get install git
```
参见我的[这篇文章](http://nulls.cc/2017/09/04/use_python2_python3_under_linux/)安装好python2与python3 
linux mint默认安装了python2.7和python3.5

### 安装vim的依赖包
vim主要有以下依赖包，`#`后注释了它是用来干嘛的，如果发现太慢，自备代理(也可以email找我要个临时的)或换源
```bash
	sudo apt-get install gcc g++						# gcc编译器，源码安装必备的东东了
	sudo apt-get install xorg-dev						# debian系发行版vim与系统自带剪切板交互必备
	sudo apt-get install libncurses5-dev				# vim 本身的编译依赖
	sudo apt install ack-grep							# 用来支持vim下ack全文搜索
	sudo apt-get install build-essential cmake3/cmake	# 用来支持YouCompleteMe插件(这里使用cmake，较老的系统(老于ubuntu14.04可以使用cmake3))
	sudo apt-get install python-dev python3-dev			# vim的python2、python3支持依赖
```
### 源码编译安装vim
由于一个vim无法同时加载python2与python3，动态加载貌似也不能实现，所以安装vim需要安装两个版本，这里先安装支持python2的vim

1. 下载源码
```bash
		git clone https://github.com/vim/vim.git
```
2. 配置并编译安装

先确认下下面要配置的`--with-python-config-dir=/usr/lib/python2.7/config-x86_64-linux-gnu`对应的python config目录是否存在对应的文件(如果没有，想办法找到你自己的):
```bash
	null@null ~/vim $ ls /usr/lib/python2.7/config-x86_64-linux-gnu
	config.c  config.c.in  install-sh  libpython2.7.a  libpython2.7-pic.a  libpython2.7.so  Makefile  makesetup  python.o  Setup  Setup.config  Setup.local
```
这里先安装支持python2.7的vim
```bash
	cd vim
	./configure --with-features=huge --enable-cscope --enable-fontset  --enable-pythoninterp=yes --with-python-config-dir=/usr/lib/python2.7/config-x86_64-linux-gnu --bindir=/usr/bin --datarootdir=/usr/share
	make
	sudo make install	
```
解释上面`configure`命令参数的意义:

1. `--with-features=huge`，启用系统粘贴板(debian系确认`apt-get install xorg-dev`了)
2. `--enable-pythoninterp=yes`，开启python2支持(确认`apt-get install python-dev`了)
3. `--with-python-config-dir=/usr/lib/python2.7/config-x86_64-linux-gnu`，对应python2的config目录
4. `--bindir=/usr/bin`，vim要安装到哪里去
5. `--datarootdir=/usr/share`，vim的周边支持要安装到哪里去，比如库文件、man手册

这时支持python2的vim已经搞定了，看看是否满足使用要求吧:

1. `vim --version | grep clipboard`返回`+clipboard`
2. `vim --version | grep python`返回`+python`，并且`-python3`

如果满足要求了，就配置吧，没满足？仔细看看跟着做没有哟？(debian系跟着做就好了，其他的发行版可以搜索对应的安装包)

### 配置vim
接下来的操作和你的网络相关，如果你执行`git clone`慢如龟速，那么劝你先想好怎么解决这个问题先～，不知道怎么解决也可以email我～

先安装vim的包管理工具，这里使用`Bundle`
```bash
	git clone http://github.com/gmarik/vundle.git ~/.vim/bundle/Vundle.vim
```
然后下载我的配置文件(如果你已经有自己的`.vimrc`，记得先备份):
```bash
	wget https://gist.githubusercontent.com/nullscc/86240b01898a178525f80533ff84610f/raw/8e070baff4ac5abcf8a10f748dfe7bdc9aeefb36/.vimrc ~/.vimrc
```
然后输入在终端输入`vim`(对三个字母就够了)，启动后(会报一大堆错误，暂时直接回车进入即可)按英文字母的冒号(`:`)输入如下命令：
```bash
	PluginInstall
```
如果你的网络ok的话应该要不了多久。

#### 编译YouCompleteMe
然后你再次打开vim，会发现下方会提示一个红色的错误
```bash
	YouCompleteMe unavailable: No module named builtins
```
这是因为YouCompleteMe和其他插件不同，它是一个需要编译的插件，接下来我们编译它(确保文章前的依赖已经安装完成)
```bash
	cd /home/null/.vim/bundle/YouCompleteMe
	git submodule update --init --recursive
 	python install.py
	
	# python ./install.py --clang-completer  # 如果你的vim需要c语言跳转支持则加上--clang-completer
```
编译完成后vim配置就算完成了

有些童鞋可能打开后会发现下方又报了个错：
```bash
	Tagbar: Exuberant ctags not found!
```
这是因为`tagbar`是依赖于`ctags`的，所以我们需要安装`ctags`，去[http://ctags.sourceforge.net/](http://ctags.sourceforge.net/)下载ctags源码包，并编译安装即可
```bash
	wget https://nchc.dl.sourceforge.net/project/ctags/ctags/5.8/ctags-5.8.tar.gz
	tar xzfv ctags-5.8.tar.gz
	cd ctags-5.8/
	./configure
	make
	sudo make install
```

## tmux 与tmuxinator 安装与配置
单单有vim可能还不是太爽，你还是免不了要用鼠标(或触控板)疯狂乱划，tmux能很好的解决这个问题，tmux + vim 能让你在代码编辑、编译(python并无编译～)、调试的全程都不用离开鼠标

### tmux安装
安装tmux很简单(先别急着执行下面的命令～)
```bash
	sudo apt-get install tmux
```
执行完上面的命令后tmux的确会安装了，但是这个tmux版本不满足要求，我们需要tmux版本大于2.2的，不然后面tmux会和vim的颜色相冲突，所以我们采用源码安装吧

如果你已经安装了tmux，那么先卸载它:
```bash
	sudo apt-get purge --auto-remove tmux
```
去tmux官网我们会发现tmux依赖于`libevent`和`ncurses`，`ncurses`我们已经安装过了，先安装`libevent`，`libevent`官网的源码压缩包貌似不能直接用wget下载，所以我们去github上下吧～
```bash
	git clone https://github.com/nmathewson/Libevent.git
	cd Libevent/
	sudo apt install libtool
	sudo apt install libsysfs-dev
	./autogen.sh
	./configure
	make
	sudo make install
```
然后下载并编译安装tmux(我这里为了演示是从github上下载源码，你最好直接从官网下载)
```bash
	git clone https://github.com/tmux/tmux.git
	cd tmux/
	./autogen.sh
	./configure
	make
	sudo make install
```
完成后可能有的童鞋会报如下错误：
```bash
	null@null ~ $ tmux -V
	tmux: error while loading shared libraries: libevent-2.2.so.1: cannot open shared object file: No such file or directory
```
这种情况就要看看是确实不存在这个文件还是路径不对，这里用如下命令是能找到相应的so文件的(因为之前已经安装过了):
```bash
	sudo find -name / "libevent*so*"
```
然后创建一个软链接指向它即可：
```bash
	sudo ln -s /usr/local/lib/libevent-2.2.so.1 /usr/lib/libevent-2.2.so.1
```
这时候已经能使用`tmux -V`查看版本号了，不过貌似在github上下载的tmux会展示成：`tmux master`，不过你知道自己的是最新的就好啦，或者去官网下载压缩包编译安装

嗯，这个时候已经能启动tmux啦。

### tmux配置
tmux的配置很简单，这里最重要的是需要更改它的前缀键:`C-b`，因为它和vim本身的翻页按键冲突，我这里改成`C-n`，将以下内容粘贴到`～/.tmux.conf`中(没有就新建一个)：
```bash
	unbind C-b
	set -g prefix C-n

	setw -g mode-keys vi
	set-option -ga terminal-overrides ",xterm-256color:Tc"
	set -g default-terminal "xterm-256color"
	set-option -g default-command "reattach-to-user-namespace -l bash"

	bind -r ^k resizep -U 5
	bind -r ^j resizep -D 5
	bind -r ^h resizep -L 5
	bind -r ^l resizep -R 5
	
	bind h select-pane -L
	bind j select-pane -D
	bind k select-pane -U
	bind l select-pane -R
```
这里的配置分为4部分：

1. 把tmux的前缀按键改为`C-n`
2. 让vim的主题配色和tmux不冲突
3. `C-n` + C-h/j/k/l 可以调整tmux的窗格大小
4. `C-n` + h/j/k/l 可以聚焦到不同分隔的窗格

其中为了防止tmux和vim颜色会冲突可能还要在`~./bashrc`下增加(如果没问题就不用加了)：
```bash
	if [ "$TERM" = "xterm" ]; then
	  export TERM=xterm-256color
	fi
	alias tmux='tmux -2'  # for 256color
	alias tmux='tmux -u' 
```
到这里其实已经算是配置完成了，这时候vim已经算是一个ide了，但是有时候我们的工作模式可能是这种：  
tmux开启多个session，每个session对应一个项目工程(我现在的模式就是这样的)，但是我们有时候要运行可能要在这个session里面创建多个窗格，这样看起来就更像一个ide了。

但是这样的话，每次我们需要新打开一个工程都需要在每个session里面慢慢创建窗格，这样实在太麻烦，我们需要一个命令就能打开工程开干的东西，tmux的基佬伴侣`tmuxinator`就是干这个事儿的

### tmuxinator安装与配置
tmuxinator的安装使用gem最方便所以安装它需要两个命令才能搞定：
```bash
	sudo apt install gem
	sudo gem install tmuxinator
```
那么安装完怎么配置呢？很简单，一个命令搞定(这里以一个名为api的工程为例)
```bash
	tmuxinator open api
```
我这里首次运行这个命令会出现这样：
```bash
	null@null ~ $ tmuxinator open api
	sh: 1: /home/null/.config/tmuxinator/api.yml: Permission denied
	Checking if tmux is installed ==> Yes
	Checking if $EDITOR is set ==> No
	Checking if $SHELL is set ==> Yes
```
这里还要设置`EDITOR`的环境变量，因为要用到这个变量指定的编辑器来编辑配置文件，当然使用vim啦，把这个环境变量加入到`.bashrc`（当然你也可以放到全局的环境变量中）  
这里需要将以下内容加入到`~/.bashrc`中：
```bash
	export EDITOR=vim
```
然后:
```bash
	source ~/.bashrc
```
然后再执行：
```bash
	tmuxinator open api
```
tmuxinator会自动帮你创建一个默认的配置文件，我们只需要修改就好了，这里贴下我主要会修改的内容(下图的projectpath就是你的工程项目的根目录):
```bash
name: api
root: ~/projectpath

pre_window: . ~/Desktop/shein/env/bin/activate
windows:
  - api:
      layout: main-horizontal
      panes:
        - sleep 1 && vim config.py
        - 
```
这里由于我一般使用虚拟环境，所以每次打开session都激活虚拟环境(pre_window)

然后你可以执行以下命令打开刚刚创建的工程：
```bash
	null@null ~ $ tmuxinator start api
	    DEPRECATION: You are running tmuxinator with an unsupported version of tmux.
	    Please consider using a supported version:
	    (1.5, 1.6, 1.7, 1.8, 1.9, 2.0, 2.1, 2.2, 2.3, 2.4, 2.5, 2.6)
	
	Press ENTER to continue.
```
上面的错误是因为我是从github上下载的tmux源码包，如果你是从官网下载的源码包编译则不会这样，不过这个警告貌似也不影响使用

## 安装支持python3的vim
虽然安装支持python3的vim和之前的步骤一摸一样，但是这里还是说下把～
```bash
	cd vim
	make distclean
	./configure --with-features=huge --enable-cscope --enable-fontset  --enable-python3interp=yes --with-python-config-dir=/usr/lib/python3.5/config-3.5m-x86_64-linux-gnu/ --bindir=/usr/local/vim3 --datarootdir=/usr/local/share/vim3
	make
	sudo make install
	sudo ln -s /usr/local/vim3/vim /usr/bin/vim3
```
注意上面的configure后面的参数换成你自己的，参数的之前已经说了。这时候再运行`vim3 --version`就可以看到`+python3`的字样了。

其他的配置完全不用动，`.vimrc`和之前那个vim是共享的。在你需要的地方使用vim和vim3即可。

ok，到这里就算完全配置完了，有时间再写篇文章说说怎么使用的，其实如果vim的快捷键你比较熟悉的话，基本上只需要看看那些插件的使用很快就能上手了