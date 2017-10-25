---
layout: post
title: 解决Fcitx输入法在Qt应用中无法使用的问题
comment: on
---

作为重度网瘾患者，我一直避免在博客里谈计算机那些事。毕竟在被屏幕包围的生活中，写作是难得的浮出水面透口气的片刻。然而这次略施小技，解决了生活中的困扰，不免洋洋得意，跟诸位唠叨两句。

<!--excerpt-->

我使用Linux作为桌面操作系统，具体到发行版则是Debian。因为最近阅读和下载的电子书比较多，就安装了[Calibre](https://calibre-ebook.com/)电子书管理软件（版本3.9.0），不料我的中文输入法在这个软件中压根无法使用，按切换输入法的快捷键一点反应都没有。网上一查，大家都说是Calibre使用了[Qt](https://www.qt.io/)图形库，而[Fcitx](https://fcitx-im.org/wiki/Fcitx)输入法框架与Qt图形库不兼容。其实，之前我在使用一款名为[Anki](https://apps.ankiweb.net/)的辅助记忆软件（版本2.0.47）记单词的时候，已经遇到过一次这个问题，一直靠先把字写到记事本里再复制粘贴的法子凑合着。这回居然又碰到了，忍无可忍，仔细研究了一番，才算彻底地解决了这个问题。

实际上，Fcitx框架的开发者已经为Qt提供了支持，但我按照提示安装fcitx-libs-qt5和fcitx-frontend-qt5两个软件包后仍然无法在Qt应用中使用输入法。参考Fcitx官网的常见问题解答[页面](https://fcitx-im.org/wiki/FAQ#Is_it_a_Qt_application_that_bundles_its_own_Qt_library.3F)，发现Calibre和Anki没有使用系统提供的Qt图形库及附带的插件，而是自带了一套Qt，保存在软件的安装目录下。
```
$ ls /opt/calibre/lib/qt_plugins/platforminputcontexts
libcomposeplatforminputcontextplugin.so  libibusplatforminputcontextplugin.so
```

可见，Calibre自带了IBus插件，能使用IBus框架下的输入法，却没有Fcitx对应的插件。我们要做的就是把fcitx-libs-qt5中的动态库文件拷贝到/opt/calibre/lib/目录下，将fcitx-frontend-qt5中的动态库文件拷贝到/opt/calibre/lib/qt_plugins/platforminputcontexts/目录下。

系统已安装的动态库文件位置可以通过以下命令查看：
```
$ dpkg -L fcitx-libs-qt5
$ dpkg -L fcitx-frontend-qt5
```
将其中形如lib\*.so.\*的文件拷贝至对应目录，在命令行运行Calibre进行验证。如果仍然没有反应，很有可能是fcitx-libs-qt5和fcitx-frontend-qt5两个软件包与Calibre所使用的Qt的版本不匹配。参考[此处](https://groups.google.com/forum/#!topic/fcitx/9e4TI39_4sk)的讨论，到发行版对应的在线软件仓库中寻找其他版本，一一进行试验。

我使用的Debian版本为jessie，fcitx-libs-qt5和fcitx-frontend-qt5两个包的版本为0.1.2，而jessie的后继stretch，对应的两个软件包的版本已经达到了1.0.5，找到下载链接（[fcitx-libs-qt5](https://packages.debian.org/stretch/libfcitx-qt5-1)，[fcitx-frontend-qt5](https://packages.debian.org/stretch/fcitx-frontend-qt5)），下载完成后，通过以下命令提取其中的库文件：
```
$ dpkg-deb -R libfcitx-qt5-1_1.0.5-1+b1_amd64.deb SOME_TEMP_DIR
$ dpkg-deb -R fcitx-frontend-qt5_1.0.5-1+b1_amd64.deb ANOTHER_TEMP_DIR
```

删除之前不好使的库文件:
```
$ cd /opt/calibre/lib/
$ rm $(find -iname '*fcitx*')
```
再将新文件拷贝至对应目录，运行Calibre测试。我的系统此时报错：
```
xts/libfcitxplatforminputcontextplugin.so: symbol xkb_compose_table_new_from_locale, version V_0.5.0 not defined in file libxkbcommon.so.0 with link time reference
```
报错原因是libxkbcommon.so.0的版本太旧，fcitx-frontend-qt5的1.0.5版本要求libxkbcommon的版本至少为0.5.0。libxkbcommon是用于处理键盘输入的通用软件库，为避免影响系统整体的稳定性，我不想单独更新libxkbcommon库，而是从Debian软件仓库中下载[新版](https://packages.debian.org/stretch/libxkbcommon0)后，提取库文件，放置于/opt/calibre/lib/目录下。因为Calibre运行时会优先使用自身安装目录下的库文件，之后再搜索系统路径，借助这一点，我们能很好地解决兼容问题。

Anki使用的是Qt4，Calibre用的是Qt5，对应的Fcitx库也不一样，但处理的方法大同小异，这里不再赘述。

通过这件事，总结几点：一、出问题先去官网找解决方案；二、发布软件时，捆绑安装所依赖的库，还是使用系统提供的库，是一个令人困扰的难题。捆绑的好处是无需用户单独安装依赖，确保库文件的版本，缺点是无法与系统其他的软件协同，且当所依赖的库有更多的依赖时会引发更大的麻烦。使用系统提供的库则正好相反，虽然不用为软件依赖关系操太多的心，但发行版提供的软件包往往是很旧的版本，不能满足软件的功能需要。如何取舍，就得依情况而定了。
