# React - Basic Theoretical Concepts

元記事：https://github.com/reactjs/react-basic

このドキュメントは、React に関する私のメンタルモデルを形式的に説明するための試みです。この設計に至った演繹的推論の観点から、このメンタルモデルを説明することを目的としています。

確かに議論の余地のある前提もいくつかあるでしょうし、この例の実際の設計にはバグや欠陥があるかもしれません。これは形式化の始まりに過ぎません。形式化の方法についてより良いアイデアがあれば、お気軽にプルリクエストを送ってください。シンプルなものから複雑なものへと進化していく過程は、ライブラリの詳細があまり見えなくても、筋が通っているはずです。

React.js の実際の実装は、実用的なソリューション、段階的なステップ、アルゴリズムの最適化、レガシーコード、デバッグツールなど、実際に使えるようにするために必要なもので溢れています。これらのものは一時的なもので、価値が高く優先度が高ければ時間の経過とともに変化する可能性があります。実際の実装は、推論するのがはるかに困難です。

私は、よりシンプルなメンタルモデルで、自分自身の基盤を築きたいと思っています。

### Transformation

React の根底にある前提は、UI は単にデータを異なる形式のデータに投影したものに過ぎないということです。同じ入力から同じ出力が得られます。シンプルな純粋関数です。

```tsx
function NameBox(name) {
  return { fontWeight: "bold", labelContent: name }
}
```

```tsx
'Sebastian Markbåge' ->
{ fontWeight: 'bold', labelContent: 'Sebastian Markbåge' };
```

### Abstraction

ただし、複雑な UI を単一の関数に収めることはできません。UI は、実装の詳細を漏らさずに再利用可能な部分に抽象化できることが重要です。例えば、ある関数から別の関数を呼び出すといったことが挙げられます。

```tsx
function FancyUserBox(user) {
  return {
    borderStyle: "1px solid blue",
    childContent: ["Name: ", NameBox(user.firstName + " " + user.lastName)],
  }
}
```

```tsx
{ firstName: 'Sebastian', lastName: 'Markbåge' } ->
{
  borderStyle: '1px solid blue',
  childContent: [
    'Name: ',
    { fontWeight: 'bold', labelContent: 'Sebastian Markbåge' }
  ]
};
```

### Composition

真に再利用可能な機能を実現するには、単にリーフを再利用して新しいコンテナを構築するだけでは不十分です。他の抽象化を構成するコンテナから、さらに抽象化を構築できることも必要です。私が考える「コンポジション」とは、2 つ以上の異なる抽象化を 1 つの新しい抽象化に組み合わせることです。

```tsx
function FancyBox(children) {
  return {
    borderStyle: "1px solid blue",
    children: children,
  }
}

function UserBox(user) {
  return FancyBox(["Name: ", NameBox(user.firstName + " " + user.lastName)])
}
```

### State

UI は、サーバーやビジネスロジックの状態を単純に複製するものではありません。実際には、特定のプロジェクションに固有の状態が多く存在します。例えば、テキストフィールドに入力を開始した場合、その状態は他のタブやモバイルデバイスに複製される場合もあれば、されない場合もあります。スクロール位置は、複数のプロジェクション間で複製されることがほとんどない典型的な例です。

私たちはデータモデルを不変にすることを好みます。状態を更新できる関数を、最上部の単一のアトムとしてスレッド化します。

```tsx
function FancyNameBox(user, likes, onClick) {
  return FancyBox([
    "Name: ",
    NameBox(user.firstName + " " + user.lastName),
    "Likes: ",
    LikeBox(likes),
    LikeButton(onClick),
  ])
}

// Implementation Details

var likes = 0
function addOneMoreLike() {
  likes++
  rerender()
}

// Init

FancyNameBox(
  { firstName: "Sebastian", lastName: "Markbåge" },
  likes,
  addOneMoreLike
)
```

注: これらの例では、状態を更新するために副作用を使用しています。私の実際のメンタルモデルでは、「更新」パス中に次のバージョンの状態を返すというものです。副作用なしで説明した方が簡潔でしたが、将来的にはこれらの例を変更する予定です。

### Memoization

関数が純粋関数だと分かっている場合、同じ関数を何度も呼び出すのは無駄です。最後の引数と最後の結果を記録する関数のメモ化バージョンを作成できます。そうすれば、同じ値を使い続ける場合でも、再実行する必要がなくなります。

```tsx
function memoize(fn) {
  var cachedArg
  var cachedResult
  return function (arg) {
    if (cachedArg === arg) {
      return cachedResult
    }
    cachedArg = arg
    cachedResult = fn(arg)
    return cachedResult
  }
}

var MemoizedNameBox = memoize(NameBox)

function NameAndAgeBox(user, currentTime) {
  return FancyBox([
    "Name: ",
    MemoizedNameBox(user.firstName + " " + user.lastName),
    "Age in milliseconds: ",
    currentTime - user.dateOfBirth,
  ])
}
```

### Lists

ほとんどの UI は何らかの形のリストであり、リスト内の各項目に対して複数の異なる値を生成します。これにより、自然な階層構造が形成されます。

リスト内の各項目の状態を管理するには、特定の項目の状態を保持するマップを作成します。

```tsx
function UserList(users, likesPerUser, updateUserLikes) {
  return users.map((user) =>
    FancyNameBox(user, likesPerUser.get(user.id), () =>
      updateUserLikes(user.id, likesPerUser.get(user.id) + 1)
    )
  )
}

var likesPerUser = new Map()
function updateUserLikes(id, likeCount) {
  likesPerUser.set(id, likeCount)
  rerender()
}

UserList(data.users, likesPerUser, updateUserLikes)
```

注: FancyNameBox に複数の異なる引数が渡されるようになりました。これにより、一度に 1 つの値しか記憶できないため、メモ化が崩れてしまいます。詳細は後述します。

### Continuations

残念ながら、UI にはリストのリストが至る所に散在しているため、それを明示的に管理しようとすると、かなりの量の定型コードが残ってしまいます。

関数の実行を遅延させることで、こうした定型コードの一部を重要なビジネスロジックから外すことができます。例えば、「カリー化」（JavaScript では bind）を使用します。そして、定型コードがなくなったコア関数の外部から状態を渡します。

bind👇
https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Function/bind

これで定型コードが減るわけではありませんが、少なくとも重要なビジネスロジックから外すことにはなります。

```tsx
function FancyUserList(users) {
  return FancyBox(UserList.bind(null, users))
}

const box = FancyUserList(data.users)
const resolvedChildren = box.children(likesPerUser, updateUserLikes)
const resolvedBox = {
  ...box,
  children: resolvedChildren,
}
```

### State Map

先ほども述べたように、繰り返しパターンを見つけたら、コンポジションを使うことで同じパターンを何度も再実装する必要がなくなります。状態の抽出と受け渡しのロジックを、頻繁に再利用する低レベル関数に移すことができます。

```tsx
function FancyBoxWithState(
  children,
  stateMap,
  updateState
) {
  return FancyBox(
    children.map(child => child.continuation(
      stateMap.get(child.key),
      updateState
    ))
  );
}

function UserList(users) {
  return users.map(user => {
    continuation: FancyNameBox.bind(null, user),
    key: user.id
  });
}

function FancyUserList(users) {
  return FancyBoxWithState.bind(null,
    UserList(users)
  );
}

const continuation = FancyUserList(data.users);
continuation(likesPerUser, updateUserLikes);
```

### Memoization Map

リスト内の複数の項目をメモ化しようとすると、メモ化は非常に難しくなります。メモリ使用量と頻度のバランスをとる複雑なキャッシュアルゴリズムを考え出す必要があります。

幸いなことに、UI は同じ位置に比較的安定している傾向があります。ツリー内の同じ位置は常に同じ値を持ちます。このツリーは、メモ化に非常に役立つ戦略であることがわかります。

状態の場合と同じトリックを使用して、メモ化キャッシュを composable 関数に渡すことができます。

```tsx
function memoize(fn) {
  return function (arg, memoizationCache) {
    if (memoizationCache.arg === arg) {
      return memoizationCache.result
    }
    const result = fn(arg)
    memoizationCache.arg = arg
    memoizationCache.result = result
    return result
  }
}

function FancyBoxWithState(children, stateMap, updateState, memoizationCache) {
  return FancyBox(
    children.map((child) =>
      child.continuation(
        stateMap.get(child.key),
        updateState,
        memoizationCache.get(child.key)
      )
    )
  )
}

const MemoizedFancyNameBox = memoize(FancyNameBox)
```

### Algebraic Effects

必要な小さな値をすべて複数の抽象化レベルに渡していくのは、かなり面倒な作業であることがわかりました。中間層を介さずに 2 つの抽象化間でデータを渡すショートカットがあると、便利な場合があります。React ではこれを「コンテキスト」と呼んでいます。

データの依存関係が抽象化ツリーにきちんと従わない場合があります。例えば、レイアウトアルゴリズムでは、子要素の位置を完全に満たす前に、そのサイズについてある程度知っておく必要があります。

さて、この例は少し「突飛」です。ECMAScript で提案されている代数的効果を使用します。関数型プログラミングに詳しい方ならご存知でしょうが、代数的効果はモナドによって課される中間的な儀式を回避しています。

```tsx
function ThemeBorderColorRequest() { }

function FancyBox(children) {
  const color = raise new ThemeBorderColorRequest();
  return {
    borderWidth: '1px',
    borderColor: color,
    children: children
  };
}

function BlueTheme(children) {
  return try {
    children();
  } catch effect ThemeBorderColorRequest -> [, continuation] {
    continuation('blue');
  }
}

function App(data) {
  return BlueTheme(
    FancyUserList.bind(null, data.users)
  );
}
```
