# How Does React State Actually Work? React.js Deep Dive #4

# React State는 실제로 어떻게 동작할까?

```jsx
setState(...args) {
	this.updater.enqueueSetstate(this, ..args);
}.

// Inside React DOM
const instance = new Component();
instance.updater = ReactDOMUpdater;
```

- 위 코드에서 updater는 React DOM에서 사용 하는 프로퍼티 입니다.
- 그래서 setState를 사용하는데 이것은 렌더러로 전달이 됩니다
- 우리가 컴포넌트를 만들 수 있는데, 이 컴포넌트 인스턴스가 업데이트가 발생하기 전에 항상 이 필드는 리액트 돔에서 미리 셋팅이 됩니다.

```jsx
useState(initialState) {
	return React.__currentDispatcher.useState(initialState);
}
```

- React hooks 측면에서 봤을 때에도 updater filed 대신에 디스패처라고 사용합니다.
- 그래도 두개의 원칙은 동일합니다.

# The Update

- 우리는 setState 또는 useState를 호출할 때, 리액트는 큐에 전달된 데이터와 함께 추가합니다.
- updates는 후에 하나씩 큐 순서에 따라 핸들링된다.
- 하지만 중요한것은 변화들은 모두 `동시에` 적용이 됩니다.

## 리액트 updates는 React Fiber와 관계적으로 가깝습니다.

- 첫번째(render phase) 단계에서 리액트는 새로운 상태를 계산하고, 이팩트 리스트들을 생성합니다.
    - 이팩트: DOM 변형 또는 라이프사이클 메서드 호출과 같은 행동
    - 이 단계에서는 비동기적으로 동작
- 첫번째 단계 이후에는 이팩트가 적용이 됩니다.
- 이 단계를 두번째 단계라고 하는데, 이건 commit phase라고도 부릅니다.
    - 이 단계에서는 동기적으로 동작

```jsx
static getDerivedStateFromProps(props, state) {
	console.log("getDerivedStateFromProps");
	return state;
}

shouldComponentUpdate(nextProps, nextState) {
	console.log("shouldComponentUpdate");
	return true;
}

componentDidUpdate(prevProps, prevState, snapshot) {
	console.log("componentDidUpdate");
}

render() {
	console.log("render");
	return (
		<button onClick={() => this.setState(() => ({ text: 'I have been updated' }))}>
			{this.state.text}
		</button>
	);
}

/*
* console.log 결과
* getDerivedStateFromProps - 1st(render) phase
*	shouldComponentUpdate - 1st(render) phase
* render
* componentDidUpdate - 2st(commit) phase
*/ 
```

## getDrivedStateFromProps ?

- 최초 마운트 시와 갱신 시 모두에서 `render` 메서드를 호출하기 직전에 호출됩니다.
- state를 갱신하기 위한 객체를 반환하거나, `null`을 반환하여 아무 것도 갱신하지 않을 수 있습니다.
- render 메서드가 호출하기 전에 항상 발생하므로, props에 state가 의존하는 상황을 위해 존재합니다. 이전과 현재의 자식 엘리먼트를 비교하는 <Transition>과 같은 컴포넌트를 구현할 때 사용한다고 합니다. (ex: 변하는 offset props을 기반으로 한 현재 스크롤링 기록하는 기능, prop을 통해 외부 데이터를 로드해야 하는 기능)
- 해당 메서드는 컴포넌트 인스턴스에 접근할 수 없음
- 필요하다면.. class 정의 외부에서 컴포넌트의 props와 state에 대한 순수 함수를 추출..
    - 이런 경우가 있을까요..?
- 하지만…. 다른 대안들을 사용하는 것을 권장!
    - props 변화에 대응한 **부수 효과를 발생**시켜야 한다면 (예를 들어, 데이터 가져오기 또는 애니메이션), `[componentDidUpdate](https://ko.reactjs.org/docs/react-component.html#componentdidupdate)` 생명주기를 대신해서 사용하세요.
    - **props가 변화했을 때에만 일부 데이터를 다시 계산** 하고 싶다면, [Memoization Helper](https://ko.reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html#what-about-memoization)를 대신해서 사용하세요.
        - 예시

            ```jsx
            class Example extends Component {
              state = {
                filterText: "",
              };
            
              // *******************************************************
              // NOTE: this example is NOT the recommended approach.
              // See the examples below for our recommendations instead.
              // *******************************************************
            
              static getDerivedStateFromProps(props, state) {
                // Re-run the filter whenever the list array or filter text change.
                // Note we need to store prevPropsList and prevFilterText to detect changes.
                if (
                  props.list !== state.prevPropsList ||
                  state.prevFilterText !== state.filterText
                ) {
                  return {
                    prevPropsList: props.list,
                    prevFilterText: state.filterText,
                    filteredList: props.list.filter(item => item.text.includes(state.filterText))
                  };
                }
                return null;
              }
            
              handleChange = event => {
                this.setState({ filterText: event.target.value });
              };
            
              render() {
                return (
                  <Fragment>
                    <input onChange={this.handleChange} value={this.state.filterText} />
                    <ul>{this.state.filteredList.map(item => <li key={item.id}>{item.text}</li>)}</ul>
                  </Fragment>
                );
              }
            }
            ```

            ```jsx
            // PureComponents only rerender if at least one state or prop value changes.
            // Change is determined by doing a shallow comparison of state and prop keys.
            class Example extends PureComponent {
              // State only needs to hold the current filter text value:
              state = {
                filterText: ""
              };
            
              handleChange = event => {
                this.setState({ filterText: event.target.value });
              };
            
              render() {
                // The render method on this PureComponent is called only if
                // props.list or state.filterText has changed.
                const filteredList = this.props.list.filter(
                  item => item.text.includes(this.state.filterText)
                )
            
                return (
                  <Fragment>
                    <input onChange={this.handleChange} value={this.state.filterText} />
                    <ul>{filteredList.map(item => <li key={item.id}>{item.text}</li>)}</ul>
                  </Fragment>
                );
              }
            }
            ```

    - **props가 변화할 때에 일부 state를 재설정** 하고 싶다면, [완전 제어 컴포넌트](https://ko.reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html#recommendation-fully-controlled-component) 또는 `[key`를 사용하는 완전 비제어 컴포넌트](https://ko.reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html#recommendation-fully-uncontrolled-component-with-a-key)로 만들어서 사용하세요.

## sholudComponentUpdate ?

- 해당 메서드를 통해서 re-rendering을 예방할 수 있다.

    ```jsx
    sholudComponentUpdate(nextProps, nextState) {
    	console.log("sholudComponentUpdate");
    	return true;
    }
    ```

- 성능 최적화로써 단독으로 존재하는 메서드이다.
- return이 true라면 componentDidUpdate가 호출이 된다.
- 우리는 위 1st phase에서 어떤 사이드 이펙트도 가질 수 없습니다.

## componentDidUpdate ?

- side effects는 오직 commit phase에서만 허락 되어진다. 이 단계가 적절하다.
- DOM changes, 네트워크 요청 등 할 수 있다.
- componentWillReceiveProps and componentWillUpdate 단계가 있었지만, 현재는 구식이며 사용하기에 안전하지 않습니다.

## 생명주기를 이용한 예시 코드

```jsx
class Clock extends React.Component {
  constructor(props) {
    super(props);
    this.state = {date: new Date()};
  }

  componentDidMount() {
    this.timerID = setInterval(
      () => this.tick(),
      1000
    );
  }

  componentWillUnmount() {
    clearInterval(this.timerID);
  }

  tick() {
    this.setState({
      date: new Date()
    });
  }

  render() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <h2>It is {this.state.date.toLocaleTimeString()}.</h2>
      </div>
    );
  }
}

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<Clock />);
```

1. `<Clock />`가 `root.render()`로 전달되었을 때 React는 `Clock` 컴포넌트의 constructor를 호출합니다. `Clock`이 현재 시각을 표시해야 하기 때문에 현재 시각이 포함된 객체로 `this.state`를 초기화합니다.
2. React는 `Clock` 컴포넌트의 `render()` 메서드를 호출합니다. 이를 통해 React는 화면에 표시되어야 할 내용을 알게 됩니다. 그 다음 React는 `Clock`의 렌더링 출력값을 일치시키기 위해 DOM을 업데이트합니다.
3. `Clock` 출력값이 DOM에 삽입되면, React는 `componentDidMount()` 생명주기 메서드를 호출합니다. 그 안에서 `Clock` 컴포넌트는 매초 컴포넌트의 `tick()` 메서드를 호출하기 위한 타이머를 설정하도록 브라우저에 요청합니다.
4. 매초 브라우저가 `tick()` 메서드를 호출합니다. 그 안에서 `Clock` 컴포넌트는 `setState()`에 현재 시각을 포함하는 객체를 호출하면서 UI 업데이트를 진행합니다. `setState()` 호출 덕분에 React는 state가 변경된 것을 인지하고 화면에 표시될 내용을 알아내기 위해 `render()` 메서드를 다시 호출합니다. 이 때 `render()` 메서드 안의 `this.state.date`가 달라지고 렌더링 출력값은 업데이트된 시각을 포함합니다. React는 이에 따라 DOM을 업데이트합니다.
5. `Clock` 컴포넌트가 DOM으로부터 한 번이라도 삭제된 적이 있다면 React는 타이머를 멈추기 위해 `componentWillUnmount()` 생명주기 메서드를 호출합니다. ( useEffect return과 동일)

## 높은 단계로부터 전체 업데이트 검토하기(정리하기 …직역의 한계)

1. setState 호출
2. 데이터 큐에 담기
3. React는 updates를 담당하기 시작
4. 새로운 상태를 계산, getDerivedStateFromProps 호출
5. shouldComponentUpdate가 마지막 state와 함께 호출된다.
    1. true인 경우에만
6. rerender 불린다.
7. DOM 변화 또는 생성
8. componentDidUpdate 호출

# 부록 - 함께 보면 좋을 페이지

- getDrivedStateFromProps

[React.Component - React](https://ko.reactjs.org/docs/react-component.html)

[You Probably Don't Need Derived State - React Blog](https://ko.reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html#when-to-use-derived-state)

[리액트 라이프사이클의 이해](https://kyun2da.dev/react/%EB%A6%AC%EC%95%A1%ED%8A%B8-%EB%9D%BC%EC%9D%B4%ED%94%84%EC%82%AC%EC%9D%B4%ED%81%B4%EC%9D%98-%EC%9D%B4%ED%95%B4/)
