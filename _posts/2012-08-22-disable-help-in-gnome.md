---
layout: post
title: "Disable help in Gnome (关闭gnome里面的帮助）"
description: "关闭 gnome 里面的帮助"
category: Linux
tags: [linux, gnome]
---
{% include JB/setup %}

最近公司发了一个笔记本，拿来装上了CentOS。这电脑的键盘F1的位置和标准键盘的ESC位置一样，在使用Terminal/VIM等工具时按Esc键时经常会按上F1，F1是Gnome里面默认的帮助快捷键，这样老是会弹出Gnome的帮助，甚为不爽。

查了一下，可以使用gconf-editor来修改其快捷键达到关闭目的。

check if gconf-editor installed

    $ rpm -qa | grep gconf-editor

or use yum to install:

    $ yum install -y gconf-editor

After installed, we can use `gconf-edit` to start the editor.

    $ gconf-edit

Then, in /apps/gnome-terminal/keybindings find the key Help and change the value from F1 to other value. suggest change to F12 or leave empty.

Close the editor. And you will find the help never popup again. 

Have a good day.:)

With help from <http://www.bytebot.net/blog/archives/2007/09/06/disabling-help-in-gnome>
