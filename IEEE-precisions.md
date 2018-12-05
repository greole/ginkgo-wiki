This is a document describing useful information about IEEE 754 floating point standard.

### Round-off error

The error introduced depends on the number of bits used for the significand **s**, and the rounding mode used:

- with round-to-nearest mode, an additional bit is added and the error is **2<sup>-(s+1)</sup>**
- with round-to-zero mode, the error is **2<sup>-s</sup>**

### Range

The range depends on the number of exponent bits **e**. The bits encode an unsigned integer **u**, and the fixed-point base 2 number **1.s...s** (where **s...s** is the significand) is multiplied by **2<sup>u-bias</sup>**, where **bias = 2<sup>e-1</sup> - 1**. The largest and the smallest representable values of **u** are reserved for infinity and denormalized numbers, so the largest possible exponent is reached for **u = 2<sup>e</sup> - 2**, thus **u - bias = 2<sup>e</sup> - 2<sup>e-1</sup> - 1 = 2<sup>e-1</sup> - 1**. Which means that the largest representable value is:


**max(e, s) = 2<sup>2<sup>e-1</sup> - 1</sup> * 1.1...1 = 2<sup>2<sup>e-1</sup> - 1</sup> * (2 - 2<sup>-s</sup>)**

The smallest exponent is reached for **u = 1**, so **u - bias = 1 - 2<sup>e-1</sup> + 1 = -(2<sup>e-1</sup> - 2)**. Thus, the smallest (normalized) representable value is reached for:

**min(e, s) = 2<sup>-(2<sup>e-1</sup> - 2)</sup> * 1.0...0 = 2<sup>-(2<sup>e-1</sup> - 2)</sup>**

### Table of useful properties

<table>
  <tr><th>name<th>#bits<th>e<th>s<th>R2N error<th>R2n digits<th>R2Z error<th>R2Z digits<th>min<th>max
  <tr><td>double<td>64<td>11<td>52<td><strong>1.11e-16</strong><td><strong>15.95</strong><td>2.22e-16<td>15.65<td>2.23e-308<td>1.80e+308
  <tr><td><td>32<td>11<td>20<td>4.77e-7<td>6.32<td><strong>9.54e-7</strong><td><strong>6.02</strong><td>2.23e-308<td>1.80e+308
  <tr><td><td>16<td>11<td>4<td>0.03125<td>1.51<td><strong>0.0625</strong><td><strong>1.20</strong><td>2.23e-308<td>1.74e+308
  <tr><td>single<td>32<td>8<td>23<td><strong>5.96e-8</strong><td><strong>7.22</strong><td>1.19e-7<td>6.92<td>1.18e-38<td>3.40e+38
  <tr><td><td>16<td>8<td>7<td>0.00391<td>2.41<td><strong>0.0078125</strong><td><strong>2.11</strong><td>1.18e-38<td>3.39e+38
  <tr><td>half<td>16<td>5<td>10<td><strong>0.00048828125</strong><td><strong>3.31</strong><td>0.0009765625<td>3.01<td>6.10e-5<td>6.55e+4
</table>