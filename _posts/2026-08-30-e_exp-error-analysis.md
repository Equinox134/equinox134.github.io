---
title: e_exp.c Error Analysis
author: Equinox134
date: 2026-08-30
categories:
  - Computer Science
  - Math
tags:
  - Computer
  - Science
math: true
mermaid: true
excerpt: An analysis on the error of glibc's exp function
---
# Worst-Case ULP Error Bound for exp (binary64, $N = 128$)

**Goal.** Find a worst-case bound on

$$E(x) = \frac{\lvert \mathrm{exp}(x) - e^x \rvert}{\mathrm{ulp}(e^x)}$$

for all $x$ on the main path.

**Result.** $E(x) \leq 0.5123$ on the main path, and $E(x) \leq 0.506156$ on the subnormal branch.

---

## 1. Notation

|Symbol|Meaning|
|---|---|
|$u = 2^{-53}$|unit roundoff of binary64 under round-to-nearest|
|$\mathrm{RN}(\cdot)$|correct rounding to nearest, ties to even|
|$\mathrm{ulp}(t) = 2^{e-52}$|for positive real $t \in [2^{e}, 2^{e+1})$|
|$\hat{v}$|the _computed_ (floating-point) value of a quantity $v$|

In the Gappa scripts below, `tmp`, `r`, … denote computed values and `Mtmp`, `Mr2`, … denote their exact real-arithmetic counterparts.

---

## 2. The Algorithm

With $N = 128$:

$$ \begin{aligned} \mathrm{exp}(x) &= 2^{k/N} \cdot e^{r}, &&\quad r = x - k \frac{\ln 2}{N}, \quad k = \mathrm{round}\left(x \cdot \frac{N}{\ln 2}\right) \cr 2^{k/N} &= \mathrm{scale} (1 + \mathrm{tail}) (1 + \varepsilon_{\mathrm{tab}})^{-1}, &&\quad \text{table entry: exact power of two} \times H_i \cr e^{r} &\approx 1 + p(r), &&\quad p(r) = r + C_2 r^2 + C_3 r^3 + C_4 r^4 + C_5 r^5 \cr \mathrm{result} &= \mathrm{RN}\left(\mathrm{scale} + \mathrm{scale}\cdot \widehat{\mathrm{tmp}}\right), &&\quad \widehat{\mathrm{tmp}} \approx \mathrm{tail} + p(r) \end{aligned} $$

### 2.1 Error sources

|Name|Definition|
|---|---|
|$\varepsilon_r$ — reduction|$\lvert \hat{r} - r \rvert$|
|$B_{\mathrm{eval}}$ — evaluation|$\bigl\lvert \widehat{\mathrm{tmp}} - (\mathrm{tail} + p(\hat{r})) \bigr\rvert$|
|$\mathrm{prop}$ — propagation|$\lvert p(\hat{r}) - p(r) \rvert$|
|$E_{\mathrm{approx}}$ — approximation|$\lvert p(r) - (e^{r} - 1) \rvert$|
|$\mathrm{cross}$ — dropped term|$\lvert \mathrm{tail}\cdot(e^{r} - 1) \rvert$|
|$\varepsilon_{\mathrm{tab}}$ — table residual|$\bigl\lvert \mathrm{scale} (1 + \mathrm{tail})/2^{k/N} - 1 \bigr\rvert$|
|$\delta_s$ — final rounding|rounding of the last addition|

---

## 3. Error Decomposition

$$ \begin{aligned} \lvert e^{x} - \mathrm{exp}(x) \rvert &= \left\lvert 2^{k/N} e^{r} - \mathrm{result} \right\rvert \cr &\leq \underbrace{\left\lvert \left(2^{k/N} - \mathrm{scale}(1+\mathrm{tail})\right) e^{r} \right\rvert}_{\text{(A) table}} + \underbrace{\left\lvert \mathrm{scale}(1+\mathrm{tail})e^{r} - \left(\mathrm{scale} + \mathrm{scale}\cdot\widehat{\mathrm{tmp}}\right) \right\rvert}_{\text{(B) evaluation of } \mathrm{tmp}} \cr &\quad + \underbrace{\left\lvert \left(\mathrm{scale} + \mathrm{scale}\cdot\widehat{\mathrm{tmp}}\right) - \mathrm{RN}\left(\mathrm{scale} + \mathrm{scale}\cdot\widehat{\mathrm{tmp}}\right) \right\rvert}_{\text{(C) final rounding} \leq \tfrac{1}{2}\mathrm{ulp}} \end{aligned} $$

Term **(A)** is bounded by maximizing $\bigl\lvert \mathrm{scale}(1+\mathrm{tail})/2^{k/N} - 1 \bigr\rvert$ over the table; term **(C)** is at most $\tfrac{1}{2} \mathrm{ulp}$.

Term **(B)** expands as

$$ \begin{aligned} \left\lvert \mathrm{scale}(1+\mathrm{tail})e^{r} - \left(\mathrm{scale} + \mathrm{scale}\cdot\widehat{\mathrm{tmp}}\right) \right\rvert &= \lvert \mathrm{scale} \rvert \cdot \left\lvert \left((1+\mathrm{tail})e^{r} - 1\right) - \widehat{\mathrm{tmp}} \right\rvert \cr &\leq \lvert \mathrm{scale}\rvert \Bigl( \underbrace{\bigl\lvert \widehat{\mathrm{tmp}} - (\mathrm{tail} + p(\hat{r})) \bigr\rvert}_{\text{Gappa}} + \underbrace{\lvert p(\hat{r}) - p(r) \rvert}_{\text{MVT}} \cr &\qquad\qquad + \underbrace{\lvert p(r) - (e^{r}-1) \rvert}_{\text{Sollya}} + \underbrace{\lvert \mathrm{tail} (e^{r}-1) \rvert}_{\max\lvert\mathrm{tail}\rvert \cdot \max\lvert e^r-1\rvert} \Bigr) \end{aligned} $$

The mean-value-theorem term uses bounds on $\lvert r - \hat{r}\rvert$ and $\lvert p'(\cdot)\rvert$.

---

## 4. Exact Constants

From `e_exp_data.c` and `math_config.h`, with `EXP_TABLE_BITS = 7` and `EXP_POLY_ORDER = 5`:

|Constant|Hex value|
|---|---|
|$\mathrm{InvLn2N}$|`0x1.71547652b82fep7`|
|$\mathrm{NegLn2hiN}$|`-0x1.62e42fefa0000p-8`|
|$\mathrm{NegLn2loN}$|`-0x1.cf79abc9e3b3ap-47`|
|$\mathrm{Shift}$|`0x1.8p52`|
|$C_2$|`0x1.ffffffffffdbdp-2`|
|$C_3$|`0x1.555555555543cp-3`|
|$C_4$|`0x1.55555cf172b91p-5`|
|$C_5$|`0x1.1111167a4d017p-7`|
|$\mathrm{tab}[256]$|128 pairs (`tail`, `sbits`)|

### 4.1 Properties

$$ \frac{\mathrm{InvLn2N} - N/\ln 2}{N/\ln 2} = 2^{-55.98} \qquad \text{(correctly rounded)} $$

$$ \rho := \left\lvert -\frac{\ln 2}{N} - (\mathrm{NegLn2hiN} + \mathrm{NegLn2loN}) \right\rvert = 7.8734 \times 10^{-34} < 2^{-100} $$

$$ \mathrm{NegLn2hiN} \text{ has 17 trailing zero bits} \Longrightarrow 36 \text{ significant bits} $$

$$ \max_i \lvert \mathrm{tail}_i \rvert = 9.4479\times10^{-17} < 2^{-53.23} $$

$$ \varepsilon_{\mathrm{tab}} := \max_i \left\lvert \frac{\mathrm{scale}_i (1+\mathrm{tail}_i)}{2^{i/N}} - 1 \right\rvert = 6.03\times10^{-33} < 2^{-107} $$

### 4.2 The `sbits` mechanism

Pure bit manipulation: for $k = mN + i$, `sbits = template_i + (k << 45)` reconstructs $\mathrm{asuint64}(2^{m} H_i)$ exactly (the template stores $\mathrm{asuint64}(H_i) - (i \ll 45)$). This is valid as long as the exponent field neither overflows nor underflows, i.e. $-1023N < k < 1024N$.

Consequently $\mathrm{scale}$ is **exact** — no error enters here, and the only table error is $\varepsilon_{\mathrm{tab}} < 2^{-107}$.

---

## 5. Bounds on $k$ and $r$; Exactness of $k_d \cdot \mathrm{NegLn2hiN}$

### 5.1 Range of $x$

Outside the special cases, $2^{-54} \leq \lvert x \rvert < 512$. Since $e^{x}$ is a normal double only for $x \in (-708.40, 709.79)$, and the widest $\lvert x \rvert$ producing a normal or subnormal result is about $745.14$, it suffices for the theorem's scope to take

$$\lvert k \rvert \leq 137630, \qquad \text{since } 745.14 \cdot \frac{N}{\ln 2} + 1 < 137630 .$$

### 5.2 Which integer $k$ is chosen

Let $z = \mathrm{RN}(\mathrm{InvLn2N}\cdot x)$. Then

$$\left\lvert z - \frac{xN}{\ln 2} \right\rvert \leq \lvert x\rvert \cdot 2^{-55.98}\cdot\frac{N}{\ln 2} + \tfrac{1}{2}\mathrm{ulp}(z) \leq 2^{-34.38}.$$

The shift trick works because $\lvert z\rvert < 2^{51}$, so $\mathrm{RN}(z + 1.5\cdot 2^{52})$ rounds $z$ to an integer. This yields an integer $k_d = k$ with $\lvert z - k\rvert \leq \tfrac{1}{2}$, hence

$$\lvert r \rvert = \left\lvert x - k \frac{\ln 2}{N}\right\rvert \leq \frac{\ln 2}{N}\left(\frac{1}{2} + 2^{-34.38}\right) =: R_{\mathrm{exact}} \leq 0.00270760618 .$$

### 5.3 Exactness of $k_d \cdot \mathrm{NegLn2hiN}$

Write $\mathrm{NegLn2hiN} = -m_{hi}\cdot 2^{-43-17}$ with the odd integer

$$m_{hi} = 6243314768150528 \gg 17 = 47633801636 .$$

The product $k_d \cdot \mathrm{NegLn2hiN}$ is the integer $k_d m_{hi}$ scaled by a power of two, so it is representable iff $\lvert k_d m_{hi}\rvert < 2^{53}$, i.e.

$$\lvert k \rvert \leq \left\lfloor \frac{2^{53}-1}{47633801636} \right\rfloor = 189096 .$$

Since $\lvert k\rvert \leq 137630 < 189096$ on every path, **the product is exact**.

---

## 6. Approximation Error $E_{\mathrm{approx}}$

Compare $p$ against $e^{r}-1$ on the reduction interval $[-0.0027076062, 0.0027076062]$ from §5.2.

```
prec = 300!;
C2 = 0x1.ffffffffffdbdp-2; C3 = 0x1.555555555543cp-3;
C4 = 0x1.55555cf172b91p-5; C5 = 0x1.1111167a4d017p-7;
p = x + C2*x^2 + C3*x^3 + C4*x^4 + C5*x^5;
sn = supnorm(p, exp(x)-1, [-0.0027076062; 0.0027076062], absolute, 2^(-30));
print("supnorm interval:", sn);
```

Output:

```
supnorm interval: [2.107373220630388324761992063610839415446540152952926592752613821346585609717294573783874512e-20;
                   2.10737322253169994535291047195736929004607953320699758735806158001477607356338278335857123e-20]
```

$$\boxed{ E_{\mathrm{approx}} \leq 2.1073733\times10^{-20} \approx 1.55497\cdot 2^{-66} }$$

---

## 7. Reduction Error $\varepsilon_r$

Let $t_2 = \mathrm{RN}(x + k_d\cdot \mathrm{hi})$ and $\hat{r} = \mathrm{RN}(t_2 + k_d\cdot \mathrm{lo})$, with exact counterpart $r_2 = x + k_d(\mathrm{hi} + \mathrm{lo})$.

```
# t2 = RN(x + kd*hi); r = RN(t2 + kd*lo)
@rnd = float<ieee_64,ne>;
@Int = fixed<0,dn>;
hi = -0x1.62e42fefa0000p-8;
lo = -0x1.cf79abc9e3b3ap-47;
kd = Int(kd_dummy);
x  = rnd(x_dummy);
t2 = rnd(x + kd*hi);
r  = rnd(t2 + kd*lo);
r2 = x + kd * (hi + lo);
{ ( kd in [-137630, 137630] /\ r2 in [-0.00270760618, 0.00270760618] )
 -> ( r - r2 in ? /\ r in ? /\ t2 - (r2 - kd*lo) in ? ) }
x + kd*hi -> r2 - kd*lo;
r - Mr2 -> (r - (t2 + kd*lo)) + (t2 - (x + kd*hi));
```

The last two lines are algebraic identities (hints). They show that the large-operand addition $x + k_d\cdot\mathrm{hi}$ cancels down to $r_2 - k_d\cdot\mathrm{lo}$, so its rounding error is proportional to the _small_ result rather than to the operands.

Output:

```
r in [-6243314781856795b-61 {-0.00270761, -2^(-8.52876)}, 6243314781856795b-61 {0.00270761, 2^(-8.52876)}]
r - r2 in [-1b-61 {-4.33681e-19, -2^(-61)}, 1b-61 {4.33681e-19, 2^(-61)}]
t2 - (r2 - kd * lo) in [-1b-62 {-2.1684e-19, -2^(-62)}, 1b-62 {2.1684e-19, 2^(-62)}]
```

So $\hat{r} \in [-0.0027076098, 0.0027076098]$ and $\hat{r} - r_2 \in \pm 2^{-61}$.

**Tightening $\hat{r}$.** The Gappa interval for $\hat{r}$ is wider than the Sollya interval used in §6. Instead use

$$\lvert \hat{r}\rvert \leq \lvert r_2\rvert_{\max} + \lvert \hat{r} - r_2\rvert \leq 0.00270760618 + 4.34\times10^{-19} < 0.0027076062 ,$$

which is exactly the interval fed to Sollya.

**Adding the constant residual.** Since $r = r_2 + k\rho$,

$$\boxed{ \varepsilon_r := \lvert \hat{r} - r\rvert \leq 2^{-61} + 137630\cdot 2^{-100} \leq 4.336811\times10^{-19} \approx 2^{-61.000} }$$

---

## 8. Evaluation Error $B_{\mathrm{eval}}$

The C expression

```c
tmp = tail + r + r2*(C2 + r*C3) + r2*r2*(C4 + r*C5);   // r2 = r*r
```

parses left-associatively into eleven roundings:

$$ \begin{aligned} &r_2 = \mathrm{RN}(r\cdot r), \quad s_1 = \mathrm{RN}(\mathrm{tail} + r), \quad p_2 = \mathrm{RN}(C_2 + r C_3), \quad s_2 = \mathrm{RN}(s_1 + r_2 p_2) \cr &q_1 = \mathrm{RN}(r_2\cdot r_2), \quad p_5 = \mathrm{RN}(C_4 + r C_5), \quad \widehat{\mathrm{tmp}} = \mathrm{RN}(s_2 + q_1 p_5) \end{aligned} $$

Writing the exact model as

$$\mathrm{Mtmp} = \bigl((\mathrm{tail}+r) + (r\cdot r)(C_2 + r C_3)\bigr) + \bigl((r\cdot r)(r\cdot r)\bigr)(C_4 + r C_5)$$

 algebraically identical to $\mathrm{tail} + p(r)$ by distributivity and associativity of real arithmetic — lets Gappa correlate each rounded node with its exact counterpart.

```
# r2=RN(r*r); s1=RN(tail+r); p2=RN(C2+r*C3); s2=RN(s1+r2*p2);
# q1=RN(r2*r2); p5=RN(C4+r*C5); tmp=RN(s2+q1*p5)
@rnd = float<ieee_64,ne>;
@Int = fixed<0,dn>;

hi = -0x1.62e42fefa0000p-8;
lo = -0x1.cf79abc9e3b3ap-47;
C2 = 0x1.ffffffffffdbdp-2;
C3 = 0x1.555555555543cp-3;
C4 = 0x1.55555cf172b91p-5;
C5 = 0x1.1111167a4d017p-7;

kd = Int(kd_dummy);
x  = rnd(x_dummy);

t2  = rnd(x + kd*hi);
r   = rnd(t2 + kd*lo);
Mr2 = x + kd * (hi + lo);

tail = rnd(tail_dummy);
r2 = rnd(r * r);
s1 = rnd(tail + r);
p2 = rnd(C2 + r*C3);
s2 = rnd(s1 + r2*p2);
q1 = rnd(r2 * r2);
p5 = rnd(C4 + r*C5);
tmp = rnd(s2 + q1*p5);

Mtmp = ((tail + r) + (r*r) * (C2 + r*C3)) + ((r*r)*(r*r)) * (C4 + r*C5);

{ ( kd in [-137630, 137630]
 /\ Mr2 in [-0.00270760618, 0.00270760618]
 /\ tail in [-9.4479e-17, 9.4479e-17] )
 ->
  ( tmp - Mtmp in ?  /\  tmp in ? )
}

x + kd*hi -> Mr2 - kd*lo;
t2 + kd*lo - Mr2 -> (t2 - (x + kd*hi)) + (x + kd*hi + kd*lo - Mr2);
```

Output:

```
tmp in [-3121661473367063b-60 {-0.00270761, -2^(-8.52876)}, 3125891410654019b-60 {0.00271128, 2^(-8.52681)}]
tmp - Mtmp in [-216378450416295001b-118 {-6.5114e-19, -2^(-60.4137)}, 216378450416295001b-118 {6.5114e-19, 2^(-60.4137)}]
```

$$\boxed{ B_{\mathrm{eval}} = \bigl\lvert \widehat{\mathrm{tmp}} - \mathrm{Mtmp} \bigr\rvert \leq 6.5114\times10^{-19} \approx 2^{-60.4137} }$$

---

## 9. Propagation and Cross Terms

### 9.1 Propagation: $\lvert p(\hat{r}) - p(r)\rvert \leq dP \cdot \varepsilon_r$

By the mean value theorem, with $dP = \max \lvert p'(x)\rvert$ over the reduction interval. Appending to the Sollya script of §6:

```
I = [-0.0027076062; 0.0027076062];

dp = diff(p);
dpmax = supnorm(dp, 0, I, absolute, 2^(-30));

print("sup |p'(x)| interval:", dpmax);
```

Output:

```
sup |p'(x)| interval: [1.0027112750762086079703294672071933746337890625;
                       1.002711275980873515323400114636175959703483229201737003677408210933208465576171875]
```

So $\lvert p'(x)\rvert \leq 1.0027113 =: dP$. This is confirmed by the triangle inequality with $R = R_{\mathrm{exact}}$:

$$\max \lvert p'(r)\rvert \leq 1 + 2C_2 R + 3C_3 R^2 + 4C_4 R^3 + 5C_5 R^4 \leq 1.0027113 = dP .$$

Hence

$$\mathrm{prop} \leq dP \cdot \varepsilon_r = 1.0027113 \times 4.336811\times10^{-19} = 4.348569\times10^{-19}.$$

### 9.2 Cross term

$$\lvert \mathrm{tail} (e^{r}-1)\rvert \leq \max_i\lvert \mathrm{tail}_i\rvert \cdot \left(e^{R}-1\right), \qquad e^{0.00270761} - 1 \leq 0.0027113 ,$$

$$\mathrm{cross} \leq 9.4479\times10^{-17} \times 0.0027113 = 2.561609\times10^{-19}.$$

---

## 10. Total Error of $\mathrm{tmp}$

Define the exact target

$$r_{\mathrm{tmp}} := (1+\mathrm{tail})e^{r} - 1 = \mathrm{tail} + (e^{r}-1) + \mathrm{tail} (e^{r}-1),$$

so that

$$\mathrm{scale} (1 + r_{\mathrm{tmp}}) = \mathrm{scale} (1+\mathrm{tail}) e^{r} \approx 2^{k/N}e^{r} .$$

Then

$$ \widehat{\mathrm{tmp}} - r_{\mathrm{tmp}} = \underbrace{\bigl[\widehat{\mathrm{tmp}} - (\mathrm{tail} + p(\hat{r}))\bigr]}_{B_{\mathrm{eval}}} + \underbrace{\bigl[p(\hat{r}) - p(r)\bigr]}_{\mathrm{prop}} + \underbrace{\bigl[p(r) - (e^{r}-1)\bigr]}_{E_{\mathrm{approx}}} - \underbrace{\mathrm{tail} (e^{r}-1)}_{\mathrm{cross}} $$

|Contribution|Value|
|---|---|
|$B_{\mathrm{eval}}$|$6.5114\times10^{-19}$|
|$\mathrm{prop}$|$4.348569\times10^{-19}$|
|$E_{\mathrm{approx}}$|$2.1073733\times10^{-20}$|
|$\mathrm{cross}$|$2.561609\times10^{-19}$|
|**$\varepsilon_{\mathrm{tmp}}$**|**$1.363231\times10^{-18}$**|

---

## 11. Final Bound (Main Path)

Let

$$y = \mathrm{RN}(w), \qquad w = \mathrm{scale} + \mathrm{scale}\cdot\widehat{\mathrm{tmp}}, \qquad t = e^{x}.$$

Split as

$$\lvert w - t\rvert \leq \bigl\lvert w - (\mathrm{scale} + \mathrm{scale}\cdot r_{\mathrm{tmp}})\bigr\rvert + \bigl\lvert (\mathrm{scale} + \mathrm{scale}\cdot r_{\mathrm{tmp}}) - t \bigr\rvert .$$

The first term is $\mathrm{scale}\cdot\varepsilon_{\mathrm{tmp}}$ from §10. For the second, using $\mathrm{scale}(1+\mathrm{tail}) = 2^{k/N}(1+\varepsilon_{\mathrm{tab}})$ and $e^{x} = 2^{k/N}e^{r}$:

$$\bigl\lvert \mathrm{scale}(1+\mathrm{tail})e^{r} - e^{x}\bigr\rvert = \bigl\lvert 2^{k/N}(1+\varepsilon_{\mathrm{tab}})e^{r} - e^{x}\bigr\rvert = e^{x}\bigl\lvert 1 - (1+\varepsilon_{\mathrm{tab}})\bigr\rvert .$$

Hence

$$D := \lvert w - t\rvert \leq \mathrm{scale}\cdot\varepsilon_{\mathrm{tmp}} + t \varepsilon_{\mathrm{tab}} .$$

### 11.1 Comparing $\mathrm{ulp}(t)$ with $\mathrm{ulp}(\mathrm{scale})$

We want $D/\mathrm{ulp}(t)$, where $t = e^{x} = \mathrm{scale} (1+r_{\mathrm{tmp}})(1+\varepsilon_{\mathrm{tab}})^{-1}$. Let $\mathrm{scale}\in[2^{m}, 2^{m+1})$. Two cases:

**Case $t \geq 2^{m}$**, so $\mathrm{ulp}(t) \geq 2^{m-52}$:

$$\frac{\mathrm{scale}\cdot\varepsilon_{\mathrm{tmp}}}{\mathrm{ulp}(t)} \leq \frac{2^{m+1}\varepsilon_{\mathrm{tmp}}}{2^{m-52}} = \varepsilon_{\mathrm{tmp}}\cdot 2^{53}.$$

**Case $t < 2^{m}$**, so $\mathrm{ulp}(t) \leq 2^{m-53}$. Here

$$\mathrm{scale} \leq \frac{t (1+\varepsilon_{\mathrm{tab}})}{1 - 0.00270761} \leq 2^{m}\cdot 1.00271,$$

and since $t/\mathrm{ulp}(t) < 2^{53}$,

$$\frac{\mathrm{scale}\cdot\varepsilon_{\mathrm{tmp}}}{\mathrm{ulp}(t)} \leq 1.00271 \varepsilon_{\mathrm{tmp}} \frac{t}{\mathrm{ulp}(t)} \leq 1.00271 \varepsilon_{\mathrm{tmp}}\cdot 2^{53}.$$

### 11.2 Result

$$\frac{D}{\mathrm{ulp}(t)} \leq 1.00271 \varepsilon_{\mathrm{tmp}}\cdot 2^{53} + \varepsilon_{\mathrm{tab}}\cdot 2^{53} \approx 0.0123 .$$

The final rounding from $w$ to $y$ adds at most $\tfrac{1}{2}\mathrm{ulp}(t)$, giving

$$\boxed{ E(x) \leq 0.5 + \frac{D}{\mathrm{ulp}(t)} = 0.5123 }$$

The code comment claims $0.5 + 1.11/N \approx 0.5087$, so this result is slightly looser than the documented bound.

---

## 12. Special Cases

Three branches follow the scaling step.

1. **$k > 0$.** The code only manipulates the exponent of the result, so the error is unchanged from the main path.
2. **$k < 0$ and $y \geq 1$.** Same reasoning — exponent manipulation only.
3. **$k < 0$ and $y < 1$ (subnormal output).** Analyzed below.

### 12.1 The subnormal branch

```cpp
y  = scale + scale*tmp;
lo = scale - y + scale * tmp;
hi = 1.0 + y;
lo = 1.0 - hi + y + lo;
y  = (hi + lo) - 1.0;
return 0x1p-1022 * y;
```

**Step 1 - Fast2Sum on $(\mathrm{scale}, \mathrm{scale}\cdot\mathrm{tmp})$.** Let $a = \mathrm{scale}$, $b = \mathrm{scale}\cdot\mathrm{tmp}$, $s = a + b$, and $\mathrm{lo} = a - s + b$. Since $z = \mathrm{RN}(s - a)$ is exactly representable (Dekker), we get $\mathrm{lo} = t = \mathrm{RN}(b - z)$ and

$$a + b = s + t = y + \mathrm{lo} \qquad \text{exactly.}$$

**Step 2 - Fast2Sum on $(1, y)$.** Because $0 < y < 1$, with $e = 1.0 - \mathrm{hi} + y$ we again get

$$1 + y = \mathrm{hi} + e \qquad \text{exactly}, \qquad \lvert e\rvert \leq \tfrac{1}{2}\mathrm{ulp}(\mathrm{hi}) \leq 2^{-53}.$$

**Step 3 - Combining the low parts.** Since $\lvert \mathrm{lo}\rvert \leq \tfrac{1}{2}\mathrm{ulp}(y) \leq 2^{-54}$,

$$\lvert e + \mathrm{lo}\rvert \leq 2^{-53} + 2^{-54} \leq 2^{-52},$$

so for $\mathrm{lo}_2 = \mathrm{RN}(e + \mathrm{lo})$ the residual satisfies

$$\lvert e_2\rvert = \lvert \mathrm{lo}_2 - (e + \mathrm{lo})\rvert \leq \tfrac{1}{2}\mathrm{ulp}(e+\mathrm{lo}) \leq \tfrac{1}{2}\cdot 2^{-105} = 2^{-106},$$

and

$$\mathrm{hi} + \mathrm{lo}_2 = 1 + \mathrm{scale} + \mathrm{scale}\cdot\mathrm{tmp} + e_2 .$$

**Step 4 - Exactness of the final subtraction.** Let $v = \mathrm{RN}(\mathrm{hi} + \mathrm{lo}_2)$. Since $\mathrm{scale} + \mathrm{scale}\cdot\mathrm{tmp} > 0$ and $\lvert e_2\rvert \leq 2^{-106}$, we have $\mathrm{hi} + \mathrm{lo}_2 > 1 - 2^{-106}$, so the nearest floats are all $\geq 1$ and $< 2$, giving $v \in [1,2)$. Then $v - 1.0$ is exact by Sterbenz, and $v = 1.0 + j\cdot 2^{-52}$ yields $y = j\cdot 2^{-52}$ for an integer $j$. Finally

$$y_1 = v - 1.0 \Longrightarrow y_1 \cdot 2^{-1022} = j\cdot 2^{-1074} \quad \text{is representable.}$$

Therefore

$$\lvert y_1 - 2^{1022}e^x\rvert \leq \underbrace{\lvert v - (\mathrm{hi}+\mathrm{lo}_2)\rvert}_{\leq\,\frac12\mathrm{ulp}(v)\,=\,2^{-53}} + \lvert e_2\rvert + \lvert (\text{scale} + \text{scale} \cdot \text{tmp}) - 2^{1022}e^x\rvert \leq \tfrac12\cdot 2^{-52} + 2^{-106} + D .$$

### 12.2 Error on the subnormal branch

$$E(x)  = \frac{\lvert 2^{-1022}y_1 - e^x \rvert}{\text{ulp}(e^x)} = \frac{\lvert 2^{-1022}y_1 - e^x \rvert}{2^{-1074}} = \frac{\lvert y_1 - 2^{1022}e^x \rvert}{2^{-52}} \leq 0.5 + \frac{\mathrm{scale}\cdot\varepsilon_{\mathrm{tmp}} + \lvert e_2\rvert + (\varepsilon_{\mathrm{tab}}\text{ term})}{2^{-52}}$$

$$1.00271 \varepsilon_{\mathrm{tmp}}\cdot 2^{52} + \varepsilon_{\mathrm{tab}}\cdot 2^{52} \approx 0.006156$$

(the $\lvert e_2\rvert / 2^{-52} \leq 2^{-54}$ contribution is negligible), so

$$\boxed{ E(x) \leq 0.506156 \quad \text{on the subnormal branch.} }$$

---

## 13. Summary

| Quantity                                  | Bound                                     |
| ----------------------------------------- | ----------------------------------------- |
| $R_{\mathrm{exact}} = \max\lvert r\rvert$ | $0.00270760618$                           |
| $\varepsilon_r$ (reduction)               | $4.336811\times10^{-19} \approx 2^{-61}$  |
| $B_{\mathrm{eval}}$ (evaluation)          | $6.5114\times10^{-19} \approx 2^{-60.41}$ |
| $\mathrm{prop}$ (propagation)             | $4.348569\times10^{-19}$                  |
| $E_{\mathrm{approx}}$ (approximation)     | $2.1073733\times10^{-20}$                 |
| $\mathrm{cross}$ (dropped term)           | $2.561609\times10^{-19}$                  |
| $\varepsilon_{\mathrm{tmp}}$ (total)      | $1.363231\times10^{-18}$                  |
| $\varepsilon_{\mathrm{tab}}$ (table)      | $6.03\times10^{-33} < 2^{-107}$           |
| **$E(x)$, main path**                     | **$\leq 0.5123$**                         |
| **$E(x)$, subnormal branch**              | **$\leq 0.506156$**                       |
