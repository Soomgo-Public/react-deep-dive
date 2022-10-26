# [React elements vs React components vs Component instances](https://decorous-skipjack-927.notion.site/React-elements-vs-React-components-vs-Component-instances-89ce9f91cfad40099b50b61607c15b64)

# 배경

## 전통적인 객체 지향 UI 프로그래밍에서의 UI 모델

### 전통적인 객체 지향 UI 모델

- 컴포넌트 클래스와 인스턴스로 구현
- 앱이 실행중 일 때 화면에 여러 컴포넌트의 인스턴스들이 있고, 각각 고유한 속성과 로컬 상태가 존재

### 인스턴스 관리

- 자식 컴포넌트의 인스턴스를 파괴하고 생성하는 것은 모두 작성자에게 달려있다.
    
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
    
    Form 컴포넌트가 Button 컴포넌트를 렌더링 하려면 이것의 인스턴스를 생성해주고 수동으로 새 정보로 최신 상태를 유지해야 한다.
    
    각 컴포넌트 인스턴스들은 그것의 DOM node와 자식 컴포넌트 인스턴스에 대한 참조를 유지해야 하고  그들이 적절한 시점에 생성되고, 업데이트 되고, 파괴되도록 해야한다.
    
    코드 라인은 컴포넌트가 지닐 수 있는 상태 수의 제곱으로 늘어나고 부모는 자식에 컴포넌트 인스터스들에 직접 접근할 수 있어 나중에 이들을 분리하기 어려워진다.
    

## React가 다른점

> *In React, this is where the elements come to rescue.*
> 

### React Element는 Tree를 설명한다.

- element는 컴포넌트 인스턴스와 DOM node 및 그것의 프로퍼티들을 설명하는 plain object이다.
- element에는 컴포넌트의 type(ex. `Button` ⇒컴포넌트 인스턴스), 그것의 프로퍼티들(ex. color), 그리고 모든 자식 요소들에 대한 정보가 있다.
- element는 인스턴스가 아니다. **화면에서 렌더링할 것을 React에게 알려주는 방법이다.**
- element는 `type: (string | Class or Function)` 과 `props: Object` 두개의 필드가 있는 불변 객체이다.****

### **DOM Elements**

`type: string`

```jsx
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

이 element는 다음 HTML을 plain object로 표현하는 방법일 뿐이다.

```html
<button class='button button-blue'>
  <b>
    OK!
  </b>
</button>
```

### **Component** elements

element의 type은 리액트 컴포넌트에 해당하는 함수 혹은 class일 수도 있다.

```jsx
{
  type: Button,
  props: {
    color: 'blue',
    children: 'OK!'
  }
}
```

This is the core idea of React.

컴포넌트를 설명하는 element 또한 DOM node를 설명하는 element와 마찬가지로 element이다. ***이들은 서로 중첩되고 mixed될 수 있다.***

```jsx
// DangerButton component as a Button with a specific color property value without worrying about whether Button renders to a DOM <button>, a <div>, or something else entirely:
const DangerButton = ({ children }) => ({
  type: Button,
  props: {
    color: 'red',
    children: children
  }
});
```

```jsx
// You can mix and match DOM and component elements in a single element tree
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

```jsx
// Or, if you prefer JSX:
const DeleteAccount = () => (
  <div>
    <p>Are you sure?</p>
    <DangerButton>Yep</DangerButton>
    <Button color='blue'>Cancel</Button>
  </div>
);
```

이 mix와 matching는 컴포넌트를 서로 분리시키는 데 도움이 되는데, 이는 이들이 이러한 구성을 통해 독점적으로 [is-a와 has-a관계](https://stackoverflow.com/questions/36162714/what-is-the-difference-between-is-a-relationship-and-has-a-relationship-in)를 표현할 수 있기 때문이다.

- `Button` is a DOM `<button>` with specific properties.
- `DangerButton` is a `Button` with specific properties.
- `DeleteAccount` contains a `Button` and a `DangerButton` inside a `<div>`.

### React Component는 **Element Trees를 캡슐화한다**

리액트는 함수나 클래스 타입의 element를 만나면 주어진 props로 이 컴포넌트가 어떤 element를 렌더링 해야 하는지 알려준다.

```jsx
// Component element
{
  type: Button,
  props: {
    color: 'blue',
    children: 'OK!'
  }
}
```

리액트는 Button에게 무엇을 렌더링 해야 할 지 물어볼 것이다. 그러면 Button은 다음 element를 반환한다.

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

리액트는 페이지의 모든 element에 대한 기본 DOM tag element들을 알 때까지 이러한 과정을 계속 반복한다. (리액트는 아이들이 세상의 모든 작은 것들(기본 DOM tag elements)을 알아낼 때까지 우리가 설명하는 모든 **"X는 Y"**에 대해 *"Y"가 무엇이냐고* 묻는 것과 같다.)

전에 본 Form을 React에서는 하기와 같이 작성할 수 있다.

 

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
};
```

리액트 컴포넌트에서는 props가 input이고 엘리먼트 트리가 output이 된다.

이렇게 리턴된 엘리먼트 트리는 여러 엘리먼트들이 중첩, 섞여있는 구조이다. 이런 특성들로 인해 내부적인 DOM 구조에 의존하지 않고 UI를 독립적으로 조합할 수 있도록 한다.

코드 작성자가 일일이 인스턴스를 만들고, 업데이트하고, 삭제할 필요가 없다. 그냥 무엇을 화면에 나타내고 싶은지 컴포넌트들로 나타내면(=엘리먼트로 설명해주면) 리액트는 인스턴스를 알아서 관리한다.

# React element

```jsx
const element = (
  <h1>
    Hello, world!
  </h1>
);
```

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
		// 보안적인 이유로 존재
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

- element는 component를 이루는 작은 단위
- 엘리먼트는 인스턴스가 아니다. 엘리먼트는 immutable한 **plain object*이다.(엘리먼트가 생성되면, 절대로 변화되지 않는다.)
- 엘리먼트는 컴포넌트 인스턴스나 DOM node에 관한 정보를 묘사하고 있다.
- Element는 바로 사용되지는 않으며, Component에서 리턴받아서 사용되곤 한다.
- 엘리먼트는 타입(type)과 속성(props) 2가지 필드로 구성된다.
- type으로 이것이 DOM node인지, 아니면 컴포넌트 인스턴스(Class or Function)인지 알려준다.
- 그리고 props로 해당 객체가 갖고 있는 속성들, 클래스네임이나 자식 엘리먼트 등을 나타낸다.
- 컴포넌트를 묘사하는 엘리먼트도, DOM 노드를 묘사하는 엘리먼트도 모두 같은 엘리먼트이다. 그들은 중첩될 수 있고 섞일 수 있다.
- type에 지금처럼 string으로 적으면 DOM node를 나타내는 것이고 ex) Button 이런식으로 적으면 component instance를 나타내는 것이다.

<aside>
💡 *PlainObject : 오래된 방식의 단순 자바 객체입니다. 조금 더 디테일한 의미는 특별한 환경(클래스나 인터페이스 등)에 종속되지 않는 일반적인 Java 객체를 의미합니다.이 단순 자바 객체는 다른 클래스나 인터페이스를 extends 및 implements 받아 메서드를 구현해야 하는 클래스가 아닌, getter/setter 와 같이 기본적인 기능만 가진 자바 객체를 뜻한다.*

</aside>

# React component

```jsx
const DeleteAccount = () => (
 <div>
   <p>Are you sure?</p>
   <DangerButton>Yep</DangerButton>
   <Button color="blue">Cancel</Button>
 </div>
)
```

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
    "type": f App(),
		"$$typeof": Symbol(react.element),
    "key": null,
    "ref": null,
    "props": {},   // 엘리먼트 트리 캡슐화
    "_owner": null,
    "_store": {validated: false}
}
```

- 컴포넌트는 엘리먼트 트리를 캡슐화한다.
- 함수 컴포넌트는 데이터를 가진 props 객체인자를 받아 element를 반환한다.
- UI를 재사용 가능한 개별적인 여러 조각으로 나눈 것이다.
- react는 component들의 조합으로 이루어지기 때문에 어떤 element 트리를 리턴할지는 보이지 않는다. 그래서 React는 이것을 구현할 때 가장 밑에 깔려있는 DOM 태그 엘리먼트가 나올 때까지 계속적으로 해당 컴포넌트가 무엇을 리턴할 것인지 묻는다. *(React is like a child asking “what is Y” for every “X is Y” you explain to them until they figure out every little thing in the world.)*
- 리액트 컴포넌트에서는 props가 input이고 엘리먼트 트리가 output이 된다.
- 이렇게 리턴된 엘리먼트 트리는 여러 엘리먼트들이 중첩, 섞여있는 구조이다. 이런 특성들로 인해 내부적인 DOM 구조에 의존하지 않고 UI를 독립적으로 조합할 수 있도록 한다.
- 우리는 일일이 인스턴스를 만들고, 업데이트하고, 삭제할 필요가 없다. 그냥 무엇을 화면에 나타내고 싶은지 컴포넌트들로 나타내면(=엘리먼트로 설명해주면) 리액트는 인스턴스를 알아서 관리한다.

# Component instances

**리액트에서** Instance는 위에서 설명한 Element와 Component에 비해 별로 중요하지 않다.
단지 리액트의 클래스형 컴포넌트를 인스턴스화한걸 Instance라고 표현한다. (함수형 컴포넌트는 Instance가 없다.)

리액트를 쓸 때 직접 Instance를 생성하지 않는다. (리액트가 알아서 해준다.)

 
Element의 type란에 Class or Function을 명시해야지 Class의 Instance를 명시하면 리액트는 그걸 해석을 못한다.