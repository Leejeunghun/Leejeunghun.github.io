---
title: "React Type TypeScript 공부 "
date: 2024-06-24
last_modified_at: 2024-06-24
categories:
  - React
tags:
  - React
  - TypeScript


published: true
toc: true
toc_sticky: true
---

# 목적

1. 기본 함수 바꾸기


기존
```ts
funciton App()

-> 

const App:React.FC = () => {

}

```

2. useState 사용법

useState는 React에서 제공하는 Hook 중 하나로, 함수형 컴포넌트에서 상태 관리를 가능하게 해주는 기능입니다. 기본적으로 useState 함수는 하나의 인자를 받으며, 이 인자는 상태의 초기값을 나타냅니다useState는 배열을 반환하고, 이 배열의 첫 번째 원소는 현재 상태를 나타내며, 두 번째 원소는 상태를 변경하는 함수입니다1.

<h2>Hook 이란</h2>
```
React Hook은 React 버전 16.8에서 도입된 새로운 기능으로, 함수형 컴포넌트에서도 클래스 컴포넌트의 상태 관리(state)와 생명주기 기능(lifecycle features)을 사용할 수 있게 해주는 함수들입니다

```


다음은 useState를 사용하는 기본적인 예시입니다:

JavaScript
```ts
import React, { useState } from 'react';

function Counter() {
  const [value, setValue] = useState(0); // 0은 초기값입니다.

  return (
    <div>
      <p>현재 값은 {value} 입니다.</p>
      <button onClick={() => setValue(value + 1)}>+1</button>
      <button onClick={() => setValue(value - 1)}>-1</button>
    </div>
  );
}
```
AI가 생성한 코드입니다. 신중하게 검토하고 사용하세요.

위 예제에서는 useState(0)을 통해 value라는 상태를 만들고 초기값을 0으로 설정했습니다. 그리고 버튼을 클릭할 때마다 setValue 함수를 통해 value의 값을 더하거나 빼서 갱신하고 있습니다.

useState로 상태를 갱신하는 작업은 비동기적으로 이루어지기 때문에, 값이 항상 바로바로 갱신되지 않을 수 있습니다. 이러한 문제를 해결하기 위해서는, setValue 함수에 함수를 인자로 전달하는 방법을 사용할 수 있습니다. 이 함수는 이전 상태 값을 매개변수로 받아 새로운 상태 값을 반환합니다1.

예를 들어:

JavaScript

setValue(prevValue => prevValue + 1);
AI가 생성한 코드입니다. 신중하게 검토하고 사용하세요. FAQ의 자세한 정보.
이렇게 하면, setValue 호출 후에 value는 증가된 값으로 갱신됩니다. 이 방법을 통해 useState의 비동기적인 동작으로 인한 문제를 해결할 수 있습니다1.



```ts


```



# 자료 출처
1. 메인 강의 자료
https://www.youtube.com/watch?v=FJDVKeh7RJI&ab_channel=freeCodeCamp.org

2. 강의자 유투버
https://www.youtube.com/@RoadsideCoder

3. typeScipt 설치 링크
https://create-react-app.dev/docs/adding-typescript/

4. typeScirpt 참조
https://www.typescriptlang.org/ko/

5. 코드 출처
https://github.com/piyush-eon/react-typescript-taskify

6. 자동완성 프로그램 추천
ES7 React/Redux/GraphQL/R