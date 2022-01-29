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
4. 求e关于r的模逆元d，即$ed \equiv 1 \bmod r$
5. RSA组成
   * 公钥：(e, n)
   * 私钥：(d, n)
   * 原文：m ([0-n-1])
   * 密文：c
6. 加密：，c为密文，$c=m^e \bmod n$
7. 解密：$m=c^d \bmod n$

# RSA证明

## m与n互质
$$
c^d \bmod n = 
$$
$$
m^{ed} \bmod n = 
$$
$$
m*m^{ed - 1} \bmod n = 
$$
$$
m*m^{kr} \bmod n = 
$$
$$
m\bmod n * ((m^r \bmod n)^k \bmod n) = 
$$
根据欧拉定理，因为m与n互质，则$m^r \bmod n = 1$，则
$$
m\bmod n *((1\bmod n)^k \bmod n) = 
$$
$$
m \bmod n = 
$$
$$
m
$$

## m与n不互质
因为n=pq且pq是质数，则m=tp或者m=tq，这里假设m=tp，计算$m^{ed} \bmod q$
$$
m^{ed} \bmod q= 
$$
$$
(tp)^{ed} \bmod q = 
$$
$$
(tp)^{kr+1} \bmod q = 
$$
$$
(tp)^{k(p-1)(q-1)+1} \bmod q = 
$$
$$
(tp)((tp)^{(q-1)})^{k(p-1)} \bmod q
$$
此时tp与q互质（反证法：若tp与q不互质，因为pq互质，则t与q不互质，因为q是质数，所以t=kq，即m=tp=kpq=kn，则m大于n，不满足条件m在[0-n-1]范围）
根据欧拉定理：$(tp)^{(q-1)} \equiv 1 \bmod q$，所以
$$
(tp)^{eb} \bmod q= 
$$
$$
(tp)((tp)^{(q-1)})^{k(p-1)} \bmod q = 
$$
$$
(tp) \bmod q
$$

所以，
$$
(tp)^{ed} \equiv (tp) \bmod q
$$
即
$$
(tp)^{ed} = jq + tp
$$
又因为
$$
(tp)^{ed} \bmod p = 0
$$
所以
$$
(jq + tp) \bmod p = 0, 即 jq \bmod p = 0
$$
因为pq互质，所以
$$
j \bmod p = 0, 即 j = lp
$$
即
$$
(tp)^{ed} = jq + tp = lpq + tp = ln + tp
$$
所以
$$
(tp)^{ed} \bmod n = (ln+tp) \bmod n = tp \bmod n
$$
由假设m=tp，可证得：
$$
m^{eb} \bmod n = m \bmod n = m
$$

# RSA安全性分析
* 公开的：e，n，c
* 私有的：d，m
* 生成秘钥对后销毁 p，q，r
  
分解n为两个大素数p，q，不存在非多项式时间的解，因此无法求出r，无法求出d

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

# 参考资料

- [1] [https://tools.ietf.org/html/rfc8017](https://tools.ietf.org/html/rfc8017)