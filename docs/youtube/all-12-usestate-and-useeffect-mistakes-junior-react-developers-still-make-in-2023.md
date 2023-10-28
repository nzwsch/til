# All 12 useState & useEffect Mistakes Junior React Developers Still Make in 2023

[All 12 useState & useEffect Mistakes Junior React Developers Still Make in 2023](https://www.youtube.com/watch?v=-yIsQPp31L0)

この動画の内容をまとめてみる。
さすがにすべて写経するのは無意味だと思ったので、気になったものだけ残すことにした。

### State updates aren't immediate

```javascript title="BAD"
import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1); // 0 + 1
    setCount(count + 1); // 0 + 1
    setCount(count + 1); // 0 + 1
    setCount(count + 1); // 0 + 1
  };

  return (
    <>
      <button onClick={handleClick}>Click me</button>

      <p>
        Count is: {count} {/* 1 */}
      </p>
    </>
  );
}
```

必ずしも`state`を上書きするのは悪くはないのだけれども、このコードの意図する出力は1回のクリックにつき`1`ではなく`4`を期待するはず。

```javascript title="GOOD"
import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount((prev) => prev + 1);
    setCount((prev) => prev + 1);
    setCount((prev) => prev + 1);
    setCount((prev) => prev + 1);
  };

  return (
    <>
      <button onClick={handleClick}>Click me</button>

      <p>
        Count is: {count} {/* 4 */}
      </p>
    </>
  );
}
```

直前の`state`を使いましょうという内容。
一概にすべてがだめというわけではないのは意外であった。

### Primitives vs non-primitives

```javascript title="primitives"
import { useState } from "react";

export default function Price() {
  console.log("Component rendering...");
  const [price, setPrice] = useState(0);
  const [price, setPrice] = useState("test");
  const [price, setPrice] = useState(true);

  const handleClick = () => {
    setPrice(0);
    setPrice("test");
    setPrice(true);
  };

  return <button onClick={handleClick}>Click me</button>;
}
```

これはあまり意識したことがなかったのだが、JavaScriptにおけるプリミティブな値(数値、文字列、論理値)は常に一定なのでボタンをクリックしてもコンポーネントが再描画されることはない。

```javascript title="non-primitives"
import { useState } from "react";

export default function Price() {
  console.log("Component rendering...");
  const [price, setPrice] = useState({
    number: 100,
    totalPrice: true,
  });

  const handleClick = () => {
    setPrice({
      number: 100,
      totalPrice: true,
    });
  };

  return <button onClick={handleClick}>Click me</button>;
}
```

これがオブジェクトになった場合はボタンを押すたびにコンポーネントが再描画される。
たとえ同じ内容のキーと値のオブジェクトだとしても、オブジェクトは同一ではないということだ。

どのみちESLintでこういうケースでは`useCallback`を使えと表示されるのであまり見落とすこともないだろう。

### Fetching in useEffect

```javascript title="BAD"
export default function Post() {
  const [id, setId] = useState(1);

  return (
    <div>
      <button onClick={() => setId(Math.floor(Math.random() * 100))}>
        Show me a different post
      </button>

      <PostBody id={id} />
    </div>
  );
}

export function PostBody({ id }) {
  const [text, setText] = useState("");

  useEffect(() => {
    fetch(`https://dummyjson.com/posts/${id}`)
      .then((res) => res.json())
      .then((data) => setText(data.body));
  }, [id]);

  return <p>{text}</p>;
}
```

正直12もあったが、ほとんど知っていたので最初からすべて写経しなくて本当によかったと思っている。
ただしこの最後の例は非常に興味深くて、一見するとこのコンポーネントの設計の何がおかしいところがあるかは一見わかりにくい。

コンポーネントの作り方としては教科書的で問題はないのだけれども、実際に動かしてみるとユーザーが複数回クリックを行うと、クリックした分だけ`text`の内容が非同期に切り替わる。
これはUIというか、使い勝手の点では非常によくない。
こういった例ではスピナーアイコンを表示するのがベターな選択肢なので、その間はボタンを非活性化(`disabled`)するとかの方法があるので、この問題もそもそも起こりにくいといえば起こりにくい。

```javascript title="GOOD"
export default function Post() {
  const [id, setId] = useState(1);

  return (
    <div>
      <button onClick={() => setId(Math.floor(Math.random() * 100))}>
        Show me a different post
      </button>

      <PostBody id={id} />
    </div>
  );
}

export function PostBody({ id }) {
  const [text, setText] = useState("");

  useEffect(() => {
    const controller = new AbortController();

    fetch(`https://dummyjson.com/posts/${id}`, {
      signal: controller.signal,
    })
      .then((res) => res.json())
      .then((data) => setText(data.body));

    return () => controller.abort();
  }, [id]);

  return <p>{text}</p>;
}
```

こういうケースでは[`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)を使うことができる。

別のIDが呼び出されてコンポーネントが新しく切り替わるタイミングでコンポーネントのクリーンアップ関数が呼び出される際に`controller.abort()`を実行すると、`fetch`関数で実行している非同期処理がキャンセルできるという仕組みだそうだ。

[Axios](https://axios-http.com/docs/cancellation)にも同様に`signal`を受け付けているようで、ほぼ同じコードで使えるようだ。
どちらかというとReactよりはFetch APIの小ネタだけれども汎用性は高いと思う。
