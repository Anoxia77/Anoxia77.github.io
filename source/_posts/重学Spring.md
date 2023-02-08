---
title: 重学Spring
---
## 前言
![](https://s3.bmp.ovh/imgs/2023/02/07/9a1d1648ec5b65b1.jpg#id=pHeEJ&originHeight=1440&originWidth=2560&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
从上一次学完spring到现在已经1年多了，学完之后感觉自己提升了不少，对项目中一些spring的用法也清晰了些。<br />第一次学习过于匆忙，没有记录，没有复盘，就是一路走马观花是的跟着敲完了代码，虽然学完之后自己对spring有了简单的认识，但是在实际开发过程中用起来还是磕磕绊绊。<br />这一次，重新去跟着学一遍spring，这一次，一步一记录，可能对于以前啥都不会来说，能更好的去组织自己所学到的知识。<br />

下面是我目前能记起的spring提供的一些东西<br />![](https://cdn.nlark.com/yuque/0/2023/jpeg/2869098/1675318460071-b4de6a74-1e3f-4d4e-9436-4c88309065eb.jpeg)<br />第一次学完spring后，我对spring的认识就是 `spring 只是一个工厂，帮助我们管理 Bean（对象）的创建与销毁`，这其实也是 `IOC`控制反转，提供的功能。<br />spring 接管了以前由 人手动 new 一个对象的过程，通过一个工厂统一来管理项目中所有的 Bean。<br />所以 spring 的核心，应该是围绕这 对象的创建进行的。

#### 类加载过程
![](https://cdn.nlark.com/yuque/0/2023/jpeg/2869098/1675321986350-2946785c-2c8b-4f2c-8899-1fd3efebcdda.jpeg)
#### 对象的创建过程
![](https://cdn.nlark.com/yuque/0/2023/jpeg/2869098/1675322171166-392ad12a-1c2f-41aa-92f5-c57ff7ca51ef.jpeg)
