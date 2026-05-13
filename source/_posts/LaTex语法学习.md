---
title: LaTex语法学习
katex: true
abbrlink: 4defbc1f
date: 2022-09-04 23:48:43
tags:
cover:
top_img:
---

### LaTex简单介绍

LaTex是一种简单文件生成系统，文件后缀为.tex



### 基本用法

- 新建一个文件，后缀名为.tex
- 输入基本语法框架

```latex
\documentclass{article}
\begin{document} 

Hello world! 

\end{document}
```

### 基本语法

1. 文档类型 ' article ', 'book ',等等
2. 注释使用%，转义使用\

### 由于时间仓促就不介绍其他参数了，直接上用法

### 字体设置

使用fontspec包设置字体

```latex
\usepackage{fontspec} 
\usepackage{amsmath} 
\setmainfont{Times New Roman}
```

不过一般论文使用TNR差不多了:ocean:

### 字体大小设置

```latex
\tiny 
\scriptsize 
\footnotesize 
\small 
\normalsize 
\large 
\large 
\LARGE 
\huge 
\Huge
```

### 一、公式部分

 1. 行内公式 \$. . .\$，独立成行使用\$\$...\$$

    ```
    Einstein 's $E=mc^2$.   %行内
    ```

    $E=mc^2$

    $x^2 + 2x + 5 + \sqrt x = 0$

    $F=G \frac {m_{1}m_{2}}{R^{2}}$

    $i\hbar\frac{\partial \psi}{\partial t} = \frac{-\hbar^2}{2m} \left(\frac{\partial^2}{\partial x^2} + \frac{\partial^2}{\partial y^2}+\frac{\partial^2}{\partial z^2} \right) \psi + V \psi
    $

 2. 行间公式 \[ ...]

    ```
    [E=mc^2]   %行内
    \[ E=mc^2. \]
    ```

    \[ E=mc^2\]

 3. 一行插入多个公式环境为flalign

    ```latex
    \begin{flalign} 
    S_{n+1} = S_{n} + S_{n},  
    S_{n}=1=2^{n} 
    \end{flalign}
    ```

    $\begin{flalign} 
    S_{n+1} = S_{n} + S_{n}, 
    S_{n}=1=2^{n} 
    \end{flalign}$

 4. 对行间公式编号

    ```latex
    \begin{equation} 
    ... 
    \end{equation}
    ```

    $\begin{equation} 
    ... 
    \end{equation}$

 5. 公式上下标

    ```latex
    ^{} %上标 
    _{} %下标
    ```

 6. 分式

    ```latex
    \frac{m}{n} %n分之m
    ```

 7. 开方

    ```latex
    \sqrt{} %开平方 
    \sqrt[m]{n} %n开m次方
    ```

 8. 求和

    ```
    \sum_{i=m}^{n}  %从m到n求和
    ```

    $\sum_{i=m}^{n}$

 9. 求积

    ```
    \prod_{i=m}^{n} %从m到n求积
    ```

    $\prod_{i=m}^{n}$ %从m到n求积

 10. 积分

    \int_{i=m}^{n}  %从m到n积分

​		$\int_{i=m}^{n}$  %从m到n积分

 11. 向量

     ```
     \vec a  %a向量 
     \overrightarrow{AB} %A到B的向量
     ```

     $\vec a$ %a向量 
     $\overrightarrow{AB}$ %A到B的向量

 12. 省略号

     ```
     a+b+\cdots+z    %a+b+…+z
     ```

     $a+b+\cdots+z    \%a+b+…+z$

 13. 大括号

     ```
     \underbrace{a+b+\cdots+z}_{26}  %a+b+…+z
     ```

     $\underbrace{a+b+\cdots+z}_{26}$ %a+b+…+z

 14. 横杠

     ```
     \overline{m+n}  %m+n公式上面加上横杠 
     \underline{m+n} %m+n公式下面加上横杠
     ```

     $\begin{flalign}\overline{m+n},\ \underline{m+n}\end{flalign}$

### 二、其他内容





