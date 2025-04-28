
## 介绍

### 作者介绍

Stefan Gustavson拥有理学硕士背景及博士学位，1965年出生，现居瑞典国内，是林雪平大学计算机图形学与图像处理专业的高级讲师。多年来，他的研究兴趣有所转变，从色彩与图像再现转向高动态
范围成像，但过程纹理与噪声一直是他职业生涯中持续关注的一个方面，他也在该领域发表了一些经过同行评审的学术论文。他将大量时间用于教学，并乐在其中。 


### 封面介绍

封面图片是作者Shadertoy账号中的一个GLSL演示着色器。它展示了本书中涉及的许多技术，其中包括几种不同类型的噪声、凹凸和位移映射、散射（斐波那契球体）、混合方法（文本的预计算距离场）以及放大和缩小的抗锯齿。

基础对象是一个通过单次光线追踪渲染的光滑球体，不使用普通纹理图像。该着色器即使在手机上也能实时流畅运行。

该演示可在以下网址实时观看： https://www.shadertoy.com/view/sdfyWX


## License and contact information
All material in this book is my own original creation,
except for some of the images and a small amount
of code. The sources for these are clearly stated.
Everything is of course used with permission.
Released under the terms of the CC-BY-NC 4.0 license:
https://creativecommons.org/licenses/by-nc/4.0/
TL;DR: Use and remix for any purpose, but give credit,
and don’t make actual money from it. If you want to use
any of this for commercial purposes, contact me to
negotiate a separate license. I’m reasonable.
If you find errors, omissions, bad explanations or figures that
could be made more clear, or if you just want to get in contact
with me, feel free to post a comment on Github:
https://github.com/stegu/noiseisbeautiful/



## 前言

这本书的撰写是为了填补一项空白。几十年来，我教授的有关程序化方法的理学硕士课程所选用的教科书，实际上也是市面上唯一一本详细聚焦于该主题的专业教科书，就是《Texturing and
Modeling: a Procedural Approach”》。其最新的第三版于2003年出版，由大卫·埃伯特编辑，他与众多杰出的合著者共同撰写，这些合著者都对该领域的发展起到了推动作用： Ken Musgrave, Darwin Peachey, Ken Perlin and Steven
Worley。人们亲切地将这本书称为 “程序化圣经”，这是有充分理由的。 

然而，由于需求低迷，这本书几年前就被出版商停止印刷了。非主流学术科目的教科书往往销量不佳，遗憾的是，当已印书籍售罄后，这本书就不再重印了。诚然，书中有几个章节已经过时，其他一些章节也需要修订，但那本书是我们唯一的选择。

在一次用电子书取代纸质版的尝试失败后，出版商选择重新推出1991年出版的该书第一版，这一版既严重过时，写作水平也相当欠佳。2003年出版的大幅改进且广受好评的第三版，如今除了从图书馆借阅，在二手市场购买单本，以及非法获取盗版PDF外，已无法获得。该书第三版停止印刷后，我开始发放更详细的讲义以及我自己撰写的短文，这些内容逐年变得更加丰富。本书的总体架构在很大程度上基于那些讲义，但大部分内容是新写的。 

如果你能（当然是通过合法途径）搞到这本经典教材的第三版，我还是推荐阅读的。你可以跳过关于硬件着色的章节，但其余部分都值得一读。不过，除非出于历史研究的好奇，否则我不建议看第一版。花钱买它的电子版是浪费钱。

我希望这本书能达到它的目的。我自己一定会找到它的好用处。
*<p align="center">Stefan Gustavson, Norrköping, Sweden, April 2025</p>*

