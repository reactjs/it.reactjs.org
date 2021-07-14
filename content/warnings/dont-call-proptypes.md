---
title: Don't Call PropTypes Warning
layout: single
permalink: warnings/dont-call-proptypes.html
---

> Note:
>
> `React.PropTypes` sono stati spostati in un pacchetto diverso dalla React v15.5. Per favore utilizza invece [la libreria `prop-types`](https://www.npmjs.com/package/prop-types).
>
>Esiste [un codemod script](/blog/2017/04/07/react-v15.5.0.html#migrating-from-react.proptypes) per automatizzare questa conversazione.

In una futura pubblicazione attualizzata di React, il codice che implementa le funzioni di validazione dei PropType saranno rimosse. Quando questo accadrà, qualsiasi codice che chiamasse queste funzioni manualmente (che non sono state rimosse in produzione) lanceranno un errore.

### Dichiarare i PropTypes continua a essere corretto {#declaring-proptypes-is-still-fine}

L'utilizzo normale degli PropTypes continua a essere supportata:

```javascript
Button.propTypes = {
  highlighted: PropTypes.bool
};
```

Nulla da cambiare qui.

### Non chiamare PropTypes direttamente {#dont-call-proptypes-directly}

Utilizzare PropTypes in qualsiasi altro modo che quello annotato con i rispettivi componenti di React non è più supportato:

```javascript
var apiShape = PropTypes.shape({
  body: PropTypes.object,
  statusCode: PropTypes.number.isRequired
}).isRequired;

// No supportato!
var error = apiShape(json, 'response');
```

Quando dipendi dall'utilizzo dei PropTypes come questo, ti incoraggiamo a utilizzare o creare un fork dei PropTypes (come [questi](https://github.com/aackerman/PropTypes) [due](https://github.com/developit/proptypes) pacchetti)

Quando non risolvi il _warning_, questo codice si romperà in produzione con React 16.

### Se non chiami PropTypes direttamente però continui ad avere il _warning_ {#if-you-dont-call-proptypes-directly-but-still-get-the-warning}

Inspezionando lo stack trace prodotto dal _warning_. Troverai il componente definito responsabile per la chiamata diretta ai PropTypes. Il più delle volte, il problema è dovuto da un Proptypes di terze parti che ingloba i PropTypes di React, per esempio:

```js
Button.propTypes = {
  highlighted: ThirdPartyPropTypes.deprecated(
    PropTypes.bool,
    'Use `active` prop instead'
  )
}
```
In questo caso, `ThirdPartyPropTypes.deprecated` è un involucro chiamato `PropTypes.bool`. Questo pattern di per se va bene, ma innesca un falso positivo perché React pensa che tu stia chiamando PropTypes direttamente. La prossima sezione spiega come sistemare questo problema per una libreria implementando qualcosa come `ThirdPartyPropTypes`. Nel caso non fosse una libreria che hai scritto tu, puoi riportare un problema su di essa.

### Riparando un falso positivo in PropTypes di terze parti {#fixing-the-false-positive-in-third-party-proptypes}

Nel caso tu sia un autore di una libreria di PropTypes di terze parti e lasci i consumitori inglobare i React PropTypes esistenti, possono incominciare a vedere questo _warning_ venire dalla stessa. Questo succede perchè React non vede un "segreto" come ultimo argomento che [è passato](https://github.com/facebook/react/pull/7132) per detettare chiamate manuali dei PropTypes

Ecco come soluzionarlo. Utilizzeremo `deprecated` da [react-bootstrap/react-prop-types](https://github.com/react-bootstrap/react-prop-types/blob/0d1cd3a49a93e513325e3258b28a82ce7d38e690/src/deprecated.js) come esempio. Questa implementazioen passa solamente sotto gli argomenti: `props`, `propName` e `componentName`:

```javascript
export default function deprecated(propType, explanation) {
  return function validate(props, propName, componentName) {
    if (props[propName] != null) {
      const message = `"${propName}" la proprietá di "${componentName}" è stata deprecata.\n${explanation}`;
      if (!warned[message]) {
        warning(false, message);
        warned[message] = true;
      }
    }

    return propType(props, propName, componentName);
  };
}
```
Per riparare il falso positivo, fai attenzione passare l'argomento **all** sotto il PropType inglobato. Questo è facile da fare con la notazione `...rest` di ES6:

```javascript
export default function deprecated(propType, explanation) {
  return function validate(props, propName, componentName, ...rest) { // Nota ...rest qui
    if (props[propName] != null) {
      const message = `"${propName}" la proprietá di "${componentName}" è stata deprecata.\n${explanation}`;
      if (!warned[message]) {
        warning(false, message);
        warned[message] = true;
      }
    }

    return propType(props, propName, componentName, ...rest); // e qui
  };
}
```
Questo silenzierá il _warning_.
