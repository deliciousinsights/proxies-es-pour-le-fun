---
theme: ./theme
titleTemplate: 'Les proxies ES pour le fun et la gloire'
background: /riccardo-annandale-7e2pe9wjL9M-unsplash.jpg
download: true
exportFilename: proxies-es-pour-le-fun
class: text-center
highlighter: shiki
lineNumbers: false
favicon: https://delicious-insights.com/apple-touch-icon.png
info: |
  ## Les proxies ES pour le fun et la gloire

  Une présentation de Christophe Porteneuve à [BDX I/O 2022](https://www.bdxio.fr/).

  Envie de plus ? Notre [chaîne YouTube](https://www.youtube.com/c/DeliciousInsights) et nos [super formations](https://delicious-insights.com/fr/formations/) sont pour toi !
drawings:
  persist: false
  syncAll: false
css: unocss
---

# Les proxies ES<br/>pour le fun et la gloire

Une présentation de Christophe Porteneuve à [BDX I/O 2022](https://www.bdxio.fr/)

---

# `whoami`

```js
const christophe = {
  family: { wife: 'Élodie', sons: ['Maxence', 'Elliott'] },
  city: 'Paris, FR',
  company: 'Delicious Insights',
  trainings: ['Web Apps Modernes', 'Node.js', 'ES Total'],
  jsSince: 1995,
  claimsToFame: [
    'Prototype.js',
    'script.aculo.us',
    'Bien Développer pour le Web 2.0',
    'NodeSchool Paris',
    'Paris Web',
    'dotJS'
  ]
}
```


---

# « Proxy »… 🤔

Les proxies ES nous permettent de **redéfinir la sémantique de certains aspects clés du langage**.

<v-clicks>

C’est de la **métaprogrammation**, comme les méthodes `Object.*` et les **symboles prédéfinis**.

Ça n'altère pas l'objet d'origine : **ça l'enrobe**.

```js {1:20-30}
const proxy = new Proxy(origObject, handler)
```

En gros tout l'AOP : réactivité / *data binding*, RBAC, monitoring, logs, chronométrage, délégation…

```js
const chris = { age: 41.91170431211499 }
const proxy = new Proxy(christophe, {
  set(target, prop, value, recipient) {
    if (prop === 'age' && (typeof value !== 'number' || value < 0)) {
      throw new Error(`Invalid age: ${prop}.  Must be a non-negative number.`)
    }
    Reflect.set(target, prop, value, recipient)
  }
})
```

</v-clicks>

---

# Vocabulaire

## Trappe (*trap*)

Une **fonction** au nom prédéfini qui intercepte une **interaction de langage** pour la remplacer ou la personnaliser.  Nous verrons qu'elle peut **déléguer** au comportement d'origine grâce à l'API `Reflect`.

<v-click>

## Gestionnaire (*handler*)

Un **objet constitué de _traps_**.  Généralement mono-sujet, il implémente juste les trappes nécessaires à son besoin.  Par exemple, les indices négatifs sur les tableaux ne nécessitent que `get` et `set`.

</v-click>
<v-click>

## Proxy

Un objet qui **en enrobe un autre** et intercepte tout ou partie des interactions de langage potentielles avec cet objet.  Celles-ci sont déduites des méthodes du gestionnaire passé lors de la création du proxy.

</v-click>

---

# Trappes disponibles (1/2)

| Trappe                      | Intercepte…                                                                         |
| --------------------------- | ----------------------------------------------------------------------------------- |
| `get`                       | Lecture de propriété                                                                |
| `set`                       | Écriture de propriété                                                               |
| `has`                       | Opérateur `in` (test d'existence de propriété)                                      |
| `ownKeys`                   | `Object.keys()`, `Object.getOwnPropertyNames()` et `Object.getOwnPropertySymbols()` |
| `getOwnProperrtyDescriptor` | `Object.getOwnPropertyDescriptor()` / `…Descriptors()`                              |
| `defineProperty`            | `Object.defineProperty()`                                                           |

---

# Trappes disponibles (2/2)

| Trappe                              | Intercepte…                                                     |
| ----------------------------------- | --------------------------------------------------------------- |
| `deleteProperty`                    | Opérateur `delete` (retrait de propriété)                       |
| `isExtensible`, `preventExtensions` | `Object.isExtensible()`, `Object.preventExtensions()`           |
| `getPrototypeOf`, `setPrototypeOf`  | `Object.getPrototypeOf()`, `Object.setPrototypeOf()`            |
|                                     |                                                                 |
| `apply`                             | Appel de la fonction                                            |
| `construct`                         | Utilisation de la fonction comme constructeur (opérateur `new`) |

---

# Accéder au comportement d'origine

L'espace de noms `Reflect` fournit des méthodes pour chaque trappe, à signature identique.

<v-clicks>

On a parfois l'impression d'une **duplication** des méthodes de `Object`, mais en fait il y a des **différences subtiles** (ex. pas de transtypage, renvoi de booléens au lieu de levées d'exceptions).

D'une façon général c'est **plus léger** que les méthodes d'`Object`.

Ça s'approche pas mal de ce que la spec d'ES appelle des **_internal slots_**, tels que `[[Call]]`.

J'utilise **presque toujours** l'API `Reflect` dans mes trappes, même s'il peut sembler plus facile de recourir à `in`, `delete` ou des propriétés en accès direct.  Ainsi, je m'assure de n'oublier aucun cas à la marge.

</v-clicks>

---

# Trappes `get` et `set`

```js
get(target, prop, receiver)
```

Intercepte les lectures de propriétés *(y compris inexistantes)*.  Notez que c'est un préalable à _l'appel d'une méthode_, qui est une propriété comme une autre.

Par défaut : utilise l'éventuel accesseur lecteur, et renvoie `undefined` pour les propriétés absentes.

<v-click>

```js
set(target, prop, value, receiver)
```

Intercepte les écritures de propriétés.

Par défaut : utilise l'éventuel accesseur écrivain, et si la propriété est absente, la crée à la volée (sauf si l'objet est non-extensible).

</v-click>

<footer v-click>

Note : `prop` est toujours de type `String` ou `Symbol` (conformément aux contraintes de noms de propriétés).

</footer>

---

# La démo LOL de `get` : [tpyo](https://github.com/mathiasbynens/tpyo)

Par Mathias Bynens, ingénieur v8.

Redéfinit l'accès aux propriétés en utilisant la _distance de Levenstein_ la plus faible en cas de propriété manquante 😂

```js {all|3-5|6|7|8|9|10|12-13|all}
const tpyo = require('tpyo')

const speakers = tpyo(
  ['Anaïs', 'Bérengère', 'Cécile', 'Justine', 'Manon', 'Amélie', 'Alice', 'Noémie']
)
speakers.longueur         // => 8 (et rien que pour le matin !)
speakers.flop()           // => 'Noémie' (mais pas du tout !)
speakers.splif(-4)        // => ['Justine', 'Manon', 'Amélie', 'Alice']
speakers.joint(' 😄 ')    // => 'Anaïs 😄 Bérengère 😄 Cécile'
speakers.full('of win')   // => Ben carrément (3 x 'of win')

const math = tpyo(Math)
math.skirt(9) // => 3. Ben voyons.
```

---

# 😈 Quelle est cette diablerie ?!

```js {all|4|5|6-8|10|10-11|10-12|all}
// Simplifié un poil pour tenir sur la diapo…

function tpyo(something) {
  return new Proxy(something, {
    get(target, name) {
      if (name in target) {
        return target[name]
      }

      const properties = getProperties(target)
      const closestProperty = findSimilarProperty(name, properties)
      return target[closestProperty]
    }
  })
}
```

---

# Une démo `get` utile : client API à la volée

Tu te souviens des jours ~~heureux~~ maudits de COM, DCOM, CORBA et de la génération de proxy client ?

On peut faire beaucoup mieux désormais !

```js {1|2-3|5-6|5-8|10-11|all}
const api = makeRestProxy('https://jsonplaceholder.typicode.com')
await api.users()
// => [{ id: 1, name: 'Leanne Graham' … }, { id: 2, name: 'Ervin Howell', … }, …]

await api.users(1)
// => { id: 1 name: 'Leanne Graham', username: 'Bret', email: 'Sincere@april.biz', … }

// (Paie ta cohérence de ouf entre les champs factices 😅)

await api.posts(42)
// => { userId: 5, id: 42, title: 'commodi ullam…', body: 'odio fugit…' }
```

---

# Code d'une démo `get` utile : client API à la volée

<div className="flex-cols">

```js {1-2,11-12|3,10|4,8|5-7|9|all}
function makeRestProxy(baseURL) {
  return new Proxy({}, {
    get(target, prop, receiver) {
      if (!(prop in target)) {
        Reflect.defineProperty(target, prop, {
          value: makeFetchCall(baseURL, prop)
        })
      }
      return Reflect.get(target, prop, receiver)
    }
  })
}
```

<v-click>

```js
// Considérablement simplifié pour la diapo

function makeFetchCall(baseURL, prop) {
  return async function fetch(id) {
    const path = id == null ? '' : `/${id}`
    const res = await fetch(`${baseURL}/${prop}${path}`, {
      headers: {
        Accept: 'application/json',
        'Content-Type': 'application/json',
      },
    })
    return res.json()
  }
}
```

</v-click>
</div>

---

# Démo `get` + `get` : indices négatifs de tableaux

Il faut dire qu'ils nous manquent (ex. Ruby), et `.at()` c'est bien mais ça ne suffit pas.

```js{1-2|1,4-5|1,7-9}
const names = ['Alice', 'Bob', 'Claire', 'David']
const coolNames = allowNegativeIndices(names)

coolNames[-1]
// => 'David'

coolNames[-2] = 'Clara'
names
// => ['Alice', 'Bob', 'Clara', 'David']
```

---

# Code des indices négatifs de tableaux

```js {1-2,15-16|3-8|9-14|all}
function allowNegativeIndices(arr) {
  return new Proxy(arr, {
    get(target, prop, receiver) {
      if (prop < 0) {
        prop = target.length + Number(prop)
      }
      return Reflect.get(target, prop, receiver)
    },
    set(target, prop, value, receiver) {
      if (prop < 0) {
        prop = target.length + Number(prop)
      }
      return Reflect.set(target, prop, value, receiver)
    }
  })
}
```

---

# Démo `get` (+ `set` ?) utile : objets défensifs

Parfois on ne veut pas `undefined` sur une propriété manquante : on veut se prendre une exception !

```js
const basis = { first: 'Odile', last: 'Deray' }
const defensive = makeDefensive(basis)
defensive.first  // => ✅ 'Odile'
defensive.middle // => 💥 ReferenceError: No middle property on object
```

---

# Code des objets défensifs

```js {all|3-8}
function makeDefensive(obj) {
  return new Proxy(obj, {
    get(target, prop, receiver) {
      if (!(prop in target)) {
        throw new ReferenceError(`No ${prop} property on object`)
      }
      return Reflect.get(target, prop, receiver)
    }
  })
}
```

---

# Un mot sur `apply` et `construct`

Pour le coup, l'objet enrobé doit être **une fonction**.  Je sais, 🤯.

```js
apply(target, thisArg, argumentsList)
```

- Intercepte l'appel à la fonction (opérateur `(…)`) aini que les équivalents programmatiques (`apply()` et `call()`).
- Super utile pour du *copy-on-write* qui permet d'enrober automatiquement les valeurs de retours des méthodes d'objets enrobés par un proxy.

<br/>
<v-click>

```js
construct(target, argumentsList, newTarget)
```

- Intercepte l'opérateur `new` sur une fonction. Le résultat **doit** être un objet.
- On se moque généralement de `newTarget`, sauf si on butte sur une vérification de `new.target` dans le code utilisateur.

</v-click>

---

# Proxies révocables

Au lieu d'instancier un proxy par construction, on peut utiliser une _factory_ qui nous permettra de révoquer à tout moment l'accès (_via_ le proxy s'entend) à l'objet d'origine.

En gros, ce sont des « références périssables ».

```js
const { proxy, revoke } = Proxy.revocable(target, handler)
```

<v-clicks>

Y'a clairement des super cas d'usages en termes de sécurité.

```js
const { proxy, revoke } = Proxy.revocable({ first: 'John' }, {})

proxy.first // => 'John'
revoke()
proxy.first // => TypeError: Cannot perform 'get' on a proxy that has been revoked
```

</v-clicks>

---

# Proxy révocable utile : quota d'appels

```js {1-2|3|3-4|3-5}
const fx = (...args) => args
const meteredFx = meter(fx, { max: 2 })
meteredFx('foo')        // => ✅ ['foo']
meteredFx('bar', 'baz') // => ✅ ['bar', 'baz']
meteredFx('fuu')        // => ⛔ TypeError: Cannot perform 'apply' on a proxy that has been revoked
```

<br/>
<v-click>

```js {1-2,9-12|3,8|4-6|7}
function meter(fx, { max }) {
  const { proxy, revoke } = Proxy.revocable(fx, {
    apply(target, thisArg, argumentsList) {
      if (--max <= 0) {
        revoke()
      }
      return Reflect.apply(target, thisArg, argumentsList)
    }
  })
  return proxy
}
```

</v-click>

---

# Proxy révocable utile : TTL

```js {1-2|4|5|6}
const obj = { first: 'John' }
const moth = scheduleExpiry(obj, { ttl: 50 })

moth.first // => 'John'
setTimeout(() => console.log(moth.first), 40) // => 'John' après 40ms
setTimeout(() => console.log(moth.first), 60) // => TypeError après 60ms
```

<br/>
<v-click>

```js {1-2,4-5|3}
function scheduleExpiry(obj, { ttl = 100 } = {}) {
  const { proxy, revoke } =  Proxy.revocable(obj, {})
  setTimeout(revoke, ttl)
  return proxy
}
```

</v-click>

---

# Une dinguerie : [Immer](https://immerjs.github.io/immer/)

Aide à l'immutabilité 😍 **extraordinaire** 😍 de Michel Westrate (également auteur de MobX fame).  Nous permet d'écrire du **code mutatif classique** !

**Fait du copy-on-write récursif** à l'aide de proxies révocables récursifs sur à peu près toutes les trappes 😅

```js {1-7|8,11|9|10|8-11}
import produce from 'immer'

const baseState = [
  { todo: 'Apprendre React', done: true },
  { todo: 'Explorer Immer', done: false },
]

const nextState = produce(baseState, (draft) => {
  draft.push({ todo: 'Tooter à ce sujet' })
  draft[1].done = true
}) // => baseState intact, nextState correct.
```

<footer style='text-align: center'>

  [Super leçon Egghead](https://egghead.io/lessons/redux-simplify-creating-immutable-data-trees-with-immer) •
  [Billet d'intro](https://medium.com/@mweststrate/introducing-immer-immutability-the-easy-way-9d73d8f71cb3) (juin 2018) •
  [Site officiel](https://immerjs.github.io/immer/)

</footer>

---

# Immer et les états locaux React

<div class="flex-cols">

<div>

**Avant** 😒

```js {1,8|2|3-5,7|6|all}
setState((prev) => ({
  ...prev,
  user: {
    ...prev.user,
    age: prev.user.age + 1,
    daysUsed: [...prev.user.daysUsed, new Date()]
  }
}))
```

</div>
<div v-click=4>

**Après** 🤩

```js {1,4|2|3|all}
setState(produce(({ user }) => {
  user.age += 1
  user.daysUsed.push(new Date())
}))
```

</div>
</div>

---

# Immer et les réducteurs Redux

_(Au fait, Immer est pré-configuré par défaut dans [Redux Toolkit](https://redux-toolkit.js.org/usage/immer-reducers))_

<div class="flex-cols">

<div>

**Avant** 😒

```js {all|4-10}
export function byId(state, action) {
  switch (action.type) {
    case RECEIVE_PRODUCTS:
      return {
        ...state,
        ...action.products.reduce((obj, product) => {
          obj[product.id] = product
          return obj
        }, {})
      }
    default:
      return state
  }
}
```

</div>
<div v-click=2>

**Après** 🤩

<!-- all|all here is so the incorrect custom-click-rank handling fudges code highlight later steps. -->
```js {all|all|3-7}
export default createReducer(DEFAULT_STATE, (builder) =>
  builder
    .addCase(RECEIVE_PRODUCTS, (draft, { products }) => {
      for (const product of products) {
        draft[product.id] = product
      }
    })
)
```

</div>
</div>

---

# Immer : à quoi ça ressemble sous le capot ?

```js {all|4|12}
// Extraits choisis, simplifiés pour cette diapo
export function createProxy(base, parent) {
  // …
  const {revoke, proxy} = Proxy.revocable(state, {
    get(state, prop) {
      if (prop === DRAFT_STATE) return state
      const { drafts } = state
      if (!state.modified && has(drafts, prop)) {
        return drafts[prop]
      }
      // …
      return (drafts[prop] = createProxy(value, state))
    },
    // …
  })
  // …
}
```

---

# _&lt;Insérer avertissement obligatoire ici&gt;_

Il faut faire attention à `this` : l'enrobage par proxy l’affecte (il référence le proxy).

<v-clicks>

Les mécanismes comparant l'identité, tels que `WeakMap` (pour transpiler les champs d'instances privés par exemple), sont donc affectés.

Idem pour les **constructeurs prédéfinis** dont les méthodes utiliseraient des _internal slots_ (ex. `Date#getDate()`) : ça contourne les trappes `get` / `set`.

</v-clicks>

<div class="flex-cols">
<div v-click>

**Problème**

```js
const target = new Date('2022-12-02')
const proxy = new Proxy(target, {})

proxy.getDate()
// => TypeError: this is not a Date object
// (car `getDate` exploite [[NumberData]])
```

</div>
<div v-click>

**Solution de contournement**

```js {3-6}
const target = new Date('2022-12-02')
const proxy = new Proxy(target, {
  get(target, prop, receiver) {
    const result = Reflect.get(target, prop, receiver)
    return prop === 'getDate' ? result.bind(target) : result
  }
})
proxy.getDate() // => 5 (vendredi)
```

</div>
</div>

---
layout: center
---

# Viens nous voir !

On est sympas.

Chez [Delicious Insights](https://delicious-insights.com/), on fait des **[formations](https://delicious-insights.com/fr/formations/) qui déchirent tout**, notamment sur [100% de JS pur](https://delicious-insights.com/fr/formations/es-total/), [React et les PWA](https://delicious-insights.com/fr/formations/web-apps-modernes/), [Node.js](https://delicious-insights.com/fr/formations/node-js/) et [Git](https://delicious-insights.com/fr/formations/git-total/).

_(Franchement, elles envoient du bois.)_

On peut aussi venir [gronder ton archi / ta codebase](https://delicious-insights.com/fr/services/) (mais gentiment), voire réaliser tes **preuves de concept** pour toi, en mode pas jetable du tout™.


À côté de ça, tu devrais **carrément** t'abonner à notre fabuleuse [chaîne YouTube](), qui déborde de tutos, cours, livestreams, talks en conférences, etc. et c'est évidemment **gratuit** !

---
layout: cover
background: /riccardo-annandale-7e2pe9wjL9M-unsplash.jpg
---

# Merci

.

<style>
  .feedback { display: flex; flex-direction: column; gap: 1em; align-items: center; }
  .feedback img {
    max-height: 7em;
    display: block;
  }
  .feedback p { margin: 0; }
</style>

<div class="feedback">

[![](/qrcode.png)](https://openfeedback.io/r46KviPgLYMQfQnFpaGS/2022-10-31/556)

[Laisse tes impressions ici !](https://openfeedback.io/r46KviPgLYMQfQnFpaGS/2022-10-31/556)  Ça ne prend qu'un instant.

Cette présentation est sur [`bit.ly/proxies-es`](https://bit.ly/proxies-es).

[`@porteneuve@piaille.fr`](https://piaille.fr/@porteneuve) / [`@DelicioInsights`](https://twitter.com/DelicioInsights) / [YouTube](https://youtube.com/c/DeliciousInsights)

</div>

<div class="mt-12 text-sm" style="opacity: 0.5">


Crédits : photos de couverture par <a href="https://unsplash.com/@pavement_special?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Riccardo Annandale</a> sur <a href="https://unsplash.com/s/photos/creativity?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>

</div>
