# [React elements vs React components vs Component instances](https://decorous-skipjack-927.notion.site/React-elements-vs-React-components-vs-Component-instances-c30c5558b67c4453b2ac359d0d669375)

# 배경

## 전통적인 객체 지향 UI 프로그래밍에서의 UI 모델

- 자식 컴포넌트의 인스턴스를 파괴하고 생성하는 책임을 부모 컴포넌트에게 돌림
- 부모 컴포넌트의 값이 바뀌면 자식을 새롭게 갱신하는 작업이 필요함
- 부모 컴포넌트는 자식 컴포넌트 인스턴스 관리 뿐만 아니라 자신의 DOM node도 관리해야 함
- 상태가 늘어날 수록 코드 라인은 제곱으로 늘어남
- 부모가 자식을 직접 참조하고 있으므로 이들을 분리하기 어려워짐

```jsx
class Form extends TraditionalObjectOrientedView {
  render() {
    // Read some data passed to the view
    const { isSubmitted, buttonText } = this.attrs;

    if (!isSubmitted && !this.button) {
      // Form is not yet submitted. Create the button!
      this.button = new Button({
        children: buttonText,
        color: 'blue'
      });
      this.el.appendChild(this.button.el);
    }

    if (this.button) {
      // The button is visible. Update its text!
      this.button.attrs.children = buttonText;
      this.button.render();
    }

    if (isSubmitted && this.button) {
      // Form was submitted. Destroy the button!
      this.el.removeChild(this.button.el);
      this.button.destroy();
    }

    if (isSubmitted && !this.message) {
      // Form was submitted. Show the success message!
      this.message = new Message({ text: 'Success!' });
      this.el.appendChild(this.message.el);
    }
  }
}
```

## React는 이를 어떻게 해결했을까

> *In React, this is where the elements come to rescue.*
> 

### Element는 Tree를 설명한다.

React에서 element란 컴포넌트를 plain object로 표현한 것이다. (*이것은 인스턴스가 아니다*)

React에서는 JSX는

```jsx
const App = () => {
	return (
      <button className='button button-blue'>
        <b>
          OK!
        </b>
      </button>
    )
}

const App = () => {
  return /*#__PURE__*/React.createElement("button", {
    className: "button button-blue"
  }, /*#__PURE__*/React.createElement("b", null, "OK!"));
};
```

[아래와 같은 (React.createElement 함수를 통해) React Element로 변환된다.](https://babeljs.io/repl)

```jsx

console.log(
  React.createElement("button", {
    className: "button button-blue"
  }, /*#__PURE__*/React.createElement("b", null, "OK!"))
)

// element: React.createElement()의 반환값
{
  type: 'button',  // type: (string | Class or Function)
  props: {
    className: 'button button-blue',
    children: {
      type: 'b',
      children: 'OK!'
    }
  }
}
```

type이 string인 경우 type은 해당 컴포넌트가 어떤 HTML Tag인지를 표현하고, props는 해당 HTML Tag의 속성들을 명시한다.

```jsx
{
  type: Button,
  props: {
    color: 'blue',
    children: 'OK!'
  }
}
```

이와 같이 element의 type이 Class나 Function이면 어떨까?

여기서 바로 리액트의 핵심 아이디어가 나온다.

***element(Dom element, Component element)들은 서로 중첩되고 섞일 수 있다.***

```jsx
const DangerButton = ({ children }) => ({
  type: Button,
  props: {
    color: 'red',
    children: children
  }
});
```

`DangerButton`은 `Button`이 `div`이나 `button` 같은 HTML tag인지, Class 혹은 Function으로 만들어진 컴포넌트인지 전혀 신경 쓸 필요가 없다. 그냥 Button 컴포넌트에 color props와 children을 전달해주고 있을 뿐이다.

```jsx
const DeleteAccount = () => (
  <div>
    <p>Are you sure?</p>
    <DangerButton>Yep</DangerButton>
    <Button color='blue'>Cancel</Button>
  </div>
);
```

이 함수형 컴포넌트는 아래와 같은 React element로 해석된다.

```jsx
const DeleteAccount = () => ({
  type: 'div',
  props: {
    children: [{
      type: 'p',
      props: {
        children: 'Are you sure?'
      }
    }, {
      type: DangerButton,
      props: {
        children: 'Yep'
      }
    }, {
      type: Button,
      props: {
        color: 'blue',
        children: 'Cancel'
      }
   }]
});

```

이런 mix and matching을 통해 이들의 is-a와 has-a관계가 형성되고, 컴포넌트를 서로 분리시킬 수 있다.

- `Button` is a DOM `<button>` with specific properties.
- `DangerButton` is a `Button` with specific properties.
- `DeleteAccount` contains a `Button` and a `DangerButton` inside a `<div>`.

### Component는 **Element Trees를 캡슐화한다**

리액트가 함수나 클래스 타입(컴포넌트)의 element를 만나면

```jsx
// Component element
{
  type: Button,  // Component: Element Trees 캡슐화
  props: {
    color: 'blue',
    children: 'OK!'
  }
}
```

리액트는 Button이 무슨 element를 뱉어내는지 찾아서 다음과 같은 결과를 반환한다.

```jsx
// DOM element
{
  type: 'button',
  props: {
    className: 'button button-blue',
    children: {
      type: 'b',
      props: {
        children: 'OK!'
      }
    }
  }
}
```

리액트는 페이지의 모든 element에 대한 기본 DOM tag element들을 알 때까지 이러한 과정을 계속 반복한다. (type이 string이 될 때 까지)

(리액트는 아이들이 세상의 모든 작은 것들(기본 DOM tag elements)을 알아낼 때까지 우리가 설명하는 모든 **"X는 Y"**에 대해 *"Y"가 무엇이냐고* 묻는 것과 같다.)

최종적으로, 아까 본 Form을 React에서는 다음과 같이 작성할 수 있다.

```jsx
const Form = ({ isSubmitted, buttonText }) => {
  if (isSubmitted) {
    // Form submitted! Return a message element.
    return {
      type: Message,
      props: {
        text: 'Success!'
      }
    };
  }

  // Form is still visible! Return a button element.
  return {
    type: Button,
    props: {
      children: buttonText,
      color: 'blue'
    }
  };
```

컴포넌트로부터 리턴된 Element Tree는 DOM node를 설명할 수도 있고 다른 컴포넌트를 설명하고 있을수도 있다. 이런 방식때문에 리액트가 다른 컴포넌트의 내부 구조를 몰라도 서로 독립적으로 합쳐질수 있는 것이다.

우리는 리액트에게 Element(설명서)를 전달하기만 했고 실제로 컴포넌트를 생성하고 제거하고 업데이트하는건 리액트가 알아서 해준다.

# React element

```jsx
function App() {
  return (
		// React element
    <div>
      <HelloWorld />
    </div>
  );
}

console.log(App())

{
    "type": "div",
		// 보안상의 이유로 존재
		// https://github.com/facebook/react/pull/4832
		// https://overreacted.io/why-do-react-elements-have-typeof-property/
		"$$typeof": Symbol(react.element),
		// props에 속해 있지 않고 분리되어 있는 이유
		// 특별한 속성
		// key: Diffing algorithm의 기준이 되는 값
		// ref: DOM에 직접 접근
    "key": null,
    "ref": null,
    "props": {
        "children": {
            "key": null,
            "ref": null,
            "props": {},
            "type": f HelloWorld(),   // 함수형 컴포넌트
            "_owner": null,
            "_store": {validated: true}
        }
    },
    "_owner": null,
    "_store": {validated: false}
}
```

- React.createElement()의 반환값
- element는 component를 이루는 작은 단위
- 탐색하기 쉽고 구문 분석할 필요가 없으며 실제 DOM 요소보다 훨씬 가볍다.****
- 엘리먼트는 인스턴스가 아니다. 엘리먼트는 immutable한 **plain object*이다.(엘리먼트가 생성되면, 절대로 변화되지 않는다.)
- 엘리먼트는 컴포넌트 인스턴스나 DOM node에 관한 정보를 묘사하고 있다.
- Element는 바로 사용되지는 않으며, Component에서 리턴받아서 사용되곤 한다.
- 엘리먼트는 타입(type)과 속성(props) 2가지 필드로 구성된다.
    - type으로 이것이 DOM node(string)인지, 아니면 컴포넌트 인스턴스(Class or Function)인지 알려준다.
    - 그리고 props로 해당 객체가 갖고 있는 속성들(클래스네임이나 자식 엘리먼트 등)을 나타낸다.

# React component

```jsx
// React component
function App() {
  return (
    <div>
      <HelloWorld />
    </div>
  );
}

console.log(<App/>)

{
    "type": f App(),  // 캡슐화 된 트리
		"$$typeof": Symbol(react.element),
    "key": null,
    "ref": null,
    "props": {},
    "_owner": null,
    "_store": {validated: false}
}
```

- 컴포넌트는 엘리먼트 트리를 캡슐화한다.
    - React component는 element tree를 캡슐화하고 있으므로 어떤 element 트리를 리턴할지는 보이지 않는다. 그래서 React는 이것을 구현할 때 가장 밑에 깔려있는 DOM 태그 엘리먼트가 나올 때까지 계속적으로 해당 컴포넌트가 무엇을 리턴할 것인지 묻는다.
- 리액트 컴포넌트에서는 props가 input이고 엘리먼트 트리가 output이 된다.
- UI를 재사용 가능한 개별적인 여러 조각으로 나눈 것이다.
- 이렇게 리턴된 엘리먼트 트리는 여러 엘리먼트들이 중첩, 섞여있는 구조이다. 이런 특성들로 인해 내부적인 DOM 구조에 의존하지 않고 UI를 독립적으로 조합할 수 있도록 한다.
- 우리는 일일이 인스턴스를 만들고, 업데이트하고, 삭제할 필요가 없다. 그냥 무엇을 화면에 나타내고 싶은지 컴포넌트들로 나타내면(=엘리먼트로 설명해주면) 리액트는 인스턴스를 알아서 관리한다.

# Component instances

**리액트에서** Instance는 위에서 설명한 Element와 Component에 비해 별로 중요하지 않다.
단지 리액트의 클래스형 컴포넌트를 인스턴스화한걸 Instance라고 표현한다. (함수형 컴포넌트는 Instance가 없다.)

엘리먼트의 type이 Class이면 리액트는 새로운 인스턴스를 생성하고 이것의 render메소드를 실행한다. 또한, 클래스 컴포넌트를 사용하더라도 리액트를 쓸 때 우리는 직접 instance를 관리하지 않는다. (컴포넌트 인스턴스를 직접 생성하거나 파괴하거나 수정하거나 등등..). 리액트가 알아서 해준다.

# 참고자료

- [https://www.youtube.com/watch?v=7YhdqIR2Yzo&list=PLxRVWC-K96b0ktvhd16l3xA6gncuGP7gJ&index=1](https://www.youtube.com/watch?v=7YhdqIR2Yzo&list=PLxRVWC-K96b0ktvhd16l3xA6gncuGP7gJ&index=1)
- [https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html](https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html)
- [https://simsimjae.tistory.com/449](https://simsimjae.tistory.com/449)
- [https://kimchunsick.me/2022-07-16-how-to-work-react/](https://kimchunsick.me/2022-07-16-how-to-work-react/)
- [https://velog.io/@codns1223/Element와-Component의-차이](https://velog.io/@codns1223/Element%EC%99%80-Component%EC%9D%98-%EC%B0%A8%EC%9D%B4)