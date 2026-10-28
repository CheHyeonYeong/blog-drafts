# TypeScript로 가능한 상태와 실패 표현하기

> 티스토리 초안 · 2026-10-26
> 출발점: [[2026-10-26]], [[2026-10-20]]
> 예제는 사용자 조회를 소재로 새로 구성했다. 코드 블록은 설명 단계에 따라 타입을 교체하므로 각각의 단계로 읽는다.
> 확인 환경: TypeScript 5.9.3. 본문 코드의 컴파일과 성공 필드 누락·명시적 `undefined` 대입·미확인 인덱스 접근의 오류 검출을 확인했다.

TypeScript를 쓰면서 `optional`, `undefined`, 유니언 타입을 각각 배웠다. 처음에는 필드가 없을 수도 있으면 `?`를 붙이고, 값이 여러 종류면 `|`로 이어 쓰는 문법으로 이해했다. 그런데 성공과 실패를 반환하는 함수를 만들다 보니 필드 하나의 타입보다 필드 사이의 관계가 더 중요해졌다.

사용자 조회에 성공했다면 사용자 정보가 있어야 한다. 실패했다면 실패 이유가 있어야 한다. 이 관계를 타입으로 표현할 수 있을까? 사용자 조회 함수 하나를 조금씩 바꾸며 살펴보려 한다.

## optional 필드 두 개로 시작하기

먼저 사용자와 조회 결과를 다음처럼 정의해 보자.

```ts
type User = {
  id: string;
  name: string;
};

type LooseResult = {
  user?: User;
  error?: "notFound" | "forbidden";
};
```

성공하면 `user`를, 실패하면 `error`를 넣겠다는 의도다. 하지만 이 타입은 다음 값도 허용한다.

```ts
const empty: LooseResult = {};

const both: LooseResult = {
  user: { id: "u1", name: "채현" },
  error: "notFound",
};
```

첫 번째 값에는 성공 데이터도 실패 이유도 없다. 두 번째 값은 사용자를 찾았는데 찾지 못했다고 말한다. 각 필드에 잘못된 타입을 넣은 것은 아니므로, 필드의 자료형만으로는 두 경우를 막을 수 없다.

조회 결과를 읽는 함수도 애매해진다.

```ts
function describeLoose(result: LooseResult): string {
  if (result.error !== undefined) {
    return `조회 실패: ${result.error}`;
  }

  return result.user?.name ?? "사용자 정보가 없습니다";
}
```

에러가 없는데 사용자도 없는 상태까지 처리하게 됐다. 조회 함수가 그런 결과를 만들지 않는다고 약속해도, 그 약속은 타입 밖에 남아 있다.

## 함께 성립해야 하는 필드를 한 상태로 묶기

성공과 실패를 각각의 객체 타입으로 나누면 관계를 더 직접적으로 표현할 수 있다.

```ts
type LookupResult =
  | { kind: "found"; user: User }
  | { kind: "notFound"; requestedId: string }
  | { kind: "forbidden" };
```

이제 성공 상태에는 `user`가 필수다. 사용자를 찾지 못했다면 조회한 ID가 있고, 접근이 거절됐다면 해당 상태만 반환한다. 모든 상태에 같은 이름의 `kind`가 있으며, 각 상태는 서로 다른 문자열 리터럴 값을 사용한다.

이런 형태를 판별 유니언(discriminated union)이라고 한다. `kind`를 검사하면 TypeScript가 가능한 구성원을 좁힐 수 있다. 이 과정이 타입 좁히기(type narrowing)다. [TypeScript의 narrowing 설명](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions)

```ts
function describe(result: LookupResult): string {
  switch (result.kind) {
    case "found":
      return `${result.user.name}님을 찾았습니다`;
    case "notFound":
      return `${result.requestedId} 사용자가 없습니다`;
    case "forbidden":
      return "조회 권한이 없습니다";
  }
}
```

`found` 분기에서는 `user`가 있다는 사실이 타입에 포함된다. 호출자가 다시 존재 여부를 확인하거나 `!`로 단언할 필요가 없다. 상태를 만드는 쪽에서도 `{ kind: "found" }`만 반환하면 필수 필드가 빠졌다는 오류가 난다.

다만 이것을 객체에 불필요한 필드가 절대로 들어올 수 없다는 뜻으로 넓히면 안 된다. TypeScript는 구조적으로 타입을 비교하므로 추가 필드가 있는 변수가 호환될 수 있다. 여기서 표현한 것은 **판별값에 따라 필요한 필드의 관계**다. 정확한 키 집합이나 외부 입력의 유효성까지 자동으로 보장한 것은 아니다. [객체 타입과 초과 속성 검사](https://www.typescriptlang.org/docs/handbook/2/objects.html#excess-property-checks)

## 새로운 상태가 생겼을 때 빠뜨리지 않기

나중에 계정이 정지된 경우를 별도 상태로 추가한다고 하자. 타입만 고치고 화면의 분기를 놓칠 수 있다. 이때 모든 상태를 처리했는지 검사하도록 작성할 수 있다.

```ts
function assertNever(value: never): never {
  throw new Error(`처리하지 않은 상태: ${JSON.stringify(value)}`);
}

function describeExhaustively(result: LookupResult): string {
  switch (result.kind) {
    case "found":
      return `${result.user.name}님을 찾았습니다`;
    case "notFound":
      return `${result.requestedId} 사용자가 없습니다`;
    case "forbidden":
      return "조회 권한이 없습니다";
    default:
      return assertNever(result);
  }
}
```

모든 구성원을 처리했다면 `default`에 남는 타입은 `never`다. `LookupResult`에 `{ kind: "suspended" }`를 추가하고 분기를 추가하지 않으면, `result`가 `never`가 아니라는 오류가 난다. 타입에 새 상태를 추가하는 작업이 그 상태를 소비하는 코드의 수정으로 이어진다. [완전성 검사](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#exhaustiveness-checking)

이 방식은 상태가 유한하고, 호출자가 각 상태에 다른 행동을 해야 할 때 특히 유용하다. 모든 실패를 세분화하는 것이 목적은 아니다. 사용자가 다시 입력할 수 있는 실패와 권한 요청이 필요한 실패처럼, 다음 행동이 다를 때 구분할 이유가 생긴다.

## 필드가 없는 것과 undefined를 넣는 것

결과의 상태를 나눴더라도 optional 필드는 여전히 필요하다. 사용자에게 별명이 없을 수도 있다고 하자.

```ts
type UserWithNickname = User & {
  nickname?: string;
};

const withoutNickname: UserWithNickname = {
  id: "u1",
  name: "채현",
};
```

이때 필드를 생략하는 것과 `nickname: undefined`를 넣는 것은 JavaScript에서 관찰 가능한 차이가 있다.

```ts
const absent = {};
const present = { nickname: undefined };

console.log("nickname" in absent);  // false
console.log("nickname" in present); // true
```

`exactOptionalPropertyTypes`를 켜면 `nickname?: string`에 `undefined`를 직접 대입할 수 없다. 그것도 허용하려면 `nickname?: string | undefined`라고 적는다. 다만 필드를 읽을 때는 여전히 없을 수 있으므로 `string | undefined`로 다뤄야 한다. [exactOptionalPropertyTypes](https://www.typescriptlang.org/tsconfig/exactOptionalPropertyTypes.html)

수정 API를 설계할 때도 이 구별을 생각해 볼 수 있다. 이 글에서는 ‘필드 생략은 유지, 문자열은 변경, `null`은 삭제’라는 규칙을 정해 보자.

```ts
type NicknamePatch = {
  nickname?: string | null;
};

function explainPatch(patch: NicknamePatch): string {
  if (!("nickname" in patch)) {
    return "기존 별명 유지";
  }
  if (patch.nickname === null) {
    return "기존 별명 삭제";
  }
  return `별명을 ${patch.nickname}으로 변경`;
}
```

이 의미는 우리가 정한 API 계약이다. 옵션을 켠다고 업데이트 정책이 자동으로 생기지는 않는다. 먼저 어떤 입력을 구별해야 하는지 정하고 그 구별이 타입에 남도록 만드는 순서다.

## 인덱스로 꺼낸 값도 존재를 확인하기

사용자 조회를 실제로 구현할 때는 자료구조에서 값을 꺼내야 한다. 예제를 단순하게 만들기 위해 ID를 키로 갖는 메모리 객체를 사용하자.

```ts
const users: Record<string, User> = {
  u1: { id: "u1", name: "채현" },
};

function findUser(id: string): LookupResult {
  const user = users[id];

  if (user === undefined) {
    return { kind: "notFound", requestedId: id };
  }

  return { kind: "found", user };
}
```

임의의 문자열로 조회한다고 그 키가 항상 존재하지는 않는다. `noUncheckedIndexedAccess`를 켜면 이 인덱스 접근 결과에 `undefined`가 포함된다. 그래서 `user.name`을 바로 사용하기 전에 존재 여부를 확인하게 된다. 일반 배열의 인덱스 접근도 같은 종류의 문제를 가진다. [noUncheckedIndexedAccess](https://www.typescriptlang.org/tsconfig/noUncheckedIndexedAccess.html)

이 예제에는 권한 모델을 넣지 않았으므로 `findUser`는 `found`와 `notFound`만 만든다. 앞에서 정의한 `forbidden`은 권한 검사를 하는 조회 흐름까지 표현하려고 남겨 둔 상태다.

이제 함수 안과 밖의 역할이 연결된다. 함수 안에서는 값이 없을 가능성을 확인하고, 바깥에는 그 결과를 이름 있는 상태로 반환한다. 호출자는 저장소가 객체인지 DB인지 몰라도 결과를 처리할 수 있다.

예제의 컴파일러 설정은 다음과 같다. 뒤의 두 옵션은 `strict`를 켰다고 함께 활성화되지는 않으므로 별도로 적었다.

```json
{
  "compilerOptions": {
    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true,
    "target": "ES2022"
  }
}
```

## 타입으로 표현한 결과의 바깥

지금까지는 이미 TypeScript 코드 안에 들어온 값을 다뤘다. 네트워크 응답에 `as LookupResult`를 붙여도 서버가 그 형태를 보냈는지 검사되지는 않는다. 타입 단언은 런타임 검사 코드를 만들지 않기 때문이다. 외부 입력은 `unknown`으로 받고 실제 구조를 검사한 뒤 내부 타입으로 다뤄야 한다. [TypeScript 타입의 제거](https://www.typescriptlang.org/docs/handbook/2/basic-types.html#erased-types)

성공과 실패를 유니언으로 반환하는 함수가 예외를 던지지 않는다는 보장도 생기지 않는다. 메모리 객체를 읽는 예제를 DB 조회로 바꾸면 연결 오류가 발생할 수 있다. `Promise<LookupResult>`는 성공적으로 이행됐을 때의 값을 설명하며, 거부되는 이유를 타입 매개변수로 제한하지 않는다.

따라서 `notFound`로 처리할 상황과 호출자에게 전파할 실행 오류를 정해야 한다. 모든 예외를 잡아 `notFound`로 바꾸면 DB 연결 실패도 사용자가 없다는 응답이 된다. 호출자가 결과를 보고 할 수 있는 판단이 오히려 부정확해진다.

타입을 설계할 때는 ‘어떤 자료형을 넣을까’와 함께 ‘어떤 조합이 실제로 가능한가’를 물어보게 된다. 사용자 데이터와 에러를 독립적으로 optional로 두었던 결과를 상태별로 나누자, 생성하는 쪽과 사용하는 쪽이 같은 규칙을 따르게 됐다.

다음으로 남는 문제는 실패한 작업의 흔적이다. 조회를 넘어 데이터를 저장하는 함수라면, 실패를 반환할 때 이미 한 쓰기가 남아도 되는지까지 결정해야 한다. 그 지점에서 타입 설계가 트랜잭션 설계로 이어진다.

---

작성 메모: [[블로그/초안/블로그 - 실패 처리와 트랜잭션 경계 설계하기|다음 초안 — 실패 처리와 트랜잭션 경계 설계하기]]
