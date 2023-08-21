---
author: "李昌"
title: "manjaro配置fcitx5输入法"
date: "2023-08-21"
tags: ["manjaro", "Tool"]
categories: ["Tool"]
ShowToc: true
TocOpen: true
---

## 1. 安装

```bash
yay -S fcitx5 fcitx5-rime fcitx5-input-support fcitx5-chinese-addons fcitx5-qt fcitx5-gtk fcitx5-configtool rime-double-pinyin
```

rime-double-pinyin 是双拼输入法，我使用小鹤双拼。

开机自启：
```bash
cp /usr/share/applications/org.fcitx.Fcitx5.desktop ~/.config/autostart/
```

## 配置

fcitx5的配置目录在：`mkdir ~/.local/share/fcitx5/rime/`

在这个目录下创建`default.custom.yaml`:
```yaml
patch:
  schema_list:
    - schema: double_pinyin_flypy
```
使用小鹤双拼输入方案。

若想对某个输入方案进行自定义，例如小鹤双拼，其原配置文件为：`~/.local/share/fcitx5/rime/double_pinyin_flypy.schema.yaml`.则需要在`~/.local/share/fcitx5/rime/`下创建`double_pinyin_flypy.custom.yaml`，然后在其中进行自定义。
例如：
```yaml
# encoding: utf-8

patch:
  "switches/@0/reset": 1
  switches:
    - name: ascii_mode
      reset: 1
      states: [ 中文, 西文 ] # 默认英语
    - name: full_shape
      reset: 0 # 默认半角
      states: [ 半角, 全角 ]
    - name: zh_simp           # 注意這裏（※1）
      reset: 1 # 默认简体
      states: [ 漢字, 汉字 ]
  switches/@next:
    name: emoji_suggestion
    reset: 1 # 默认开启emoji
    states: [ "🈚︎", "🈶" ]
  'engine/filters/@before 0':
    simplifier@emoji_suggestion
  emoji_suggestion:
    opencc_config: emoji.json
    option_name: emoji_suggestion
    tips: none
    inherit_comment: false
```

[主题设置](https://github.com/thep0y/fcitx5-themes)：
```bash
git clone https://github.com/thep0y/fcitx5-themes.git
cd fcitx5-themes
cp spring ~/.local/share/fcitx5/themes -r
```

修改fcitx5配置文件：
```bash
vim ~/.config/fcitx5/conf/classicui.conf
```

```conf
# 垂直候选列表
Vertical Candidate List=False

# 按屏幕 DPI 使用
PerScreenDPI=True

# Font (设置成你喜欢的字体)
Font="Smartisan Compact CNS 13"

# 主题(这里要改成你想要使用的主题名，主题名就在下面)
Theme=spring
```


## Reference

https://wiki.archlinuxcn.org/wiki/Fcitx5

https://wiki.archlinuxcn.org/wiki/Rime#%E9%85%8D%E7%BD%AE%E7%9B%AE%E5%BD%95

https://github.com/thep0y/fcitx5-themes