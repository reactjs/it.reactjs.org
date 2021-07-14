---
id: web-components
title: Web Components
permalink: docs/web-components.html
redirect_from:
  - "docs/webcomponents.html"
---

React e la tecnologia [Web Components](https://developer.mozilla.org/en-US/docs/Web/Web_Components) sono stati creati per risolvere problemi differenti. La tecnologia Web Components permette di creare componenti riutilizzabili completamente incapsulati, React invece è una libreria che fornisce un sistema dichiarativo per mantenere sincronizzato il DOM con i dati. I due obiettivi sono complementari. Come sviluppatore sei libero di utilizzare React all'interno dei tuoi Web Components o di impiegare la tecnologia Web Components nei tuoi componenti React o di fare entrambe le cose nello stesso progetto.

La maggior parte delle persone che usano React non utilizzano la tecnologia Web Components ma potresti volerlo fare, in particolar modo se vuoi creare l'interfaccia grafica della tua applicazione usando delle librerie di componenti implementati come Web Components.

## Usare Web Components in React {#using-web-components-in-react}

```javascript
class HelloMessage extends React.Component {
  render() {
    return <div>Hello <x-search>{this.props.name}</x-search>!</div>;
  }
}
```

> Nota:
>
> I Web Components spesso espongono una API imperativa. Per esempio, un Web Component `video` potrebbe esporre le funzioni `play()` e `pausa()`. Per accedere all'API imperativa di un Web Component avrai bisogno di un ref, il quale permette di interagire direttamente con il suo nodo nel DOM. Se stai usando dei Web Components di terze parti la soluzione migliore consiste nello scrivere dei componenti React che fungano da wrappers per gli Web Components di tuo interesse.
>
> Gli eventi emessi da un Web Component potrebbero non propagarsi correttamente attraverso l'albero dei componenti gestiti da React.
> Potresti aver bisogno di collegare manualmente degli handler per gestire questi eventi all'interno dei tuoi componenti React.

Una fonte comune di confusione è che gli Web Components usano l'attributo "class" invece di "className".

```javascript
function BrickFlipbox() {
  return (
    <brick-flipbox class="demo">
      <div>front</div>
      <div>back</div>
    </brick-flipbox>
  );
}
```

## Usare React nei tuoi Web Components {#using-react-in-your-web-components}

```javascript
class XSearch extends HTMLElement {
  connectedCallback() {
    const mountPoint = document.createElement('span');
    this.attachShadow({ mode: 'open' }).appendChild(mountPoint);

    const name = this.getAttribute('name');
    const url = 'https://www.google.com/search?q=' + encodeURIComponent(name);
    ReactDOM.render(<a href={url}>{name}</a>, mountPoint);
  }
}
customElements.define('x-search', XSearch);
```

>Nota:
>
> Questo codice **NON** funzionerà se le classi saranno convertite con con Babel. Vedi [questa issue](https://github.com/w3c/webcomponents/issues/587) per la discussione.
> Includi il pacchetto [custom-elements-es5-adapter](https://github.com/webcomponents/polyfills/tree/master/packages/webcomponentsjs#custom-elements-es5-adapterjs) prima di caricare i tuoi Web Components per risolvere questo problema.
