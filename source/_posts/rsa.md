---
title: RSA的数学知识
date: 2020-11-01 12:58:31
categories:
  - math
mathjax: true
---


# 费马小定理
$$
a^p \bmod p = a
$$
其中p为素数

# 欧拉函数
$\varphi(n)$代表小于等于n的所有与n互质的正整数。
$$
\varphi(pq) = \varphi(p)*\varphi(q) = (p-1)*(q-1)
$$
其中pq为素数。
更通用的求解：
$$
\varphi(n) = n\displaystyle\prod_{p|n}(1-\frac{1}{p})
$$
<!-- more -->

# 欧拉定理
$$
a^{\varphi(n)} \equiv 1 \bmod n
$$
其中a与n互质

# RSA
1. 生成pq，生成n=pq
2. $r=\varphi(n)=\varphi(pq)=(p-1)(q-1)$
3. 找一个小于r的数e
4. 求e关于r的模逆元b，即$eb \equiv 1 \bmod r$
5. 加密：$m=x^e \bmod n$
6. 解密：$x=m^b \bmod n$

# RSA证明
$$
m^b \bmod n = 
$$
$$
x^{eb} \bmod n = 
$$
$$
x*x^{eb - 1} \bmod n = 
$$
$$
x*x^{kr} \bmod n = 
$$
$$
x\bmod n * ((x^r \bmod n)^k \bmod n) = 
$$
$$
x\bmod n *((1\bmod n)^k \bmod n) = 
$$
$$
x \bmod n = 
$$
$$
x
$$

# RSA安全性分析
公开的：e，n，m
私有的：b，x，p，q，r
分解n为两个大素数p，q，不存在非多项式时间的解，因此无法求出r，无法求出b

# 欧拉定理的证明
证明
$$
a^{\varphi(n)} \equiv 1 \bmod n
$$

## 同余类
模n余数相同的数构成一个模n的同余类。

## 完全剩余系
从n的同余类中，各取一个数，构成的集合。

## 最小非负完全剩余系
{0, 1, ..., n-1}是n的最小非负完全剩余系。

## 缩剩余系
完全剩余系中所有与n互质的数，构成n的缩剩余系，其个数即为$\varphi(n)$。

当这个完全剩余系是最小非负完全剩余系时，构成n的最小正缩系。

## 证明

* $\phi(n)$ = {$c_1$, $c_2$, ..., $c_{\varphi(n)}$}代表模n的最小正缩系。
* $a\phi(n)$代表$\phi(n)$中的每个数乘以a，则$a\phi(n)$ = {$ac_1$, $ac_2$, ..., $ac_{\varphi(n)}$}是模n的缩剩余系。其中a与n互质
  * 任意$ac_i$和$ac_j$模n的余数不相同：假如相同，因为a与n互质，根据消去律能得到$c_i \equiv c_j \bmod n$，
  * 任意$ac_i$与n互质
* 对于任意缩系，所有数模n之后的余数构成n的最小正缩系
* 则有$\displaystyle\prod_{i=1}^{\varphi(n)}ac_i \equiv \displaystyle\prod_{i=1}^{\varphi(n)}c_i \bmod n$
* 则有$a^{\varphi(n)}\displaystyle\prod_{i=1}^{\varphi(n)}c_i \equiv \displaystyle\prod_{i=1}^{\varphi(n)}c_i \bmod n$；因为任意$c_i$与n互质，则$\displaystyle\prod_{i=1}^{\varphi(n)}c_i$与n互质
* 根据消去律得：$a^{\varphi(n)} \equiv 1 \bmod n$