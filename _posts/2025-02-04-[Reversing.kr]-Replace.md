---
title: "[Reversing.kr] Replace"
date: "2025-02-04 20:20:00 +0900"
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

아래와 같은 창이 뜨고 숫자를 입력한다. 어셈블리를 분석해 Correct가 나오도록 하는 숫자를 찾아야 한다.

![Image](https://github.com/user-attachments/assets/0e933c2b-1e6a-41d0-9024-b425335dbc07)

## 문제 풀이

GetDlgItemInt 함수로 입력받는 숫자를 16진수로 eax에 저장한다.

eax에 저장된 숫자는 4084d0 주소에 저장된다. 그리고 40466f 주소를 호출한다. `F7` 을 눌러 따라가보자.

![Image](https://github.com/user-attachments/assets/b1e19d6d-6369-4acf-a398-f9b005d678eb)

따라가 보면 바로 40467a 주소를 호출한다.  마찬가지로 `F7` 을 눌러 따라가보자.
![Image](https://github.com/user-attachments/assets/18213f55-d119-4348-ad3b-f0af4dcbf1cd)

406016주소에 619060EB를 저장하고, 404689 주소를 호출한다. 

404689주소는 바로 아래에 있는 주소인데, 이렇게 되면 ret주소가 404689가 되므로 404689 명령어를 두 번 실행한다.

![Image](https://github.com/user-attachments/assets/fe0413d6-f67d-4571-b1a8-e559aa56461e)

`call 40466F` 의 동작을 정리해보면 4084d0, 즉 사용자가 입력한 숫자를 2 증가시킨다.

그 다음 동작을 살펴보자. 

4084d0에 0x601605c7를 더한다. 그리고 1씩 두 번 증가시킨다.

![Image](https://github.com/user-attachments/assets/f19d379c-b1ba-45d3-bdd1-1695b90dabe2)

eax에 4084d0의 값을 옮기고 404689 주소를 호출한다. 4084d0의 값을 1 증가시킨다.

![Image](https://github.com/user-attachments/assets/525be120-a089-4cfe-9d0a-103454f1abee)

![Image](https://github.com/user-attachments/assets/05acb75c-65ff-44e2-baf6-bb0cd7aa3ec6)

그리고 주소 40466f에 있는 값을 C39000C6으로 바꾼다.

![Image](https://github.com/user-attachments/assets/5c4d35ae-c27e-42b4-b343-eb9befcff5e1)

원래 40466f 주소에 있는 명령어는 `call 40467a` 였으나, `mov byte ptr ds:[eax], 90` 으로 바뀌었다.

90은 NOP에 해당하는 값이므로, eax가 가리키는 주소를 NOP로 바꾼다는 의미이다.

![Image](https://github.com/user-attachments/assets/f65480ab-7363-4f9a-8e3b-c4751c221ffd)

![Image](https://github.com/user-attachments/assets/6d7ab18c-752f-4696-9294-d643804c928c)

그리고 나서 40466f 주소를 2번 호출하여 eax가 가리키는 주소값의 명령어를 NOP 명령어로 replace하고, 401071 주소로 점프한다.

![Image](https://github.com/user-attachments/assets/fc56c1d3-d8fd-4318-83f8-fb3f14dda35a)

401071주소를 살펴보면 `jmp 401084` 이다. 이 부분을 NOP로 만들면 된다.

![Image](https://github.com/user-attachments/assets/3bf74aae-a172-4c31-ba32-71096f9a6ef5)

eax값은 4084d0의 값이므로, 최종 4084d0의 값이 401071이 되도록 만들면 된다.

`mov, eax dword ptr ds:[4084d0]` 이 될때, 4084d0의 값은 **[입력값]+1 * 4 + 0x601605c7**이었다.

역산해보면 입력값은 **0x100401071 - 0x601605c7 - 4 = 0xA02A0AA6(2687109798)**이다.

따라서 플래그는 2687109798이다.

![Image](https://github.com/user-attachments/assets/05f9908d-575a-4aca-b898-76373d85138b)