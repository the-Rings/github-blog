---
title: 布尔代数
date: 2022-01-24 20:28:34
tags:
---

### 十进制小数转化为二进制数
```python
res, bit_count, max_length = 0, 0, 36
binary_number_str = "0."
while bit_count <= max_length:
    bit_count += 1
    if res + 2**(-bit_count) > 1/10:
        binary_number_str += "0"
    else:
        binary_number_str += "1"
        res += 2**(-bit_count)

print(binary_number_str, res)
```
