# Node.js의 비동기 처리와 병렬 실행 이해하기

> 티스토리 초안 · 2026-10-26
> 출발점: [[2026-10-26]]의 단일 스레드·cluster·async 메모
> 예제는 실행 모델을 관찰하기 위한 학습용 코드다. 운영 서비스의 부하 측정 결과는 포함하지 않는다. #미적용
> 확인 환경: Node.js v24.22.1. `async-blocking.mjs`와 `worker-demo.mjs`를 실행해 로그와 워커 결과 전달을 확인했다.

Node.js를 공부하면서 ‘단일 스레드인데 여러 요청을 처리한다’는 설명을 접했다. 여기에 `async`, Promise, 워커, `cluster`까지 등장하니 여러 일을 처리한다는 말이 서로 다른 의미로 쓰이고 있었다.

`async`를 붙이면 다른 스레드에서 실행될까? 요청을 여러 개 처리할 수 있다면, 오래 걸리는 계산도 다른 요청에 영향을 주지 않을까? 이 질문을 풀려면 작업의 결과를 기다리는 방식과 실제 실행 자원을 나누어 봐야 한다.

## 한 번에 하나의 JavaScript 콜백을 실행한다는 뜻

일반적인 Node.js 서버에서는 하나의 이벤트 루프 스레드가 요청 콜백과 JavaScript 코드를 실행한다. 어떤 콜백이 실행 중이면 같은 이벤트 루프에 속한 다른 콜백은 그 실행이 끝나기를 기다린다.

그렇다고 Node.js 프로세스 전체에 스레드가 하나만 있는 것은 아니다. 런타임 내부에는 libuv 워커 풀 등 다른 실행 주체가 있고, 직접 `worker_threads`를 만들어 JavaScript 작업을 별도 스레드에 보낼 수도 있다. 따라서 ‘단일 스레드’라는 표현에는 어느 실행 주체를 말하는지 붙여야 한다. [Node.js의 이벤트 루프와 워커 풀](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)

서버가 DB 응답을 기다리는 동안에는 JavaScript가 계속 계산할 필요가 없다. 비동기 I/O를 요청한 뒤 제어권을 돌려주면 이벤트 루프는 다른 요청의 콜백을 실행할 수 있다. 응답이 준비되면 그 결과를 처리할 차례가 돌아온다.

여러 요청이 진행 중이라는 것은 여러 요청의 JavaScript 코드가 같은 순간에 실행된다는 뜻과 다르다. 대기 시간을 겹쳐 여러 작업을 진행하는 동시성과, 여러 실행 자원에서 실제로 작업하는 병렬성을 구별할 수 있다.

## async 함수 안에서도 계산은 현재 스레드에서 실행된다

먼저 `async`만 붙인 함수에서 긴 계산을 해 보자. 다음 내용을 `async-blocking.mjs`에 저장하고 `node async-blocking.mjs`로 실행한다.

```js
import { performance } from "node:perf_hooks";

function burnCpu(ms) {
  const end = performance.now() + ms;
  while (performance.now() < end) {
    // 관찰을 위해 CPU를 계속 사용하는 예제
  }
}

async function calculate() {
  console.log("2. 계산 시작");
  burnCpu(250);
  console.log("3. 계산 종료");
  return "done";
}

const started = performance.now();
setTimeout(() => {
  console.log(`5. 타이머 실행: ${Math.round(performance.now() - started)}ms`);
}, 0);

console.log("1. 함수 호출 전");
const result = calculate();
console.log("4. 함수 호출 후", result instanceof Promise);
await result;
```

`함수 호출 후`가 출력되기 전에 계산이 끝난다. 타이머도 긴 반복문이 끝난 다음에야 실행된다. `async` 함수는 Promise를 반환하지만, 첫 `await`를 만날 때까지 함수 본문은 동기적으로 실행된다. 이 예제에는 `await`가 없으므로 본문 전체가 호출 중에 실행된다. [async 함수의 실행](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)

250ms는 예제에서 정한 대기 기준이며 실제 타이머 출력은 실행 환경에 따라 더 늦어질 수 있다. `setTimeout(..., 0)`은 현재 코드를 중단하고 즉시 실행하라는 명령이 아니다. 실행할 수 있는 시점이 돼도 이벤트 루프가 다른 콜백을 실행 중이면 기다린다. [Node.js 이벤트 루프와 타이머](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)

이 코드가 서버의 요청 처리 함수라면 같은 이벤트 루프의 다른 요청도 영향을 받는다. 함수 이름에 `Async`를 붙이거나 반환형을 Promise로 바꿔도 계산을 수행하는 자원은 그대로다.

## await와 Promise.all이 바꾸는 것

그렇다면 계산 전에 `await`를 넣으면 어떨까?

```js
async function calculateLater() {
  await Promise.resolve();
  burnCpu(250);
}
```

함수의 뒷부분을 이어서 실행하는 시점은 달라지지만 `burnCpu`는 여전히 같은 스레드에서 실행된다. Promise의 후속 처리는 마이크로태스크로 예약되며, 이를 계속 추가한다고 타이머나 I/O 콜백에 실행 기회가 보장되는 것도 아니다. 위 코드는 다른 스레드로 계산을 옮기지 않는다. [Promise 후속 작업과 마이크로태스크](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise#job_queue)

`Promise.all`도 실행 자원을 만들어 주지 않는다. 각각 독립적인 비동기 I/O 작업을 먼저 시작하고 완료를 함께 기다릴 때는 대기 시간을 겹칠 수 있다.

```js
// readUser와 readSettings는 서로 독립적인 비동기 조회 함수라고 가정한다.
const [user, settings] = await Promise.all([
  readUser(userId),
  readSettings(userId),
]);
```

하지만 실제 실행이 얼마나 겹치는지는 연결 풀과 서버 자원에도 달려 있다. 반대로 `Promise.all([calculate(), calculate()])`를 호출하면 앞 예제의 동기 계산은 배열 요소를 평가하는 동안 차례로 실행된다. Promise를 모았다는 사실만으로 CPU 계산이 병렬화되지는 않는다. [Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)

## 비동기 I/O는 어디에서 진행될까

Node.js가 모든 비동기 작업마다 스레드를 하나 만드는 것은 아니다. 네트워크 I/O는 보통 OS의 비동기 이벤트 통지를 이용한다. 파일 I/O나 일부 암호 연산 등은 libuv의 워커 풀에서 처리한다. 작업 종류에 따라 기다리는 곳과 실행하는 곳이 다르다. [Node.js의 작업 분배](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop#what-code-runs-on-the-worker-pool)

이 차이를 알면 ‘비동기인데 왜 느리지?’라는 질문도 구체적으로 나눌 수 있다. JavaScript 콜백이 길어 결과 처리가 늦는지, 워커 풀에 작업이 몰렸는지, DB나 네트워크 자체가 느린지 구별해야 한다.

libuv 워커 풀과 `worker_threads`도 서로 다른 용도로 봐야 한다. 앞의 것은 Node가 일부 네이티브 작업을 처리하는 풀이고, 뒤의 것은 애플리케이션이 JavaScript 코드를 별도 스레드에서 실행하도록 만드는 API다. 파일 읽기를 비동기로 바꿨다고 내 JavaScript 반복문까지 워커 풀로 이동하지는 않는다.

## CPU 작업을 워커로 옮기기

앞의 긴 계산을 별도 워커에서 실행해 보자. 다음 코드를 `worker-demo.mjs`에 저장하고 `node worker-demo.mjs`로 실행한다.

```js
import {
  Worker,
  isMainThread,
  parentPort,
  workerData,
} from "node:worker_threads";
import { performance } from "node:perf_hooks";

if (isMainThread) {
  const started = performance.now();
  const worker = new Worker(new URL(import.meta.url), {
    workerData: { durationMs: 250 },
  });

  setTimeout(() => {
    console.log(`메인 타이머: ${Math.round(performance.now() - started)}ms`);
  }, 0);

  worker.on("message", (message) => {
    console.log("워커 결과:", message);
  });
  worker.on("error", (error) => {
    console.error("워커 오류:", error);
    process.exitCode = 1;
  });
  worker.on("exit", (code) => {
    if (code !== 0) {
      console.error("워커 종료 코드:", code);
      process.exitCode = 1;
    }
  });
} else {
  const end = performance.now() + workerData.durationMs;
  while (performance.now() < end) {
    // 워커 스레드에서 CPU 사용
  }
  parentPort.postMessage({ status: "done" });
}
```

메인 스레드는 워커의 계산이 끝나기를 동기적으로 기다리지 않는다. 워커가 결과를 메시지로 보내면 메인 스레드가 그 메시지를 처리한다. 앞 예제와 비교할 것은 총 계산 시간이 줄었는지가 아니라, 메인 타이머가 긴 계산에 직접 막히는지다.

워커 생성 비용과 OS 스케줄링이 있으므로 출력 시점이나 두 로그의 순서를 모든 환경에서 보장하는 예제로 읽으면 안 된다. CPU 자원이 부족하면 별도 스레드에 옮겨도 서로 실행 시간을 경쟁한다.

실제 서비스에서 계산 요청마다 워커를 새로 만들면 생성 비용이 반복된다. 지속적인 CPU 작업에는 워커를 재사용하는 풀을 고려할 수 있다. 메시지는 일반적으로 복제되며, 전송 가능한 버퍼의 소유권을 넘기거나 공유 메모리를 사용하는 방법도 있다. 큰 입력을 보내는 비용까지 포함해야 작업 분리의 이익을 판단할 수 있다. [worker_threads 공식 문서](https://nodejs.org/api/worker_threads.html)

워커를 도입하면 기다리는 작업의 개수, 취소된 요청의 계산 처리, 워커가 실패했을 때의 응답도 설계해야 한다. 실행 스레드를 추가하는 것과 과부하를 통제하는 것은 함께 생각할 문제다.

## 프로세스를 나누는 cluster는 무엇이 다를까

워커 스레드가 CPU 계산을 옮기는 선택지라면, `cluster`는 여러 Node.js 프로세스로 서버를 구성하는 방식이다. 각 프로세스는 자체 실행 환경을 갖고 요청을 처리한다. 같은 애플리케이션을 여러 프로세스에서 실행해 여러 CPU 자원을 사용할 수 있다. [cluster 공식 문서](https://nodejs.org/api/cluster.html)

별도 프로세스이므로 프로세스의 메모리에 넣은 값이 다른 프로세스에 자동으로 전달되지는 않는다. 로그인 상태나 요청 제한 횟수를 메모리 변수에만 저장했다면 요청을 처리한 프로세스에 따라 서로 다른 상태를 볼 수 있다. 여러 프로세스가 공유해야 하는 상태는 그에 맞는 저장소나 통신 방식으로 관리해야 한다.

프로세스 하나가 종료됐을 때 다른 프로세스가 살아 있을 수는 있지만, 그것만으로 요청 성공이 보장되지는 않는다. 종료된 프로세스에서 처리하던 요청은 실패할 수 있고, 워커 재시작과 준비 상태 확인도 필요하다. 외부 DB나 호스트 전체의 장애까지 격리되는 것도 아니다. 따라서 ‘cluster를 쓰면 안 죽는 서버가 된다’고 설명하면 필요한 조건이 빠진다.

| 방식 | 실행을 나누는 단위 | 이 글에서의 용도 | 함께 고려할 것 |
| --- | --- | --- | --- |
| 비동기 I/O | 대기와 후속 처리 | DB·네트워크 응답을 기다리며 다른 요청 처리 | 연결 풀, 외부 서비스 지연 |
| `worker_threads` | 같은 프로세스의 별도 JS 스레드 | 긴 CPU 계산을 메인 이벤트 루프에서 분리 | 워커 생성, 데이터 전달, 작업 대기열 |
| `cluster` | 별도 Node.js 프로세스 | 여러 서버 프로세스로 요청 처리 | 상태 공유, 프로세스 수명, 자원 사용 |

이 표는 세 가지 중 하나만 고르라는 의미가 아니다. 여러 서버 프로세스 각각이 비동기 I/O를 사용하고, 필요한 CPU 작업을 워커에 보낼 수도 있다. 선택의 출발점은 지금 어디에서 대기하고 무엇이 CPU를 쓰는지다.

## 느려졌을 때 확인할 지점

요청이 느리다는 사실만으로 워커 수부터 늘리기는 어렵다. 큰 JSON을 처리하는 콜백이 이벤트 루프를 오래 점유하는 상황과 DB 연결을 얻지 못해 기다리는 상황은 해결할 대상이 다르다.

Node.js의 `monitorEventLoopDelay()`로 이벤트 루프 지연을 관찰하고, `eventLoopUtilization()`으로 활성 시간을 살펴볼 수 있다. 이 값은 요청 지연, 프로세스 CPU 사용량, 외부 호출 시간과 함께 해석해야 한다. 이벤트 루프 활성 비율을 프로세스 전체 CPU 사용률로 읽으면 실행 주체의 차이를 다시 놓치게 된다. [Node.js 성능 측정 API](https://nodejs.org/api/perf_hooks.html)

이번 예제에서 `async` 함수는 결과를 Promise로 전달했지만 계산을 옮기지는 않았다. 워커 예제는 계산을 수행하는 스레드 자체를 나눴다. `cluster`는 서버 프로세스를 나누며, 그 결과 상태 공유와 요청 처리 방식도 달라진다.

앞으로 비동기 코드를 읽을 때는 두 가지를 함께 확인하려 한다. 호출자는 언제 제어권을 돌려받는지, 실제 작업은 어떤 자원에서 실행되는지다. 이 둘을 구분하면 Promise를 반환한다는 이유만으로 긴 계산까지 다른 요청과 잘 섞일 것이라고 기대하지 않게 된다.

---

작성 메모: [[블로그/초안/블로그 - 실패 처리와 트랜잭션 경계 설계하기|관련 초안 — 실패 처리와 트랜잭션 경계 설계하기]]의 동기·비동기 콜백 구분과 함께 읽기.
