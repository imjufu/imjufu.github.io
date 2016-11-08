---
layout: post
title: Javascript, gardez le meilleur !
date: 2015-02-27
---

Javascript est un langage remarquablement puissant. C'est une sorte de Lisp revêtu des habits du C.

Javascript est basé sur de très bonnes idées : fonctions, couplage lâche, objets dynamiques, notation littérale expressive des objets… et quelques mauvaises : modèle de programmation axé sur les variables globales.

## Typage faible

Javascript est un langage faiblement typé. Contrairement aux langages à typage fort, les compilateurs Javascript ne détectent pas les erreurs de type. Cela peut inquiéter mais est en fait libérateur. Il permet d'éviter des hiérarchies de classes complexes, de se battre avec le transtypage…

## Héritage

Il existe plusieurs patterns d'héritage en Javascript.

### Pseudo-classique

Ce pattern est destiné à paraître orienté objet.

```js
var Mammal = function(name) {
  this.name = name;
};
Mammal.prototype.get_name = function() {
  return this.name;
};
Mammal.prototype.says = function() {
  return this.saying || '';
};

var myMammal = new Mammal('Herb the Mammal');
myMammal.get_name(); // Herb the Mammal

var Cat = function(name) {
  this.name = name;
  this.saying = 'meow';
}

Cat.prototype = new Mammal();
Cat.prototype.get_name = function() {
  return this.says() + ' ' + this.name;
};

var myCat = new Cat('Henrietta');
myCat.says(); // meow
myCat.get_name(); // meow Henrietta
```

Il n'y a pas de portée privée, toutes les propriétés sont publiques, il n'y a pas d'accès aux méthodes `super`.
Attention de ne pas oublier le prefix `new` lors de l'appel à la fonction constructeur sinon `this` n'est pas lié à un nouvel objet mais à l'objet global !

### Prototypal

Javascript possède un système d'objets sans classe dans lequel les objets héritent directement des propriétés d'autres objets.
C'est une relation dynamique. Si l'on ajoute une nouvelle propriété à un prototype, cette propriété est immédiatement visible dans tous les objets qui héritent de ce prototype.

L'héritage pas prototype est plus simple que l'héritage classique. Commencez par créer un objet utile puis créer de nombreux objets analogues à ce premier.

```js
var myMammal = {
  name: 'Herb the mammal',
  get_name: function() {
    return this.name;
  },
  says: function() {
    return this.saying || '';
  }
};

var myCat = Object.create(myMammal);
myCat.name = 'Henrietta';
myCat.saying = 'meow';
myCat.get_name = function() {
  return this.says() + ' ' + this.name;
};
```

On parle d'héritage différentiel.
Ce pattern n'offre pas de portée privée. Toutes les propriétés sont visibles.

### Fonctionnel

Appliquons ce pattern à notre exemple `mammal`.

```js
var mammal = function(spec) {
  var that = {};
  that.get_name = function() {
    return spec.name;
  };
  that.says = function() {
    return spec.saying || '';
  };
  return that;
};
var myMammal = mammal({ name: 'Herb' });

var cat = function(spec) {
  spec.saying = 'meow';
  var that = mammal(spec);
  that.get_name = function() {
    return this.says() + ' ' + this.name;
  };
  return that;
}
var myCat = cat({ name: 'Henrietta' });
```

L'objet `spec` contient toutes les informations dont le constructeur a besoin pour créer une instance.
Les propriétés sont maintenant complètement privées. Elles ne sont accessibles que par des méthodes.

Un des avantages du pattern fonctionnel est de pouvoir invoquer les méthodes `super`.

```js
Object.method('superior', function(name) {
  var that = this, method = that[name];
  return function() { return method.apply(that, arguments); };
});

var coolcat = function(spec) {
  var that = cat(spec), super_get_name = that.superior('get_name');
  that.get_name = function() {
    return 'like ' + super_get_name() + ' baby';
  };
  return that;
};

var myCoolCat = coolcat({ name: 'Bix' });
myCoolCat.get_name(); // 'like meow Bix baby'
```

Si tout est privé, l'objet est sécurisé. Les propriétés de l'objet peuvent être remplacées ou supprimées mais l'intégrité de l'objet n'est pas compromise.
Si toutes les méthodes de l'objet n'utilisent pas `this` ou `that` alors l'objet est durable. Un objet durable ne peut pas être compromis. Un attaquant ne peut accéder à l'état interne de l'objet que par des méthodes définies.

### Parties

Un objet peut être construit grâce à un ensemble de parties. Une fonction peut très bien ajouter des fonctionnalités à un objet.

Par exemple, on peut créer une fonction capable d'ajouter des fonctionnalités de traitement d'évènements simples à n'importe quel objet.

```js
var eventuality = function(that) {
  var registry = {};
  that.fire = function(event) {
    var array, func, handler, i, type = typeof event == 'string' ? event : event.type;
    if(registry.hasOwnProperty(type)) {
      array = registry[type];
      for(i = 0; i < array.length; i++) {
        handler = array();
        func = handler.method;
        if(typeof func == 'string') {
          func = this[func];
        }
        func.apply(this, handler.parameters || [events]);
      }
    }
    return this;
  };
  that.on = function(type, method, parameters) {
    var handler = {
      method: method,
      parameters: parameters
    };
    if(registry.hasOwnProperty(type)) {
      registry[type].push(handler);
    } else {
      registry[type] = [handler];
    }
    return this;
  };
  return that;
};
```

## Grammaire

### Nombres

Javascript possède un seul type numérique représenté sous la forme d'un nombre 64 bits à virgule flottante, comme le `double` Java. Il n'existe pas de type entier séparé, `1` et `1.0` correspondent à la même valeur. Il est toutefois possible d'étendre le type `Number` pour contourner ce problème.

```js
Number.prototype.method('integer', function () {
  return Math[this < 0 ? 'ceiling' : 'floor'](this);
});
(-10 / 3).integer(); // -3
```

La valeur `NaN` est une réprésentation numérique d'une valeur non numérique. On l'obtient lorsqu'une opération numérique ne peut pas renvoyer de nombre.
`NaN` est différente de toute valeur, y compris d'elle-même. `NaN` peut être détecté avec la fonction `isNaN(nombre)`.

### Chaînes

Tous les caractères en Javascript sont codés sur 16 bits. Les chaînes sont immuables. Une fois créée, une chaîne ne peut pas être changée.
`\` correspond au caractère d'échappement.
Les chaînes possèdent une propriété `length`.

Il n'existe pas de méthode pour supprimer les espaces en début et fin de chaîne. Solution simple en étendant le type `String` :

```js
String.prototype.method('trim', function() {
  return this.replace(/^\s+|\s+$/g, '');
});
```

### Valeurs égalent à `false`

* `false`
* `null`
* `undefined`
* chaîne vide
* `0`
* `NaN`

Toutes les autres valeurs sont `true`, dont `true`, la chaîne `false` et tous les objets.

### Tableaux

Les tableaux en Javascript sont sensiblement plus lent qu'un véritable tableau. Javascript convertit les indices de tableau en chaînes qui sont utilisées pour créer des propriétés. La récupération et la mise à jour fonctionnent comme avec les objets.
Les tableaux ont des méthodes intégrées de base très utiles.

```js
var empty = [];
var numbers = ['zero', 'one', 'two', 'three'];
// Objet littéral :
var numbers_object = { '0': 'zero', '1': 'one', '2': 'two', '3': 'three' };

empty.length; // O
numbers.length; // 4
```

L'objet littéral `numbers_object` produit un résultat similaire à `numbers` : mêmes propriétés, même longueur.
Toutefois, `numbers_object` hérite de `Object.prototype` alors que `numbers` hérite de `Array.prototype`. C'est pour cela qu'il n'est pas possible d'utiliser la propriété `length` sur `numbers_object`.

Javascript permet aux tableaux de contenir n'importe quel type de valeurs.

```js
var misc = ['string', 42, true, null, undefined, ['nested'], {object: true}, NaN, Infinity];
misc.length; // 9
```

#### length

`length` n'est pas une limite supérieure en Javascript. Si un élément est stocké avec un indice supérieur à la valeur de `length`, alors la propriété `length` est augmentée. Il n'y a pas d'erreur de limite. `length` correspond juste à l'index entier le plus grand du tableau + 1.

```js
var myArray = [];
myArray.length; // 0
myArray[10000] = true;
myArray.length; // 10001
```

`length` peut être définie de manière explicite. Si on l'augmente, cela n'alloue pas plus d'espace au tableau. Si on le diminue, cela supprime toutes les propriétés dont l'index est supérieur ou égal à la nouvelle valeur de `length`.

```js
numbers.length = 1;
// numbers == ['zero']
numbers.push('one');
numbers.length; // 2
```

#### delete

```js
var numbers = ['zero', 'one', 'two', 'three'];
delete numbers[2];
// numbers == ['zero', 'one', undefined, 'three']
```

`delete` laisse un trou dans le tableau. Si on souhaite décaler les éléments du tableau, il faut préférer la méthode `splice`.
Toutefois, cette méthode peut être lente pour les tableaux de grande taille.

```js
numbers.splice(2, 1);
// Cela supprime 1 élément du tableau en partant de l'index 2
// numbers == ['zero', 'one', 'three']
```

#### concat

`concat` produit un nouveau tableau contenant une copie superficielle de ce tableau à laquelle sont ajoutés les éléments. Si un des éléments est un tableau, chacun de ses éléments est ajouté individuellement.

```js
var a = ['a', 'b', 'c'];
var b = ['x', 'y', 'z'];
var c = a.concat(b, true);
// c == ['a', 'b', 'c', 'x', 'y', 'z', true]
```

#### join

`join` permet de créer une chaîne à partir d'un tableau en concaténant ses éléments séparés par un séparateur. Par défaut, `,`.

```js
var a = ['a', 'b', 'c'];
var c = a.join('-');
// c == 'a-b-c'
```

#### push et pop

Ces méthodes permettent de faire fonctionner un tableau comme une pile. `pop` supprimer et renvoie le dernier élément. `push` ajoute un élément à la fin et retourne la longueur du tableau.

```js
var a = ['a', 'b', 'c'];
var c = a.pop();
// c == 'c'
// a == ['a', 'b']
var b = a.push('w');
// a == ['a', 'b', 'w']
// b == 3
```

#### reverse

Cette méthode inverse l'ordre des éléments d'un tableau et retourne le tableau.

```js
var a = ['a', 'b', 'c'];
var b = a.reverse();
// a == b == ['c', 'b', 'a']
```

#### shift

`shift` supprime et retourne le premier élément d'un tableau. Attention, cette méthode est bien plus lente que `pop`.

```js
var a = ['a', 'b', 'c'];
var c = a.shift();
// a == ['b', 'c']
// c = 'a'
```

#### slice

Cette méthode crée une copie superficielle d'une portion d'un tableau. Elle copie de `tableau[début]` à `tableau[fin - 1]`. Par défaut, le paramètre `fin` est facultatif et il vaut `tableau.length`. Si un des paramètres est négatif, `tableau.length` est ajouté afin de le rendre positif.

```js
var a = ['a', 'b', 'c'];
var b = a.slice(0, 1);
// b == ['a']
var c = a.slice(1);
// c == ['b', 'c']
```

#### sort

`sort` trie le contenu d'un tableau et le remplace. Elle présuppose que les éléments doivent être triés comme des chaînes. Elle ne vérifie pas le type des éléments avant de les comparer ce qui produit le résultat aberrant suivant :

```js
var n = [4, 15, 8, 23, 16, 42];
n.sort();
// n == [15, 16, 23, 4, 42, 8]
```

Toutefois, la fonction de comparaison peut être remplacé. Celle-ci doit prendre deux paramètres et renvoyer :

* 0 si égalité
* un nombre négatif si le premier paramètre doit être placé en premier
* un nombre positif si le second paramètre doit être placé en premier


```js
n.sort(function (a, b) {
  return a - b;
});
// n == [4, 8, 15, 16, 23, 42]
```

Et on peut même rendre cette fonction de tri plus intelligente,

```js
var m = ['aa', 'bb', 'a', 4, 15, 8, 23, 16, 42];
m.sort(function (a, b) {
  if (a === b) {
    return 0;
  }
  if (typeof a === typeof b) {
    return a < b ? -1 : 1;
  }
  return typeof a < typeof b ? -1 : 1;
});
// m == [4, 8, 15, 16, 23, 42, 'a', 'aa', 'bb']
```

## Portée

La portée contrôle la visibilité et les temps de vie des variables et des paramètres. Cela réduit les collisions de noms et fournit une gestion de la mémoire automatique.

```js
var foo = function() {
  var a = 3, b = 5;
  var bar = function() {
    var b = 7, c = 11;
    // a == 3, b == 7, c == 11
    a += b + c;
    // a == 21, b == 7, c == 11
  };
  bar();
  // a == 21, b == 5
};
```

La plupart des langages disposent d'une portée de bloc. Les variables sont invisibles en dehors du bloc et sont libérées à la fin de l'exécution du bloc.

Ce n'est pas le cas en Javascript. Il possède une portée de fonction. Les paramètres et les variables sont invisibles en dehors de la fonction. Il est préférable de déclarer toutes les variables utilisées dans une fonction en haut du corps de la fonction.

### Closure

Les fonctions internes ont accès aux paramètres et aux variables des fonctions à l'intérieur desquelles elles sont définies (à l'exception de `this` et `arguments`). La fonction interne a accès aux variables elles-mêmes des fonctions externes et non à des copies. Cela est possible car la fonction a accès au contexte dans lequel elle a été créée.

C'est une bonne chose car cela permet d'éviter le problème suivant :

```js
// Objectif : Créer une fonction qui attribue correctement des fonctions de gestionnaire d'évènements à un tableau de noeuds. Lorsqu'on clique sur un noeud, une boîte d'alerte affiche l'ordinal du noeud.

// Mauvais exemple
var add_the_handlers = function(nodes) {
  var i;
  for (i = 0; i < nodes.length; i += 1) {
    nodes[i].onclick = function(e) {
      alert(i);
    };
  }
};
// Au lieu d'afficher l'ordinal du noeud, la boîte d'alerte va afficher le nombre de noeuds.
// La fonction échoue car les fonctions de gestionnaire sont liées à la variable i et non à la valeur de la variable i au moment où la fonction a été créée.

// Bon exemple
var add_the_handlers = function(nodes) {
  var i;
  for (i = 0; i < nodes.length; i += 1) {
    nodes[i].onclick = function(i) {
      return function(e) {
        alert(i);
      }
    }(i);
};
// Au lieu d'attribuer une fonction à onclick, on invoque une fonction, en lui passant i, qui retourne une fonction de gestionnaire d'évènements liée à la valeur de i qui a été passée (et non au i défini dans add_the_handlers).
```

## Modules

Grâce aux closures et aux fonctions, il est possible de créer des fonctions ou objets présentant une interface mais masquant son implémentation. C'est ce qu'on appelle un module.
Grâce aux fonctions pour produire des modules, il est possible d'éviter complètement les variables globales. Il est très efficace pour encapsuler des applications et produire des objets sécurisés.

```js
// Objectif : créer un objet qui produit des chaînes uniques composées d'un préfixe et d'un numéro de séquence.

var serial_maker = function() {
  var prefix = '', seq = 0;
  return {
    set_prefix: function(p) {
      prefix = String(p);
    },
    set_seq: function(s) {
      seq = s;
    },
    gensym: function() {
      var result = prefix + seq;
      seq += 1;
      return result;
    }
  };
}();

var seqer = serial_maker();
seqer.set_prefix('Q');
seqer.set_seq(1000);
seqer.gensym(); // Q1000

// Les méthodes n'utilisent pas this ou that. seqer ne peut donc pas être compromis. prefix ou seq ne peuvent pas être modifié en dehors des méthodes. seqer est mutable, ses méthodes peuvent être remplacées mais cela ne donne pas accès à prefix et seq. seqer est juste une collection de fonctions.
// Si seqer.gensym est passé à une fonction tierce, cette fonction ne peut que générer des chaînes uniques.
```

## Objets

Les types simples du Javascript sont les nombres, les chaînes, les booléens, `null` et `undefined`. Toutes les autres valeurs sont des objets.
Les objets en Javascript sont des collections modifiables à clés. Les tableaux, les fonctions, les expressions régulières… sont des objets.
Les objets sont passés par référence. Ils ne sont jamais copiés.

Les objets littéraux héritent de `Object.prototype`, un objet standard.


### Récupération

La valeur d'un objet peut être récupérée en y accédant grâce à `[]` ou `.`.

```js
foo['bar'] // 42
foo.bar // 42
```

### Mise à jour

Une valeur d'un objet est mise à jour par attribution. Si l'objet ne possède pas déjà la propriété, il est augmenté.

```js
foo['bar'] = 42;
foo.bar = 777;
foo.status = 'up';
```

### Suppression

`delete` permet de supprimer la propriété d'un objet mais ne touche à aucun objet dans la chaîne des prototypes. Toutefois, la suppression d'une propriété d'un objet peut permettre à une propriété dans la chaîne des prototypes de ressurgir.

```js
another_foo.nickname // 'Bart'
delete another_foo.nickname;
another_foo.nickname // Surnom du prototype : 'Homer'
```

## Fonctions

Les fonctions héritent de `Function.prototype`, qui hérite de `Object.prototype`. Elles disposent de deux propriétés supplémentaires : le contexte et le code qui implémente le comportement.

### Patterns d'invocation

Il existe 4 patterns d'invocation de fonction :

**Pattern d'invocation de méthode**
Une fonction stockée sous forme de propriété d'un objet, autrement dit une méthode, est liée à cet objet lors de son invocation. Elle peut utiliser `this` pour accéder (et/ou modifier) le contexte de l'objet, elle est alors considéré comme "méthode publique".

**Pattern d'invocation de fonction**
En dehors d'un objet, une fonction est invoquée comme… une fonction ! Invoquée de cette manière, une fonction est liée à l'objet global, c'est une erreur de conception du langage. Cette erreur implique qu'une méthode ne peut pas employer de fonction interne pour l'aider car la fonction interne ne partage pas l'accès de la méthode à l'objet. `this` est lié à la mauvaise valeur. Toutefois, il est possible de contourner cela en attribuant `this` à une autre variable dans la méthode. Par convention, cette variable s'appelle `that`.

**Pattern d'invocation de constructeur**
Une fonction invoquée avec le préfixe `new` crée un nouvel objet. Par convention, ces fonctions sont affectées à des variables dont la première lettre est en majuscule.
Il est déconseillé d'utiliser ce type de fonction constructeur.

```js
var Foo = function (string) {
  this.nickname = string;
};
Foo.prototype.get_nickname = function () { return this.nickname; };

var myFoo = new Foo('bar');
myFoo.get_nickname(); // bar
```

**Pattern d'invocation avec `apply`**
La méthode `apply` permet de passer un tableau d'arguments à une fonction. Elle prend deux paramètres : la valeur qui doit être liée à `this` et le tableau de paramètres.

```js
var add = function (a, b) { return a + b; };
var array = [3, 4];
var sum = add.apply(null, array); // sum == 7

var nicknameObject = { nickname: 'Chuck' };
var nickname = Foo.prototype.get_nickname.apply(nicknameObject); // status == 'Chuck'
```

### Arguments

`arguments` donne à la fonction tous les arguments fournis à l'invocation.

```js
var sum = function () {
  var i, sum = 0;
  for (i = 0; i < arguments.length; i += 1) {
    sum += arguments[i];
  }
  return sum;
}
sum(1, 30, 11); // 42
```

`arguments` n'est pas un vrai tableau à cause d'une erreur de conception du langage. Il est dépourvu de toutes les méthodes de tableau sauf de `length`.

### Return

`return` peut être utilisé pour provoquer le retour anticipé de la fonction.
Une fonction renvoie toujours une valeur. Si elle n'est pas spécifiée, `undefined` est retournée.

## Exceptions

`throw` interrompt l'exécution. Un objet `exception`, contenant les propriétés `name` et `message`, est alors créé.
Si une exception est levée dans un bloc `try`, le contrôle passe à un unique bloc `catch` qui capture toutes les exceptions. Il faudra alors inspecter le nom de l'exception pour déterminer son type.

## Bonnes pratiques

### Commentaires

Il est préférable d'utiliser les commentaires de fin de ligne commençant par `//`. Les commentaires de bloc `/* */` ne sont pas sûrs. Par exemple, `/* var rm_a = /a*/.match(s); */` provoque une erreur de syntaxe.

### Variable globale par application

Les variables globales doivent être évitées.
L'un des moyens de minimiser leur utilisation consiste à créer une variable globale unique par application. Elle représente alors le conteneur de l'application.
Cela permettra de limiter les risques de conflits avec d'autres applications, modules ou bibliothèques. De plus, on y gagne en cohérence et lisibilité.

```js
var MYAPP = {};
MYAPP.foo = {
  bar: 42
};
```

### Tableau ou Objet ?

Javascript entretient lui-même une confusion entre les deux. `typeof` sur un tableau renvoit `object`.
Il est possible de corriger cela.

```js
var is_array = function(value) {
  return value && // permet de rejeter null et false
         typeof value === 'object' && // vrai pour les objets, les tableaux et null
         typeof value.length === 'number' && // vrai pour les tableaux et généralement faux pour les objets
         typeof value.splice === 'function' && // vrai pour les tableaux
         !(value.propertyIsEnumerable('length')); // length n'est pas énumérable (produit par une boucle for in ?) pour les tableaux
};
```

Quoi qu'il en soit, lorsque les noms de propriété sont de petits entiers séquentiels, vous devez utiliser un tableau sinon, utilisez un objet.

## Les choses pas terribles

### Variables globales

La pire de toute. Une variable globale est une variable visible dans toutes les portées. Elle peut être modifié à tout moment par n'importe quelle partie du programme. De plus, si des sous-programmes utilisent des variables globales dont le nom est identique, celles-ci risquent de s'interférer. Et malheureusement, Javascript ne fait pas que de les autoriser mais il en a besoin. Toutes les unités de compilation sont chargées dans un objet global commun.

### Portée

Javascript ne fournit pas de portée de bloc. Une variable déclarée dans un bloc est visible partout dans la fonction qui contient ce bloc. Il vaut alors mieux déclarer toutes les variables en haut de chaque fonction.

### Insertion du point-virgule

Javascript vise à corriger les programmes erronés en insérant des `;`. C'est assez horrible car il peut masquer des erreurs plus graves. Exemple pour l'instruction `return` :

```js
return
{
  status: true
};
// Ce code renvoie `undefined`
return {
  status: true
};
// Ce code renvoie `{ status: true }`
```

### Mots réservés

```js
break else new var case finally return void catch for switch
while continue function this with default if throw delete in
try do instanceof typeof abstract enum int short boolean export
interface static byte extends long super char final native
synchronized class float package throws const goto private
transient debugger implements protected volatile double import public
```

La plupart de ces mots ne sont pas utilisés dans le langage, toutefois, on ne peut y recourir pour nommer des variables ou des paramètres.

### parseInt

`parseInt` est une fonction qui convertit les chaînes en entier. Elle s'arrête lorsqu'elle voit un caractère qui n'est pas un chiffre. Du coup, `parseInt("16")` est égal à `parseInt("16plop")` sans nous alerter.
Si le premier caractère de la chaîne est "0", la chaîne est évaluée en base 8 au lieu d'être en base 10. En base 8, 8 et 9 ne sont pas des chiffres. Du coup, `parseInt("08")` et `parseInt("09")` produisent le résultat 0. Cela est gênant quand on analyse des dates et des heures. Il est recommandé de toujours utiliser le paramètre de base de `parseInt` pour éviter cette erreur. `parseInt("08", 10)` vaut alors 8.

### Virgule flottante

Les nombres binaires à virgule flottante ne sont pas capables de gérer les fractions décimales. Du coup, `0.1 + 0.2` n'est pas égal à `0.3`. Ceci est dû à l'adoption volontaire du standard IEEE 754. Cela convient à la majorité des applications mais va à l'encontre de ce que l'on a appris au collège.
Pour contourner ce problème, les valeurs monétaires peuvent être converties en valeur entières en centimes.
