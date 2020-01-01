---
id: hooks-reference
title: API di Riferimento degli Hooks
permalink: docs/hooks-reference.html
prev: hooks-custom.html
next: hooks-faq.html
---

Gli *Hooks* sono stati aggiunti in React 16.8. Ti permettono di utilizzare `state` ed altre funzioni di React senza dover scrivere una classe.

Questa pagina descrive le API per gli Hooks predefiniti di React.

Se non sei familiare con gli Hooks, ti consigliamo prima di leggere [la panoramica](/docs/hooks-overview.html). Inoltre, potresti trovare delle informazioni utili nella sezione delle [domande più frequenti](/docs/hooks-faq.html).

- [Hooks base](#basic-hooks)
  - [`useState`](#usestate)
  - [`useEffect`](#useeffect)
  - [`useContext`](#usecontext)
- [Hooks addizionali](#additional-hooks)
  - [`useReducer`](#usereducer)
  - [`useCallback`](#usecallback)
  - [`useMemo`](#usememo)
  - [`useRef`](#useref)
  - [`useImperativeHandle`](#useimperativehandle)
  - [`useLayoutEffect`](#uselayouteffect)
  - [`useDebugValue`](#usedebugvalue)

## Hooks base {#basic-hooks}

### `useState` {#usestate}

```js
const [state, setState] = useState(statoIniziale);
```

Ritorna un valore stateful (appartenente allo stato), ed una funzione per aggiornarlo.

Durante il render iniziale, lo stato ritornato (`state`) è lo stesso del valore passato come primo argomento (`statoIniziale`).

La funzione `setState` è utilizzata per aggiornare lo stato. Accetta un nuovo valore stato e mette in coda un re-render del componente.

```js
setState(nuovoStato);
```

Durante i successivi re-render, il primo valore ritornato da `useState` sarà sempre lo stato più recente dopo aver effettuato gli aggiornamenti.

>Nota
>
>React garantisce che l'identità della funzione `setState` è stabile e non cambierà nei re-render. Per questo, è sicuro ometterla dalla lista di dipendenze di `useEffect` or `useCallback`.

#### Update funzionali {#functional-updates}

Se il nuovo stato è calcolato a partire dallo stato precedente, è possibile passare una funziona a `setState`. La funzione riceverà il valore precedente e ritornerà un valore aggiornato. Ecco un esempio di un componente contatore che usa entrambe le forme di `setState`:

```js
function Contatore({contatoreIniziale}) {
  const [contatore, setContatore] = useState(contatoreIniziale);
  return (
    <>
      Contatore: {contatore}
      <button onClick={() => setContatore(contatoreIniziale)}>Reset</button>
      <button onClick={() => setContatore(contatorePrecedente => contatorePrecedente - 1)}>-</button>
      <button onClick={() => setContatore(contatorePrecedente => contatorePrecedente + 1)}>+</button>
    </>
  );
}
```

I bottoni "+" e "-" utilizzano la forma funzionale, perchè il valore aggiornato dipende da quello precedente. Il bottone "Reset", invece, utilizza la forma normale, poiché imposterà sempre il contatore al valore iniziale.

> Nota
>
> A differenza del metodo `setState` dei componenti classe, `useState` non unisce automaticamente gli oggetti. Si può replicare lo stesso comportamento combinando la forma funzionale con la spread syntax (sintassi espansa) degli oggetti:
>
> ```js
> setState(statoPrecedente => {
>   // Object.assign funzionerebbe allo stesso modo
>   return {...statoPrecedente, ...valoriAggiornati};
> });
> ```
>
> Un'altra opzione è `useReducer`, più adatto alla gestione di oggetti stato i quali contengono più sotto-valori.

#### Stato iniziale lazy {#lazy-initial-state}

L'argomento `statoIniziale` è lo stato utilizzato durante il primo render. Nei render successivi, è tralasciato. Se lo stato iniziale è ottenuto da un calcolo complesso, è possibile utilizzare una funzione al suo posto, che verrà eseguita unicamente durante il primo render:

```js
const [state, setState] = useState(() => {
  const statoIniziale = qualcheCalcoloComplesso(props);
  return statoIniziale;
});
```

#### Abbandonare un aggiornamento di stato {#bailing-out-of-a-state-update}

Se uno State Hook viene aggiornato con lo stesso valore dello stato iniziale, React abbandonerà l'aggiornamento, senza renderizzare i componenti figli e senza lanciare eventi. (React utilizza l'[algoritmo di comparazione di `Object.is`](https://developer.mozilla.org/it/docs/Web/JavaScript/Reference/Global_Objects/Object/is#Descrizione).)

Tieni a mente che React potrebbe comunque aver bisogno di renderizzare quello specifico componente di nuovo prima di abbandonare l'aggiornamento. Non dovrebbe comunque crearti preoccupazioni, in quanto React non scenderà "più a fondo" nell'albero senza motivo. Se stai effettuando dei calcoli complessi nel rendering, puoi ottimizzarli con `useMemo`.

### `useEffect` {#useeffect}

```js
useEffect(didUpdate);
```

Accetta una funziona che contiene del codice imperativo, possibilmente con effetti.

Mutazioni, sottoscrizioni, timer, log ed altri side effect (effetti collaterali) non sono permessi all'interno del corpo principale di un componente funzionale (riferito come _fase di rendering_ di React). Così facendo, potrebbero causarsi strani bug ed incosistenze nell'UI.

Invece, puoi usare `useEffect`. La funzione passata a `useEffect` verrà eseguita dopo che il render viene committato nello schermo. Pensa a questi effetti come una via di fuga dal mondo puramente funzionale di React, al mondo imperativo.

Di default, gli effetti vengono eseguiti dopo ogni render, ma puoi scegliere di eseguirli [solamente quando cambiano determinati valori](#conditionally-firing-an-effect).

#### Ripulire un effetto {#cleaning-up-an-effect}

Spesso, gli effetti creano risorse che devono essere ripuliti prima che il componente lasci lo schermo, come una sottoscrizione o un timer. Per farlo, la funzione passata a `useEffect` può ritornare una funzione di pulizia. Ad esempio, per creare una sottoscrizione:

```js
useEffect(() => {
  const sottoscrizione = props.source.subscribe();
  return () => {
    // Rimuoviamo la sottoscrizione
    sottoscrizione.unsubscribe();
  };
});
```

La funzione di pulizia viene eseguita prima che il componente sia rimosso dall'UI, in modo da prevenire memory leaks. Inoltre, se un componente viene renderizzato più volte (come avviene di solito), **l'effetto precedente viene ripulito prima del successivo**. Nel nostro esempio, questo significa che viene creata una nuova sottoscrizione ad ogni aggiornamento. Per evitare di eseguire effetti ad ogni update, dai un'occhiata alla prossima sezione.

#### Tempistica degli effetti {#timing-of-effects}

A differenza di `componentDidMount` e `componentDidUpdate`, la funzione passata a `useEffect` viene eseguita **dopo** la visualizzazione della pagina, durante un evento in differita. Questo la rende adatta a molti side effect piuttosto comuni, come la creazione di una sottoscrizione e degli event handler (gestori di eventi), poiché la maggior parte di essi non dovrebbe impedire al browser di aggiornare la schermata.

Tuttavia, non tutti gli effetti possono essere eseguiti in differita. Ad esempio, una mutazione del DOM visibile all'utente deve essere eseguita in sincrono prima del prossimo aggiornamento dello schermo, così che l'utente eviti di percepire alcuna incosistenza grafica. (La differenza è concettualmente molto simile a quella tra event listener attivi e passivi.) Per questo tipo di eventi, React offre un ulteriore Hook, chiamato [`useLayoutEffect`](#uselayouteffect). Esso accetta gli stessi parametri di `useEffect` e si differenzia unicamente dal momento nel quale viene eseguito.

Nonostante `useEffect` sia eseguito dopo che il browser abbia disegnato la pagina, è garantito che la sua esecuzione avvenga prima di nuovi render. React smaltirà sempre gli effetti del render precedente prima di iniziare un nuovo update.

#### Conditionally firing an effect {#conditionally-firing-an-effect}

Il comportamento predefinito per gli effetti è vengano eseguiti dopo ogni render. In questo modo, un effetto è sempre ricreato se una delle sue dipendenze cambia.

Tuttavia questo potrebbe non essere sempre necessario, come ad esempio nel caso delle sottoscrizioni della sezione precedente. Non è necessario creare una nuova sottoscrizione ad ogni aggiornamento, ma solo se il prop `source` è cambiato.

Per ottenere questo comportamento bisogna passare un secondo argomento alla funzione `useEffect`, ovvero un vettore di valori dai quali l'effetto dipende. L'esempio aggiornato sarà quindi così:

```js
useEffect(
  () => {
    const sottoscrizione = props.source.subscribe();
    return () => {
      sottoscrizione.unsubscribe();
    };
  },
  [props.source],
);
```

Adesso la sottoscrizione verrà ricreata solamente al cambiamento di `props.source`.

>Nota
>
>Se usi questa ottimizzazione, accertati che il vettore includa **tutti i valori del componente (come lo stato ed i props) che cambiano nel tempo e che sono usati dall'effetto**. Altrimenti, il codice farà riferimento a valori datati provenienti da render precedenti. Approfondisci pure [come gestire le funzioni](/docs/hooks-faq.html#is-it-safe-to-omit-functions-from-the-list-of-dependencies) e cose fare quando [i valori del vettore cambiano spesso](/docs/hooks-faq.html#what-can-i-do-if-my-effect-dependencies-change-too-often).
>
>Se vuoi eseguire e ripulire un effetto una sola volta (quando il componente viene montato e smontato), puoi passare un vettore vuoto (`[]`) come secondo argomento. Questo dirà a React che il tuo effetto non dipenderà da *nessun* valore presente nei props o nello stato, perciò non sarà mai necessario ri-eseguirlo. Nota che questo comportamento non viene gestito internamente come caso speciale, ma segue la logica del vettore di dipendenze.
>
>Se passi un array vuoto (`[]`), i prop e lo stato all'interno dell'effetto avranno sempre i loro valori iniziali. Sebbene passare `[]` come secondo argomento è molto simile al più familiare modello concettuale di  `componentDidMount` e `componentWillUnmount`, spesso ci sono [soluzioni](/docs/hooks-faq.html#is-it-safe-to-omit-functions-from-the-list-of-dependencies) [migliori](/docs/hooks-faq.html#what-can-i-do-if-my-effect-dependencies-change-too-often) per evitare di ri-eseguire degli effetti troppo spesso. Inoltre, non dimenticare che React pospone l'esecuzione di `useEffect` a dopo che il browser ha disegnato la pagina, quindi un po' di lavoro extra è meno problematico.
>
>
>Consigliamo l'utilizzo della regola [`exhaustive-deps`](https://github.com/facebook/react/issues/14920), parte del nostro pacchetto [`eslint-plugin-react-hooks`](https://www.npmjs.com/package/eslint-plugin-react-hooks#installation). Ti segnalerà quando le dipendenze non sono specificate correttamente e proporrà una soluzione.

Il vettore di dipendenze non viene passato come argomento alla funzione effetto. Concettualmente, però, quella è la loro rappresentazione: ogni valore utilizzato all'interno della funzione effetto dovrebbe essere presente nel vettore delle dipendenze. In futuro, un compilatore sufficientemente sofisticato potrebbe essere in grado di creare questo vettore automaticamente.

### `useContext` {#usecontext}

```js
const value = useContext(MyContext);
```

Accepts a context object (the value returned from `React.createContext`) and returns the current context value for that context. The current context value is determined by the `value` prop of the nearest `<MyContext.Provider>` above the calling component in the tree.

When the nearest `<MyContext.Provider>` above the component updates, this Hook will trigger a rerender with the latest context `value` passed to that `MyContext` provider.

Don't forget that the argument to `useContext` must be the *context object itself*:

 * **Correct:** `useContext(MyContext)`
 * **Incorrect:** `useContext(MyContext.Consumer)`
 * **Incorrect:** `useContext(MyContext.Provider)`

A component calling `useContext` will always re-render when the context value changes. If re-rendering the component is expensive, you can [optimize it by using memoization](https://github.com/facebook/react/issues/15156#issuecomment-474590693).

>Tip
>
>If you're familiar with the context API before Hooks, `useContext(MyContext)` is equivalent to `static contextType = MyContext` in a class, or to `<MyContext.Consumer>`.
>
>`useContext(MyContext)` only lets you *read* the context and subscribe to its changes. You still need a `<MyContext.Provider>` above in the tree to *provide* the value for this context.

**Putting it together with Context.Provider**
```js{31-36}
const themes = {
  light: {
    foreground: "#000000",
    background: "#eeeeee"
  },
  dark: {
    foreground: "#ffffff",
    background: "#222222"
  }
};

const ThemeContext = React.createContext(themes.light);

function App() {
  return (
    <ThemeContext.Provider value={themes.dark}>
      <Toolbar />
    </ThemeContext.Provider>
  );
}

function Toolbar(props) {
  return (
    <div>
      <ThemedButton />
    </div>
  );
}

function ThemedButton() {
  const theme = useContext(ThemeContext);

  return (
    <button style={{ background: theme.background, color: theme.foreground }}>
      I am styled by theme context!
    </button>
  );
}
```
This example is modified for hooks from a previous example in the [Context Advanced Guide](/docs/context.html), where you can find more information about when and how to use Context.


## Additional Hooks {#additional-hooks}

The following Hooks are either variants of the basic ones from the previous section, or only needed for specific edge cases. Don't stress about learning them up front.

### `useReducer` {#usereducer}

```js
const [state, dispatch] = useReducer(reducer, initialArg, init);
```

An alternative to [`useState`](#usestate). Accepts a reducer of type `(state, action) => newState`, and returns the current state paired with a `dispatch` method. (If you're familiar with Redux, you already know how this works.)

`useReducer` is usually preferable to `useState` when you have complex state logic that involves multiple sub-values or when the next state depends on the previous one. `useReducer` also lets you optimize performance for components that trigger deep updates because [you can pass `dispatch` down instead of callbacks](/docs/hooks-faq.html#how-to-avoid-passing-callbacks-down).

Here's the counter example from the [`useState`](#usestate) section, rewritten to use a reducer:

```js
const initialState = {count: 0};

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return {count: state.count + 1};
    case 'decrement':
      return {count: state.count - 1};
    default:
      throw new Error();
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <>
      Count: {state.count}
      <button onClick={() => dispatch({type: 'decrement'})}>-</button>
      <button onClick={() => dispatch({type: 'increment'})}>+</button>
    </>
  );
}
```

>Note
>
>React guarantees that `dispatch` function identity is stable and won't change on re-renders. This is why it's safe to omit from the `useEffect` or `useCallback` dependency list.

#### Specifying the initial state {#specifying-the-initial-state}

There are two different ways to initialize `useReducer` state. You may choose either one depending on the use case. The simplest way is to pass the initial state as a second argument:

```js{3}
  const [state, dispatch] = useReducer(
    reducer,
    {count: initialCount}
  );
```

>Note
>
>React doesn’t use the `state = initialState` argument convention popularized by Redux. The initial value sometimes needs to depend on props and so is specified from the Hook call instead. If you feel strongly about this, you can call `useReducer(reducer, undefined, reducer)` to emulate the Redux behavior, but it's not encouraged.

#### Lazy initialization {#lazy-initialization}

You can also create the initial state lazily. To do this, you can pass an `init` function as the third argument. The initial state will be set to `init(initialArg)`.

It lets you extract the logic for calculating the initial state outside the reducer. This is also handy for resetting the state later in response to an action:

```js{1-3,11-12,19,24}
function init(initialCount) {
  return {count: initialCount};
}

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return {count: state.count + 1};
    case 'decrement':
      return {count: state.count - 1};
    case 'reset':
      return init(action.payload);
    default:
      throw new Error();
  }
}

function Counter({initialCount}) {
  const [state, dispatch] = useReducer(reducer, initialCount, init);
  return (
    <>
      Count: {state.count}
      <button
        onClick={() => dispatch({type: 'reset', payload: initialCount})}>
        Reset
      </button>
      <button onClick={() => dispatch({type: 'decrement'})}>-</button>
      <button onClick={() => dispatch({type: 'increment'})}>+</button>
    </>
  );
}
```

#### Bailing out of a dispatch {#bailing-out-of-a-dispatch}

If you return the same value from a Reducer Hook as the current state, React will bail out without rendering the children or firing effects. (React uses the [`Object.is` comparison algorithm](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is#Description).)

Note that React may still need to render that specific component again before bailing out. That shouldn't be a concern because React won't unnecessarily go "deeper" into the tree. If you're doing expensive calculations while rendering, you can optimize them with `useMemo`.

### `useCallback` {#usecallback}

```js
const memoizedCallback = useCallback(
  () => {
    doSomething(a, b);
  },
  [a, b],
);
```

Returns a [memoized](https://en.wikipedia.org/wiki/Memoization) callback.

Pass an inline callback and an array of dependencies. `useCallback` will return a memoized version of the callback that only changes if one of the dependencies has changed. This is useful when passing callbacks to optimized child components that rely on reference equality to prevent unnecessary renders (e.g. `shouldComponentUpdate`).

`useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.

> Note
>
> The array of dependencies is not passed as arguments to the callback. Conceptually, though, that's what they represent: every value referenced inside the callback should also appear in the dependencies array. In the future, a sufficiently advanced compiler could create this array automatically.
>
> We recommend using the [`exhaustive-deps`](https://github.com/facebook/react/issues/14920) rule as part of our [`eslint-plugin-react-hooks`](https://www.npmjs.com/package/eslint-plugin-react-hooks#installation) package. It warns when dependencies are specified incorrectly and suggests a fix.

### `useMemo` {#usememo}

```js
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

Returns a [memoized](https://en.wikipedia.org/wiki/Memoization) value.

Pass a "create" function and an array of dependencies. `useMemo` will only recompute the memoized value when one of the dependencies has changed. This optimization helps to avoid expensive calculations on every render.

Remember that the function passed to `useMemo` runs during rendering. Don't do anything there that you wouldn't normally do while rendering. For example, side effects belong in `useEffect`, not `useMemo`.

If no array is provided, a new value will be computed on every render.

**You may rely on `useMemo` as a performance optimization, not as a semantic guarantee.** In the future, React may choose to "forget" some previously memoized values and recalculate them on next render, e.g. to free memory for offscreen components. Write your code so that it still works without `useMemo` — and then add it to optimize performance.

> Note
>
> The array of dependencies is not passed as arguments to the function. Conceptually, though, that's what they represent: every value referenced inside the function should also appear in the dependencies array. In the future, a sufficiently advanced compiler could create this array automatically.
>
> We recommend using the [`exhaustive-deps`](https://github.com/facebook/react/issues/14920) rule as part of our [`eslint-plugin-react-hooks`](https://www.npmjs.com/package/eslint-plugin-react-hooks#installation) package. It warns when dependencies are specified incorrectly and suggests a fix.

### `useRef` {#useref}

```js
const refContainer = useRef(initialValue);
```

`useRef` returns a mutable ref object whose `.current` property is initialized to the passed argument (`initialValue`). The returned object will persist for the full lifetime of the component.

A common use case is to access a child imperatively:

```js
function TextInputWithFocusButton() {
  const inputEl = useRef(null);
  const onButtonClick = () => {
    // `current` points to the mounted text input element
    inputEl.current.focus();
  };
  return (
    <>
      <input ref={inputEl} type="text" />
      <button onClick={onButtonClick}>Focus the input</button>
    </>
  );
}
```

Essentially, `useRef` is like a "box" that can hold a mutable value in its `.current` property.

You might be familiar with refs primarily as a way to [access the DOM](/docs/refs-and-the-dom.html). If you pass a ref object to React with `<div ref={myRef} />`, React will set its `.current` property to the corresponding DOM node whenever that node changes.

However, `useRef()` is useful for more than the `ref` attribute. It's [handy for keeping any mutable value around](/docs/hooks-faq.html#is-there-something-like-instance-variables) similar to how you'd use instance fields in classes.

This works because `useRef()` creates a plain JavaScript object. The only difference between `useRef()` and creating a `{current: ...}` object yourself is that `useRef` will give you the same ref object on every render.

Keep in mind that `useRef` *doesn't* notify you when its content changes. Mutating the `.current` property doesn't cause a re-render. If you want to run some code when React attaches or detaches a ref to a DOM node, you may want to use a [callback ref](/docs/hooks-faq.html#how-can-i-measure-a-dom-node) instead.


### `useImperativeHandle` {#useimperativehandle}

```js
useImperativeHandle(ref, createHandle, [deps])
```

`useImperativeHandle` customizes the instance value that is exposed to parent components when using `ref`. As always, imperative code using refs should be avoided in most cases. `useImperativeHandle` should be used with `forwardRef`:

```js
function FancyInput(props, ref) {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    }
  }));
  return <input ref={inputRef} ... />;
}
FancyInput = forwardRef(FancyInput);
```

In this example, a parent component that renders `<FancyInput ref={inputRef} />` would be able to call `inputRef.current.focus()`.

### `useLayoutEffect` {#uselayouteffect}

The signature is identical to `useEffect`, but it fires synchronously after all DOM mutations. Use this to read layout from the DOM and synchronously re-render. Updates scheduled inside `useLayoutEffect` will be flushed synchronously, before the browser has a chance to paint.

Prefer the standard `useEffect` when possible to avoid blocking visual updates.

> Tip
>
> If you're migrating code from a class component, note `useLayoutEffect` fires in the same phase as `componentDidMount` and `componentDidUpdate`. However, **we recommend starting with `useEffect` first** and only trying `useLayoutEffect` if that causes a problem.
>
>If you use server rendering, keep in mind that *neither* `useLayoutEffect` nor `useEffect` can run until the JavaScript is downloaded. This is why React warns when a server-rendered component contains `useLayoutEffect`. To fix this, either move that logic to `useEffect` (if it isn't necessary for the first render), or delay showing that component until after the client renders (if the HTML looks broken until `useLayoutEffect` runs).
>
>To exclude a component that needs layout effects from the server-rendered HTML, render it conditionally with `showChild && <Child />` and defer showing it with `useEffect(() => { setShowChild(true); }, [])`. This way, the UI doesn't appear broken before hydration.

### `useDebugValue` {#usedebugvalue}

```js
useDebugValue(value)
```

`useDebugValue` can be used to display a label for custom hooks in React DevTools.

For example, consider the `useFriendStatus` custom Hook described in ["Building Your Own Hooks"](/docs/hooks-custom.html):

```js{6-8}
function useFriendStatus(friendID) {
  const [isOnline, setIsOnline] = useState(null);

  // ...

  // Show a label in DevTools next to this Hook
  // e.g. "FriendStatus: Online"
  useDebugValue(isOnline ? 'Online' : 'Offline');

  return isOnline;
}
```

> Tip
>
> We don't recommend adding debug values to every custom Hook. It's most valuable for custom Hooks that are part of shared libraries.

#### Defer formatting debug values {#defer-formatting-debug-values}

In some cases formatting a value for display might be an expensive operation. It's also unnecessary unless a Hook is actually inspected.

For this reason `useDebugValue` accepts a formatting function as an optional second parameter. This function is only called if the Hooks are inspected. It receives the debug value as a parameter and should return a formatted display value.

For example a custom Hook that returned a `Date` value could avoid calling the `toDateString` function unnecessarily by passing the following formatter:

```js
useDebugValue(date, date => date.toDateString());
```
