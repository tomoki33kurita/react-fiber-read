# React Components, Elements, and Instances

元記事：https://legacy.reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html

コンポーネント、そのインスタンス、要素の違いは、多くの React 初心者を混乱させる。画面に描かれるものを指すのに、なぜ 3 つの異なる用語があるのだろうか？

## インスタンスの管理

React を初めて使う人は、コンポーネント・クラスとインスタンスしか扱ったことがないだろう。例えば、クラスを作って Button コンポーネントを宣言する。アプリの実行時には、このコンポーネントのインスタンスが画面上に複数存在し、それぞれが独自のプロパティとローカルステートを持つかもしれない。これが伝統的なオブジェクト指向の UI プログラミングです。なぜエレメントを導入するのか？

この伝統的な UI モデルでは、子コンポーネントのインスタンスを生成したり破棄したりするのはあなた次第です。Form コンポーネントが Button コンポーネントをレンダリングしたい場合、そのインスタンスを生成し、新しい情報を手動で更新する必要があります。

```tsx
class Form extends TraditionalObjectOrientedView {
  render() {
    // Read some data passed to the view
    const { isSubmitted, buttonText } = this.attrs

    if (!isSubmitted && !this.button) {
      // Form is not yet submitted. Create the button!
      this.button = new Button({
        children: buttonText,
        color: "blue",
      })
      this.el.appendChild(this.button.el)
    }

    if (this.button) {
      // The button is visible. Update its text!
      this.button.attrs.children = buttonText
      this.button.render()
    }

    if (isSubmitted && this.button) {
      // Form was submitted. Destroy the button!
      this.el.removeChild(this.button.el)
      this.button.destroy()
    }

    if (isSubmitted && !this.message) {
      // Form was submitted. Show the success message!
      this.message = new Message({ text: "Success!" })
      this.el.appendChild(this.message.el)
    }
  }
}
```

これは擬似コードだが、Backbone のようなライブラリを使ってオブジェクト指向で一貫した振る舞いをする複合 UI コードを書くと、多かれ少なかれこうなる。

各コンポーネントのインスタンスは、その DOM ノードと子コンポーネントのインスタンスへの参照を保持し、適切なタイミングでそれらを作成、更新、破棄しなければならない。コードの行数は、コンポーネントの可能な状態の数の 2 乗として増加し、親は子コンポーネントのインスタンスに直接アクセスできるため、将来的にそれらを切り離すことは難しくなります。

では、リアクトはどう違うのか？

## ツリーを構成する要素

React では、ここでエレメントが役に立つ。要素は、コンポーネントインスタンスまたは DOM ノードとそのプロパティを記述するプレーンなオブジェクトです。エレメントには、コンポーネントのタイプ（たとえばボタン）、プロパティ（たとえば色）、およびその中の子エレメントに関する情報のみが含まれます。

要素は実際のインスタンスではない。むしろ、画面に何を表示したいかを React に伝えるためのものだ。要素のメソッドを呼び出すことはできません。これは、type: (string | ReactClass)と props の 2 つのフィールドを持つイミュータブルな記述オブジェクトです： Object1 です。

### DOM Elements

要素のタイプが文字列の場合、そのタグ名の DOM ノードを表し、props はその属性に対応する。これは React がレンダリングするものです。例えば

```tsx
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

この要素は、以下の HTML をプレーン・オブジェクトとして表現するための単なる方法である：

```tsx
<button class="button button-blue">
  <b>OK!</b>
</button>
```

要素がどのように入れ子になるかに注意してください。慣例として、要素ツリーを作成したい場合、1 つ以上の子要素を、それらを含む要素の子プロップとして指定します。

重要なのは、子要素も親要素も単なる説明であって、実際のインスタンスではないということだ。子要素も親要素も単なる説明であって、実際のインスタンスではないということだ。作って捨ててしまっても、さほど問題にはならないだろう。

React の要素はトラバースが簡単で、パースする必要がなく、もちろん実際の DOM 要素よりもずっと軽量だ！

### Components Elements

しかし、要素の型は、React コンポーネントに対応する関数やクラスであることもできる：

```tsx
{
  type: Button,
  props: {
    color: 'blue',
    children: 'OK!'
  }
}
```

これが React の核となる考え方だ。

**DOM ノードを記述する要素と同じように、コンポーネントを記述する要素も要素である。これらは入れ子にすることができ、互いに混在させることができます。**

この機能により、Button が DOM の<button>にレンダリングされるか、<div>にレンダリングされるか、あるいはまったく別のものにレンダリングされるかを気にすることなく、DangerButton コンポーネントを特定のカラー・プロパティ値を持つ Button として定義することができます：

```tsx
const DangerButton = ({ children }) => ({
  type: Button,
  props: {
    color: "red",
    children: children,
  },
})
```

1 つの要素ツリー内で、DOM 要素とコンポーネント要素を混在させることができます：

```tsx
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

あるいは、JSX がお望みなら：

```tsx
const DeleteAccount = () => (
  <div>
    <p>Are you sure?</p>
    <DangerButton>Yep</DangerButton>
    <Button color="blue">Cancel</Button>
  </div>
)
```

このミックス＆マッチングによって、コンポーネントは互いに切り離された状態を保つことができ、is-a と has-a の両方の関係をコンポジションだけで表現することができる：

- button は特定のプロパティを持つ DOM の<button>である。
- DangerButton は特定のプロパティを持つ Button です。
- DeleteAccount は<div>の中に Button と DangerButton を含んでいます。

### Components Encapsulate Element Trees

React は、関数型やクラス型を持つ要素を見たとき、対応する props が与えられれば、どの要素にレンダリングするかをそのコンポーネントに尋ねることを知っている。

このエレメントを見ると

```tsx
{
  type: Button,
  props: {
    color: 'blue',
    children: 'OK!'
  }
}
```

React は Button に何をレンダリングするかを尋ねます。Button はこの要素を返します：

```tsx
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

React は、ページ上のすべてのコンポーネントの基礎となる DOM タグ要素を知るまで、このプロセスを繰り返す。

React is like a child asking “what is Y” for every “X is Y” you explain to them until they figure out every little thing in the world.

上のフォームの例を覚えていますか？React では次のように書くことができる 1：

```tsx
const Form = ({ isSubmitted, buttonText }) => {
  if (isSubmitted) {
    // Form submitted! Return a message element.
    return {
      type: Message,
      props: {
        text: "Success!",
      },
    }
  }

  // Form is still visible! Return a button element.
  return {
    type: Button,
    props: {
      children: buttonText,
      color: "blue",
    },
  }
}
```

それだけだ！React コンポーネントでは、props が入力で、要素ツリーが出力です。

**返される要素ツリーには、DOM ノードを記述する要素と、他のコンポーネントを記述する要素の両方を含めることができます。これにより、DOM の内部構造に依存することなく、UI の独立した部分を構成することができます。**

インスタンスの作成、更新、破棄は React に任せる。コンポーネントから返す要素でインスタンスを記述し、React がインスタンスの管理を行う。

### Components Can Be Classes or Functions

上のコードでは、Form、Message、Button が React コンポーネントです。これらのコンポーネントは、上記のように関数として記述することもできるし、React.Component.Form.Message.Button から派生したクラスとして記述することもできる。コンポーネントを宣言するこれら 3 つの方法は、ほぼ等価です：

```tsx
// 1) As a function of props
const Button = ({ children, color }) => ({
  type: "button",
  props: {
    className: "button button-" + color,
    children: {
      type: "b",
      props: {
        children: children,
      },
    },
  },
})

// 2) Using the React.createClass() factory
const Button = React.createClass({
  render() {
    const { children, color } = this.props
    return {
      type: "button",
      props: {
        className: "button button-" + color,
        children: {
          type: "b",
          props: {
            children: children,
          },
        },
      },
    }
  },
})

// 3) As an ES6 class descending from React.Component
class Button extends React.Component {
  render() {
    const { children, color } = this.props
    return {
      type: "button",
      props: {
        className: "button button-" + color,
        children: {
          type: "b",
          props: {
            children: children,
          },
        },
      },
    }
  }
}
```

コンポーネントがクラスとして定義されている場合、ファンクション・コンポーネントよりも少し強力になります。ローカルの状態を保存したり、対応する DOM ノードが生成または破棄されたときにカスタムロジックを実行したりすることができます。

ファンクション・コンポーネントは強力ではありませんが、よりシンプルで、`render()`メソッド 1 つでクラス・コンポーネントのように動作します。クラスでしか使えない機能が必要な場合以外は、ファンクションコンポーネントを使うことをお勧めします。

**しかし、関数であれクラスであれ、基本的にはすべて React のコンポーネントである。これらは入力として props を受け取り、出力として要素を返します。**

### Top-Down Reconciliation

When you call:

```tsx
ReactDOM.render(
  {
    type: Form,
    props: {
      isSubmitted: false,
      buttonText: "OK!",
    },
  },
  document.getElementById("root")
)
```

React は Form コンポーネントに、これらの props が与えられたときにどのような要素ツリーを返すかを尋ねます。React は、コンポーネント・ツリーをより単純なプリミティブの観点から徐々に「洗練」していきます：

```tsx
// React: You told me this...
{
  type: Form,
  props: {
    isSubmitted: false,
    buttonText: 'OK!'
  }
}

// React: ...And Form told me this...
{
  type: Button,
  props: {
    children: 'OK!',
    color: 'blue'
  }
}

// React: ...and Button told me this! I guess I'm done.
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

これは、ReactDOM.render()または setState()を呼び出したときに始まる、React が和解と呼ぶプロセスの一部である。リコンシリエーションが終了するまでに、React は結果の DOM ツリーを知っており、react-dom や react-native のようなレンダラーは、DOM ノード（または React Native の場合はプラットフォーム固有のビュー）を更新するために必要な最小限の変更セットを適用する。

React アプリが最適化しやすい理由も、この段階的な絞り込みプロセスにある。コンポーネントツリーの一部が大きくなりすぎて React が効率的に訪問できなくなった場合、関連する props が変更されていなければ、この「精製」をスキップしてツリーの特定の部分を差分化するように指示することができる。プロップが不変であれば、プロップが変更されたかどうかを計算するのは非常に高速なので、React と不変性は非常に相性がよく、最小限の労力で大きな最適化を実現できます。

このブログのエントリーでは、コンポーネントやエレメントについて多くを語り、インスタンスについてはあまり語っていないことにお気づきかもしれない。実は、React ではインスタンスの重要性は、ほとんどのオブジェクト指向 UI フレームワークよりもずっと低いのだ。

クラスとして宣言されたコンポーネントだけがインスタンスを持ち、それを直接作成することはない： React がそれをやってくれる。親コンポーネントのインスタンスが子コンポーネントのインスタンスにアクセスする仕組みは存在するが、それらは命令的なアクション（フィールドにフォーカスを設定するなど）にしか使われないので、一般的には避けるべきである。

React はクラス・コンポーネントごとにインスタンスを生成してくれるので、メソッドやローカル・ステートを使ってオブジェクト指向でコンポーネントを書くことができるが、それ以外は React のプログラミング・モデルにおいてインスタンスはあまり重要ではなく、React 自身が管理する。

### Summary

要素とは、DOM ノードやその他のコンポーネントのうち、画面に表示させたいものを記述したプレーンなオブジェクトのことです。要素は、そのプロパティの中に他の要素を含むことができます。React 要素を作成するのは簡単だ。一度要素が作成されると、それが変更されることはない。

コンポーネントは、いくつかの異なる方法で宣言できる。render()メソッドを持つクラスにすることもできる。また、単純なケースでは、関数として定義することもできる。いずれの場合も、入力として props を受け取り、出力として要素ツリーを返します。

コンポーネントが入力としていくつかの props を受け取るとき、それは特定の親コンポーネントがその型とこれらの props を持つ要素を返したからです。これが、React では props の流れは一方通行であると言われる理由です：親から子へ。

インスタンスは、あなたが書くコンポーネントクラスでこのように参照するものです。ローカルの状態を保存したり、ライフサイクルイベントに反応したりするのに便利だ。

ファンクション・コンポーネントはインスタンスをまったく持たない。クラス・コンポーネントはインスタンスを持つが、コンポーネントのインスタンスを直接作成する必要はない。

最後に、要素を作成するには、React.createElement()、JSX、または要素ファクトリ・ヘルパーを使用します。実際のコードでは、要素をプレーンなオブジェクトとして書かないでください。
