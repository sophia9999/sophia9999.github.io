---
title: 운영체제 관점에서 이해하는 Python의 async/await
date: 2025-03-22 20:00:00 +0900
categories: [tech]
tags: [python, async, coroutine, os, 이벤트루프, 협력적스케줄링, 개발개념정리]
description: 🛠 운영체제 관점에서 async/await가 진짜 CPU에 뭘 하는지 알고싶은 나의 꼬리에 꼬리를 무는 질문
mermaid: true
pin: true
---
> 운영체제 관점에서 **Python의 async/await가 어떤 흐름 위에서 작동하는 건지** 정리해보려합니다.
{: .prompt-info }

Python의 `async/await`를 처음 접했을 때,  
“비동기 처리를 위한 문법”이라는 건 알았지만  
솔직히 뭐가 어떻게 돌아가는지는 잘 몰랐습니다.

운영체제를 먼저 공부한 입장에서  
> “프로세스는 CPU에 올라가 있어야 실행되는 거잖아?  
> 그런데 비동기 처리라는 게 그 CPU 위에서 어떤 식으로 이루어지는 걸까?”

이런 의문이 자꾸 생겼죠.

GIL 때문이라는 말은 많이 들었지만,  
그게 정확히 어떤 구조를 만들고,  
왜 CPU 바운드 작업에서는 async가 힘을 못 쓰는 건지  
제대로 납득이 되진 않았습니다.\
*※ GIL(Global Interpreter Lock)은 Python의 멀티스레딩에서 중요한 개념이지만,  
이 글에서는 async/await와 OS의 흐름 관계에 집중하기 위해 별도로 다루지 않습니다.*

거창한 이론보다,  
> “내가 작성한 이 코드가, 실제 시스템 안에서는 어떤 위치에 있을까?”  
그걸 그림으로 그리듯 풀어보는 게 목적이에요.

---

## 이 글에서는:

- 프로세스/스레드/GIL 같은 기본 구조를 짚고  
- 코루틴과 이벤트 루프가 어떻게 협력적으로 흐름을 제어하는지  
- 왜 CPU 바운드 작업에서는 async가 효과를 못 내는지  

이런 것들을 차근차근 풀어봅니다.

기본적인 내용이지만,  
운영체제랑 async 개념이 서로 어떻게 맞물리는지  
헷갈렸던 분들에게 조금이라도 도움이 되면 좋겠습니다.

---

> 🤓 참고로  
> 글에 나오는 코드나 도식은 비교를 위해 넣은 이미지를 제외하고는 모두 **CPU 1개 환경 기준**으로 확인한 내용입니다.
> 가능한 실제 출력 로그와 함께 정리했어요!
![cpu_info](/assets/img/posts/250322.cpuInfo.jpg)
> multi core와 비교하기 위해서는 local 컴퓨터를 사용했습니다. 
![local_cpuinfo](/assets/img/posts/250322.localCpu.png)\
wsl에서 CPU가 8개라고 보이지만
물리코어는 4개입니다. 
![local_physical_cpuinfo](/assets/img/posts/250322.physicalCpu.png)
---

## 1. Python 프로세스 안에서 async는 어디서 돌아가는가

운영체제 입장에서 보면,  
우리가 만든 Python 프로그램은 그저 **하나의 프로세스**일 뿐입니다.  
그 안에서 어떤 코드를 실행하든, 어떤 모듈을 쓰든  
**OS는 전혀 알지 못합니다.**

우리가 작성한 Python 애플리케이션도 결국  
운영체제 입장에서는 **하나의 프로세스**일 뿐입니다.  
다른 앱(예: Chrome, VSCode)들과 나란히 놓인 실행 단위 중 하나죠.

그리고 이 프로세스 안에서 Python 런타임이 돌아가고,  
그 내부에서 `asyncio`, `코루틴`, `GIL` 같은 구조가 동작합니다.

```mermaid
flowchart TB
  subgraph OS["운영체제 (커널 공간)"]
    subgraph PYPROC["프로세스: Python 앱"]
      GIL["GIL"]
      LOOP["asyncio"]
      CORO["코루틴"]
    end

    OTHERPROC["프로세스: 다른 앱 (ex. Chrome, VSCode)"]
    CPU["CPU (실행 자원)"]
  end

  PYPROC --> CPU
  OTHERPROC --> CPU

```

즉, 우리가 async def를 만들고 await을 써도
운영체제는 아무것도 하지 않습니다.
**오직 Python 런타임 내부에서만** 실행 흐름이 바뀝니다.

그래서 이걸 **협력적(cooperative) 스케줄링**이라고 부릅니다.
→ await이 없으면, Python은 다른 작업에게 CPU를 양보하지 않습니다.

여기까지 정리하면,
우리가 작성한 async/await 코드는 OS가 직접 다뤄주는 구조가 아니고,
그저 **Python 프로세스 안에서 일어나는 흐름 제어일 뿐**이라는 걸 알 수 있습니다.

## 2. async 코드에서 흐름이 바뀐다는 건 무슨 뜻일까?

Python의 `async/await` 구조는  
운영체제가 CPU를 직접 제어하는 구조가 아니라,  
**코드 내부에서 실행 흐름을 스스로 양보하는 방식**입니다.

아래 코드는 그걸 실험해본 예제입니다.
```python
import asyncio
import time

# sleep을 통해 서로 cpu를 양보하며 실행하는 경우
async def yield_task(name, delay):
    for i in range(3):
        print(f"{name}: step {i} at {time.strftime('%X')}")
        await asyncio.sleep(delay)
    print(f"{name} 완료 at {time.strftime('%X')}")

# 양보하지 않고 CPU를 점유하는 경우
async def task(name, delay):
    for i in range(3):
        print(f"{name}: step {i} at {time.strftime('%X')}")
    print(f"{name} 완료 at {time.strftime('%X')}")

async def main():
    await asyncio.gather(
        yield_task("yield_task1", 1),
        yield_task("yield_task2", 1)
    )

    await asyncio.gather(
        task("task1", 1),
        task("task2", 1)
    )

asyncio.run(main())
```
### 🖥 실행 결과 (CPU 1개 환경)
![yielding async](/assets/img/posts/250322.python.png)

### 코루틴이 서로 번갈아 실행되는 경우 (`await` 사용)

- `yield task1`와 `yield task2`가 시간 단위로 교차하며 실행됩니다.
- `await asyncio.sleep(1)`이 들어간 순간 이벤트 루프가 다른 코루틴을 실행시킵니다.

### CPU를 양보하지 않는 경우 (`await` 없음)

실행 로그에서는 `task1`의 모든 step이 먼저 출력되고,
그 뒤에 `task2`의 로그가 출력됩니다.
-> 이는 이벤트 루프가 **다른 코루틴을 끼워 넣을 기회를 얻지 못했다는 뜻**입니다.

이를 통해 우리는 `async`가 **단일 스레드에서 여러 코루틴을 빠르게 전환하며 실행**되는 것임을 확인할 수 있습니다.

이처럼 `await`를 통해 코루틴이 스스로 실행 흐름을 양보할 때,
이벤트 루프는 다른 코루틴을 실행시켜 **병렬처럼 보이는 실행 흐름**을 만들어냅니다.

## 3. CPU 바운드 작업도 흐름을 양보할 수 있을까?

async는 I/O 바운드 작업에 최적화된 구조입니다.  
그런데 CPU를 오래 점유하는 작업에서도  
중간에 흐름을 나눌 수 있을까요?

결론부터 말하면 **"가능은 하지만, 협조가 필요합니다."**

아래 예제는 CPU를 계속 점유하는 작업 안에  
일정 조건마다 `await asyncio.sleep(0)`을 넣어  
**다른 코루틴에게 실행 기회를 넘겨주는 구조**입니다.

```python
import asyncio
import time

async def cpu_heavy(name, delay):
    print(f"{name}: start at {time.strftime('%X')}")
    for i in range(10**8):
        if i % 2_000_000 == 0 :
            # 이 조건이 해당될 때 await를 걸어 이벤트 루프가 다른 코루틴을 실행할 수 있게 해준다.
            await asyncio.sleep(0)
    print(f"{name}: end at {time.strftime('%X')}")

async def io_task(name, delay):
    print(f"{name} started at {time.strftime('%X')}")
    await asyncio.sleep(delay)
    print(f"{name} finished at {time.strftime('%X')}")

async def main():
    await asyncio.gather(
        cpu_heavy("cpu_heavy_task", 1),
        io_task("io_task", 1)
    )

asyncio.run(main())
```
### 🖥 실행 결과 (CPU 1개 환경)
![cpu_bound_yield](/assets/img/posts/250322.cpuBoundYield.png)

여기서 중요한 포인트 `await asyncio.sleep(0)`은
**"다른 코루틴이 있다면 실행 기회를 양보하겠다"**는 뜻입니다.

Python은 이 지점에서 이벤트 루프에게 제어권을 넘깁니다.

따라서 CPU를 혼자 독점하지 않고, 다른 작업에게도 실행 기회를 나누는 구조가 됩니다.

그런데 성능은 오히려 느려질 수도 있습니다. 
너무 자주 await을 만나면 컨텍스트 전환 비용이 발생합니다.

실제로는 연산이 멈췄다 재개되기를 반복하기 때문에
총 소요 시간이 더 길어질 수도 있습니다.

```python
import asyncio
import time

async def cpu_heavy(name, delay):
    print(f"{name}: start at {time.strftime('%X')}")
    for i in range(10**8):
        pass
    print(f"{name}: end at {time.strftime('%X')}")

async def io_task(name, delay):
    print(f"{name} started at {time.strftime('%X')}")
    await asyncio.sleep(delay)
    print(f"{name} finished at {time.strftime('%X')}")

async def main():
    await asyncio.gather(
        cpu_heavy("cpu_heavy_task", 1),
        io_task("io_task", 1)
    )

asyncio.run(main())
```
### 🖥 실행 결과 (CPU 1개 환경)
![cpu_bound_task](/assets/img/posts/250322.cpuBoundTask.png)\
실제로 이 결과를 보면 총 소요시간은 위에서 await 처리를 했을 때 더 걸렸음을 확인할 수 있죠.

결국 async는 “병렬 처리 도구”가 아니라
**“동시성을 협력적으로 제어하는 방식”**이라는 점이 핵심입니다.

---

## 4. CPU 바운드 작업에는 결국 멀티프로세싱?

async는 I/O 바운드 작업에 잘 맞지만,  
CPU를 오래 쓰는 작업에는 큰 도움이 되지 않습니다.

그 이유는 간단합니다.  
**async는 흐름을 넘겨주는 방식일 뿐,  
하나의 CPU를 더 효율적으로 쓰게 해주는 구조는 아니기 때문입니다.**

---

Python에서는 GIL(Global Interpreter Lock) 때문에  
**멀티스레딩으로는 CPU를 병렬로 사용할 수 없습니다.**

그래서 진짜 CPU를 병렬로 쓰고 싶다면  
**멀티프로세싱**을 사용해야 합니다.

멀티프로세싱은  
- 운영체제가 직접 여러 프로세스를 관리하고  
- 각 프로세스를 서로 다른 CPU 코어에 배분할 수 있습니다.

---

### 💡 그럼 멀티코어가 아니면?

맞습니다.  
**멀티코어가 아니면, 멀티프로세싱도 사실상 동시에 실행되는 건 아닙니다.**

하지만 운영체제는 각 프로세스에 **타임슬라이스를 배분**해서  
각각이 돌아가듯 보이게 만들 수는 있죠.  
(바로 그게 OS 스케줄링이 하는 일입니다)

---

### ✅ 정리하자면

| 처리 방식       | 구조         | CPU 바운드에 적합? | 병렬성 |
|----------------|--------------|--------------------|--------|
| async/await    | 단일 프로세스 내부 흐름 전환 | ❌        | ✕ (동시성만 있음) |
| 멀티스레딩     | Python에서는 GIL 때문에 제한 | ❌        | ✕ |
| 멀티프로세싱   | OS가 여러 프로세스를 스케줄링 | ✅        | ⭕ (멀티코어에서 효과적) |

---

그래서 Python에서  
> **I/O 바운드 작업 → `async/await`**,  
> **CPU 바운드 작업 → `multiprocessing`**  
으로 처리하는 게 일반적인 전략입니다.


## 🧪 실험: 멀티코어 vs 싱글코어 환경에서 멀티프로세싱 성능 차이

Python에서 CPU 바운드 작업을 `multiprocessing`으로 처리했을 때,  
**CPU 코어 수에 따라 실제 실행 시간이 어떻게 달라지는지** 실험해보았습니다.

### 💻 실험 환경

| 항목             | Cloud 환경                | Local PC 환경           |
|------------------|---------------------------|-------------------------|
| CPU 코어 수       | 1개 (논리코어 2)            | 4 (논리코어 8)         |
| OS               | Ubuntu 24.04.1 LTS        | Ubuntu 24.04.2 LTS      |
| Python 버전       | 3.12.3                    | 3.12.3                  |

### 🧾 테스트 코드
```python
import multiprocessing
import time

def cpu_bound_task(n):
    count = 0
    for i in range(10 ** 8):
        count += i
    return count

if __name__ == "__main__":

    print(f"start at {time.strftime('%X')}")
    start = time.time()

    processes = []
    # 프로세스를 CPU 코어 수만큼 돌려 계산하는 작업
    for _ in range(multiprocessing.cpu_count()):
        p = multiprocessing.Process(target=cpu_bound_task, args=(1,))
        processes.append(p)
        p.start()

    for p in processes:
        p.join()

    print(f"end at {time.strftime('%X')} 소요 시간: {time.time() - start:.2f}초")

```
### 🖥 실행 결과 (CPU 1개 환경)
![cpu_1](/assets/img/posts/250322.singleCpu.png)

### 🖥 실행 결과 (CPU 4개 환경)
![cpu_multiple](/assets/img/posts/250322.multiCpu.png)

## ✍️ 마무리하며

이 글은 async 문법을 소개하는 글이 아닙니다.  
운영체제 관점에서, 그리고 CPU와의 관계를 기준으로  
Python의 async/await가 어떤 의미를 가지는지를 이해하려는 시도였습니다.

처음엔 단순히  
> "await하면 멈추고, 그 사이 다른 거 하는 거 아냐?"  
정도로만 알고 있었던 개념이,

실제로는  
> “CPU가 누구에게 실행 시간을 줄 것인가”,  
> “그 실행 시간 안에서 내가 어떤 흐름을 만들어낼 수 있는가”  
라는 더 깊은 질문으로 이어졌습니다.

---

저는 이 질문에 답을 찾아가며

- async는 **운영체제가 제어하는 흐름이 아니라**,  
  **사용자 코드 안에서의 협력적인 흐름 전환**이라는 점,
- CPU 바운드 작업은 async로 해결할 수 없고,  
  **운영체제가 직접 스케줄링하는 멀티프로세싱**이 필요하다는 점,
- 그리고 **멀티코어 환경이 진짜 병렬 처리의 핵심**이라는 걸  
  직접 실험을 통해 확인할 수 있었습니다.

---

비동기 프로그래밍은 단순한 문법이 아니라,  
**운영체제 위에서 돌아가는 코드의 본질을 이해하는 방법**일지도 모릅니다.

---

> 이 글은 처음 `async`와 `await`이 진짜 뭔지 궁금했던  
> 저 스스로의 질문에서 출발한 기록입니다.  
> 알아본다고 이리저리 파보긴 했지만...
> 저도 틀릴 수 있습니다! 피드백은 언제든 환영입니다.