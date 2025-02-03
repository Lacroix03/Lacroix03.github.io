---
title: [Reversing.kr] Easy Crack
date: 2025-02-03 20:42:00 +0900
categories: [Security]
tags: [Reversing.kr]
toc: true
toc_sticky: true
render_with_liquid: false
published : true
---
# Easy Crack

## 문제 내용

x32dbg로 파일을 열어 비밀 번호를 알아내는 간단한 문제.

## 문제 풀이

`f9` 를 눌러 실행해보면, easy_crackme.exe의 EP가 나온다. 여기서부터 차근 차근 `F8` 과 `F7` 을 눌러가며 어떤 동작을 하는지 살펴보자.

![Image](https://github.com/user-attachments/assets/90db6483-4828-4945-a8cc-39bfd5b1b7d4)

여러 번 실행시켜보니, 이 부분을 넘어갈 때 대화 상자가 뜬다. 401000 주소의 어셈블리를 잘 살펴보자.

![Image](https://github.com/user-attachments/assets/eb979d04-38d8-48de-87c1-c41de8848085)

**DialogBoxParamA**함수가 호출되고 있다. 대화 상자를 만드는 함수이다.

```cpp
INT_PTR DialogBoxParamA(
  [in, optional] HINSTANCE hInstance,
  [in]           LPCSTR    lpTemplateName,
  [in, optional] HWND      hWndParent,
  [in, optional] DLGPROC   lpDialogFunc,
  [in]           LPARAM    dwInitParam
);
[출처 : https://learn.microsoft.com/ko-kr/windows/win32/api/winuser/nf-winuser-dialogboxparama]
```

이 대화 상자의 입력 검증에 대한 동작은 **GetDlgItemText** 함수 호출 이후를 보면 된다. 대화 상자에 입력된 문자열을 가져오는 함수이다.

```cpp
UINT GetDlgItemTextA(
  [in]  HWND  hDlg,
  [in]  int   nIDDlgItem,
  [out] LPSTR lpString,
  [in]  int   cchMax
);
[출처 : https://learn.microsoft.com/ko-kr/windows/win32/api/winuser/nf-winuser-getdlgitemtexta]
```

![Image](https://github.com/user-attachments/assets/3e2bbac0-c991-4c8a-95a7-5aa2445d4282)

살펴보면 **GetDlgItemText** 함수가 호출되고 스택에 있는 바이트(esp+5, 즉 두번째로 입력된 문자)와 0x61(’a’)을 비교하고 있다. 서로 같으면 넘어가고, 같지 않으면 “Incorrect Password”를 띄우고 ret주소로 돌아간다.

![Image](https://github.com/user-attachments/assets/cd1b0b46-df79-40fe-9e6c-2150952d8d3b)

대화 상자에 ‘aaaa’를 입력한 다음 스택을 살펴보면, esp+4부터 차례대로 입력값이 들어간 것을 확인할 수 있다. 두번 째 문자가 a이므로 첫번째 검증은 통과된다.

![Image](https://github.com/user-attachments/assets/63a247e1-35ac-4503-b6bc-1f4a95c1514c)

두번째 검증 루틴은 esp+A와 ‘5y’가 일치하면 통과하는 루틴이다. 주소 401150에서는 문자 두 개를 서로 비교한다.

![Image](https://github.com/user-attachments/assets/03998edd-82a3-4329-aba0-7d322756d88d)

스택 상황을 살펴보면 esp+A는 입력 문자열의 세번째 문자부터이다. 따라서 비밀번호의 세번째, 네번째 문자는 ‘5y’이다.

![Image](https://github.com/user-attachments/assets/a0eb9f35-0884-4557-9880-89f9d5dbd8a4)

![Image](https://github.com/user-attachments/assets/8f113f67-7842-4736-a457-3ec8e09e2417)

세번째 검증 루틴은 esp+0x10부터 ‘R3versing’과 일치하면 통과하는 루틴이다. 

스택을 살펴보면 esp+0x10은 입력 문자열의 5번째 문자이다. 따라서 비밀번호의 다섯번째 문자열부터는 ‘R3versing’이다.

![Image](https://github.com/user-attachments/assets/25540ccc-1a53-40bf-9be0-23dbfa0499e6)

마지막, 네번째 검증 루틴은 esp+4가 ‘E’와 일치하면 통과하는 루틴이다. esp+4는 입력 문자열의 첫번째 문자이다.

따라서, 비밀번호는 ‘Ea5yR3versing’이다.