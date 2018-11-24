---
layout: post
title: "Comparative Cheat Sheet: Python 3 and JavaScript (ECMAScript 6)"
date: 2018-11-13
---
## Primitive Types

<table border="1">
<tr><th width="50%"> Python </th><th> JavaScript </th></tr>
<tr>
  <td> <code><b>int</b></code>: <code>14 0 -22 0b101 0o64 0xA2B</code><br> No limit on number of digits.</td>
  <td rowspan="2"> <code><b>number</b></code>: <code>14 0 -22 1.2 -0.7e-23 Infinity NaN</code><br>
  64-bit IEEE 754 rep. on 64 bit systems for both integer and floats<br>integer value range: <code>Number.MIN_SAFE_INTEGER</code> - <code>Number.MAX_SAFE_INTEGER</code></td>
</tr>
<tr>
  <td><code><b>float</b></code>: <code>1.2 0.0 -0.7e-23</code><br>64 bit IEEE 754 rep.</td>
</tr>
<tr>
<td><code><b>bool</b></code>: <code>True False</code></td>
<td> <code><b>boolean</b></code>: <code>true false</code></td>
</tr>
<tr>
<td><code><b>str</b></code>: <code>"Hello\nWorld" '100\u20B9'</code><br> sequence of unicode code points<br>str.encode() => bytes</td>
<td><code><b>string</b></code>: <code>"Hello, World"</code><br>
a set of 16-bit value "elements"
Each element is a UTF-16 code unit.
</td>
</tr>
<tr>
<td><code><b>bytes</b></code>: <code>b"Hel\x6Co,Wor\154d"</code><br>sequence of 8-bit octets<br>bytes.decode() => str</td>
</tr>
<tr>
<td> </td><td><code><b>symbol</b></code></td>
</tr>
<tr>
<td> </td><td><code><b>undefined</b></code></td>
</tr>
<tr>
<td> <code>type(14)</code> => <code>&lt;class 'int'&gt;</code></td>
<td><code>typeof(14)</code> => <code>"number"</code></td>
</tr>
</table>
