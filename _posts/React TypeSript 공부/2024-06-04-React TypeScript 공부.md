---
title: "React Type Scipt 공부 "
date: 2024-06-04
last_modified_at: 2024-06-04
categories:
  - 컴퓨터비전
tags:
  - 컴퓨터비전


published: true
toc: true
toc_sticky: true
---

# 목적
React와 자바 스크립트를 이용하여 좋은 프로젝트를 만들기위해


# 순서
1. [Type Scipt가 무엇인가?](#type-scipt가-무엇인가)
2. [프로젝트 결과 미리 보기](#프로젝트-결과-미리-보기)
3. [React Typescript 프로젝트 세팅](#react-typescript-프로젝트-세팅)
4. [TypeScipt 기본](#TypeScipt-기본)
#  Type Scipt가 무엇인가?

TypeScipt는 java Script를 기반으로 만들어 졌다.

# 프로젝트 결과 미리 보기
[최종 결과 ](/assets/img/React/1%EA%B0%95%20%EC%98%81%EC%83%81.png)


# React Typescript 프로젝트 세팅

 타입 스크립트를 시작하는 리액트 만들기 위해서 다음을 시행한다.
```sh
npx create-react-app typescript-tutorial --template typescript
```

 해당 프로젝트에 들어가서 프로그램이 제대로 돌아가는확인한다.

 ```
 npm start
 ```

 사용하지 않을 파일 삭제한다
 1. logo.svg( 파일안에 들어가서 쓰는곳도 지워야한다)
 2. setupTest.ts
 3. reportWebVitals.ts (파일에서 쓰는곳이 있는데 찾아서 지우면된다)
 4. App.test.tsx

# TypeScipt 기본
1. 변수 선언시 자료형을 밝혀줘야한다

``` Ts
let name: String  = "Leejeunghun";
```

2. 자료형을 맞는 형태를 써야한다

```Ts
let name: string;
let age : number;
let isStudent : boolean;
let hobbies: string[]; 
let role:[number,string] ;//튜플



// 오브젝트도 선언이 가능하다
type Person= {
    name : string;
    age? : number; // 옵션으로 추가할 수 있따.
}

let person:Person = {
    name: "piysh",
    age : 22,
}

let lotsOfPeople: Person[]; //오브젝트 리스트 선언이 가능하다.

//union 선언 숫자 글자 둘다 쓰고 싶을때
let age : number | string;

/** 함수 선언 방법 1번
function printName (name: string){
    console.log(name);
}
printName("piysh")
*/

/** 함수 선언 방법 2번 
let printName : (name: String) => never;
let printName : (name: String) => void;
차이가 있다.
*/

let name : any; // 제한이 없는 함수 권장하지 않음 
let personName: unknown; // 알지 못하는 경우

name = 5; // 에러
role = [5,"something"];



와
interface person = {
    name: "piysh",
    age : 22,
}

interface Guy extends Person {
    Profession:string;
}
연장하면서 쓸수 있다.


type X = {
    a: string;
    b: number;
};

type Y = X &{
    c: string;
    d: number;
};

// 에러 발생
let y:Y = { 
    c: 'efas'
    d: 42,
};

```


# 참조
 1. 확장 프로그램 사용후 해당 명령어 수행시 기본적인 틀 만들어준다
```
rafce

```
2. css 선언 참조 규칙 공부 필요


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