---
layout: post
title: "Sublime Text使用"
description: ""
category: 
tags: []

---
{% include JB/setup %}

MacOS Sublime Text在Vim模式下无法连续按键

在终端下运行如下命令：
defaults write com.sublimetext.3 ApplePressAndHoldEnabled -bool false

恢复：
defaults write -g ApplePressAndHoldEnabled -bool false

引用：
https://gist.github.com/kconragan/2510186