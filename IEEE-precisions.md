This is a document describing useful information about IEEE 754 floating point standard.

### Round-off error

The error introduced depends on the number of bits used for the significand `s`, and the rounding mode used:

- with round-to-nearest mode, an additional bit is added and the error is **2<sup>-(s+1)</sup>**
- with round-to-zero mode, the error is **2<sup>-s</sup>**

### Range

The range depends on the number of exponent bits `e`.


### Table of useful types

<table>
  <tr><th>name<th>e<th>s<th>R2N round-off<th>R2Z round-off
  <tr><td>double<td>11<td>52<td><strong>1.11e-16</strong><td>2.22e-16
  <tr><td><td>11<td>20<td>4.77e-7<td><strong>9.54e-7</strong>
  <tr><td><td>11<td>4<td>0.03125<td><strong>0.0625</strong>
  <tr><td>single<td>8<td>23<td><strong>5.96e-8</strong><td>1.19e-7
  <tr><td><td>8<td>7<td>0.00391<td><strong>0.0078125</strong>
  <tr><td>half<td>7<td>10<td><strong>0.00048828125</strong><td>0.0009765625
</table>