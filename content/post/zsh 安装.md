---
author: "bufferflies"
date: 2021-02-20
title: "zsh"
tags: [
    "tools",
]
categories: [
    "shell"
]
---

## 安装oh-my-zsh

```
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```
## 安装插件
1. 进入oh-my-zsh插件库中
    ```
    cd ~/.oh-my-zsh/custom/plugins/
    ```
2. 下载插件（自动补全为例）


    ```
    git clone git://github.com/zsh-users/zsh-autosuggestions
    ```

    其实插件名就是仓库名（zsh-autosuggestions）
3. 插件生效
    vim ~/.zshrc
    找到plugin配置，添加如下：
    
    ```
    plugins=(
        git
        zsh-autosuggestions
    )
    ```
4. 生效.zshrc 文件
    source ~/.zsh

## 安装fzf 
支持历史命令查看
https://www.jianshu.com/p/74ff1e637b2c
## 参考
- [安装oh-my-zsh](https://www.jianshu.com/p/b6af829ec2dc)
- [插件安装](https://blog.csdn.net/xfxf0520/article/details/84589446)
- [autojumo](https://learnku.com/tensorflow/t/5790/ultimate-terminal-zshautojump) 记录cd去过的路径 后面可以直接通过 j 文件夹进行跳转
- [fzf](https://juejin.im/post/5d905a886fb9a04e3902d7a9) 记录历史命令，非常直观显示最近使用的命令
    
     