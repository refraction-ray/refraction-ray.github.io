---
layout: post
title: "云上的 Jupyter Notebook"
date: 2018-02-08
excerpt: "Jupyter Notebook 在 web 端的一些剪影和利用"
tags: [machine learning, python, finance, web, data science]
comments: true
---
* toc 
{:toc}
*别怕，浏览器那个 hello 提醒框是我弹的，没有安全问题，请继续阅读。*

# 引言

知道 Ipython Notebook 差不多和开始接触 python 同一时间，交互式的设计和我适应的 mathematica 的 notebook 设计异曲同工。这种边做边试边反馈边记录的方式，无疑非常适合科研试验或数据处理分析可视化等任务，比传统的运行单文件的方式效率高了不少。

但是，凡事都有个但是，我以前从没将 notebook 作为主力用过。首先这种在浏览器里写代码的体验稍显诡异；其次，我似乎也没有这样的需求。原因在于，在 sublime text 安装了SublimeREPL 插件后，可以直接在 sublime text 里得到类似 ipython 的交互式体验（介于 terminal 和 ipython 的交互体验之间），大多数时候，学学函数和库，处理处理数据，这种程度交互也够用了，毕竟我复杂可视化还是更倾向 mathematica。

不过 notebook 天生和 web 契合的基因也使得这种形式无比容易分享和标准化（ipynb 其实就是一坨 json），特别对于基于云端的计算引擎或大数据资源的服务，这种 notebook 是相当合适的现成的接口。

Ipython notebook 早些时候已经升级为 Jupyter notebook，其目标已更加宏大，作了进一步的抽象，把前段界面接口和内核语言引擎分离，使得 notebook 现在可以支持多种语言，甚至可以说是大多数语言（详情见此[列表](https://github.com/jupyter/jupyter/wiki/Jupyter-kernels)）。

最近越来越多的云服务使用 ipython 作为默认的接口和交互方式，jupyter notebook 也重回我视线，本文记录了 jupyter notebook 与 web 相关的一些思考，折腾和资源。

# Web 搭建 notebook 服务

## 本地使用

怎么安装 Jupyter Notebook 这种事，就不废口舌了，conda 自带或者 pip 安装。关于多个版本 python 并存，虚拟环境，包管理等等可能出现的烂事，和主旨无关。假设你已经成功搞定 notebook 安装，那么作为服务器非常简单，在命令行输入 `jupyter notebook` 即可直接开启一个默认在`localhost:8888`的web服务，其默认的打开文件夹位置为命令行的 `pwd`。如果电脑有图形界面和默认浏览器的话，这一web界面会默认用浏览器自动打开。happy coding。

## 远程使用

### 配置文件

最朴素的方法，修改配置文件，将监听 ip 改一下，使得相应端口外网可以访问即可。简单来讲，先命令行生成 jupyter 的配置文件 `jupyter notebook --generate-config`，配置文件默认位置在 ~／.jupyter／jupyter_notebook_config.py。配置文件只需将以下几行去掉注释改一下即可，实际上只有第一行是必须的，把ip从默认的127.0.0.1改掉即可。

~~~bash
c.NotebookApp.ip = '*' # MUST
c.NotebookApp.open_browser = False # Optional, if the remote sever has browser 
c.NotebookApp.port = 8888 # Optional, if you wanna change the default listening port
c.NotebookApp.notebook_dir = "\path" # Optional, if you wanna change the working path of jupyter project
~~~

此时你已经可以从其他终端，用 ip 或者域名（如果你绑定了的话）来访问远程的 jupyter notebook了（当然前提是服务器 `jupyter notebook` 一下）。访问第一次会要求你输入 token，去服务器命令行 `jupyter notebook list` 一下，在输出运行的 jupyter 的url里就找到对应的 token 值。之后你就可以在前端 web 界面输入 token 并设置密码，此后就可以通过密码登陆了。

### 安全保证

不过，在直接按上一段的做法远程连接 notebook 之前，最好先注意下安全问题。刚才的方式直接通过 http 通讯，token 密码 cookie 等信息可以轻易被中间人截获，从而使得其他人拥有进入 notebook 的权限。所以为了安全需要套上一层加密，最容易想到的就是本来的 http 下面垫上 TLS，**https** 的设置这里就不多讲了，免费的话关键字 Let's Encrypt，记得在配置文件里加上以下两行，指定https证书密钥的位置。

~~~bash
c.NotebookApp.certfile = u'/absolute/path/to/your/certificate/mycert.pem'
c.NotebookApp.keyfile = u'/absolute/path/to/your/certificate/mykey.key'`
~~~

当然 https 的安全方式也有一些缺点。比如配置 https 需要花些时间，有些服务器没有绑定域名只有 IP 导致获得 https 证书困难等等。所以在推荐另外两种加密与服务器间通讯的方式。一种是利用 ssh 的端口映射功能，在本地的 terminal 使用如下命令,

~~~bash
$  ssh -N -f -L localhost:8888:localhost:9999 user@server.ip -p 22
~~~

该命令通过22端口的**ssh隧道**将本地8888端口映射到服务器的9999端口，假设服务器端的 jupyter notebook 建立在9999端口，本地只需要浏览器访问 `localhost:8888`即可访问到notebook，同时访问的数据都走了加密的ssh 隧道。相同原理曾经在11年作为科学上网的神器来用，不用试了，现在这条路上网基本是废了。这种方式都不需要配置 jupyter notebook 监听IP范围，服务器本地运行即可端口转发实现外网访问。

说到了怎么正常上网，就带出了第二种 https 的替代加密方案，**socks5加密代理**，那些常用的shadowsocks，v2ray之类的，都可以。只需在部署 jupyter 相同的服务器上部署一个 socks5 加密代理的服务端，并将对应 IP 列入客户端 pac 文件，那么本地访问时，信息则加密转发到服务器，而服务器上的代理会根据 IP （其实就是对应本机）再访问 notebook 资源，如此流量数据在网络通讯中也被加密。这种方案适合服务器本来就部署有 socks5 代理的使用，只需将服务器本身 IP 加入代理列表，即可保证 notebook 访问的安全。

### 服务部署

既然是借助web框架的web服务，那么可以把web框架服务器部署运维那一套都拿过来用。只要你愿意，你甚至可以用 docker 来打包和部署自己的 notebook 服务。比较基本的操作是在监听本地端口的 jupyter 前边套一个 nginx 反代，如果你本来在对应的服务器上就有基于 nginx 建好的 https 网站（不推荐生产环境乱折腾），那直接在 nginx 配置里 url 重定向到服务器本地 jupyter 的监听端口即可（记得 jupyter 用到了 websocket，nginx 配置时注意下加 header）。

最后还有一点，既然是做成了远程 web 服务，当然要保证 jupyter notebook 在后台的稳定持续运行。最原始的方式就是执行`jupyter notebook`时加上nohup啥的，再高级点就是用 screen 或者 tmux 将命令跑在相应的 session 里。最好的办法是做成 service 进行管理，这里我推荐用 supervisor 来管理和自定义服务，比系统级的 systemctl 更易上手。通过 pip 或是 apt 等包管理工具安装 supervisor。在 /etc/supervisor/conf.d 添加一个配置文件 jupyter.conf，内容如下

~~~bash
[program:jupyter]
command=/usr/local/bin/jupyter notebook
user=username
autostart=true
autorestart=true
~~~

之后 `supervisord`就可启动 supervisor 的管理进程，如果 jupyter 没有自动启动的话可以 `sudo supervisorctl reload`一下。具体关于 supervisor 的使用与架构请自行谷歌。 至此，我们已经配置好了稳定运行在 web 上的 notebook 服务。你可以随时随地，通过不同设备（只要他们有浏览器），来访问你的笔记，记录和测试你的想法，可以作为一个云上的类笔记服务，同时还具有支持多种语言的强大计算功能。

# Notebook 显示类 web 内容

## markdown&html

Jupyter notebook 已经远超过了交互式 python 笔记本的范畴，其不仅后端 kernel 可以支持多种语言引擎（包括!开始的输入自动调用命令行），前端天生的 web 气质更让其充满想象力。话不多说，直接看下面这些例子。

<iframe src="https://nbviewer.jupyter.org/gist/refraction-ray/2ead8ac5efdc0382d8a5e3d863f5425b" width="100%" height="2000"></iframe>

结果非常震撼，基于 web 的交互，这个 web 基因不是随便说说的，这货已经强悍到前端用它来调试了吧。无论是 html5，css，javascript，甚至是引入 jQuery 库，notebook 全可以完美消化。就不要提对 markdown 的支持和 latex 的简单支持了。我强烈建议读者将上面的全部代码和内容细读一边，这才能更好的认识到notebook在web展示方面的潜力。当然也要感谢 nbviewer 可以给出这么大的宽容度，允许用户的js脚本发挥效应，使得这个ipynb在互联网上就能得到充分的显示（不得不说，这样做面临XSS攻击的压力还挺大的，比如里面我写的没有恶意的js `alert`）。*所以当你打开此页看到弹出的 hello 对话框时也别慌*。唯一的一点遗憾是 nbviewer 似乎不支持 mathematica 语法高亮，而在原始的 jupyter notebook 里是支持的。

## pyecharts

有了前面小节的探索，既然前端的js完全可以在notebook里发光发热，那么无比丰富的前端生态和库用在里边岂不是很炫。比如说可视化方面，基于 web 的 js 可视化库可以做出交互性更强也更现代更漂亮的数据可视化，自然的想法就是抛弃老相好python的matplotlib，而借助 js 的可视化库画图。以百度出品的[echarts.js](http://echarts.baidu.com/)为例（这个库还是很良心的），已经有人做好了 python 的封装，做的基本的事情就是一个 wrapper：用 python 风格的绘图 API 来生成 echarts js代码。这就是 pyecharts，官方 repo 请参考[这里](https://github.com/pyecharts/pyecharts)。当然还有其他一些好用的可视化js库和对应的python接口库，可以自行探索。

pyecharts 基本用法是生成可以被浏览器渲染成图片的含 echart.js 代码的html。不过正好借助 notebook 本身的 web 特性，pyecharts 可以直接出图，显示在notebook里面。这个库的另一个用途是结合在 web 框架里，这样后端 python 可以直接通过该库生成 js 代码返回给前端，从而把画图的任务甩锅到后端。同时 pyecharts 还直接支持 numpy 和 pandas 的数据结构来画图，简直幸福到没朋友。

具体事例和用法这里就不详细介绍了，你可以去官方实例的[repo](https://github.com/pyecharts/pyecharts-users-cases)学习一个。唯一要强调的，用这库在 Jupyter 内部显示都没问题，如果想要导出的 html 格式也能正确的嵌入可以被渲染成交互图片的 js 代码，需要在notebook里加上这句 `pyecharts.online()`，无视那个提醒就好了，据说这个提醒和函数的取代计划已经终止，见这个[issue](https://github.com/pyecharts/pyecharts/issues/324)。加了这行命令，notebook 导出成 html 后就能正常显示图片了。但是，经我测试，这里似乎有个小问题，page对象生成的图片，似乎加了online也无法以能被浏览器渲染的正确形式导出（version 0.3.1）。

既然说到了可视化，就也提一句 jupyter 项目官推的 [widgets](http://jupyter.org/widgets)，目的也是实现各种可以交互的基于web的可视化。

# Notebook 分享到 web 页面

## 格式转化方案

谈完了怎么在 notebook 里显示 web 内容，再谈谈怎么在 web 里显示 notebook 内容。简单来讲就是找个办法把 ipynb 转化成 html 格式。

最直接的就是从notebook的前端界面里，选择下载和保存格式的选项，存成 html 格式即可。更灵活的方式是在服务器上命令行转码 ipynb 文件。

~~~bash
$ jupyter nbconvert --to html --template full demo.ipynb
~~~

这么干最讲究就是模板选项，你可以自定义自己的模板或者魔改原有的模板，并保存在对应 package 的 template/html 路径下。具体package的路径可以通过以下命令输出来了解。

~~~python
import nbconvert
print(nbconvert)
~~~

nbconvert 的自带的 html 模板包括 full 和 basic，如果不指定则默认是使用 full.tpl。两者的区别是，full 提供的是生成独立完整的 html 页面，自带了 notebook 默认的 css 样式（甚至不是外链，而是未压缩放在了生成的 html 文件内，导致生成的 html 文件超级长，并且，这种格式转换会自动把 matplotlib 的画图转化成 base64 编码留在 html 文件里，有点丧心病狂）。而 basic 模版则会生成所谓的 html 片段，可以轻易的将其复制到其他 html 文件内来显示，这种方案会使用宿主 html 文件原有的 css 样式渲染，所以可能会很别扭。

既然模板可以定制魔改，就可以写出自带 css 的 basic（千万注意不要让引入的 notebook 的 css 污染到页面其他内容），或是定制自己喜欢的 css 样式的 full，能导出什么样的 html 文件，就看自己对模板有什么想象力了。如果考虑到 nbconvert 还可以导出 latex，markdown 等格式，结合模板的定制，可玩性还是不错的。更多关于 templates 的内容，可以参考[这里](https://github.com/ipython/ipython/wiki/Cookbook:-nbconvert-templates)。

## 网页嵌入方案 

在 web 上分享和展示 notebook 更好方案是 nbview。比起没头没脑生成一个 html 文件，或者需要魔改模板，小心处理 css 冲突等问题生成对应 notebook 的可嵌入 html 片段；更省心的方式还是直接在网站渲染生成一个网页，这样既可以完整的将 notebook 的结果以网站链接的形式分享给他人，又能通过 iframe 的形式嵌入自己的博客。

具体来讲就是首先将自己 ipynb 的源码（用编辑器打开复制即可，就是一个 json 文件）上传到公网的某处，这里推荐使用 github 的 [gist](https://gist.github.com/) 服务。事实上，github 已经自带了一定的 ipynb 的渲染能力了。不过这个渲染能力比较残废，css样式也怪怪的，很多东西无法好好渲染，因此不推荐直接分享对应 gist。否则的话在 jekyll 博客对应的md文件里直接加个 \{\% gist gist_no \%\} 就完美嵌入了。

之后访问官方的 [nbviewer](https://nbviewer.jupyter.org/) 服务，将对应gist的url复制到选框，即可得到 ipynb 的最终渲染效果，其使用的后台引擎就是前文提到的 nbconvert。这样就可以将最后得到的 nbview 的url分享出去，供大家欣赏批判了。而如果想在博客里引用的话，直接加一个 iframe 的 html 标签，并把 src 指向该网址就大功告成了。这一系列工作流的最后效果正如[上节](#markdownhtml)引用 notebook 展示出的那样。

*更新*： 现在 github gist 上 host 的 ipynb 甚至在右上角都会有按钮，提示 gist 渲染不完整并自动给出 nbviewer 网址的跳转，所以这节的展示工作流已经属于钦定的了。

# Google Colaboratory 机器学习

聊完了 Jupyter Notebook 的基础实践，再来看看网络服务以 Jupyter notebook 为交互接口的实例。

google 的 colaboratory 项目，说白了就是 google drive 里的 ipynb。这一集成巧妙结合了 google drive 的优势（共享交流，协作办公，随时随地同步）和 jupyter notebook 的优势 （交互式命令，所见即所得）。最吸引人之处还在于，这些notebook可以直接在 google drive 里创建和运行，不需要上传下载，并且，运行的后台可以选择 google 免费提供的 GPU Tesla K80！这羊毛不薅，简直对不起社会主义核心价值观！

colaboratory 的使用也很简单，直接在 google drive 里右击，在 more 里选择 colaboratory。如果没有该选项的话，先在右击－更多－关联更多应用里找到 colaboratory 并关联。之后打开新建的 ipynb 文件，在编辑－笔记本设置里，将硬件加速器设置为GPU即可。接下来一切都和普通的 notebook 无疑，除了其背后运行的是免费的 GPU 羊腿， enjoy！

这一项目的后台，已经安装好了基本的数据处理库 numpy，pandas等；谷歌全家桶，也即和谷歌各种服务API交互的python库；以及重头戏 tensorflow 的机器学习库等。至于 keras 之类，也可以在 notebook 里 `!pip install keras`来安装。方便的开箱即用，配上后面的强劲 GPU，再加上一键式的分享和协作体验，不来跑跑神经网络都觉得亏了。于是我在上面跑了个[上篇博客](/VAE的逻辑与实践)提到的识别生成手写数字的 VAE 网络，速度在GPU里，我没什么比较，但至少比我用 CPU 跑，提升了一个数量级。羊腿不仅免费还这么粗，大写的服！

使用 google colaboratory 的详细步骤可以参考[这篇文章](https://medium.com/deep-learning-turkey/google-colab-free-gpu-tutorial-e113627b9f5d)。这篇博客里讲解了将自己的 google drive mount到 notebook 后台硬盘的方法。用这种方式可以更灵活的运行 .py 文件和上传下载数据。

同时这篇博客还介绍了查看 notebook 后台硬件信息的方法。其实我们还可以更进一步，比如 `!cat /etc/issue`得知google在notebook的后台用的操作系统是 Ubuntu 17.10，用`!df -h`查看后台的硬盘分区情况。最恐怖的是，`whoami` 的返回结果是 root，apt install 的权限是放开的可以佐证root身份。这意味着除了前端的 notebook，后端似乎想象空间更大，毕竟 notebook 命令前加个感叹号，就和 ssh 到服务器后台没什么区别（而且你还有 sudo）。比如谷歌虽然阉割了后台的 ping 命令，但可以直接 `apt install -y hping3`，马上就获得了更加强大的 ping 工具。总之 colaboratory 服务器后台到底可以实现哪些东西，就看你的想象力啦，但个人不建议玩坏了。

*更新*： google colaboratory 和 nbviewer 一样，现在接受任意 github 地址 ipynb 文件的渲染和导入，同时可以将 google drive 上的 notebook 直接存储在 github 或 gist 上。这就相当于一个可以运行版的 nbviewer，还可以免费蹭 google 的 GPU 和新出的 TPU 引擎，可以作为之后分享 ipynb 格式的首推方案（如果不考虑国内受众的网络因素的话）。不过 colab 的渲染不支持跨域，因此没有办法以 iframe 的方式在其他网站嵌入来呈现给读者，只能以分享链接的方式。值得一提的是 colab 的 UI 不完全和 jupyter notebook 一致，比较突出的是 colab 的笔记本具有按照标题级别进行内容折叠的功能。这一功能和 mathematica 中的对应功能一致，也是原生 jupyter notebook 的一个痛点，方便了文件变大之后的浏览和管理。具体 colab 和 github 互动的各种操作可以参见[这里](https://colab.research.google.com/github/googlecolab/colabtools/blob/master/notebooks/colab-github-demo.ipynb#scrollTo=WzIRIt9d2huC)，渲染服务基本和 nbviewer 这种后接 url 的方式一致。另外，ipynb 文件保持和 jupyter notbook UI 一致的渲染，也有了可以运行版本的选择，也即 mybinder 服务，可以查阅其[主页](https://mybinder.org/)。该服务在 nbviewer 渲染网页里，以最上方的一个按钮的形式出现。

# Joinquant 量化分析

另一个以 notebook 为交互界面的网络服务，就是最近几年在国内兴起的互联网量化平台，这种平台以 [quantopian](https://www.quantopian.com/) 为模板，如雨后春笋般出现（[Ricequant 米筐科技](https://www.ricequant.com/)，[Joinquant 聚宽](https://www.joinquant.com/)，[Uqer 优矿](https://uqer.io/home/)）。我们这里以聚宽 Joinquant 为例，做一点简单的介绍。

这类网站基本都使用 python 语言。有两种基本的交互方式，一种是编写固定范式的内容，从而实现交易策略，进而利用平台的数据和引擎做回测分析和模拟交易。当然国内实时自动化交易并不放开，因此除了私下里使用破解某几家券商交易接口 API 的方案，正规渠道最多只能做到模拟交易和交易提醒。

另一种交互方式就是以notebook为交互界面，利用平台的数据 API，做一些数据研究和想法验证。这种模式在上文 google 的 colaboratory 的基础上更进一步。 google 只提供免费的计算资源，而这些量化网站，不只提供免费的计算资源，还提供免费的金融数据。要知道，靠谱的金融数据可都是很昂贵的。

以聚宽为例，网站免费提供计算所需的CPU，内存（限量1G）和硬盘空间（没有找到限制说明，也很有想象空间哦）。另外还提供 A 股股票价格公司财务指数成份经济基本面等大量05年以来的历史数据。因此在平台上利用这些数据 API，和 pandas 库，简单的几十行 python 就能做出很多有趣的分析和量化研究。比如自己计算指数市盈率并分析历史分位，用来指导指数定投等。此外，聚宽的研究模块支持 pyecharts，可以直接将数据结果可靠可交互的可视化，比 matplotlib 高到不知哪里去了，对于细节丰富的金融数据可视化，图片可交互性是非常关键的。此外聚宽的回测模块还支持研究模块的 py 文件直接 import，使得在 notebook 的研究成果和编写的函数，能够方便地应用到策略编写上。

关于数据的获取，平台原则上是不允许数据在平台之外使用和导出的，但量化平台往往都没有特别限制这一点。因此，对于自己想要的数据，只需平台上用 API 拿到，并 `to_csv()` 将拿到的数据作为文件导入到个人空间，之后逐一文件下载下来即可。如果嫌麻烦，也可以抓个包，看看文件下载的 http 请求，自己写个命令行自动批量下载即可。还是再提一句，注意别玩坏哦。

最后，还是把目光放到 notebook 后台服务器上，聚宽的后台用的是 Debian，让我们看看这一次感叹号魔法又如何。

<iframe src="https://nbviewer.jupyter.org/gist/refraction-ray/be904f07dcb7a6b34dbc73e96fad6b91" width="100%" height="600"></iframe>

EOF

