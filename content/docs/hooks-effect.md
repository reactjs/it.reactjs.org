---
id: hooks-state
title: Usare l'Hook Effect
permalink: docs/hooks-effect.html
next: hooks-rules.html
prev: hooks-state.html
---

Gli *Hooks* sono una novità di React 16.8. Ti permettono di utilizzare lo state ed altre funzioni di React senza dover scrivere una classe.

L' *Effect Hook* ti permette di effettuare side effects in componenti funzione:

```js{1,6-10}
import React, { useState, useEffect } from 'react';

function Example() {
  const [contatore, setContatore] = useState(0);

  // Simile a componentDidMount e componentDidUpdate:
  useEffect(() => {
    // Aggiorna il titolo della pagina usando le browser API
    document.title = `Hai cliccato ${contatore} volte`;
  });

  return (
    <div>
      <p>Hai cliccato{contatore} volte</p>
      <button onClick={() => setContatore(contatore + 1)}>
        Cliccami
      </button>
    </div>
  );
}
```

Questo snippet è basato sull' [esempio del contatore della pagina precedente](/docs/hooks-state.html), ma abbiamo aggiunto una nuova funzione: settiamo il titolo del documento con un messaggio custom contenente il numero di click.

Il fetching dei dati, il settaggio di una subscription, ed il cambiamento manuale del DOM nei componenti React sono tutti esempi di side effects. Abituato o meno a chiamare queste operazioni "side effects" (o solo "effects"), è probabile che tu li abbia già utilizzati.

>Consiglio
>
>Se hai familiarità con i metodi del ciclo di vita di React, puoi pensare all'Hook `useEffect` come ad una combinazione di `componentDidMount`, `componentDidUpdate` e `componentWillMount`.

Ci sono due tipi comuni di side effects nei componenti React: quelli che non richiedono un cleanup, e quelli che lo richiedono. Vediamo nel dettaglio come distinguerli.

## Effects senza Cleanup {#effects-without-cleanup}

Qualche volta, vogliamo **eseguire del codice aggiuntivo dopo che React ha aggiornato il DOM.** Richieste di rete, mutazioni manuali del DOM, e logging sono esempi comuni di effetti che non richiedono un cleanup. Li chiamiamo così perché possono essere eseguiti e subito dimenticati. Confrontiamo come classi e Hook ci consentono di eseguire questi side effects.

### Esempio con le classi {#example-using-classes}

Nei componenti classe React, il metodo `render` non dovrebbe provocare side effects. Sarebbe troppo presto -- tipicamente vogliamo eseguire i nostri effects *dopo* che React ha aggiornato il DOM.

Questo è il motivo per cui nelle classi React mettiamo i side effects in `componentDidMount` e `componentDidUpdate`. Tornando al nostro esempio, questo è un componente classe React contatore che aggiorna il titolo del documento appena React effettua cambiamenti al DOM:

```js{9-15}
class Example extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      contatore: 0
    };
  }

  componentDidMount() {
    document.title = `Hai cliccato ${this.state.contatore} volte`;
  }

  componentDidUpdate() {
    document.title = `Hai cliccato ${this.state.contatore} volte`;
  }

  render() {
    return (
      <div>
        <p>Hai cliccato {this.state.contatore} volte</p>
        <button onClick={() => this.setState({ contatore: this.state.contatore + 1 })}>
          Cliccami
        </button>
      </div>
    );
  }
}
```

Nota come  **abbiamo dovuto duplicare il codice tra questi due metodi del ciclo di vita nella classe.**

Questo perché in molti casi vogliamo eseguire lo stesso side effect indipendentemente dal fatto che il componente sia stato montato, o solo aggiornato. Concettualmente, vogliamo che questo avvenga dopo ogni render -- ma i componenti classe React non hanno un metodo per fare questo. Possiamo estrarre un metodo distinto ma dobbiamo comunque richiamarlo in due posti.

Adesso vediamo come possiamo fare la stessa cosa con l'Hook `useEffect`.

### Esempio con gli Hooks {#example-using-hooks}

Abbiamo già visto questo esempio all'inizio di questa pagina, ma guardiamolo più attentamente:

```js{1,6-8}
import React, { useState, useEffect } from 'react';

function Example() {
  const [contatore, setContatore] = useState(0);

  useEffect(() => {
    document.title = `Hai cliccato ${contatore} volte`;
  });

  return (
    <div>
      <p>Hai cliccato {contatore} volte</p>
      <button onClick={() => setContatore(contatore + 1)}>
        Cliccami
      </button>
    </div>
  );
}
```

**Cosa fa `useEffect`?** Usando questo Hook, stai dicendo a React che il tuo componente necessita di eseguire qualcosa dopo il render. React si ricorderà della funzione che hai passato (che sarà il nostro "effect") , e la chiamerà dopo aver aggiornato il DOM. In questo effect, settiamo il titolo del documento, ma potremmo anche eseguire un fetch dei dati o chiamare una qualsiasi altra API.

**Perchè `useEffect` viene chiamato dentro un componente?** Mettendo `useEffect` nel nostro componente abbiamo accesso alla variabile di state `contatore` (o una qualsiasi props) direttamente dall'effect. Non abbiamo bisogno di una API speciale per leggerlo -- è già nello scope della funzione. Gli Hooks fanno uso delle closures di Javascript evitando l'introduzione di API React specifiche poichè Javascript fornisce già una soluzione.

**`useEffect` viene eseguito dopo ogni render?** Si! Di default, viene eseguito dopo il primo render *e* dopo ogni update. (Parleremo più tardi di [come personalizzarlo](#tip-optimizing-performance-by-skipping-effects).) Invece di pensare in termini di "mounting" e "updating", potresti trovare più facile pensare che gli effects avvengono "dopo il render". React garantisce che il DOM è aggiornato nel momento in cui vengono eseguiti gli effects.

### Spiegazione dettagliata {#detailed-explanation}

Adesso che sappiamo di più riguardo gli effects, queste righe dovrebbero prendere significato:

```js
function Example() {
  const [contatore, setContatore] = useState(0);

  useEffect(() => {
    document.title = `Hai cliccato ${contatore} volte`;
  });
}
```

Dichiariamo la variabile di state `contatore`, e diciamo a React che abbiamo bisogno di utilizzare un effect. Passiamo la funzione all'Hook `useEffect`. Questa funzione che passiamo *è* il nostro effect. Dentro il nostro effect, settiamo il titolo del documento usanto l'API del browser `document.title`. Possiamo leggere l'ultimo `contatore` dentro l'effect perché è nello scope della nostra funzione. Quando React renderizza nuovamente il nostro componente, si ricorderà dell'effect che abbiamo utilizzato, e lo eseguirà nuovamente dopo aver aggiornato il DOM. Questo avviene per ogni render, incluso il primo.

Gli sviluppatore JavaScript più esperti potrebbero notare che la funzione passata a `useEffect` sarà diversa ad ogni render. Questo è voluto. Infatti, questo è ciò che ci permette di leggere il valore `contatore` dentro l'effect senza troppe preoccupazioni. Ogni volta che facciamo un re-render, pianifichiamo un effect _diverso_, rimpiazzando il precedente. In questo modo, questo permette agli effects di comportarsi come parte del risultato del render -- ogni effect "appartiene" ad un particolare render. Vedremo più chiaramente perchè questo è utile [più tardi in questa pagina](#explanation-why-effects-run-on-each-update).

>Consiglio
>
>A differenza di `componentDidMount` o `componentDidUpdate`, gli effects schedulati con `useEffect` non bloccano il browser dall'aggiornare lo schermo. Questo rende l'app più responsive. La maggior parte degli effects non necessitano di funzionare in modo sincrono. Nei casi meno comuni dove ne hanno bisogno (come la misurazione del layout), c'è un altro Hook [`useLayoutEffect`](/docs/hooks-reference.html#uselayouteffect) con un API identica a `useEffect`.

## Effects con Cleanup {#effects-with-cleanup}

Prima, abbiamo visto come eseguire side effects che non richiedono alcun cleanup. In ogni caso, qualche effect lo fa. Per esempio, **potremmo voler fare il setup di una subscription** a qualche sorgente dati esterna. In questo caso, è importante fare pulizia così da non introdurre alcun memory leak! Confrontiamo come possiamo farlo con classi ed Hooks.

### Esempio con le classi {#example-using-classes-1}

In una classe React, tipicamente setti una subscription in `componentDidMount`, e la pulisci in `componentWillUnmount`. Per esempio, supponiamo di avere un modulo `ChatAPI` che ci permette di sottoscriverci allo status online di un amico. Questo è come potremmo sottoscriverci e visualizzare lo status usando una classe:

```js{8-26}
class StatusAmico extends React.Component {
  constructor(props) {
    super(props);
    this.state = { isOnline: null };
    this.gestisciCambioStatus = this.gestisciCambioStatus.bind(this);
  }

  componentDidMount() {
    ChatAPI.sottoscriviStatusAmico(
      this.props.amico.id,
      this.gestisciCambioStatus
    );
  }

  componentWillUnmount() {
    ChatAPI.annullaSottoscrizioneStatusAmico(
      this.props.friend.id,
      this.gestisciCambioStatus
    );
  }

  gestisciCambioStatus(status) {
    this.setState({
      isOnline: status.isOnline
    });
  }

  render() {
    if (this.state.isOnline === null) {
      return 'Caricamento...';
    }
    return this.state.isOnline ? 'Online' : 'Offline';
  }
}
```

Nota come `componentDidMount` e `componentWillUnmount` necessitano di essere speculari. I metodi del ciclo di vita ci forzano a splittare questa logica anche se concettualmente il codice di entrambe le sezioni è relativo allo stesso effect.

>Nota
>
>I lettori più attenti potrebbero notare che questo esempio ha bisogno anche del metodo `componentDidUpdate` per essere completamente corretto. Per adesso lo ignoriamo ma ci torneremo più tardi nella [relativa sezione](#explanation-why-effects-run-on-each-update) di questa pagina.

### Esempio con gli Hooks {#example-using-hooks-1}

Vediamo come possiamo scrivere questo componente con gli Hooks.

Potresti pensare che abbiamo bisogno di un effect separato per eseguire il cleanup. Ma il codice per aggiungere e rimuovere una subscription è così fortemente correlato che `useEffect` è progettato per tenerlo insieme. Se il tuo effect ritorna una funzione, React la eseguira quando sarà necessario eseguire la pulizia:

```js{6-16}
import React, { useState, useEffect } from 'react';

function StatusAmico(props) {
  const [isOnline, setIsOnline] = useState(null);

  useEffect(() => {
    function gestisciCambioStatus(status) {
      setIsOnline(status.isOnline);
    }

    ChatAPI.sottoscriviStatusAmico(props.amico.id, gestisciCambioStatus);
    // Specifica come avviene la pulizia dopo questo effect:
    return function pulizia() {
      ChatAPI.annullaSottoscrizioneStatusAmico(props.amico.id, gestisciCambioStatus);
    };
  });

  if (isOnline === null) {
    return 'Caricamento...';
  }
  return isOnline ? 'Online' : 'Offline';
}
```
**Perchè abbiamo ritornato una funzione dal nostro effect?** Questo è il meccanismo opzionale di pulizia per gli effects. Ogni effect può ritornare una funzione che esegue la pulizia. Questo ci permette di mantenere unita la logica di aggiunta e rimozione delle subscriptions. Sono parte dello stesso effect!

**Precisamente, quando React esegue la pulizia di un effect?** React esegue il cleanup quando il componente viene smontato. Comunque, come abbiamo appreso precedentemente, gli effects vengono eseguiti per ogni render e non solo una volta. Questo è *anche* il motivo per cui React pulisce gli effects del render precedente prima di eseguire nuovamente gli effects. Parleremo più tardi del [perché questo aiuta a prevenire i bugs](#explanation-why-effects-run-on-each-update) e [come prevenire questo comportamento in caso di problemi di performance](#tip-optimizing-performance-by-skipping-effects).

>Nota
>
>Non dobbiamo ritornare una funzione denominata dall'effect. La chiamiamo `cleanup` per definire bene il suo scopo, ma potresti ritornare una funzione freccia o chiamarla in un altro modo.

## Riepilogo {#recap}

Abbiamo appreso che `useEffect` ci permette di eseguire diversi tipi di side effects dopo il render di un componente. Alcuni effects potrebbero richiedere un cleanup quindi ritornano una funzione:

```js
  useEffect(() => {
    function gestisciCambioStatus(status) {
      setIsOnline(status.isOnline);
    }

    ChatAPI.sottoscriviStatusAmico(props.amico.id, gestisciCambioStatus);
    return () => {
      ChatAPI.annullaSottoscrizioneStatusAmico(props.amico.id, gestisciCambioStatus);
    };
  });
```

Altri effects potrebbero non avere una fase di cleanup, e non ritornare nulla.

```js
  useEffect(() => {
    document.title = `Hai cliccato ${contatore} volte`;
  });
```

Questo Effect Hook unisce entrambi i casi d'uso con una singola API.

-------------

**Se credi di avere compreso bene come l'Effect Hook funziona, o se ti senti sopraffatto, puoi saltare subito alla [prossima pagina riguardo le Regole degli Hooks](/docs/hooks-rules.html)**

-------------

## Consigli per usare gli Effects {#tips-for-using-effects}

Continueremo in questa pagina con l'approfondimento di alcuni aspetti di `useEffect` di cui gli utenti React più esperti saranno curiosi. Non sertirti obbligato ad affrontarlo adesso. Puoi sempre tornare in questa pagina ed apprendere dettagli riguardo l'Effect Hook.

### Consiglio: Usare Effects multipli per separare le preoccupazioni {#tip-use-multiple-effects-to-separate-concerns}

Uno dei problemi di cui abbiamo discusso nella sezione [Motivazione](/docs/hooks-intro.html#complex-components-become-hard-to-understand) degli Hooks è che i metodi del ciclo di vita delle classi spesso contengono logica non correlata, ma la logica correlata viene suddivisa su più metodi. Quello che segue è un componente che unisce la logica del contatore con quella dell'indicatore dello status di un amico direttamente dall'esempio precedente:

```js
class StatusAmicoConContatore extends React.Component {
  constructor(props) {
    super(props);
    this.state = { contatore: 0, isOnline: null };
    this.gestisciStatusAmico = this.gestisciStatusAmico.bind(this);
  }

  componentDidMount() {
    document.title = `Hai cliccato ${this.state.contatore} volte`;
    ChatAPI.sottoscriviStatusAmico(
      this.props.amico.id,
      this.gestisciStatusAmico
    );
  }

  componentDidUpdate() {
    document.title = `Hai cliccato ${this.state.contatore} volte`;
  }

  componentWillUnmount() {
    ChatAPI.annullaSottoscrizioneStatusAmico(
      this.props.amico.id,
      this.gestisciStatusAmico
    );
  }

  gestisciStatusAmico(status) {
    this.setState({
      isOnline: status.isOnline
    });
  }
  // ...
```

Nota come la logica che imposta il `document.title` è suddivisa tra `componentDidMount` e `componentDidUpdate`. Anche la logica della sottoscrizione è separata tra `componentDidMount` e `componentDidUpdate`. E `componentDidMount` contiene codice per entrambi i task.

Quindi, come possono gli Hooks risolvere questo problema? Dal momento che [è possibile usare l'Hook *State* più di una volta](/docs/hooks-state.html#tip-using-multiple-state-variables), si può utilizzare più di un effetto. Questo ci permette di separare logica non relazionata in effects differenti:

```js{3,8}
function StatusAmicoConContatore(props) {
  const [contatore, setContatore] = useState(0);
  useEffect(() => {
    document.title = `Hai cliccato ${contatore} volte`;
  });

  const [isOnline, setIsOnline] = useState(null);
  useEffect(() => {
    function gestisciStatusAmico(status) {
      setIsOnline(status.isOnline);
    }

    ChatAPI.sottoscriviStatusAmico(props.amico.id, gestisciStatusAmico);
    return () => {
      ChatAPI.annullaSottoscrizioneStatusAmico(props.amico.id, gestisciStatusAmico);
    };
  });
  // ...
}
```

**Gli Hooks ci permettono di dividere il codice in base a cosa si sta facendo** e non in base ad un metodo del ciclo di vita. React applicherà *ogni* effetto usato dal componente, nell'ordine in cui sono specificati.

### Spiegazione: Perché gli effects vengono eseguiti ad ogni update {#explanation-why-effects-run-on-each-update}

Se sei solito utilizzare le classi, potresti chiederti perchè la fase di cleanup di un effect avviene dopo ogni re-render, e non sono una volta durante l'unmounting. Guardiamo un esempio pratico per capire il motivo per cui questa architettura ci aiuta nella creazione di componenti con meno bug.

[Prima in questa pagina](#example-using-classes-1), abbiamo introdotto un componente di esempio `StatusAmico` che visualizza se un amico è online o no. La nostra classe legge `amico.id` da `this.props`, si sottoscrive allo status dell'amico dopo che il componente viene montato, e dismette la sottoscrizione durante l'unmounting:

```js
  componentDidMount() {
    ChatAPI.sottoscriviStatusAmico(
      this.props.amico.id,
      this.gestisciStatusAmico
    );
  }

  componentWillUnmount() {
    ChatAPI.annullaSottoscrizioneStatusAmico(
      this.props.amico.id,
      this.gestisciStatusAmico
    );
  }
```
** Ma cosa succede se la prop `amico` cambia** mentre il componente è visualizzato sullo schermo? Il nostro componente continuerebbe a mostrare lo status online di un amico diverso. Questo è un bug. Inoltre causeremmo anche un memory leak o un crash quando avviene l'unmounting dal momento che la chiamata per la cancellazione della sottoscrizione userebbe un ID amico errato.

In un componente classe, avremmo bisogno di aggiungere `componentDidUpdate` per gestire questo caso:

```js{8-19}
  componentDidMount() {
    ChatAPI.sottoscriviStatusAmico(
      this.props.amico.id,
      this.gestisciStatusAmico
    );
  }

  componentDidUpdate(prevProps) {
    // Cancella la sottoscrizione al precedente amico.id
    ChatAPI.annullaSottoscrizioneStatusAmico(
      prevProps.amico.id,
      this.gestisciStatusAmico
    );
    // Sottoscrivi al successivo amico.id
    ChatAPI.sottoscriviStatusAmico(
      this.props.amico.id,
      this.gestisciStatusAmico
    );
  }

  componentWillUnmount() {
    ChatAPI.annullaSottoscrizioneStatusAmico(
      this.props.amico.id,
      this.gestisciStatusAmico
    );
  }
```

Dimenticarsi di gestire correttamente `componentDidUpdate` è una fonte comune di bug nelle applicazioni React.

Adesso considera la versione di questo componente che usa gli Hooks:

```js
function StatusAmico(props) {
  // ...
  useEffect(() => {
    // ...
    ChatAPI.sottoscriviStatusAmico(props.amico.id, gestisciStatusAmico);
    return () => {
      ChatAPI.annullaSottoscrizioneStatusAmico(props.amico.id, gestisciStatusAmico);
    };
  });
```

In questo caso il codice non è soggetto a questo bug. (Ma non abbiamo nemmeno fatto alcuna modifica.)

Non c'è un codice speciale per gestire gli aggiornamenti perchè `useEffect` li gestisce *di default*. Esegue il cleanup degli effetti precedenti prima di applicare i successivi. Per illustrare questo, ecco una sequenza di sottoscrizione e cancellazione della sottoscrizione che questo componente può produrre nel tempo:

```js
// Monta con le props { amico: { id: 100 } }
ChatAPI.sottoscriviStatusAmico(100, gestisciStatusAmico);     // Esegue il primo effect

// Aggiorna con le props { amico: { id: 200 } }
ChatAPI.annullaSottoscrizioneStatusAmico(100, gestisciStatusAmico); // Esegue il cleanup dell'effect precedente
ChatAPI.sottoscriviStatusAmico(200, gestisciStatusAmico);     // Esegue l'effect successivo

// Aggiorna con le props { amico: { id: 300 } }
ChatAPI.annullaSottoscrizioneStatusAmico(200, gestisciStatusAmico); // Esegue il cleanup dell'effect precedente
ChatAPI.sottoscriviStatusAmico(300, gestisciStatusAmico);     // Esegue l'effect successivo

// Unmount
ChatAPI.annullaSottoscrizioneStatusAmico(300, gestisciStatusAmico); // Esegue il cleanup dell'ultimo effect
```

Questo comportamento assicura consistenza di default e previene bug che sono comuni nei componenti classe dovuti alla mancanza di logica di aggiornamento.

### Tip: Ottimizzazione delle Performance evitando gli Effects {#tip-optimizing-performance-by-skipping-effects}

In alcuni casi, pulire o applicare l'effetto dopo ogni render potrebbe creare problemi di performance. Nei componenti classe, possiamo risolvere scrivendo un confronto extra con `prevProps` o `prevState` nel metodo `componentDidUpdate`:

```js
componentDidUpdate(prevProps, prevState) {
  if (prevState.contatore !== this.state.contatore) {
    document.title = `Hai cliccato ${this.state.contatore} volte`;
  }
}
```

Questo requisito è comune al punto da essere stato inserito nell'API dell'Hook `useEffect`. Puoi dire a React di *saltare* l'applicazione dell'effect se certi valori non sono cambiati durante i re-render. Per fare ciò, si passa un array come secondo parametro opzionale a `useEffect`.

```js{3}
useEffect(() => {
  document.title = `Hai cliccato ${contatore} volte`;
}, [contatore]); // Riesegue l'effect solamente se contatore cambia
```

Nell'esempio quì sopra, passiamo `[contatore]` come secondo parametro. Cosa significa questo? Se `contatore` è `5`, e poi il nostro componente re-renderizza con `contatore` ancora a `5`, React confronterà `[5]` del render precedente con `[5]` del render successivo. Dal momento che tutti gli elementi dell'array sono gli stessi (`5 === 5`), React salterà l'effect. Questa è la nostra ottimizzazione.

Quando renderizziamo con `contatore` aggiornato a `6`, React confronterà gli elementi dell'array `[5]` del render precedente con gli elementi dell'array `[6]` del render successivo. Questa volta, React applicherà nuovamente l'effect perchè `5 !== 6`. Se ci sono più elementi nell'array, React eseguirà di nuovo l'effect anche se solo uno di essi sarà diverso.

Questo funziona anche per gli effects che hanno una fase di cleanup:

```js{10}
useEffect(() => {
  function gestisciStatusAmico(status) {
    setIsOnline(status.isOnline);
  }

  ChatAPI.sottoscriviStatusAmico(props.amico.id, gestisciStatusAmico);
  return () => {
    ChatAPI.annullaSottoscrizioneStatusAmico(props.amico.id, gestisciStatusAmico);
  };
}, [props.amico.id]); // Sottoscrive di nuovo solo se props.amico.id cambia
```

In futuro, il secondo parametro potrebbe essere aggiunto automaticamente durante la build.

>Nota
>
>Se usi questa ottimizzazione, assicurati che gli array includano **tutti i valori dallo scope del componente (come props e state) che cambiano nel tempo e che sono utilizzati dall'effect**. Altrimenti, il tuo codice referenzierà vecchi valori dai render precedenti. Puoi saperne di più su [come operare con le funzioni](/docs/hooks-faq.html#is-it-safe-to-omit-functions-from-the-list-of-dependencies) e [cosa fare quando l'array cambia troppo spesso](/docs/hooks-faq.html#what-can-i-do-if-my-effect-dependencies-change-too-often).
>
>Se vuoi eseguire e pulire un effect solamente una volta (al mount e unmount), puoi passare un array vuoto (`[]`) come secondo parametro. Questo indica a React che il tuo effect non dipende da *alcun* valore da props e state, quindi non ha mai bisogno di essere rieseguito. Questo non è gestito come caso speciale -- è semplicemente come gli array dipendenti funzionano da sempre.
>
>Se passi un array vuoto (`[]`), le props e lo state nell'effect avranno sempre il loro valore iniziale. Mentre passare `[]` come secondo argomento è più vicino al familiare modello mentale dei `componentDidMount` e `componentWillUnmount`, di solito ci sono [migliori](/docs/hooks-faq.html#is-it-safe-to-omit-functions-from-the-list-of-dependencies) [soluzioni](/docs/hooks-faq.html#what-can-i-do-if-my-effect-dependencies-change-too-often) per evitare di rieseguire effects troppo spesso. Non dimenticare che React rimanda l'esecuzione di `useEffect` fino a che il browser non ha renderizzato, quindi fare lavoro extra non è un problema.
>
>Raccomandiamo l'uso della regola delle [exhaustive-deps](https://github.com/facebook/react/issues/14920) come parte del nostro package [`eslint-plugin-react-hooks`](https://www.npmjs.com/package/eslint-plugin-react-hooks#installation). Questo ci avvertirà quando le dipendenze non sono corrette e suggerirà un fix.

## Prossimi Steps {#next-steps}

Congratulazioni! Questa era una lunga pagina, ma siamo convinti di aver risolto molti dei tuoi dubbi legati agli effects. Hai imparato ad utilizzare sia l'Hook State che l'Hook Effect, ed utilizzandoli insieme puoi fare *molto*. Coprono la maggior parte dei casi d'uso per le classi -- e quando non lo fanno, potrai trovare utile [altri Hooks](/docs/hooks-reference.html).

Stiamo anche iniziando a vedere come gli Hooks risolvono problemi emersi in [Motivazione](/docs/hooks-intro.html#motivation). Abbiamo visto come il cleanup previene duplicazioni in `componentDidUpdate` e `componentWillUnmount`, mettendo insieme codice correlato, che ci aiuta a prevenire bug. Abbiamo inoltre visto come separare gli effects per scopo, il che è qualcosa che non potremmo assolutamente fare con le classi.

A questo punto potresti chiederti come funzionano gli Hooks. Come può React sapere quale chiamata `useState` corrisponde a quale variabile di state durante i re-render? Come fa React a mettere insieme effects precedenti e successivi ad ogni update? **Nella prossima pagina impareremo circa le [Regole degli Hooks](/docs/hooks-rules.html) -- sono essenziali per far funzionare gli Hooks.**
