---
title: How does C implement exp()?
author: Equinox134
date: 2026-03-11
categories:
  - Math
  - Research
math: true
excerpt: A deep dive into the C implementation of the exp() function
---
Recently I've started doing a lab internship (sort of) at the Foundations of Programming & Computing Lab. As starters, I've started studying about floating-point numbers and looking at the implementations of math functions in `glibc`.

These functions are interesting, as they use floating-point numbers to calculate a continuous function as accurately as possible. Many tricks are used to achieve this, and while no one cares about this stuff when actually using the functions, I thought it was worth giving it some spotlight.

I'm still new to this, and have only really looked at the `exp()` function, but this should be enough to give some idea on how these continuous functions are calculate using floating-point numbers.

The implementations of the `glibc` math library can found [in this link](https://sourceware.org/git/?p=glibc.git;a=tree;f=sysdeps/ieee754;hb=HEAD). The function we will be looking at is `e_exp().c` under `dbl-64`, which basically means double(64 bit) precision.

## Overview

If you look at the code, there are two functions: `__exp()` and `specialcase()`. `__exp()` is the actual function that calculates $e^x$. `specialcase()` is a function that exists to take care of cases where calculations inside the `__exp()` function might underflow or overflow. Thus, we will start by looking at the `__exp()` function, then the `specialcase()` function.

Throughout the code, there are a lot of devices made to optimize the code in many ways. I won't focus too much on those, as I'm not familiar with them myself. Instead, I'd like to focus more on the logic and math of the code.

Before we start, I would like to give a brief explanation of floating-point numbers and the idea used to implement the function.

### Floating-Point Numbers in a Nutshell

A floating point number $x$ is represented as follows:

$$x = M \cdot \beta^{e-p+1}$$

Each of the numbers means the following:
* $\beta \geq 2$ is called the radix or base
* $p \geq 2$ is the precision
* $M$ is an integer such that $\vert M \vert \leq \beta^p - 1$
* $e$ is an integer such that $e_{\text{min}} \leq e \leq e_{\text{max}}$

Each of those values have names, but I won't go that deep here. Maybe someday I'll write about just floating point numbers.

It is easy to understand the above representation as the following:
* $\beta$ is the base of the number. Obviously, most computers nowadays use $\beta = 2$.
* $p$ represents the number of "significant digits" of $x$.
* $M$ represents the $p$ significant digits.
* $e$ represents where the decimal point is at.

It's basically the same as scientific notation. Also, the existence of $e$ makes it possible to change the location of the decimal point, hence the name "floating-point". Imagine a $p$ digit number, and think of $e$ as an indicator of where to put the decimal point.

In C, the `double` datatype uses the following values:
* $\beta = 2$
* $p = 53$, but because binary numbers in scientific notation always start with 1 (like $1.01101 \times 2^e$), we don't actually store 1 bit, resulting in this part using 52 bits
* $e_{\text{min}} = -1022$, $e_{\text{max}} = 1023$, this part uses 11 bits.

In addition to everything above, floating-point numbers use an extra bit for the sign. This results in a total of 1 + 11 + 52 = 64 bits, which is exactly the number of bits used in a `double` datatype.

On C, the IEEE-754 standard represents a floating-point number $x$ in binary like the following:

|                   | sign bit | exponent ($e$) | significand $m$                                      |
| ----------------- | -------- | -------------- | ---------------------------------------------------- |
| number of bits    | 1        | 11             | 52                                                   |
| example (-13.625) | 1        | 10000000010    | 1011010000000000000000000000000000000000000000000000 |

Note that $-13.625 = -1.101101 \times 2^3$ in binary, and that the exponent is shifted by 1023(= $e_{\text{max}}$) when being stored.

### The Basic Idea

Now I'll briefly mention the overall idea used to calculate $e^x$ (and many other math functions) with floating-point arithmetic. The basic idea is to use a polynomial expansion of the function (like the Taylor expansion, although in practice people use something else). This way, instead of having to compute a transcendental function, we can use simple additions and multiplications to calculate the function value.

However, this will lead to errors. Now, errors are inevitable, as floating-point numbers themselves cannot represent every single real number. Still, we would like to be as accurate as possible. At the same time, we don't want the calculation to take a million years either.

If we simply calculate the polynomial expansion for every input $x$, we would need to use a lot of factors in the expansion to accurately calculate it. More factors mean more time, so to avoid this, most math functions start by reducing the range of the input. In the case of $\sin x$, the input can be shrunken down to $[0,\frac{\pi}{2})$. We'll see the case of $e^x$ shortly. Shrinking the input down to a smaller range allows for easier and faster computation.

Now, with that in mind, let's start looking at the function.

## `__exp()`

Before the actual function, lets take a moment to look at the very top of the code. After a bunch of `#include` statements, there are a bunch of macro definitions.

```c
#define N (1 << EXP_TABLE_BITS)
#define InvLn2N __exp_data.invln2N
#define NegLn2hiN __exp_data.negln2hiN
#define NegLn2loN __exp_data.negln2loN
#define Shift __exp_data.shift
#define T __exp_data.tab
#define C2 __exp_data.poly[5 - EXP_POLY_ORDER]
#define C3 __exp_data.poly[6 - EXP_POLY_ORDER]
#define C4 __exp_data.poly[7 - EXP_POLY_ORDER]
#define C5 __exp_data.poly[8 - EXP_POLY_ORDER]
```

These are basically constants that will be used in calculation. The exact values are all stored in the `e_exp_data.c` file, along with some explanations.

Now lets start looking at the `__exp()` function. The start of the function looks like this.

```c
double
SECTION
__exp (double x)
{
  uint32_t abstop;
  uint64_t ki, idx, top, sbits;
  double kd, z, r, r2, scale, tail, tmp;
```

The `SECTION` macro is used for some kind optimization I don't know about. Other than that, `__exp()` has a single double as a parameter (as expected), and declares a bunch of variables. These variables will be used throughout the code, and I'll explain them when they are used.

### First `if` statement

After all the declarations, there is immediately an `if` statement waiting.

```c
abstop = top12 (x) & 0x7ff;
if (__glibc_unlikely (abstop - top12 (0x1p-54)
                    >= top12 (512.0) - top12 (0x1p-54)))
    {
      if (abstop - top12 (0x1p-54) >= 0x80000000)
        /* Avoid spurious underflow for tiny x.  */
        /* Note: 0 is common input.  */
        return WANT_ROUNDING ? 1.0 + x : 1.0;
      if (abstop >= top12 (1024.0))
        {
          if (asuint64 (x) == asuint64 (-INFINITY))
            return 0.0;
          if (abstop >= top12 (INFINITY))
            return 1.0 + x;
          if (asuint64 (x) >> 63)
            return __math_uflow (0);
          else
            return __math_oflow (0);
        }
      /* Large x is special cased below.  */
      abstop = 0;
    }

```

First, `abstop` is assigned the value `top12(x) & 0x7ff`. `top12(x)` is a macro defined previously, and looks like this.

```c
/* Top 12 bits of a double (sign and exponent bits).  */
static inline uint32_t
top12 (double x)
{
  return asuint64 (x) >> 52;
}
```

Basically, it takes the double $x$, treats the bits as a `uint64` type, and fetches the top 12 bits of $x$. As the comment says, this is to fetch the sign and exponent of $x$. As we've seen above, in `double`, the first bit represents the sign, and the following 11 bits represent the exponent.

`0x7ff` in binary (shown with 12 bits) is `011111111111`, so basically `abstop` stores the bits that represent the exponent in $x$.

Moving on, inside the `if` statement we are checking this condition.

```c
__glibc_unlikely (abstop - top12 (0x1p-54) >= top12 (512.0) - top12 (0x1p-54))
```

This part is basically checking if the exponent of `x` is too large or too small. If the value of `x` is too large, an overflow might occur in later computations, while an underflow could occur if `x` is too small. Therefore this `if` statement does a check to see whether or not this might happen, and does the necessary work if the conditions are met.

`top(0x1p-54)` means $2^{-54}$, and `top(512.0)` means 512. So this condition is checking if $\vert x \vert \leq 2^{-54}$ or $\vert x \vert \geq 512$. But, the way it is written doesn't make this immediately obvious. Instead of using a standard `||`, it is written as `abstop - MIN >= MAX - MIN`. This is, again, an optimization, as using an `or` statement would cause the CPU to check two separate branch conditions. To avoid this, the code makes use of underflow.

* If `x` is just a normal number, and `abstop` is between `MIN` and `MAX`, then obviously `abstop - MIN` would be smaller than `MAX - MIN` as `abstop` is smaller than `MAX`. Therefore the condition is false.
* If `x` is too large, and `abstop` is larger than `MAX`, then obviously the  `abstop - MIN >= MAX - MIN`, and the condition is true.
* If `x` is too small, and `abstop` is smaller than `MIN`, then `abstop - MIN` underflows, as both are unsigned integers, resulting in massive values. This result is larger than `MAX - MIN`, and therefore the condition becomes true.

Finally, the `__glibc_unlikely()` is a macro, again used for optimization, which tells the compiler that the chances of the given statement being false is very low (which is true). This can save CPU resources as the compiler doesn't have to waste resources to create a branch for that `if` statement.

Now lets look at the body of the `if` statement.

```c
if (abstop - top12 (0x1p-54) >= 0x80000000)
	/* Avoid spurious underflow for tiny x.  */
	/* Note: 0 is common input.  */
	return WANT_ROUNDING ? 1.0 + x : 1.0;
```

The first part is self-explanatory. The `abstop - top12 (0x1p-54) >= 0x80000000` is checking if $x$ is small, like explained before. In this case, we would like to avoid underflow. For very small $x$, $e^x \approx x+1$ which can be seen from the Taylor expansion, which is exactly what the `return` statement is doing.

In the `return` statement, the `WANT_ROUNDING` flag tells if the user is using a different rounding function. By default, the CPU uses Round to Nearest, and since $x$ is very small, it would simply round `x + 1` to 1, which is fast. However, IEEE-754 allows users to change the rounding model (ex, Round to $\infty$). In this case, `WANT ROUNDING` is true, and the program executes `1.0 + x` to use the correct rounding rules.

The next part looks like this.

```c
if (abstop >= top12 (1024.0))
	{
	  if (asuint64 (x) == asuint64 (-INFINITY))
		return 0.0;
	  if (abstop >= top12 (INFINITY))
		return 1.0 + x;
	  if (asuint64 (x) >> 63)
		return __math_uflow (0);
	  else
		return __math_oflow (0);
	}
```

This part checks for large $x$, specifically those larger than 1024. A `double` can store up to around $1.79 \times 10^{308}$. $e^{709}$ breaks this limit, so a value larger than 1024 is guaranteed to overflow.

The nested `if` statement checks the following.
* The first case checks if $x$ is exactly negative infinity, in which case it immediately returns 0, as $e^{-\infty} = 0$.
* The next case checks if $x$ is either positive infinity or `NaN`. This is possible because the IEEE-754 representation of both positive infinity and `NaN` consists of filling the exponent bit with 1s. In this case, the code simply return `1.0 + x`, as $1 + \infty = \infty$ and $1 + \text{NaN} = \text{NaN}$.
* The third case checks if $x$ is a large negative number (remember that the first bit of a `double` is the sign bit). These numbers (like $e^{-1500}$) are guaranteed to underflow the format, and thus calls `__math_uflow(0)`, which triggers the hardware underflow flag.
* The last case is the opposite of the third. If $x$ wasn't negative, then it's some large positive number. In this case it is guaranteed to overflow, so it calls `__math_oflow(0)`, which triggers the hardware overflow flag.

The final part of the `if` statement looks like this.

```c
/* Large x is special cased below.  */
abstop = 0;
```

The fact that the code reached this part tells that $x$ is large (between 512 and 1024), but are not guaranteed to overflow. So, the code sets a flag by setting `abstop` to 0, and handles them later in the function `specialcase()`.

Now that we've got this out of the way, we can now look at the main body of the function.

### Main Body

Lets start by looking at the beginning part.

```c
/* exp(x) = 2^(k/N) * exp(r), with exp(r) in [2^(-1/2N),2^(1/2N)].  */
/* x = ln2/N*k + r, with int k and r in [-ln2/2N, ln2/2N].  */
z = InvLn2N * x;
#if TOINT_INTRINSICS
	kd = roundtoint (z);
	ki = converttoint (z);
#else
	/* z - kd is in [-1, 1] in non-nearest rounding modes.  */
	kd = math_narrow_eval (z + Shift);
	ki = asuint64 (kd);
	kd -= Shift;
#endif
```

This is the range reduction part I talked about in the beginning. Like the comment says, rather than solving $e^x$ directly, we use the following identity.

$$e^x = e^{k \cdot \ln 2 / N + r} = (e^{\ln 2})^{k/N} \cdot e^r = 2^{k/N} \cdot e^r$$

Here, $N$ is a fixed constant, $k$ is an integer that represents how many chunks of $\frac{\ln 2}{N}$ fit into $x$, and $r$ is the remainder left over that is smaller than $\frac{\ln 2}{N}$. In the code above, `kd` is $k$ as a `double`, and `ki` is $k$ as a `uint64_t`.

If you remember the macro, $N$ is defined as `1 << EXP_TABLE_BITS`. This is to balance CPU resources calculations. `EXP_TABLE_BITS` is basically the size of the lookup table `T`, and if this size is large (meaning $N$ is large) it would use a lot of CPU resources, but because $N$ is large, $r$ becomes small, making calculation of polynomials quicker. The same goes the other way around. If you look up `e_exp_data.c` $N$ is set to 128.

The code snippet above is used to find $k$. It starts by multiplying `InvLn2N` (defined above, which has the value `0x1.71547652b82fep0 * N`), which is $\frac{N}{\ln 2}$ to find $z$. Obviously, the closest integer to $z$ would be $k$. To find this integer, the code either uses hardware instructions (the `TOINT_INTRINSICS`), or uses some other method.

If we have hardware instructions (meaning `TOINT_INTRINSICS` is true), then we can simply use those instructions (`roundtoint()` and `converttoint()`) to earn $k$.

Otherwise, we have to manually find $k$. This part is really hacky, and uses something called a `Shift` trick. The basic idea to round $z$ is to add a large number `Shift`, then subtract `Shift`. This might look weird, but is actually quite smart.

When simply using the casting operator `(int)`, the CPU pauses, and does some stuff, which takes several CPU cycles. Since `glibc` is optimized as hell, they avoid this by performing floating-point addition, which is extremely fast. When adding a large number `Shift` (which is defined as `0x1.8p52`), the CPU must align the decimal points. Because `Shift` is large, the CPU has to shift the bits of $z$ 52 spaces to the right. This causes the decimal part of $z$ to be pushed out of the CPUs memory, leaving only the integer part, automatically rounding $z$. Then, by subtracting `Shift`, we are left with the rounded $z$, which is $k$.

The function `math_narrow_eval()` tells the CPU to strictly round the calculation to a standard 64 bit `double`, and not to do any other weird tricks. Also, `ki` is given the value `asuint64(kd)` right after `Shift` is added. Since `Shift` is around $2^{52}$, the actual integer part of $z$ sits at the very bottom bits of the 64 `double`. Therefore, just grabbing those raw bits give us the integer version of $k$.

Next up, we have to find the remainder $r$.

```c
r = x + kd * NegLn2hiN + kd * NegLn2loN;
```

The naive way to calculate $r$ is by using $r = x - k \cdot \frac{\ln 2}{N}$. However, because of the way things are set up, the values of $x$ and $k \cdot \frac{\ln 2}{N}$ are extremely close together. In floating-point arithmetic, subtracting two very close floating-point numbers can cause catastrophic cancellation, which results in a massive loss of precision. To avoid this, the value $\frac{\ln 2}{N}$ is split into to values `hi` and `lo`. More specifically, `NegLn2hiN + NegLn2loN = -ln2/N`. The code then performs the calculation $x - k \cdot \frac{\ln 2}{N}$ in two steps, keeping $r$ accurate.

To understand this part, lets think of an example. Imagine that your computer can hold up to only 4 significant digits. Say you want to perform the calculation $x - C$, where $x = 3.142$ and $C = 3.1415926$. The actual result is $0.0004074$, which should be possible to have on out computer, but since the computer can only hold 4 significant digits, $C$ is treated as $3.142$ (in Round to Nearest), and the result becomes $0.000$, which is terrible.

Now, imagine instead of performing the calculation directly, we perform it in two steps after splitting $C$:
* Split $C$ into two parts: $C_{lo} = 3.141$ and $C_{hi} = 0.0005926$. Since the computer can hold 4 significant digits, this is valid.
* Start with $x - C_{lo} = 3.142 - 3.141$. We get $0.001$.
* Now subtract $C_{hi}$. We get $0.001 - 0.0005926 = 0.00004074$, Which is accurate.

This is basically what is going on. `NegLn2hiN` holds the first 53 bits of the constant $-\frac{\ln 2}{N}$ (stored as `-0x1.62e42fefa0000p-8`), while `NegLn2loN` holds the bits 53 to 106 of $-\frac{\ln 2}{N}$ (stored as `-0x1.cf79abc9e3b3ap-47`). This way, we can simulate 106 bit precision, keeping $r$ accurate.

With $r$ found, we have to calculate $2^{k/N}$.

```c
/* 2^(k/N) ~= scale * (1 + tail).  */
idx = 2 * (ki % N);
top = ki << (52 - EXP_TABLE_BITS);
tail = asdouble (T[idx]);
/* This is only a valid scale when -1023*N < k < 1024*N.  */
sbits = T[idx + 1] + top;
```

To calculate $2^{k/N}$, the code splits $k/N$ into an integer part, and a fractional part. The integer part becomes the main exponent of the floating-point answer we want, while the fractional part is found in a precomputed look up table.

First, we need to know the fractional remainder of $k/N$. This is done using the modulo operator, and the value is used as the index in the lookup table. 2 is multiplied as the table `T` stores pairs of values. For any index, `T[idx]` stores something called the tail, and `T[idx+1]` holds bits for the `scale` multiplier.

The next line has `top = ki << (52 - EXP_TABLE_BITS)`. This part is really smart. Remember how the exponent part of a `double` starts at bit 52? Well, were doing exactly that, by shifting `ki` (the integer $k$) to the right by 52, it lands exactly at the exponent part, simulating $2^k$ in `double`. However, we want $2^{\lfloor k/N \rfloor}$. This is also easy to do, as $N$ is defined as `1 << EXP_TABLE_BITS`, meaning $\lfloor k/N \rfloor$ is literally just bit-shifting $k$ `EXP_TABLE_BITS` to the right. Thus, combining these operations, we are left with `ki << (52 - EXP_TABLE_BITS)`, which stores the correct bits into the 11 bit exponent field of the `double` precision format without having to use any rounding operations and just integer arithmetic. Absolute genius.

Now we find the remaining part of $2^{k/N}$. The exact value has too many decimal places, so just like before when calculating $r$, we store some extra bits in `tail` that will be used later. The values are stored in the table `T` as raw bits, so we convert it to `double` via `asdouble()`.

Finally, the code builds the final floating-point multiplier (the `scale`). `T[idx + 1]` contains the precomputed bits for $2^{\text{fractional part}}$, and `top` contains the bits for $2^{\text{integer part}}$ (sitting in the exponent part) we calculated earlier. Adding these together we end up with the exact `double` bit representation of $2^{k/N}$ in `sbits` (short for scale bits). Later, we can use `asdouble()` to use this as a `double`. Again, no floating-point arithmetic needed.

As a side not, as the comment in the code says, this only works when $-1023N < k < 1024N$, which should be true, since in the very first `if` statement in the code, we already took care of $x$ values outside the range $[-1023,  1024]$.

Now, with everything in place, we just have to stitch everything together.

```c
/* exp(x) = 2^(k/N) * exp(r) ~= scale + scale * (tail + exp(r) - 1).  */
/* Evaluation is optimized assuming superscalar pipelined execution.  */
r2 = r * r;
/* Without fma the worst case error is 0.25/N ulp larger.  */
/* Worst case error is less than 0.5+1.11/N+(abs poly error * 2^53) ulp.  */
tmp = tail + r + r2 * (C2 + r * C3) + r2 * r2 * (C4 + r * C5);
```

We first have to find $e^r$. However, because we set up $r$ to be in the range $[-\frac{\ln 2}{N}, \frac{\ln 2}{N}]$, it is extremely small, we can use a basic polynomial series to approximate $e^r$. 

Recall the Taylor series for $e^x$.

$$e^r = 1 + r + \frac{r^2}{2!} + \frac{r^3}{3!} + \cdots$$

The constants `C2`, `C3`, `C4`, `C5` essentially store these coefficients (for example, $\frac{1}{2}$, $\frac{1}{6}$, $\frac{1}{24}$, $\frac{1}{120}$ in the Taylor series ) of the polynomial series. More precisely, the values that are stored comes from the minimax approximation algorithm (or $L^{\infty}$ approximation), which gives a polynomial of degree $n$ that approximates a function $f$.  This is because the minimax approximation gives a better approximation than the truncated Taylor series. I'll omit the details of what a minimax approximation actually is, as I don't know to well myself, and this post is already way to long.

One can easily confirm that the line where `tmp` is calculated is the same as evaluating the polynomial. However, you can also notice that a +1 is being omitted. This is because adding 1.0 to a tiny fraction overwrites the lowest bits of precision. Therefore, this 1 is left out until the last moment to preserve accuracy.

Lets take a closer look at how the polynomial is actually calculated. Modern CPUs have multiple math units, meaning they can do multiple calculations at once. The way the code is written abuses this.

By calculating `r2 = r * r` and grouping the terms into `(C2 + r * C3)` and `(C4 + r * C5)`, the CPU can run all of these operations at the same time, saving time. Simply laying out the full polynomial will cause too many multiplications of $r$, and something like Horner's Method cannot be run in parallel, so this is a good balance.

Now lets look at `tail`. `tail` was the leftover bits of $2^{k/N}$ that didn't fit into the standard bits. Ideally, we want to calculate the following.

$$(1 + \text{tail}) \cdot e^r$$

Since $e^r \approx 1 + P(r)$, where $P(r)$ is the polynomial $r + C_2 r^2 + \cdots$, plugging this above and expanding it gives us

$$1 + P(r) + \text{tail} + (\text{tail} \cdot P(r))$$

However, both `tail` and $P(r)$ are small values, so their multiplication results in something extremely small that will go outside the limit of `double`, so we can just ignore it. Thus we are just left with $1 + \text{tail} + P(r)$, and since we are going to add the 1 at the very end, we have $\text{tail} + P(r)$, which is precisely what `tmp` holds.

Now, we just have to do one final operation.

```c
scale = asdouble (sbits);
/* Note: tmp == 0 or |tmp| > 2^-65 and scale > 2^-739, so there
 is no spurious underflow here even without fma.  */
return scale + scale * tmp;
```

`sbits` held the raw bits of `scale`, so know we just have to change that to a double. Putting in $\text{tail} + P(r)$ instead of `tmp`, we get $\text{scale} + \text{scale}(\text{tail} + P(r)) = \text{scale}(1 + \text{tail} + P(r))$, which perfectly gives us our goal

$$2^{k/N} \cdot e^r$$

By doing `scale + scale * tmp` instead of `scale * (1 + tmp)` we can delay the 1 being added to some small fraction until the `scale` multiplier made it larger. In addition, if there exists FMA (Fused Multiply Add), a calculation like `a + b * c` can be done in one operation, making it more accurate (this is what the comment is concerning).

And that is how C calculates $e^x$. But wait, we aren't done yet. There is one snippet I skipped over.

```c
if (__glibc_unlikely (abstop == 0))
    return specialcase (tmp, sbits, ki);
```

Remember at the very beginning we set a flag by setting `abstop` to 0 when $x$ was large but not too large to guarantee an overflow? Know is the time address that issue, by calling the function `specialcase()`.

## `specialcase()`

Lets start with looking at the kindly written comment above this function.

```c
/* Handle cases that may overflow or underflow when computing the result that
   is scale*(1+TMP) without intermediate rounding.  The bit representation of
   scale is in SBITS, however it has a computed exponent that may have
   overflown into the sign bit so that needs to be adjusted before using it as
   a double.  (int32_t)KI is the k used in the argument reduction and exponent
   adjustment of scale, positive k here means the result may overflow and
   negative k means the result may underflow.  */
```

Just as the comment says, in the case that $x$ is large, when calculating `sbits`, the exponent that we computed via bit-shifting could have leaked into the sign bit of the `double` representation. The `specialcase()` function here is to fix this before returning the final result.

Lets now look at the function.

```c
static inline double
specialcase (double tmp, uint64_t sbits, uint64_t ki)
{
  double scale, y;
```

The function take in as a parameter three `double`s `tmp`, `sbits`, and `ki`. These are the same stuff we used in the `__exp()` function. Then it declares two variables.

```c
if ((ki & 0x80000000) == 0)
    {
      /* k > 0, the exponent of scale might have overflowed by <= 460.  */
      sbits -= 1009ull << 52;
      scale = asdouble (sbits);
      y = 0x1p1009 * (scale + scale * tmp);
      return check_oflow (y);
    }
```

The condition of this `if` statement, `(ki & 0x80000000)`, is checking the 32nd bit of `ki`, which represents the sign of $k$. Now, `ki` has a type of `uint64_t`, so this might be weird, but as the comment above says, because $x$ was in a certain range, $k$ fits nicely into the 32bits, meaning `(int32_t)ki` is basically the same as $k$. Therefore the 32nd bit can act as the sign bit.

In the case the $k > 0$, or in other words, the `if` statement is true, the exponent of `scale` could have overflowed, as the comment suggests. Therefore, before the CPU looks at `sbits` as a floating-point number, the code shrinks it as an integer.

The line `sbits -= 1009ull << 52` is like the one we used to calculate `top`. We push the value 1009 to the exponent field and subtract it, making dangerous exponents (ex, 1050), down to something safe (ex, 41). Now, it can safely use `scale = asdouble (sbits)` to use `sbits` as a `double`.

Then, it finishes the calculation and scales the answer back up to its original size. The `scale + scale * tmp` is exactly the same as before, calculating $2^{k/N} \cdot e^r$ using the safely shrunken scale. Then it multiplies `0x1p1009`, which is exactly $2^{1009}$ to restore the exponent we subtracted earlier.

Doing the calculations this way, we can let the CPUs dedicated floating-point math unit to handle the final multiplication (`* 0x1p1009`). This way, if the result overflows and becomes larger than the limit $1.8 \times 10^{308}$, the hardware will catch it, raise the official exception flag, and return `+INFINITY`. The `check_oflow(y)` makes sure that the flag is properly registered.

Now lets look at the rest of the function.

```c
/* k < 0, need special care in the subnormal range.  */
sbits += 1022ull << 52;
scale = asdouble (sbits);
y = scale + scale * tmp;
```

This is the opposite case, where $k < 0$. In this case, the calculated value can be extremely small and drop into something called the subnormal range.

Remember at the beginning when I was explaining floating-point numbers how binary numbers in scientific notation always start with 0? Well, in the subnormal range (the range $0$ to $\beta^{e_{\text{min}}}$), this rule is dropped to represent tiny fractions at the const of reduced precision. If possible, we would like to avoid falling into this range.

The code snippet above is essentially doing the same thing as when $k > 0$, but opposite, by adding 1022 to the exponent rather than subtracting. Then `scale = asdouble (sbits)` can be safely done, along with `y = scale + scale * tmp`. So, $y$ stores $2^{k/N} \cdot e^r$ for the reduced exponent.

The next part contains another `if` statement. Lets look at it bit by bit.

```c
if (y < 1.0)
    {
      /* Round y to the right precision before scaling it into the subnormal
         range to avoid double rounding that can cause 0.5+E/2 ulp error where
         E is the worst-case ulp error outside the subnormal range.  So this
         is only useful if the goal is better than 1 ulp worst-case error.  */
      double hi, lo;
      lo = scale - y + scale * tmp;
```

The condition in the `if` statement checks if $y < 1$. If $y$ is smaller than 1, then when we reduce the exponent by 1022, the result is guaranteed to fall into the subnormal range. In this case, the hardware calculates a math equation (which is round to 53 bits), and then it is scaled down to the subnormal range, the hardware has to round the result a second time as subnormal numbers have fewer bits. These two separate rounding can stack and give a larger error.

For example, imagine you want to round 2.49 into an integer. The obvious answer is 2. However, if you round to the first decimal first, then round to an integer, you will first get 2.5, then 3, which gives a large error. In the actual code, the machine first rounding to make the number fit 53 bits is equivalent to the process of rounding 2.49 to 2.5. Then dropping it to the subnormal range causes a second rounding with less bits, which is equivalent to rounding 2.5 to 3.

Therefore, the code uses a trick (similar to the trick used in something called Fast2Sum) to prevent this. First, `lo = scale - y + scale * tmp` is used to calculate the exact error (the tiny fractional bits) that were lost when we calculated $y$.

Moving on to the next part.

```c
hi = 1.0 + y;
lo = 1.0 - hi + y + lo;
y = math_narrow_eval (hi + lo) - 1.0;
```

This looks very weird at a glance, because if this were just regular math, then `lo = 1.0 - hi + y + lo` is basically just adding `1.0 + y` to `lo`, then subtracting `1.0 + y` again, which s very strange. However, this again, is an extremely cleverly thought out piece of code.

As I've said earlier, simply dropping the exponent down will cause $y$ to fall into the subnormal range, causing a double rounding. Instead, it would be better to round the number once, at the location where the subnormal range will cut it off, before it actually falls into the subnormal range. The process of adding `1.0` is doing exactly this.

Since $y$ has a small value, `1.0` is huge compared to $y$. To add these two, the CPU has to align their decimal points. If you recall how floating-point numbers are represented, `1.0` takes up the very top bit of the 53 bit register, causing the bits of $y$ to be shifted to the right. Here, surprisingly, the number of bits $y$ has been pushed to right so the number of bits $y$ has moved is exactly the number of bits $y$ will eventually lose in the subnormal range. So that is what `hi` is doing.

Now lets look at `lo = 1.0 - hi + y + lo`. Remember how `lo` stored the error that occurred when calculating $y$. The first part `1.0 - hi` is basically taking the truncated version of $y$ we made and making it negative. By adding $y$ here, we are left with the exact error between $y$ and the truncated version of $y$. Finally, by adding `lo`, we restore the error $y$ had, which is possible because by subtracting the truncated $y$, the number should use less bits.

The final line `y = math_narrow_eval (hi + lo) - 1.0` is basically putting this all together. `hi` can be thought of as $1.0 + y_{\text{rounded}}$, and `lo` can now be thought of as the difference between $y$ and $y_{\text{rounded}}$. Therefore by adding `hi` and `lo`, we can get a precise value of $1.0 + y$. The `math_narrow_eval()` is the same as we saw before, which crushes the result down into 53 bits, performing exactly one rounding operation.

Since `hi` still contains the `1.0` anchor, the CPUs decimal point is still locked in place. Therefore when we round the calculation down to 53 bits, the result is chopped of at the exact location required for a subnormal number. Finally, we have a perfectly rounded `double`, and the code subtracts `1.0` that acted as the anchor.

Therefore, what is left in $y$ is a small fraction, perfectly calculated and perfectly rounded according to the subnormal range, ready to be multiplied by $2^{-1022}$ and be safely dropped into the subnormal range without losing any extra precision.

There is one small last part before exiting the `if` statement.

```c
/* Avoid -0.0 with downward rounding.  */
if (WANT_ROUNDING && y == 0.0)
y = 0.0;
/* The underflow exception needs to be signaled explicitly.  */
math_force_eval (math_opt_barrier (0x1p-1022) * 0x1p-1022);
```

The first part is about rounding. Like I've said before, the default rounding mode is Round to Nearest. However, if the user is using another rounding mode, specifically Round to $-\infty$, things can go a little wrong. In math 0 is 0, but in the IEEE-754 standard for floating-point numbers, there exists `+0.0` and `-0.0`. The result of $e^x$ is always positive, but if the rounding mode is Round to $-\infty$, then when the CPU has to round to 0, it might actually become `-0.0`. This first `if` statement is there to prevent this, checking if a custom rounding mode is used and if $y = 0$, in which case it will overwrite $y$ with a hardcoded `+0.0`.

The last line is there to flag an alarm. The fact that we are in this `if` statement is because the final answer of $e^x$ is dropped into the subnormal range, which means it is permanently losing precision. According to the IEEE-754 standard, the library must flag an alarm `FE_UNDERFLOW` so that the program knows it lost accuracy.

To do this the code calculates $2^{-1022} \times 2^{-1022}$, where $2^{-1022}$ is the smallest normal floating-point number. The result, $2^{-2044}$, is extremely small, and when the CPU calculates it, it fails, rounds to 0, and flags the `FE_UNDERFLOW` alarm.

The `math_force_eval()` and `math_opt_barrier()` is there to ensure that the flag is called. This is because for calculations like `0x1p-1022 * 0x1p-1022`, the super-optimized compiler might immediately change it to 0, causing the flag to not arise. `math_opt_barrier()` is saying to not pre-calculate whatever is inside the parenthesis, and `math_force_eval()` forces the compiler to carry out the calculation, thus raising the flag.

With that, this `if` statement is done, and we have only the final part left.

```c
y = 0x1p-1022 * y;
return check_uflow (y);
```

This is exactly the same as the part when $k > 0$. We previously raised the exponent by 1022, so we multiply $2^{-1022}$ to reduce this exponent back to normal. Then we return this value, while simultaneously checking for underflow via `check_uflow()`, which can happen as $y$ is small.

And just like that we are done. This was how the code handles dangerous values, and with that out of the way, we have seen the entire process of how $e^x$ is calculated in the `glibc` library.

## Epilogue

This is the first time I looked this deep into floating-point numbers, and looked at any implementation of any math function. After looking at it, I've got to say, this is crazy.

If you've read my blog, you know I do problem solving, or competitive programming. Floating-point numbers are avoided like crazy there, due to precision issues. So seeing something like this, a meticulously engineered piece of work to calculate a precise value of some real-valued function, is just mind breaking. The sheer amount of thought that goes into each line of the code to optimize both speed and accuracy is amazing, using loopholes and tricks I would have never thought about. Like, more people must know about this.

I don't plan on writing more about other math functions (unless there's something really interesting), but I probably will look at more of them because of my lab intern stuff. Hopefully this gave you a grasp on how math functions are implemented using floating-point numbers, and interested you into looking at more yourself.

Anyways, that's all from me, cheers.
