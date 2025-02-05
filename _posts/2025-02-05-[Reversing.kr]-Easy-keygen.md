---
title: "[Reversing.kr] Easy Keygen"
date: "2025-02-05 21:53:00 +0900"
categories:
  - Security
tags:
  - Reversing.kr
toc: true
toc_sticky: true
render_with_liquid: false
published: true
---
## 문제 내용

시리얼 넘버 ‘5B134977135E7D13’에 대응하는 Name을 찾는 문제.

## 문제 풀이

IDA로 문제 코드를 확인해보자.

![Image](https://github.com/user-attachments/assets/47f4ff23-9e37-4049-a7c6-2e40561d2ec6)

Name을 입력받아 v8에 저장하고, xor 연산을 거쳐 Buffer문자열에 16진수 형식으로 추가된다.

v7의 값은 어셈블리로 확인해보니,

![Image](https://github.com/user-attachments/assets/37774087-3f99-4060-8e79-6853bfaa2745)

0x10, 0x20, 0x30이 저장되어있다.

이 Buffer의 값과 시리얼 넘버(5B134977135E7D13)가 일치해야 한다.

xor의 결합법칙을 사용하여, 역산 코드를 python으로 작성해 Name을 찾을 수 있다.

```python
arr = [0x10, 0x20, 0x30]
serial = [0x5B,0x13,0x49,0x77,0x13,0x5E,0x7D,0x13]

key=''
for i in range(8):
    key+=chr(arr[i%3]^serial[i])
print(key)
```

플래그는 `K3yg3nm3`이다.