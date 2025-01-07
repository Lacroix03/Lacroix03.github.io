---
title: 21장. Windows 메시지 후킹
date: 2025-01-07 17:45:00 +0900
categories: [Security]
tags: [Reversing]
render_with_liquid: false
---
## Hook이란?

갈고리, 낚시바늘이라는 뜻을 가지고 있는 단어이다. 이에 따라 후킹은 무엇을 낚아챈다는 의미이다.

컴퓨터 분야에서 후킹(hooking)은 함수 호출, 메시지, 이벤트 등을 엿보고 조작하는 행위를 뜻한다.

그 중 가장 기본적인 형태인 메시지 후킹의 동작 원리를 알아보자.

## Message Hook

Windows 운영체제는 GUI(Graphic User Interface)를 제공하고, 이는 Event Driven 방식으로 동작한다.

<aside>
💡

event란, 프로그램에 의해 감지되고 처리될 수 있는 동작이나 사건이다. 

*ex. 키보드를 누르는 것, 마우스를 클릭하는 것, 화면에서 창을 움직이는 것 등*

따라서 event driven 방식은 프로그램의 실행 흐름이 에빈트에 의해 결정되는 소프트웨어 설계 패턴이다. 이벤트가 일어났을 때 반응하는 방식이라고 이해하면 된다.

</aside>

키보드 메시지가 처리되는 과정을 간단히 설명하면 다음과 같다.

1. 사용자가 키보드를 입력하면 Windows 운영체제는 이를 이벤트로(`WM_KEYDOWN`) 인식하고, 해당 이벤트를 메시지로 변환한다.
2. 이 메시지를 OS Message queue에 넣어 대기하다가, 해당 응용 프로그램의 Application message queue로 이동한다.
3. 응용 프로그램은 자신의 message queue를 모니터링하다가 `WM_KEYDOWN`메시지가 추가된 것을 확인하고 event handler를 호출한다.

**OS message queue와 Application message queue, 그 중간에서 메시지를 가로채는 것을 메시지 훅이라고 한다.**

메시지 훅은 SetWindowsHookEx() API를 사용해 간단히 구현할 수 있다.

## SetWindowsHookEx()

```cpp
HHOOK SetWindowsHookEx(
  [in] int       idHook, //type of hook(WH_KEYBOARD, WH_MOUSE etc...)
  [in] HOOKPROC  lpfn,   //pointer to the hook procedure
  [in] HINSTANCE hmod,   //handle to the DLL containing the hook procedure
  [in] DWORD     dwThreadId // ID of the thread to associate the hook with
													  // if 0, it is global hook.
);
[출처 : https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowshookexa]
```

이 때, hook procedure은 운영체제가 호출해주는 콜백 함수이다. 메시지 훅을 걸 때 hook procedure은 DLL 내부에 존재해야 하고, 그 DLL에 대한 handle이 세 번째 인자인 `hmod`이다.

이렇게 SetWindowsHookEx를 이용해 훅을 설치해 놓으면, 해당 메시지가 발생했을 때 OS가 해당 DLL 파일을 해당 프로세스에 강제로 인젝션 하고, 등록된 hook procedure를 호출한다.

(DLL을 강제로 인젝션한다는 것은 굉장히 중요한 포인트이다. 자세한 내용은 23장 DLL 인젝션에 나온다.)

## 키보드 메시지 후킹 실습

### HookMain.cpp

```cpp
#include "stdio.h"
#include "conio.h"
#include "windows.h"

#define	DEF_DLL_NAME		"KeyHook.dll"
#define	DEF_HOOKSTART		"HookStart"
#define	DEF_HOOKSTOP		"HookStop"

typedef void (*PFN_HOOKSTART)();
typedef void (*PFN_HOOKSTOP)();

void main()
{
	HMODULE			hDll = NULL;
	PFN_HOOKSTART	HookStart = NULL;
	PFN_HOOKSTOP	HookStop = NULL;
	char			ch = 0;

    // KeyHook.dll 로딩
	hDll = LoadLibraryA(DEF_DLL_NAME);
    if( hDll == NULL )
    {
        printf("LoadLibrary(%s) failed!!! [%d]", DEF_DLL_NAME, GetLastError());
        return;
    }

    // export 함수 주소 얻기
	HookStart = (PFN_HOOKSTART)GetProcAddress(hDll, DEF_HOOKSTART);
	HookStop = (PFN_HOOKSTOP)GetProcAddress(hDll, DEF_HOOKSTOP);

    // 후킹 시작
	HookStart();

    // 사용자가 'q' 를 입력할 때까지 대기
	printf("press 'q' to quit!\n");
	while( _getch() != 'q' )	;

    // 후킹 종료
	HookStop();
	
    // KeyHook.dll 언로딩
	FreeLibrary(hDll);
}
```

### KeyHook.cpp

```cpp
#include "stdio.h"
#include "windows.h"

#define DEF_PROCESS_NAME		"notepad.exe"

HINSTANCE g_hInstance = NULL;
HHOOK g_hHook = NULL;
HWND g_hWnd = NULL;

//[1] DLL Attach 확인 
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD dwReason, LPVOID lpvReserved)
{
	switch( dwReason )
	{
        case DLL_PROCESS_ATTACH:
			g_hInstance = hinstDLL;
			break;

        case DLL_PROCESS_DETACH:
			break;	
	}

	return TRUE;
}

//[2] hook procedure = KeyboardProc 콜백 함수
LRESULT CALLBACK KeyboardProc(int nCode, WPARAM wParam, LPARAM lParam)
{
	char szPath[MAX_PATH] = {0,};
	char *p = NULL;

	if( nCode >= 0 )
	{
		// bit 31 : 0 => press, 1 => release
		if( !(lParam & 0x80000000) ) // 키보드가 눌렀다 떨어질 때
		{
			GetModuleFileNameA(NULL, szPath, MAX_PATH);
			p = strrchr(szPath, '\\');

            // 현재 프로세스 이름을 비교해서 만약 notepad.exe 라면 0 아닌 값을 리턴함
            // => 0 아닌 값을 리턴하면 메시지는 다음으로 전달되지 않음
			if( !_stricmp(p + 1, DEF_PROCESS_NAME) )
				return 1;
		}
	}

    // 일반적인 경우에는 CallNextHookEx() 를 호출하여
    //   응용프로그램 (혹은 다음 훅) 으로 메시지를 전달함
	return CallNextHookEx(g_hHook, nCode, wParam, lParam);
}

#ifdef __cplusplus
extern "C" {
#endif
	__declspec(dllexport) void HookStart()
	{
		g_hHook = SetWindowsHookEx(WH_KEYBOARD, KeyboardProc, g_hInstance, 0);
	}

	__declspec(dllexport) void HookStop()
	{
		if( g_hHook )
		{
			UnhookWindowsHookEx(g_hHook);
			g_hHook = NULL;
		}
	}
#ifdef __cplusplus
}
#endif
```

### 소스코드 해석

HookMain.cpp파일은 LoadLibary로 KeyHook.dll 파일을 로드하고 GetProcAddress로 HookStart함수와 HookStop함수를 호출하고 있다. HookStart함수를 호출하면 후킹이 시작된다.

KeyHook.cpp파일에서 HookStart함수를 찾아보면, 앞서 공부한 SetWindowsHookEx API를 호출하고 있다.

```cpp
	__declspec(dllexport) void HookStart()
	{
		g_hHook = SetWindowsHookEx(WH_KEYBOARD, KeyboardProc, g_hInstance, 0);
	}
```

- WH_KEYBOARD(type of hook) :  keystroke 메시지를 감시하는 훅
- KeyboardProc(hook procedure) : 키보드 입력을 처리하는 훅 프로시져
- g_hInstance(DLL handle) : hook procedure인 KeyboardProc이 속해 있는 KeyHook.dll의 핸들
- 0(thread ID) : 글로벌 훅. 모든 스레드에 대해서 모니터링.

KeyboardProc은 윈도우 운영체제에서 키보드 이벤트를 처리하는 훅 프로시저이다. 사용자가 키보드를 누르거나 뗄 때 발생하는 메시지를 가로채 특정 작업 수행하도록 설계된 콜백 함수이다.

```cpp
LRESULT CALLBACK KeyboardProc(
  int    nCode,       // Hook code(HC_ACTION(0), HC_NOREMOVE(3))
  WPARAM wParam,      // virtual-key code (키 상태 정보)
  LPARAM lParam       // extra information (키 입력의 상세 정보)
);
```

함수의 구현은 사용자가 정의한다. 위 코드에서는 키보드를 눌렀다 뗀 이벤트가 발생했을 때, 프로세스 이름이 notepad.exe면 1을 리턴해 keyboardProc함수를 종료한다. 즉, 메시지를 가로채 없앤다. 

그 외의 경우에느 CallNextHookEx를 호출해 그대로 메시지가 전달되도록 한다.