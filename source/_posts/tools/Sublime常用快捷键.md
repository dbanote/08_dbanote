---
title: Sublime Text 常用快捷键
tags:
- tools
- sublime text
---

## 通用
```
↑ ↓ ← →             上下左右移动光标
Alt                 调出菜单
Ctrl + Shift + P    调出命令板（Command Palette）
Ctrl + `            调出控制台
```

<!--more-->

## 编辑
```
Ctrl + Enter            在当前行下面新增一行然后跳至该行
Ctrl + Shift + Enter    在当前行上面增加一行并跳至该行
Ctrl + ←/→              进行逐词移动
Ctrl + Shift + ←/→      进行逐词选择
Ctrl + ↑/↓              移动当前显示区域
Ctrl + Shift + ↑/↓      移动当前行
```

## 选择
```
Ctrl + D                选择当前光标所在的词并高亮该词所有出现的位置
                        再次 Ctrl + D 选择该词出现的下一个位置，在多重选词的过程中
                        使用 Ctrl + K 进行跳过，使用 Ctrl + U 进行回退，使用 Esc 退出多重编辑
Ctrl + Shift + L        将当前选中区域打散
Ctrl + J                把当前选中区域合并为一行
Ctrl + M                在起始括号和结尾括号间切换
Ctrl + Shift + M        快速选择括号间的内容
Ctrl + Shift + J        快速选择同缩进的内容
Ctrl + Shift + Space    快速选择当前作用域（Scope）的内容
```

## 查找&替换
```
F3                  跳至当前关键字下一个位置
Shift + F3          跳到当前关键字上一个位置
Alt + F3            选中当前关键字出现的所有位置
Ctrl + F/H          进行标准查找/替换，之后：
Alt + C             切换大小写敏感（Case-sensitive）模式
Alt + W             切换整字匹配（Whole matching）模式
Alt + R             切换正则匹配（Regex matching）模式
Ctrl + Shift + H    替换当前关键字
Ctrl + Alt + Enter  替换所有关键字匹配
Ctrl + Shift + F    多文件搜索&替换
```

## 跳转
```
Ctrl + P        跳转到指定文件，输入文件名后可以：
@ 符号跳转       输入@symbol跳转到symbol符号所在的位置
# 关键字跳转     输入#keyword跳转到keyword所在的位置
: 行号跳转       输入:12跳转到文件的第12行。
Ctrl + R        跳转到指定符号
Ctrl + G        跳转到指定行号
```

## 窗口
```
Ctrl + Shift + N    创建一个新窗口
Ctrl + N            在当前窗口创建一个新标签
Ctrl + W            关闭当前标签，当窗口内没有标签时会关闭该窗口
Ctrl + Shift + T    恢复刚刚关闭的标签
```

## 屏幕
```
F11                             切换至普通全屏
Shift + F11                     切换至无干扰全屏
Alt+Shift+1       Single        切换至独屏
Alt+Shift+2       Columns:2     切换至纵向二栏分屏
Alt+Shift+3       Columns:3     切换至纵向三栏分屏
Alt+Shift+4       Columns:4     切换至纵向四栏分屏
Alt+Shift+8       Rows:2        切换至横向二栏分屏
Alt+Shift+9       Rows:3        切换至横向三栏分屏
Alt+Shift+5       Grid          切换至四格式分屏
```

## 插件"DeleteBlankLines"
```
# Windows/Linux
Ctrl+Alt+Backspace          删除所有空行
Ctrl+Alt+Shift+Backspace    删除多余空行

# Mac OS
Ctrl+Alt+Delete             删除所有空行
Ctrl+Alt+Shift+Delete       删除多余空行
```

## 插件"SublimeTmpl"
```
ctrl+alt+h          html
ctrl+alt+j          javascript
ctrl+alt+c          css
ctrl+alt+p          php
ctrl+alt+r          ruby
ctrl+alt+shift+p    python
```
