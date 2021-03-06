# React Hook mental model

## 액션을 업데이트로부터 분리하기

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  const [step, setStep] = useState(1);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + step);
    }, 1000);
    return () => clearInterval(id);
  }, [step]);
  return (
    <>
      <h1>{count}</h1>
      <input value={step} onChange={e => setStep(Number(e.target.value))} />
    </>
  );
}
```
[데모](https://codesandbox.io/s/zxn70rnkx)

위 코드는 잘 동작합니다. 다만 마음에 걸리는 것은 step이 바뀔때 마다 `interval`이 clear되고 다시 시작되는 것이죠. 이러한 초기화를 하지 않으려면 어떻게 해야할까요? 그러기 위해 deps에서 `step`을 지워야할텐데 어떻게 해야할까요?

__어떤 상태 변수(count)가 다른 상태 변수의 현재 값(step)에 연관되도록 설정하려고 한다면, 두 상태 변수 모두 useReducer 로 교체해야 할 수 있습니다.__

혹시 여러분의 코드에서 `setSomething(something => ...)` 코드가 보인다면, 리듀서를 고려해보기 좋은 타이밍입니다.

리듀서는 __컴포넌트에서 일어나는 "액션"의 표현(여기서는 setCount(...)겠죠?)과 그 반응으로 상태가 어떻게 업데이트되어야 할지를(c => c + step) 분리합니다.__

```javascript
const [state, dispatch] = useReducer(reducer, initialState);
const { count, step } = state;

useEffect(() => {
  const id = setInterval(() => {
    dispatch({ type: 'tick' }); // setCount(c => c + step) 대신에
  }, 1000);
  return () => clearInterval(id);
}, [dispatch]);
```
`dispatch`는 컴포넌트가 사라지지 않는 한 항상 같습니다. 따라서 __interval은 초기화되지 않습니다.__(문제 해결!)

이처럼 이펙트 안에서 상태를 읽는 대신 _무슨일이 일어날지_ 알려주는 정보를 인코딩하는 _액션_을 전파합니다.

이로써 이펙트는 __어떻게__ 상태를 업데이트할지 신경쓰지 않고, 단지 __무슨일이 일어날지__ 알려줍니다. 그 끝에서 리듀서는 업데이트 로직을 담아냅니다.

```javascript
const initialState = {
  count: 0,
  step: 1,
};

function reducer(state, action) {
  const { count, step } = state;
  if (action.type === 'tick') {
    return { count: count + step, step };
  } else if (action.type === 'step') {
    return { count, step: action.step };
  } else {
    throw new Error();
  }
}
```

## 왜 `useReducer`가 `Hooks` 치트 모드인가

지금까지 이펙트가 이전 상태를 기준으로 상태를 변경할 때, 의존성을 제거하는지 살펴 봤습니다. 하지만 __다음 상태를 계산하는데 props가 필요하다면?__ 다음과 같이 컴포넌트를 작성한다면 어떨까요?

```javascript
<Counter step={1} />
```

리듀서는 `props`를 알지 못하기 때문에 deps에 넣어야 합니다.

```javascript
// 하지만 reducer를 컴포넌트 안에 작성한다면?
function Counter({ step }) {
  const [count, dispatch] = useReducer(reducer, 0);

  function reducer(state, action) {
    if (action.type === 'tick') {
      return state + step;
    } else {
      throw new Error();
    }
  }

  useEffect(() => {
    const id = setInterval(() => {
      dispatch({ type: 'tick' });
    }, 1000);
    return () => clearInterval(id);
  }, [dispatch]);

  return <h1>{count}</h1>;
}
```

- `dispatch`는 액션(`{type: 'tick'}`)만을 기억한 채 rerendering을 실행합니다.
- 다음 렌더링시에 컴포넌트 내부의 reducer라는 이름의 함수가 다시 useReducer에 의해 호출됩니다.
- 이 순간의 props는 reducer함수 스코프에서 접근할 수 있으므로 이펙트 내부와는 관련이 없게됩니다.

이로써 state와 props를 함께 품은채, __업데이트 로직과 그로 인해 무엇이 일어나는지 서술하는 것을 분리할__ 수 있도록 만들어줍니다. 이로써 이펙트의 불필요한 의존성을 제거하고 이로인해 필요할 때보다 더 자주 실행되는것을 피할 수 있도록 도와줍니다.

치트모드라 부를만 하군요.

> note: 의존성을 최소화 하는 방향으로 이펙트를 작성해라?

## 함수를 이펙트 안으로 옮기기

흔한 실수

```javascript
function SearchResults() {
  const [data, setData] = useState({ hits: [] });

  async function fetchData() {
    const result = await axios(
      'https://hn.algolia.com/api/v1/search?query=react',
    );
    setData(result.data);
  }

  useEffect(() => {
    fetchData();
  }, []); // 거짓말 하지 말라 그랬을 텐데?
}
```

동작은 합니다만, __로컬 함수를 의존성에서 제외하는 것은__ 컴포넌트가 더 복잡해질 경우 일관된 동작을 보장하기 어려울 수 있다는 것입니다.

```javascript
function SearchResults() {
  // 이 함수가 길다고 상상해 봅시다
  function getFetchUrl() {
    return 'https://hn.algolia.com/api/v1/search?query=react';
  }

  // 이 함수도 길다고 상상해 봅시다
  async function fetchData() {
    const result = await axios(getFetchUrl());
    setData(result.data);
  }

  useEffect(() => {
    fetchData();
  }, []);

  // ...
}
```

시간이 흘러 `getFetchUrl`이 상태값을 하나 사용한다고 생각해봅시다.

```javascript
function SearchResults() {
  const [query, setQuery] = useState('react');

  // 이 함수가 길다고 상상해 봅시다
  function getFetchUrl() {
    return 'https://hn.algolia.com/api/v1/search?query=' + query;
  }

  // 이 함수가 길다고 상상해 봅시다
  async function fetchData() {
    const result = await axios(getFetchUrl());
    setData(result.data);
  }

  useEffect(() => {
    fetchData();
  }, []);

  // ...
}
```

이제 느낌이 옵니다. `query`가 바뀌었지만 deps를 수정하지 않았기 때문에(우리눈엔 문제가 보이지만, 주석에 써있듯 코드가 매우 길다고 상상해 봅시다. 인간의 인지능력으로 빼먹을 수도 있습니다) 문제가 됩니다.

일단 쉬운 해결책이 있습니다. __어떤 함수를 이펙트 안에서만 호출한다면, 그 함수의 선언을 이펙트 안으로 옮기세요.__

```javascript
function SearchResults() {
  // ...
  useEffect(() => {
    // 아까의 함수들을 안으로 옮겼어요!
    function getFetchUrl() {
      return 'https://hn.algolia.com/api/v1/search?query=react';
    }
    async function fetchData() {
      const result = await axios(getFetchUrl());
      setData(result.data);
    }
    fetchData();
  }, []); // ✅ Deps는 OK
  // ...
}
```

단박에 `query` state를 deps에 추가하고 싶은 욕구가 있지만, 그보다 먼저 위 코드 상태에서 살펴봐야할 것은, 이렇게 함수를 옮김으로써 __의존성 전이 현상을__ 걱정하지 않아도 된다는 것입니다.

`getFetchUrl`은 언제든 상태로 인해 오염될 수 있습니다. 다행히 이 함수는 이펙트 안에서만 사용되고 있고, 이펙트 안에 넣어도 된다는 의미입니다. 이제 이 긴 컴포넌트(예제는 짧아보이지)를 수정하더라도 문제가 발생할 확률은 줄어듭니다. 실제로 state를 사용하도록 변경해볼까요?

```javascript
function SearchResults() {
  const [query, setQuery] = useState('react');

  useEffect(() => {
    function getFetchUrl() {
      return 'https://hn.algolia.com/api/v1/search?query=' + query;
    }

    async function fetchData() {
      const result = await axios(getFetchUrl());
      setData(result.data);
    }

    fetchData();
  }, [query]); // ✅ Deps는 OK
  // ...
}
```

다행히 `eslint-plugin-react-hooks` 플러그인의 `exhaustive-deps` 린트 룰을 사용하면 쉽게 의존성을 분석할 수 있습니다.

![](https://overreacted.io/04a90dcbacb01105d634964880ebed19/exhaustive-deps.gif)

## 하지만 저는 이함수를 이펙트 안에 넣을 수 없어요

- 한 컴포넌트에 여러 이펙트가 있고
- 같은 함수를 호출할 때
- 함수를 복붙하고 싶진 않을 때

이런 함수를 이펙트의 의존성으로 정의해도 괜찮을까요?

```javascript
function SearchResults() {
  function getFetchUrl(query) {
    return 'https://hn.algolia.com/api/v1/search?query=' + query;
  }

  useEffect(() => {
    const url = getFetchUrl('react');
    // ... 데이터를 불러와서 무언가를 한다 ...
  }, []); // 🔴 빠진 dep: getFetchUrl

  useEffect(() => {
    const url = getFetchUrl('redux');
    // ... 데이터를 불러와서 무언가를 한다 ...
  }, []); // 🔴 빠진 dep: getFetchUrl

  // ...
}
```

앞 챕터에서는 하나의 이펙트만 있었기 때문에 이펙트 안으로 넣기 수월했지만 지금과 같은 경우는 고민이 생깁니다.

- 안으로 넣자니 로직 공유가 안되고
- 넣지 않고 deps에 넣어봤자 매 렌더링마다 새로 `getFetchUrl`이 선언되기 때문에 이펙트는 의도치 않게 re-rendering마다 호출되고... 

```javascript
function SearchResults() {
  // 🔴 매번 랜더링마다 모든 이펙트를 다시 실행한다
  function getFetchUrl(query) {
    return 'https://hn.algolia.com/api/v1/search?query=' + query;
  }
  useEffect(() => {
    const url = getFetchUrl('react');
    // ... 데이터를 불러와서 무언가를 한다 ...
  }, [getFetchUrl]); // 🚧 Deps는 맞지만 너무 자주 바뀐다

  useEffect(() => {
    const url = getFetchUrl('redux');
    // ... 데이터를 불러와서 무언가를 한다 ...
  }, [getFetchUrl]); // 🚧 Deps는 맞지만 너무 자주 바뀐다

  // ...
}
```

'의존성 배열에서 빼도 동작은 잘 하니 그렇게 할까' 하는 유혹도 생기지만, 그랬다간 추후에 어떤 일이 생길지 앞서 살펴봤습니다.

대신 간단한 해결책 2개가 있습니다.

```javascript
// ✅ 데이터 흐름에 영향을 받지 않는다
function getFetchUrl(query) {
  return 'https://hn.algolia.com/api/v1/search?query=' + query;
}

function SearchResults() {
  useEffect(() => {
    const url = getFetchUrl('react');
    // ... 데이터를 불러와서 무언가를 한다 ...
  }, []); // ✅ Deps는 OK

  useEffect(() => {
    const url = getFetchUrl('redux');
    // ... 데이터를 불러와서 무언가를 한다 ...
  }, []); // ✅ Deps는 OK

  // ...
}
```

컴포넌트 외부에 선언돼있기 때문에 deps에 넣을 필요도 없습니다. props나 state를 사용할 가능성도 전혀 없습니다.

```javascript
function SearchResults() {
  // ✅ 여기 정의된 deps가 같다면 항등성을 유지한다
  const getFetchUrl = useCallback((query) => {
    return 'https://hn.algolia.com/api/v1/search?query=' + query;
  }, []);  // ✅ 콜백의 deps는 OK
  useEffect(() => {
    const url = getFetchUrl('react');
    // ... 데이터를 불러와서 무언가를 한다 ...
  }, [getFetchUrl]); // ✅ 이펙트의 deps는 OK

  useEffect(() => {
    const url = getFetchUrl('redux');
    // ... 데이터를 불러와서 무언가를 한다 ...
  }, [getFetchUrl]); // ✅ 이펙트의 deps는 OK

  // ...
}
```

`useCallback`을 사용했습니다. 이로써 공통 함수를 다시 컴포넌트 안으로 불러들였지만, 렌더링마다 다시 선언되지 않습니다. 이 상태로 시간이 흘러 search query를 입력받는 기능이 생겼다고 생각해 봅시다.

```javascript
function SearchResults() {
  const [query, setQuery] = useState('react');

  // ✅ query가 바뀔 때까지 항등성을 유지한다
  const getFetchUrl = useCallback(() => {
    return 'https://hn.algolia.com/api/v1/search?query=' + query;
  }, [query]);  // ✅ 콜백 deps는 OK
  useEffect(() => {
    const url = getFetchUrl();
    // ... 데이터를 불러와서 무언가를 한다 ...
  }, [getFetchUrl]); // ✅ 이펙트의 deps는 OK

  // ...
}
```

query가 같다면 재렌더링 되더라도 이펙트는 실행되지 않습니다. 바뀐다면 `getFetchUrl`함수 또한 바뀔것이며 그로인해 이펙트도 다시 실행될것입니다.

신기하지 않나요? 이것은 그저 데이터 흐름과 동기화에 대한 개념을 받아들인 결과입니다.

```javascript
function Parent() {
  const [query, setQuery] = useState('react');

  // ✅ query가 바뀔 때까지 항등성을 유지한다
  const fetchData = useCallback(() => {
    const url = 'https://hn.algolia.com/api/v1/search?query=' + query;
    // ... 데이터를 불러와서 리턴한다 ...
  }, [query]);  // ✅ 콜백 deps는 OK
  return <Child fetchData={fetchData} />
}

function Child({ fetchData }) {
  let [data, setData] = useState(null);

  useEffect(() => {
    fetchData().then(setData);
  }, [fetchData]); // ✅ 이펙트 deps는 OK

  // ...
}
```

그리고 부모로부터 __함수 prop을__ 내려보내는 것 또한 같은 해결책이 적용됩니다. `fetchData`는 `Parent`의 `query`가 바뀔 때만 변하고, 이에따라 `Child`는 필요할 때만 데이터 요청을 실행합니다.

## 함수도 데이터 흐름의 일부인가?

위 Parent-Child 패턴은 클래스 컴포넌트에서 제대로 동작하지 않습니다. 이를 통해 이펙트와 라이프사이클 패러다임의 차이를 확인할 수 있습니다.

```javascript
class Parent extends Component {
  state = {
    query: 'react'
  };
  fetchData = () => {
    const url = 'https://hn.algolia.com/api/v1/search?query=' + this.state.query;
    // ... 데이터를 불러와서 무언가를 한다 ...
  };
  render() {
    return <Child fetchData={this.fetchData} />;
  }
}

class Child extends Component {
  state = {
    data: null
  };
  componentDidMount() {
    this.props.fetchData();
  }
  render() {
    // ...
  }
}
```

`componentDidUpdate`를 사용할 수 있을까요?

```javascript
class Child extends Component {
  // ...
  componentDidUpdate(prevProps) {
    // 🔴 이 조건문은 절대 참이 될 수 없다
    if (this.props.fetchData !== prevProps.fetchData) {
      this.props.fetchData();
    }
  }
}
```

그렇다고 조건문을 뺄수도 없습니다. 재렌더링마다 실행될테니까요.

```javascript
render() {
  return <Child fetchData={this.fetchData.bind(this, this.state.query)} />;
}
```

이렇게 특정 쿼리를 바인딩 하면? `query`가 바뀌지 않았는데도 `this.props.fetchData !== prevProps.fetchData`는 언제나 참이 돼서 매번 요청을 날리게 됩니다.

쉬운 해결책은 `query`를 `Child`에게 주는 것입니다. `Child`는 `query`를 사용하지 않는데 말이죠...

```javascript
class Child extends Component {
  // ...
  componentDidUpdate(prevProps) {
    if (this.props.query !== prevProps.query) {
      this.props.fetchData();
    }
  }
  // ...
}
```

__클래스 컴포넌트에서, 함수 prop은 실제로 데이터 흐름에서 차지하는 부분이 없습니다.__ 함수는 가변성이 있는 `this`에 묶인채 매 호출마다 일관성을 담보하기 어렵게 돼있습니다. 그래서 그동안 __함수만 필요한 상황에서도__ 온갖 다른 데이터를 전달해 차이를 비교해야 했습니다. `this.props.fetchData`가 어떤 상태에 의존성을 갖고 있는지, 그냥 상태가 바뀌기만 한것이지 알 수 없었습니다.

`useCallback`은 이러한 함수를 데이터 흐름에 포함시킵니다.
- 함수의 입력값이 바뀌면
- 함수 자체가 바뀌고
- 그렇지 않다면 같은 함수로 남아있다

비슷한 케이스로 `useMemo`또한 복잡한 객체에 대해 같은 방식의 해결책을 제공합니다.

```javascript
function ColorPicker() {
  // color가 진짜로 바뀌지 않는 한
  // Child의 얕은 props 비교를 깨트리지 않는다
  const [color, setColor] = useState('pink');
  const style = useMemo(() => ({ color }), [color]);
  return <Child style={style} />;
}
```

그렇다고 함수를 전달할 때 마다 `useCallback`만 사용하는건 꽤나 투박합니다. 컴포넌트 트리가 깊어지고 복잡해진다면 [공식 문서에서 제안하는 콜백 내리기를 피하는 법](https://reactjs.org/docs/hooks-faq.html#how-to-avoid-passing-callbacks-down)을 참고해보세요.

## 경쟁 상태에 대해

```javascript
class Article extends Component {
  state = {
    article: null
  };
  componentDidMount() {
    this.fetchData(this.props.id);
  }
  async fetchData(id) {
    const article = await API.fetchArticle(id);
    this.setState({ article });
  }
  // ...
}
```

이 코드는 컴포넌트가 업데이트 되면(props id가 바뀌면) 요청을 보내지 않습니다. 아마 다음과 같은 코드가 금방 떠오를 겁니다.

```javascript
class Article extends Component {
  // ...
  componentDidUpdate(prevProps) {
    if (prevProps.id !== this.props.id) {
      this.fetchData(this.props.id);
    }
  }
  // ...
}
```

하지만 버그가 있습니다. 네트워크 요청은 비동기적이며 무엇이 먼저 끝날지 모릅니다. 아쉽게도 __이펙트는 이런상황을 해결할 수 없습니다.__ (async함수는 이펙트에 전달하면 경고가 나타납니다...)

비동기 접근 방식이 취소 기능(abort)를 지원한다면 클린업을 사용할 수 있을 것입니다.(이게 가장 best practice) 

그렇지 않다면 직접 flag를 만들어 사용해야 합니다.

```javascript
function Article({ id }) {
  const [article, setArticle] = useState(null);

  useEffect(() => {
    let didCancel = false;
    async function fetchData() {
      const article = await API.fetchArticle(id);
      if (!didCancel) {
        setArticle(article);
      }
    }

    fetchData();

    return () => {
      didCancel = true;
    };
  }, [id]);

  // ...
}
```

[이 링크의 글은](https://www.robinwieruch.de/react-hooks-fetch-data) 어떻게 에러를 다루고 상태를 불러올지, 또한 이런 로직을 어떻게 커스텀 훅으로 추상화할 수 있는지 소개합니다. 훅으로 데이터를 페칭하는 방법을 더 자세히 알고 싶다면 꼭 읽어보세요.

## 진입장벽을 더 높이기

props와 state를 통해 UI 일관성을 보장 받지만, 사이드 이펙트는 일관적이지 않을 확률이 높습니다. 하지만 이를 리액트의 데이터 흐름(동기화!)의 일환으로 활용하려면 useEffect를 제대로 작성해야합니다.

하지만 이글의 분량처럼 이는 쉽지않은 배움의 여정이기도 합니다. 골치 아파지는군요.

다행히 `useFetch`, `useTheme`같은 커스텀훅을 만들어 앱에서 사용하거나 [Suspense가](https://reactjs.org/docs/react-api.html#reactsuspense) 정식(안정적)으로 제공되면 `useEffect`를 직접 사용할 일은 앞으로 줄어들것입니다.

그렇게된다면 `useEffect`는 더욱 로우 레벨로 내려가 파워유저들이 진정으로 props와 state를 동기화 할 필요가 있을 때 사용하는 API가 될것입니다.

---

#### Reference
https://overreacted.io/ko/a-complete-guide-to-useeffect/