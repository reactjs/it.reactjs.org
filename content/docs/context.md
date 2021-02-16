---
id: context
title: Context
permalink: docs/context.html
---

Context fornisce un modo per passare i dati attraverso l'albero dei componenti senza dover passare manualmente i props ad ogni livello.

In una tipica applicazione React, i dati sono passati dall'alto verso basso (da genitore a figlio) tramite i props, ma questo può essere
complicato per certi tipi di props (ad esempio una preferenza locale, il cambio di tema dell'interfaccia utente) che sono richiesti da molti componenti all'interno di un'applicazione. Context fornisce un modo per condividere valori come questi attraverso i diversi componenti senza dover passare esplicitamente una prop attraverso ogni livello dell'albero.


- [Quando usare il Context](#when-to-use-context)
- [Prima di usare il Context](#before-you-use-context)
- [API](#api)
  - [React.createContext](#reactcreatecontext)
  - [Context.Provider](#contextprovider)
  - [Class.contextType](#classcontexttype)
  - [Context.Consumer](#contextconsumer)
  - [Context.displayName](#contextdisplayname)
- [Esempi](#examples)
  - [Context dinamico](#dynamic-context)
  - [Aggiornare il Context da un Componente Annidato](#updating-context-from-a-nested-component)
  - [Consumare più Context](#consuming-multiple-contexts)
- [Avvertenze](#caveats)
- [API Legacy](#legacy-api)

## Quando usare il Context {#when-to-use-context}

Context é progettato per condividire i dati che possono essere considerati "globali" per un albero di componenti React, come l'attuale utente autenticato, tema, o lingua preferita. Ad esempio, nel codice seguente passiamo manualmente attraverso un prop "theme" per dare style al componente Button:

`embed:context/motivation-problem.js`

Usando il Context, possiamo evitare di passare i props attraverso elementi intermedi:

`embed:context/motivation-solution.js`

## Prima di utilizzare il Context {#before-you-use-context}

Context è principalmente utilizzato quando alcuni dati devono essere accessibili da *molti* componenti ai diversi livelli di nidificazione. Dovrebbe essere applicato con moderazione in quanto rende più difficile il riutilizzo dei componenti.

**Se vuoi solo evitare di passare alcuni prop attraverso molti livelli, [composizione del componente](/docs/composition-vs-inheritance.html) è spesso una soluzione più semplice rispetto context.**

Per esempio, considera un componente `Page` che passa i props `user` e `avatarSize` diversi livelli verso il basso affichè i componenti `Link` e `Avatar` profondamente annidati lo possano leggere:

```js
<Page user={user} avatarSize={avatarSize} />
// ... which renders ...
<PageLayout user={user} avatarSize={avatarSize} />
// ... which renders ...
<NavigationBar user={user} avatarSize={avatarSize} />
// ... which renders ...
<Link href={user.permalink}>
  <Avatar user={user} size={avatarSize} />
</Link>
```

Potrebbe sembrare ridondante passare i props `user` e `avatarSize` attraverso molti livelli se alla fine solo il componente `Avatar` ne ha davvero bisogno. È anche fastidioso che ogni volta che il componente `Avatar` ha bisogno di più props dall'alto, devi aggiungerli anche a tutti i livelli intermedi.

Un modo per risolvere questo problema **senza il context** è di [passare giù il componente `Avatar` stesso](/docs/composition-vs-inheritance.html#containment) in modo che i componenti intermedi non debbano conoscere i props `user` o `avatarSize`:

```js
function Page(props) {
  const user = props.user;
  const userLink = (
    <Link href={user.permalink}>
      <Avatar user={user} size={props.avatarSize} />
    </Link>
  );
  return <PageLayout userLink={userLink} />;
}

// Now, we have:
<Page user={user} avatarSize={avatarSize} />
// ... which renders ...
<PageLayout userLink={...} />
// ... which renders ...
<NavigationBar userLink={...} />
// ... which renders ...
{props.userLink}
```

Con questo cambiamento, solo il componente Page più in alto deve conoscere l'utilizzo di `user` e `avatarSize` da componenti `Link` e `Avatar`.

Questa *inversione del controllo* può rendere il tuo codice più pulito in molti casi riducendo il numero di props che hai bisgno di passare attraverso la tua applicazione e fornendo un maggiore controllo ai componenti radice.

Tuttavia, questa non è la scelta giusta in ogni caso: spostare più complessità nell'albero verso l'alto rende più complicati i componenti di alto livello e costringe i componenti di basso livello a essere più flessibili di quanto vorresti.

Non sei limitato a un singolo figlio per un componente. Potresti passare figli multipli, o anche avere più "slot" separati per i figli, [come documentato qui](/docs/composition-vs-inheritance.html#containment):

```js
function Page(props) {
  const user = props.user;
  const content = <Feed user={user} />;
  const topBar = (
    <NavigationBar>
      <Link href={user.permalink}>
        <Avatar user={user} size={props.avatarSize} />
      </Link>
    </NavigationBar>
  );
  return (
    <PageLayout
      topBar={topBar}
      content={content}
    />
  );
}
```
Questo modello è sufficiente per molti casi in cui è necessario disaccoppiare un figlio dai suoi genitori immediati.

Lo puoi portare ancora avanti con [render props](/docs/render-props.html) se il figlio ha bisogno di comunicare con il genitore prima di renderizzare.

Tuttavia, alcune volte i stessi dati devono essere accessbili da molti componenti nell'albero, ed ai diversi livelli di nidificazione.
Il context permette di "trasmettere" tale dati, e le modifiche ad essi , a tutte i componenti di seguito.

Esempi comuni dove context potrebbe essere più semplice delle alternative includono la gestione del locale corrente, del tema, o di una cache dei dati.

## API {#api}

### `React.createContext` {#reactcreatecontext}

```js
const MyContext = React.createContext(defaultValue);
```

Crea un oggetto Context. Quando React renderizza un componente che si iscrive a questo oggetto Context, esso leggerà il valore context corrente dal `Provider` corrispondente più vicino sopra di esso nell'albero.

L'argomento `defaultValue` è utilizzato **soltanto** quando un componente non ha un Provider corrispondente sopra di esso nell'albero.
Questo può essere utile per testare i componenti in isolamento senza avvolgerli. Nota: passando `undefined` come valore del Provider non causa ai componenti consumer di utilizzare `defaultValue`

### `Context.Provider` {#contextprovider}

```js
<MyContext.Provider value={/* some value */}>
```

Ogni oggetto Context viene con un componente React Provider che consente ai componenti consumer di iscriversi alle modifiche del context.

Il componente Provider accetta un props `value` da passare ai componenti consumer che sono discendenti di questo Provider. Un Provider può essere connesso a consumer multipli. I Provider possono essere nidificati per sovrascrivere i valori più profondi all'interno dell'albero.

Tutti i consumer che sono discendenti di uno Provider si ri-renderizzeranno ogni volta il props `value` dello Provider cambia.  La propagazione dallo Provider fino ai suoi consumer discendenti (includendo [`.contextType`](#classcontexttype) e [`useContext`](/docs/hooks-reference.html#usecontext)) non sono soggetti al metodo `shouldComponentUpdate`, quindi il consumer è aggiornato anche quando un componente antenato salta un aggiornamento.

Le modifiche sono determinate confrontando nuovi e vecchi valori usand lo stesso algoritmo di [`Object.is`](//developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is#Description).

> Nota
>
> Il modo in cui vengono determinate le modifiche può causare alcuni problemi quando si passano oggetti come "valore": vedere [Avvertenze](#caveats).

### `Class.contextType` {#classcontexttype}

```js
class MyClass extends React.Component {
  componentDidMount() {
    let value = this.context;
    /* perform a side-effect at mount using the value of MyContext */
  }
  componentDidUpdate() {
    let value = this.context;
    /* ... */
  }
  componentWillUnmount() {
    let value = this.context;
    /* ... */
  }
  render() {
    let value = this.context;
    /* render something based on the value of MyContext */
  }
}
MyClass.contextType = MyContext;
```

La proprietà `contextType` in una classe può essere assegnata ad un oggetto Context creato da [`React.createContext()`](#reactcreatecontext).
Ciò consente di consumare il valore corrente più vicino di quel tipo di Context usando `this.context`.
È possibile fare riferimento a questo in uno qualsiasi dei metodi lifecycle, inclusa la funzione render.

> Nota:
>
> Puoi iscriverti solo ad un singolo context utilizzando questa API. Se hai bisogno di leggere più di uno vedi [Consumo di Più Context](#consuming-multiple-contexts).
>
> Se si utilizza la [sintassi dei campi di classe pubblica](https://babeljs.io/docs/plugins/transform-class-properties/) sperimentale, puoi usare un campo di classe **statico** per inizializzare il tuo `contextType`

```js
class MyClass extends React.Component {
  static contextType = MyContext;
  render() {
    let value = this.context;
    /* render something based on the value */
  }
}
```

### `Context.Consumer` {#contextconsumer}

```js
<MyContext.Consumer>
  {value =>  /* render something based on the context value */}
</MyContext.Consumer>
```

Un componente React che si iscrive alle modifiche. Ciò consente di iscriverti ad un context dentro un [componente funzione](/docs/components-and-props.html#function-and-class-components).

Richiede una [funzione come un figlio](/docs/render-props.html#using-props-other-than-render). La funzione riceve il valore del context corrente e restituisce un nodo React. L'argomento `value` passato alla funzione sarà uguale al prop `value` del Provider per questo context sopra nell'albero. Se non c'è un Provider per questo context sopra, l'argomento `value` sarà uguale a `defaultValue` che era stato passato a `createContext()`.

> Nota
>
> Per ulteriori informazioni sul modello 'funzione come figlio', vedere [render props](/docs/render-props.html).

### `Context.displayName` {#contextdisplayname}

L'oggetto Context accetta una proprietà stringa `displayName`. I React DevTools utilizzano questa stringa per determinare cosa visualizzare per il context.

Per esempio, il seguente componente apparirà come MyDisplayName nei DevTools:

```js{2}
const MyContext = React.createContext(/* some value */);
MyContext.displayName = 'MyDisplayName';

<MyContext.Provider> // "MyDisplayName.Provider" in DevTools
<MyContext.Consumer> // "MyDisplayName.Consumer" in DevTools
```

## Esempi {#examples}

### Context Dinamico {#dynamic-context}

Un esempio più complesso con valori dinamici per il tema:

**theme-context.js**
`embed:context/theme-detailed-theme-context.js`

**themed-button.js**
`embed:context/theme-detailed-themed-button.js`

**app.js**
`embed:context/theme-detailed-app.js`

### Aggiornamento del Context da un Componente Nidificato {#updating-context-from-a-nested-component}

Spesso è necessario aggiornare il context da un componente che è nidificato profondamente da qualche parte nell'albero dei componenti. In questo caso è possibile passare una funzione attraverso il context per consentire ai consumer di aggiornare il context:

**theme-context.js**
`embed:context/updating-nested-context-context.js`

**theme-toggler-button.js**
`embed:context/updating-nested-context-theme-toggler-button.js`

**app.js**
`embed:context/updating-nested-context-app.js`

### Consumo dei Context Multipli {#consuming-multiple-contexts}

Per mantenere veloce la ri-renderizzazione del context, React deve rendere ogni consumer del context un nodo separato nell'albero.

`embed:context/multiple-contexts.js`

Se due o più valori del context sono spesso utilizzati insieme, si potrebbe prendere in considerazione la creazione di un proprio componente render props che fornisce entrambi.

## Avvertenze {#caveats}

Poiché il context utilizza l'identità di riferimento per determinare quando ri-renderizzare, ci sono alcuni trabocchetti che potrebbero innescare renderizzazioni non intenzionali nei consumer quando il genitore di uno provider si ri-renderizza.

Per esempio, il codice sotto ri-renderizzerà tutti i consumer ogni volta che lo Provider si ri-renderizza perchè un nuovo oggetto è sempre creato per `value`:

`embed:context/reference-caveats-problem.js`


Per aggirare questo problema, sollevare il valore nello state del genitore:

`embed:context/reference-caveats-solution.js`

## API Legacy{#legacy-api}

> Nota
>
> React precedentemente fornito con un'API sperimentale del context. La vecchia API sarà supportata in tutte le 16.x versioni, ma le applicazioni che lo utilizzano dovrebbero migrare alla nuova versione. L'API Legacy verrà rimossa in una versione futura principale di React. Leggi i [docs del context legacy](/docs/legacy-context.html).
