---
title: Buffer overflow(Buffer overrun)와 대응방안
date: 2025-03-09 21:00:00 +0900
categories: [Tech, Security]
tags: [buffer overflow, 버퍼오버플로우]
description: Buffer overflow와 그 대응방안에 대해 알아봅시다. 
# pin: true # make a pin
---
> 버퍼오버플로우와 그 대응방안에 대해 알아봅시다.
{: .prompt-info }

### 버퍼오버플로우란?
일단 버퍼는 데이터를 한 곳에서 다른 한 곳으로 전송하는 동안 일시적으로 그 데이터를 보관하는 메모리의 영역입니다.\
해당 공간이 넘쳤다고(overflow) 생각하면 될 것 같습니다. 컴퓨터 보안과 프로그래밍에서 프로세스가 버퍼에 데이터를 저장할 때 프로그래머가 지정한 곳을 넘어 인접 메모리까지 덮어쓰게 되었을 때 버퍼오버플로우라고 한다.[^1] 

### 대응방안
- 컴파일 타임에서의 대응방안과 런타임에서의 대응방안으로 나눌 수 있다.

#### 컴파일 타임에서의 대응방안
C, C++ 같은 언어에서 특히 안전한 함수를 사용해야한다.
- strncat()[^2], strncpy(), fscanf(), snprintf(), vsnprintf() ...

**경계값 및 파라미터를 체크한다.**

**Stack Guard**\
Stack 영역의 보호를 위해 Return Address의 수정이 일어나는지 확인을 하기위해 변수와 Return Address사이에 canary라는 특별한 문자를 사용한다. 해당 문자를 확인할 수 없다면 Return Address가 수정되었을 가능성이 높아 프로그램의 실행을 막는다.

**Stack shield**\
Return Address를 안전한 장소에 복사해 함수종료 시 현재 스택의 리턴 주소와 비교해 변조여부를 확인합니다.

#### 런타임에서의 대응방안
스택과 힙을 실행 불능으로 설정한다.

**Address Space Layout Randomization**\
  주소 공간 배치를 난수화한다. 실행 시 마다 메모리 주소를 변경시켜 특정 주소 호출을 방지한다. 


---
참고문헌 및 각주

[^1]: <https://ko.wikipedia.org/wiki/%EB%B2%84%ED%8D%BC_%EC%98%A4%EB%B2%84%ED%94%8C%EB%A1%9C#%EC%8A%A4%ED%83%9D_%EA%B8%B0%EB%B0%98_%EC%9D%B4%EC%9A%A9>
[^2]: <https://www.ibm.com/docs/ko/i/7.5?topic=functions-strncat-concatenate-strings>

<https://koreascience.kr/article/CFKO200634741480717.pdf>
<https://hackstoryadmin.tistory.com/entry/Linux-Memory-Protection-Stack-Canary>
<https://hackstoryadmin.tistory.com/entry/Linux-Memory-Protection-Stack-Canary>