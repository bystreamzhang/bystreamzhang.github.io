---
title: ai-vedio
date: 2025-02-28 20:23:30
tags:
    - AI
    - vedio
categories: Joy
---

# 说明

学着做做ai视频玩玩. 主要就是学别人的. 记录一些流程和问题.

目前效果可能还一般, 就自己做着玩.

## 小明剑魔

参考[AI鬼畜躺赢狗 人物替换声音替换制作教程指南](https://www.bilibili.com/video/BV1zJ9wYjE6b/?spm_id_from=333.337.search-card.all.click&vd_source=69a4ae0209c3a80cb7d7747acca0305c)

视频替换用Viggle, 网站即可: [Viggle官网](https://www.viggle.ai/create-mix)
对于我这种整活视频, 人物替换最重要的是让别人知道你换的是谁. 如果是二次元的一些形象拟人的角色, 用简洁有特色的Q版形象效果可能都比实际花里胡哨的形象要好.

音频用GPT Sovits. 可参考[GPT-Sovits指南](https://www.yuque.com/baicaigongchang1145haoyuangong/ib3g1e)

剪辑软件我使用抖音的剪映专业版. 比较好用.

b站视频直接给了模型, 根据指南,
将GPT模型（ckpt后缀）放入GPT_weights_v2文件夹，SoVITS模型（pth后缀）放入SoVITS_weights_v2文件夹，刷新下模型就能选择模型推理了

关于指南中5.1：开启推理界面部分, "自动弹出推理界面"这一步, 要有心理预期, 就是这个自动可能不会很快.
我是以为有问题, 在找问题的时候这个界面突然弹出来了. 大概是需要一段时间加载.

Viggle免费一天只能生成5个, 不过用邮箱验证码就可以注册所以换个账号就行.

ai合成的语音,整体还是不如原声带感, 所以一些原文部分还是直接用原音频比较好.
