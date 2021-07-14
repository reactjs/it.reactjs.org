---
id: faq-ajax
title: AJAX ed APIs
permalink: docs/faq-ajax.html
layout: docs
category: FAQ
---

### Come posso fare una chiamata AJAX ? {#how-can-i-make-an-ajax-call}

Puoi usare qualsiasi libreria AJAX con React. Le piu popolari sono [Axios](https://github.com/axios/axios), [jQuery AJAX](https://api.jquery.com/jQuery.ajax/), e il e il browser integrato [window.fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API).

### In quale punto del ciclo di vita del componente devo effettuare una chiamata AJAX? {#where-in-the-component-lifecycle-should-i-make-an-ajax-call}

Dovresti popolare i dati con chiamate AJAX nel metodo [`componentDidMount`](/docs/react-component.html#mounting). In questo modo puoi usare `setState` per aggiornare il tuo componente quando i dati vengono recuperati.

### Esempio: utilizzo dei risultati AJAX per impostare lo stato locale {#example-using-ajax-results-to-set-local-state}

Il componente seguente mostra come effettuare una chiamata AJAX in `componentDidMount` per popolare lo stato del componente locale.

L'API in questione restituisce un oggetto JSON di questo formato:

```
{
  "items": [
    { "id": 1, "name": "Apples",  "price": "$2" },
    { "id": 2, "name": "Peaches", "price": "$5" }
  ] 
}
```

```jsx
class MyComponent extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      error: null,
      isLoaded: false,
      items: []
    };
  }

  componentDidMount() {
    fetch("https://api.example.com/items")
      .then(res => res.json())
      .then(
        (result) => {
          this.setState({
            isLoaded: true,
            items: result.items
          });
        },
        // Nota: è importante gestire gli errori qui
        // invece di un blocco catch() in modo da non fare passare
        // eccezioni da bug reali nei componenti.
        (error) => {
          this.setState({
            isLoaded: true,
            error
          });
        }
      )
  }

  render() {
    const { error, isLoaded, items } = this.state;
    if (error) {
      return <div>Error: {error.message}</div>;
    } else if (!isLoaded) {
      return <div>Loading...</div>;
    } else {
      return (
        <ul>
          {items.map(item => (
            <li key={item.id}>
              {item.name} {item.price}
            </li>
          ))}
        </ul>
      );
    }
  }
}
```

Ecco l'equivalente con [Hooks](https://reactjs.org/docs/hooks-intro.html):

```js
function MyComponent() {
  const [error, setError] = useState(null);
  const [isLoaded, setIsLoaded] = useState(false);
  const [items, setItems] = useState([]);


  // Nota: l'array deps vuoto [] significa
  // questo useEffect verrà eseguito una volta
  // simile a componentDidMount()
  useEffect(() => {
    fetch("https://api.example.com/items")
      .then(res => res.json())
      .then(
        (result) => {
          setIsLoaded(true);
          setItems(result);
        },
        // Nota: è importante gestire gli errori qui
        // invece di un blocco catch() in modo da non fare passare
        // eccezioni da bug reali nei componenti.
        (error) => {
          setIsLoaded(true);
          setError(error);
        }
      )
  }, [])

  if (error) {
    return <div>Error: {error.message}</div>;
  } else if (!isLoaded) {
    return <div>Loading...</div>;
  } else {
    return (
      <ul>
        {items.map(item => (
          <li key={item.id}>
            {item.name} {item.price}
          </li>
        ))}
      </ul>
    );
  }
}
```
