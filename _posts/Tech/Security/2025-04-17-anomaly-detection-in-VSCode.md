---
title: VSCode에서 발생한 PowerShell탐지 이슈
date: 2025-04-17 20:00:00 +0900
categories: [Tech, Security]
tags: [보안, powershell, vscode] # TAG names should always be lowercase
description: 개발 도구도 보안 탐지 대상이 될 수 있는 이유를 알아보고 구조 분석해 보았습니다.
mermaid: true
# pin: true # make a pin
---
> 이상탐지 솔루션에서 PowerShell 탐지? 
> 개발 도구도 보안 탐지 대상이 될 수 있는 이유를 알아보고 구조 분석해 보았습니다.
{: .prompt-info }

## 들어가며
최근 개발 중 사용하던 **PowerShell 터미널**에서, 보안 솔루션에 의해 예상치 못한 탐지가 발생했다.  
나는 전혀 해당 탐지를 인지하지 못한 상태였다.  
당시에는 **Python 가상환경을 구성**하고 있었고, 정확히는 `uv` 설치와 관련된 작업 중이었다.  
도구 내부에 알림창이 하나 떴던 기억이 나지만,  
방금 내가 실행한 작업 때문일 것 같아 큰 의심 없이 닫아버렸다.  

하지만 약 30분에서 1시간쯤 후, 보안 관련 검토가 필요하다는 안내를 받았고,  
정확히 어떤 행위가 탐지되었는지 확인해보게 되었다.  
결과적으로 **탐지 트리거는 보안 솔루션에서 PowerShell 프로세스를 악성 의심으로 판단하여 강제 종료한 것**이었으며,  
이와 함께 **자동으로 실행된 VSCode 설치 경로 내의 shellIntegration.ps1 스크립트가 탐지 대상**으로 지정되었다.
---

## 탐지 내용 분석

### 1. 경로 및 파일 확인
감지된 파일의 로컬 파일시스템 위치를 따라가보니,  
해당 스크립트는 **터미널 환경 구성 시 자동으로 실행되는 PowerShell 스크립트**였고,  
**설치된 개발 도구 구성 요소 중 하나**였다.  
내가 직접 작성한 것도 아니고, 의식해서 다운로드한 것도 아니었다. `shellIntegration.ps1`이었다. 
이 스크립트는 내가 직접 작성한 것도 아니고, 별도로 의식해서 다운로드한 것도 아니었다. 
**VSCode를 설치하면서 함께 포함된 구성 요소**였고, **터미널을 사용할 때 자동으로 실행되는 구조**였다.


### 2. GitHub 소스 확인
검색을 통해 공식 GitHub 저장소에서 동일한 파일을 발견했고, 이는 VSCode의 터미널 관련 기능에 사용되는 내부 스크립트였다.  
[🔗Microsoft 공식 Github Link](https://github.com/microsoft/vscode/blob/main/src/vs/workbench/contrib/terminal/common/scripts/shellIntegration.ps1)


### 3. 공식 문서 확인
터미널 셸 통합 관련 공식 문서를 보면,  
[🔗Terminal Shell Integration](https://code.visualstudio.com/docs/terminal/shell-integration)  
VSCode는 PowerShell 등의 **터미널에서 향상된 UX를 제공하기 위해 특정 스크립트를 자동으로 삽입하고 실행**한다.  
이 과정은 사용자 개입 없이 이루어지며, 해당 스크립트는 명령어 추적, 프롬프트 장식, 빠른 명령 복원 등 다양한 기능을 담당한다.

---

## Prompt() 함수 구조

스크립트 내부를 살펴보면 다음과 같은 구조가 있다:

```powershell
#부분발췌
function global:Prompt {
    $FakeCode = [int]!$global:?
    # NOTE: We disable strict mode for the scope of this function because it unhelpfully throws an
    # error when $LastHistoryEntry is null, and is not otherwise useful.
    Set-StrictMode -Off
    ...
    # Prompt started
    # OSC 633 ; A ST
    $Result += "$([char]0x1b)]633;A`a"
    # Current working directory
    # OSC 633 ; <Property>=<Value> ST
    ...
    # Run the original prompt
    $OriginalPrompt += $Global:__VSCodeOriginalPrompt.Invoke()
    $Result += $OriginalPrompt
    ...
}
```

해당 함수를 보았을 때 전부 이해가 가능한 것은 아니었지만,   
**프롬프트를 조작하고 있는 것**은 확실해 보였다.  
공식문서를 확인하면 VSCode의 terminal에 decoration을 추가하고, **터미널 상단에 명령어를 고정시키는 등(Sticky scroll)**의 동작을 수행한다고 한다.   

보안 탐지 관점에서 보면,  
이러한 프롬프트 제어 방식이 자동 실행되며 사용자에게 명확히 인식되지 않는 구조인 만큼,  
특정 상황에서는 **이상 행위(anomalous behavior)**로 해석될 여지가 있을 수 있다.

이러한 문맥에서 **PowerShell 프로세스가 강제 종료되는 흐름 속에서 함께 실행된 해당 스크립트가 탐지 대상**으로 이어진 것으로 보인다.  
실제로 탐지 시점과 사용자 활동 로그를 종합하면,  
**SIEM 기반 이벤트 수집 시스템에서 후킹 또는 유사 메커니즘을 통해 관련 흐름이 추적되었을 가능성**이 있다.  
이는 사용자 관점에서의 합리적인 추정임을 밝힌다.

---

## 알림은 있었지만, 인식되지 않았다

흥미로운 지점은 VSCode가 터미널이 비정상적으로 종료되었다는 내용의 알림(alert)을 띄운 것으로 기억된다는 점이다.  
_정확한 문구는 기억나지 않지만, 해당 메시지가 그와 관련된 것으로 추정된다._  
하지만 당시 나는 **Python 가상환경 설정 중**이었고, 알림을 자세히 읽지 않아 **“설치 과정에서 생기는 일반적인 메시지겠지”**라고 판단하고 곧바로 닫아버렸다.

이 경험을 통해 알게 된 점은:
- **알림이 있다고 해서 사용자가 그 내용을 인식하는 것은 아니다**
- **보안상 중요한 변화도 UX에 묻혀 지나갈 수 있다**
- 사용자 입장에서는 **"내가 뭘 실행한 건지"**조차 모르는 상황이 충분히 발생할 수 있다

---

## 경험을 통해 배운 것

이 경험은 단순히 알림 하나를 놓친 게 아니라,
**개발 환경에서 “무심코 넘어가는 자동화”가 실제 보안 탐지로 이어질 수 있다는 것**을 체감하게 해줬다.

특히 `shellIntegration.ps1`처럼,
**“누가 실행했는지조차 명확하지 않은” 자동 실행 스크립트**는
사용자는 인식조차 하지 못하게되고,
보안 솔루션은 그것을 **의심스러운 행위**로 간주할 수 있다.

보안은 코드의 내용뿐 아니라 **그 행위의 시점과 맥락**을 본다.  
해당 스크립트에 대한 추가 검토가 이루어진 뒤, 이슈가 종료된 것으로 보였다. 

이 사건은 단순한 시그니처 기반 탐지가 아니라, 이벤트 수집 시스템(SIEM 등)을 통해 발생한 것으로 추정되며,  
그리고 그와 충돌할 수 있는 **개발자 경험의 자동화 구조**를 모두 경험할 수 있는 계기였다.  

---

## 유사 상황 발생 시 대응 전략

- 우선 탐지된 **파일의 경로와 출처를 먼저 확인**한다.  
- **공식 문서와 GitHub 등에서 근거 확보** 후, 정리된 형태로 설명 준비
- 개발 도구 내부에서 실행되는 스크립트도 항상 잠재적인 보안 탐지 대상임을 인지할 것

> 이 글은 업무 중 발생한 실제 사례를 개인적인 관점에서 정리한 기술적 분석입니다.

