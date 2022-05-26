---
layout: post
title:  자동접수의 타입스크립트 활용 사례
subtitle: 타입스크립트 이렇게까지 사용해봤다
gh-repo: KyungJuneKim/myrepository
gh-badge: [follow]
tags: [Typescript, Maniplation, React]
comments: true
sitemap:
  changefreq: daily
  priority : 1.0
---

# # Content

- [# Case of Application](#-case-of-application)
  - [No type casting, use type predicates](#no-type-casting-use-type-predicates)
  - [`QuestionnaireType` - type manipulation](#questionnairetype---type-manipulation)
  - [`GroupByProperty` - type manipulation](#groupbyproperty---type-manipulation)
  - [`Theme` & `variant` - declaration merging](#theme--variant---declaration-merging)
  - [`of` & `ofMap` - autocompletition](#of--ofmap---autocompletition)
    - [`of` 함수](#of-함수)
    - [`ofMap` 함수](#ofmap-함수)

# # Case of Application

본 세미나의 주제는 `<자동접수>에서 타입스크립트를 어떻게 활용했는가`이다. 예시 코드는 <자동접수>의 코드를 각색했다.

우선 타입스크립트에 대해 짧막히 얘기하자면, 타입스크립트는 `type`을 꽤나 자세히 표현할 수 있다는 장점이 있다. 하지만 코드 뿐만 아니라 `type`도 자세히 정의하며 많은 시간을 소요하게 된다. 타입스크립트를 적당히 사용한다는 것은 컴파일타임에 버그를 예방하기 위한 시간과 런타임에 디버깅하는 시간의 **trade-off** 라고 생각한다. 타입스크립트를 이제 막 익혔다면, 아직 런타임에 디버깅하는 것이 더 효율적일 수 있다. 하지만 동료들과 협업하다보면 대화와 문서로 공유하는 것보다 타입스크립트로 공유하는 것이 효율적일 것이다. 본 세미나를 통해서 타입스크립트로 동료들과 어떻게 소통하고 공유할 수 있는지 알고, 상황에 맞게 타입스크립트를 적재적소에 적용할 수 있길 바란다.

---

## No type casting, use type predicates

**Problem**

```tsx
// type of event.target.value, the parameter of onChange, is string. You cannot fix it.
const fooBar = ['foo', 'bar'] as const;
type FooBar = typeof fooBar[number];

function FooOrBar() {
  const [item, setItem] = useState<FooBar>('foo');

  return (
    <Select
      value={item}
      onChange={({ target: { value } }) => setItem(value)}  <- TS error
    >
      {fooBar.map<ReactNode>((value) => (
        <MenuItem value={value} key={value}>
          {value}
        </MenuItem>
      ))}
    </Select>
  );
}
```

`Select` 컴포넌트의 `onChange`속성은 매개변수로 `event`를 넘겨주는데, `event.target.value`의 `type`은 `string`이다. `item`의 `type`과 `setItem` 매개변수의 `type`은 `FooBar`기 때문에 `setItem`의 인자로 `event.target.value`를 넘겨줄 수 없다.

**Solution1**

```tsx
// type of event.target.value, the parameter of onChange, is string. You cannot fix it.
const fooBar = ['foo', 'bar'] as const;
type FooBar = typeof fooBar[number];

function FooOrBar() {
  const [item, setItem] = useState<FooBar>('foo');

  return (
    <Select
      value={item}
      onChange={({ target: { value } }) => setItem(value as FooBar)}
    >
      {fooBar.map<ReactNode>((value) => (
        <MenuItem value={value} key={value}>
          {value}
        </MenuItem>
      ))}
    </Select>
  );
}
```

가장 간단한 방법은 `event.target.value`를 `FooBar`로 형변환하는 것이다. 하지만 `event.target.value`가 정말 `'foo'` 또는 `'bar'` 일까? 만약 아니라면 어떻게 처리해야 할까?

**Solution2**

```tsx
// type of event.target.value, the parameter of onChange, is string. You cannot fix it.
const fooBar = ['foo', 'bar'] as const;
type FooBar = typeof fooBar[number];

function FooOrBar() {
  const [item, setItem] = useState<FooBar>('foo');

  return (
    <Select
      value={item}
      onChange={({ target: { value } }) => isFooBar(value) && setItem(value)}
    >
      {fooBar.map<ReactNode>((value) => (
        <MenuItem value={value} key={value}>
          {value}
        </MenuItem>
      ))}
    </Select>
  );
}

function isFooBar(something: unknown): something is FooBar {
  // allow type assertion in function using type predicate
  return fooBar.includes(something as FooBar);
}
```

자동접수에서는 `type predicates`를 이용하여 `event.target.value`의 `type`을 추론한다. 형변환에 비해 보일러 플레이트가 많지만, 그만큼 안전한 코드를 작성 할 수 있다. 타입스크립트는 컴파일타임에만 작동하기 때문에 실제로 다른 `type`의 값이 전달될 수 있으며, 따라서 무리한 형변환은 오히려 버그를 찾기 어렵고 개발자를 헷갈리게 한다. 얽혀있는 내용이 많다면 형변환 대신 `type predicates` 함수를 사용하는 것을 권장하며, <자동접수>는 대부분의 상황에서 `type predicates` 함수를 사용한다. `type predicates`의 내용은 [링크](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#using-type-predicates)에서 확인하길 바라며, `type predicates` 뿐만 아니라 `narrowing` 챕터에서 `type`을 추론하는 내용을 다루고 있으니 읽어보는 것을 추천한다.


## `QuestionnaireType` - type manipulation

**Problem**

1. 설문지`Questionnaire`는 여러 질문`Question`으로 구성되어있다.
2. 질문`Question`의 종류는 단수선택 객관식`SINGLE_CHOICE`, 복수선택 객관식`MULTIPLE_CHOICE`, 단답식`NARRATIVE`, 이미지형`IMAGE` 4가지이다.
3. 객관식 질문(`SINGLE_CHOICE`와 `MULTIPLE_CHOICE`)에는 여러 개의 문항`Item`이 존재하며 문항`Item`의 종류는 평문`TEXT`, 단답`INPUT`, 이미지`IMAGE` 3가지가 있다.
4. 단답식 질문`NARRATIVE`과 이미지형 질문`IMAGE` 도 문항`Item`이 존재하며 문항`Item`의 종류는 각각 `INPUT`과 `IMAGE` 이다.
5. 설문지`Questionnaire`, 질문`Question`, 문항`Item` 문항은 고유한 키가 있다.

**Solution1**

```ts
type ItemType = 'TEXT' | 'INPUT' | 'IMAGE';
type QuestionType =
  | 'SINGLE_CHOICE'
  | 'MULTIPLE_CHOICE'
  | 'NARRATIVE'
  | 'IMAGE';

type Item = {
  key: number;
  itemType: ItemType;
};

type Question = {
  key: number;
  questionType: QuestionType;
  items: Item[];
};

type Questionnaire = {
  key: number;
  Questions: Question[];
};
```

위 코드는 `QuestionnaireType`을 정의하는 간단한 방법이다. 아마 많은 언어에서 같은 문제를 직면하면 위 코드처럼 작성할 것이다. 하지만 질문과 문항의 관계를 코드만 봐서는 알 수 없다.

**Solution2**

```ts
type ChoiceItemType = 'TEXT' | 'INPUT' | 'IMAGE';
type NarrativeItempType = 'INPUT';
type ImageItemType = 'IMAGE';

type ChoiceQuestionType = 'SINGLE_CHOICE' | 'MULTIPLE_CHOICE';
type NarrativeQuestionType = 'NARRATIVE';
type ImageQuestionType = 'IMAGE';

type CommonItem = {
  key: number;
};

type ChoiceItem = CommonItem & {
  itemType: ChoiceItemType;
};
type NarrativeItem = CommonItem & {
  itemType: NarrativeItempType;
};
type ImageItem = CommonItem & {
  itemType: ImageItemType;
};

type Question = {
  key: number;
} & (
  | {
      questionType: ChoiceQuestionType;
      items: ChoiceItem[];
    }
  | {
      questionType: NarrativeQuestionType;
      items: NarrativeItem[];
    }
  | {
      questionType: ImageQuestionType;
      items: ImageItem[];
    }
);

type Questionnaire = {
  key: number;
  questions: Question[];
};
```

`union type`을 사용해서 `QuestionnaireType`을 정의했다. 질문과 문항의 관계는 표현됐지만, 구현과 동시에 관계가 표현되어 있다. 더 복잡한 구조에선 직관적이지 못하다.

**Solution3**

```ts
type QuestionItemType = {
  SINGLE_CHOICE: 'TEXT' | 'INPUT' | 'IMAGE';
  MULTIPLE_CHOICE: 'TEXT' | 'INPUT' | 'IMAGE';
  NARRATIVE: 'INPUT';
  IMAGE: 'IMAGE';
};

// 'SINGLE_CHOICE' | 'MULTIPLE_CHOICE' | 'NARRATIVE' | 'IMAGE'
type QuestionType = keyof QuestionItemType;

type Item<T extends QuestionType> = {
  [K in T]: {
    key: number;
    itemType: QuestionItemType[K];
  };
};

type Question<T extends QuestionType> = {
  [K in T]: {
    key: number;
    questionType: K;
    items: Item<K>[];
  };
};

type Questionnaire = {
  key: number;
  Questions: Question<QuestionType>[];
};
```

질문과 문항의 관계를 `QuestionItemType`에 정의하여 구현과 관계의 표현을 구분했다. 질문과 문항의 관계가 형성되어 보다 엄밀한 `QuestionnaireType`이 되었다.


## `GroupByProperty` - type manipulation

**Definition**

```ts
type FilterKeyByValueMap<T, V> = {
  [K in keyof T]: T[K] extends V ? K : never;
};

type FilterKeyByMap<T, K extends keyof T> = T[K] extends never ? never : T[K];

type GroupByProperty<
  T,
  P extends FilterKeyByMap<
    FilterKeyByValueMap<T[keyof T], keyof any>,
    keyof T[keyof T]
  >,
> = {
  [K in T[keyof T][P] as K extends keyof any ? K : never]: keyof {
    [K2 in keyof T as T[K2][P] extends K ? K2 : never]: undefined;
  };
};
```

`GroupByProperty`는 관계를 형성하기 위해 고안된 `type`이다. 이름에서도 알 수 있지만, 같은 속성끼리 묶어준다. 아래 예시를 보며 `GroupByProperty`의 역할을 설명하겠다.

**Example**

```ts
type FamilyType = 'Kim' | 'Park';

const peopleMap = {
  Juliet: { koreanName: '줄리엣', family: 'Kim', gender: 'F' },
  Romeo: { koreanName: '로미오', family: 'Park', gender: 'M' },
  Pat: { koreanName: '패트', family: 'Kim', gender: 'M' },
  Mat: { koreanName: '매트', family: 'Park', gender: 'M' },
} as const;

// 'Juliet' | 'Romeo' | 'Pat' | 'Mat'
type People = keyof typeof peopleMap;
// 'Juliet' | 'Pat'
type KimFamily = GroupByProperty<typeof peopleMap, 'family'>['Kim'];
// 'Romeo' | 'Mat'
type ParkFamily = GroupByProperty<typeof peopleMap, 'family'>['Park'];
// 'Juliet'
type FemalePeople = GroupByProperty<typeof peopleMap, 'gender'>['F'];
// 'Mat'
type MalePeople = GroupByProperty<typeof peopleMap, 'gender'>['M'];

const groupByFamily: {
  [K in FamilyType]: Array<
    GroupByProperty<typeof peopleMap, 'family'>[K]
  >;
} = {
  Kim: ['Juliet', 'Pat'],
  Park: ['Romeo', 'Mat'],
};

groupByFamily.Kim.forEach((value) => console.log(peopleMap[value].koreanName));
```

`peopleMap` 객체는 사람과 속성(국문명, 가문, 성별)의 관계를 표현하고 있다. `GroupByProperty`를 이용하면 `peopleMap` 객체의 사람들을 속성별로 나누어 `type`으로 관리할 수 있다. 어느 함수는 모든 사람을 받고 싶고 어느 함수는 김씨 가문만 받고 싶을 때, 매번 `type`을 정의하는 것이 아니라 관계를 표현하고 있는 `peopleMap`을 통해서 구현할 수 있다.

`groupByFamily`은 가문별 배열을 표현하고 있다. 이때, 타입스크립트로 실제로 해당 가문에 속하는 사람이 잘 작성되었는지 확인하기 위해 `GroupByProperty`를 사용했다.

또한 `peopleMap` 객체는 `type`이 아닌 값이기 때문에 실제 코드에서 값으로 사용할 수 있다. 마지막 줄의 코드는 김씨 가문 사람들의 국문명을 `console`에 출력한다.

**Cons of `Enum`**

1. 다른 `Enum`과 관계 형성이 어렵다.
2. `key`와 `value`가 다르면 불편한 상황이 발생할 수 있다.

```ts
enum FooBar {
  FOO = 'fff',
  BAR = 'bbb',
}

const translation: Record<FooBar, string> = {
  [FooBar.FOO]: '푸',
  [FooBar.BAR]: '바',
  // fff: '푸',
  // bbb: '바',
};

console.log(translation[FooBar.FOO]);
console.log(translation.fff);
```

`Enum`을 쓰지 말자는 얘기는 아니다. 하지만 `dot` 연산자를 쓰지 못하거나, 매번 대괄호를 써야하는 불편함 있는 것은 분명하다. 단순한 상황에선 `Enum`이 좋지만, 복잡한 관계를 구현해야 한다면 다른 방안을 고려해보자.


## `Theme` & `variant` - declaration merging

```ts
declare module '@mui/material/styles' {
  export interface Theme {
    colors: typeof COLORS;
    styles: typeof STYLES;
  }
}
```

타입스크립트의 `interface`는 같은 이름으로 여러 번 정의될 수 있으며 각기 다른 파일에 정의되어도 괘찮다. 그리고 해당 `interface`는 각 정의들의 교집합이 된다. 이 점을 이용하여 `package`에 정의된 `interface`에 원하는 속성을 추가할 수 있다. 자동접수에서는 `emotion`의 `Theme`과 `mui`의 `variant`에 원하는 속성을 추가하여 사용하고 있다. 자세한 사용법은 `mui` 공식 문서에서 확인하기 바란다.

- [theming - custom variables](https://mui.com/customization/theming/#custom-variables)
- [typography - adding & disabling variants](https://mui.com/customization/typography/#adding-amp-disabling-variants)

`intersection type`과 함께 `declaration merging`에서 **주의**할 점이 있다.

```ts
// intersection type
type A = { foo: number; };
type B = { foo: string; };
type Foo = A & B;

// declaration merging
interface Foo { foo: number; }
interface Foo { foo: string; }
```

위 예시의 `type Foo`와 `interface Foo`의 결과는 아래와 같다.

```ts
type Foo = { foo: never; };
interface Foo { foo: never; }
```

`string`과 `number`의 교집합이 `never`이므로 위와 같은 결과가 된다. 이렇게 될 경우, 대부분 원하는 대로 작동하지 않을 것이다. `intersection type`과 `declaration merging`을 사용할 때는 같은 이름의 속성이 있는지 확인해주지 않고 각 정의가 다른 파일에 있다면 확인하기 어렵기 때문에 주의가 필요하다.


## `of` & `ofMap` - autocompletition

![meme-typescript-autoplugin.png]({{ site.baseurl }}/assets/images/auto_recipt_typescript_usecase/meme-typescript-autoplugin.png)

위 이미지는 `타입스크립트는 자동완성 플러그인`이라는 유명한 밈이다. 이번에는 자동완성을 활용하는 방법에 대해 얘기한다.

P.S. 개발환경에 따라 자동완성 여부는 달라질 수 있다. 본 내용은 [타입스크립트 플레이그라운드](https://www.typescriptlang.org/play)에서  확인했다.

**Definition**

```ts
function of<T>(): (obj: T) => T;
function of<T>(obj: T): T;
function of<T1>(obj1?: T1) {
  return arguments.length !== 0 ? obj1 : <T2>(obj2: T2) => obj2;
}
```

```ts
const ofMap =
  <T>() =>
  <R extends Record<string, T>>(map: R): R =>
    map;
```

자동접수에서는 자동완성과 `type` 검사를 위해 `of`와 `ofMap`이라는 함수를 정의해서 사용한다. 둘의 공통점은 어떤 인자를 받아서 그대로 반환한다는 것이다. 즉, 이 함수들의 사용유무는 실행 결과에 영향을 미치지 않는다. 아래 문제를 통해 사용 이유와 역할을 알아보자.

### `of` 함수

**Problem**

`variant`에 따라 `CSSObject`를 반환하라. 색상과 배경색상은 `variant`와 무관하며, `variant`에 따라 폰트크기와 높이가 결정된다.

**Solution1**

```ts
const cssObject = of<CSSObject>();
const buttonStyle = (variant: 'large' | 'small'): CSSObject => ({
  color: 'black',
  backgroundColor: 'white',
  ...(variant === 'large' && {
    fontSize: '26px',
    height: '65px',
  }),
  ...(variant === 'small' && {
    fontSize: '16px',
    height: '34px',
  }),
});
```

여러 방법이 있겠지만, 위의 코드처럼 `spreading`을 이용하여 구현할 수 있다. 이 때, 반환형이 `CSSObject`로 정해져 있기 때문에 많은 개발환경에서 색상과 배경색상은 `color`와 `backgroundColor`로 자동완성된다. 하지만 폰트크기와 높이를 반환하는 객체는 자동완성되지 않을 뿐더러 올바른 `type`인지 검사되지 않는다.

**Solution2**

```jsx
const cssObject = of<CSSObject>();
const buttonStyle = (variant: 'large' | 'small'): CSSObject => ({
  color: 'black',
  backgroundColor: 'white',
  ...(variant === 'large' &&
    cssObject({
      fontSize: '26px',
      height: '65px',
    })),
  ...(variant === 'small' &&
    cssObject({
      fontSize: '16px',
      height: '34px',
    })),
});
```

`Solution1`과 같은 방법이지만 `of`함수로 `cssObject`함수를 정의하여 사용하고 있다. `cssObject`는 `CSSObject` 객체를 인자로 받고 그대로 반환한다. 이 과정에서 인자의 `type`을 검사하고 자동완성을 지원하여 안전한 코드를 작성할 수 있다.

### `ofMap` 함수

`ofMap` 함수는 `of` 함수처럼 전달받은 인자를 그대로 반환하지만 반환 `type`에서 차이를 보인다. `of` 함수는 `type`도 똑같이 반환하지만, `ofMap` 함수는 입력된 타입을 `extends`하는 `type`을 반환한다. 즉, 전달 받는 인자의 `type`은 `Record<string, T>`만 만족하면 되고 반환형은 이를 `extends`하는 `type`으로 많은 상황에서 `literal type`이 되어 강력한 자동완성 기능을 제공한다. 아래 문제를 보며 차이를 살펴보자.

**Problem**

검정색, 흰색, 회색의 색상코드를 저장하는 객체를 구현하라.
객체 작성 시 `value`의 `type` 검사 여부와 객체 호출 시 `key`의 자동완성 여부를 확인하라.

**Solution1**

```jsx
const COLORS = {
  BLACK: '#000000',
  WHITE: '#FFFFFF',
  GRAY: '#808080',
};
```

가장 간단한 구현일 것이다. 객체 호출 시 `key`의 자동완성이 확인됐지만 객체 작성 시 `value`의 `type`은 검사되지 않는다.

**Solution2**

```jsx
const COLORS: Record<string, Property.Color> = {
  BLACK: '#000000',
  WHITE: '#FFFFFF',
  GRAY: '#808080',
};
```

`value`의 `type`을 검사하기 위해 `Record`를 사용하여 `type`을 지정했다. 하지만 객체 호출 시 `key`가 자동완성되지 않았다.

**Solution3**

```ts
const createColors = ofMap<Property.Color>();
const COLORS = createColors({
  BLACK: '#000000',
  WHITE: '#FFFFFF',
  GRAY: '#808080',
} as const);
```

`ofMap` 함수로 구현하면 `value`의 `type`이 `Property.Color`인지 검사되고, `COLORS.BLACK`, `COLORS.WHITE`, `COLORS.GRAY` 호출 시 모두 자동완성된다.

`ofMap` 함수의 역할을 용례를 통해 알아봤다. 위에 나왔던 `peopleMap`도 `ofMap` 함수로 안전하게 구현할 수 있다. 아래는 `peopleMap`을 더 안전하게 구현한 코드이다.

```jsx
const peopleMap = ofMap<{
  family: FamilyType;
  gender: 'F' | 'M';
}>()({
  Juliet: { koreanName: '줄리엣', family: 'Kim', gender: 'F' },
  Romeo: { koreanName: '로미오', family: 'Park', gender: 'M' },
  Pat: { koreanName: '패트', family: 'Kim', gender: 'M' },
  Mat: { koreanName: '매트', family: 'Park', gender: 'M' },
} as const);
```

**Summary of `ofMap` function**

```ts
// support type checking when we define, but no autocomplete when we call.
const colorsWithRecord: Record<string, Property.Color> = { ... };

// support autocomplete when we call, but no type checking when we define.
const colorsWithAsConst = { ... } as const;

// support type checking and autocomplete
const colorsWithCreateColors = createColors({ ... } as const);
```

**Extra**

위의 예시를 보면 `ofMap` 함수의 인자에 `as const`가 붙어있는 것을 확인할 수 있다. 일반적인 경우, `as const`가 없어도 `value`의 `type` 검사와 `key`의 자동완성에 이상없다. `as const`는 `value`를 `literal type`으로 쓰기 위해 사용한다.

```jsx
const peopleMap = ofMap<{
  family: FamilyType;
  gender: 'F' | 'M';
}>()({
  Juliet: { koreanName: '줄리엣', family: 'Kim', gender: 'F' },
  Romeo: { koreanName: '로미오', family: 'Park', gender: 'M' },
  Pat: { koreanName: '패트', family: 'Kim', gender: 'M' },
  Mat: { koreanName: '매트', family: 'Park', gender: 'M' },
});
```

위와 같이 `peopleMap`을 `as const` 없이 정의할 경우, `peopleMap.koreanName`의 `type`은 `string`이다. 하지만 `koreanName`에 대해 `GroupByProperty`를 적용하고 싶다면 `literal type`으로 만들어야 하므로 상황에 따라 `as const`를 써야 한다.

