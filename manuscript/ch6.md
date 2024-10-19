# Functional-Light JavaScript
# Capítulo 6: Imutabilidade de Valor

No [Capitulo 5](ch5.md), falamos sobre a importância de reduzir as causas/efeitos colaterais: as maneiras pela qual o estado do seu aplicativo pode mudar inesperadamente e pode mudar e causar supresas (bugs). quanto menos lugares tivermos com tais minas terrestres, mais confiança teremos em nosso código, e mais legível vai ser. Nosso tópico para este capítulo segue diretamente esse mesmo esforço.

Se a idempotência no estilo de programação trata de definir uma operação de alteração de valor de forma que ela possa afetar o estado apenas uma vez, agora vamos focar no objetivo de reduzir o número de ocorrências de mudança de um para zero.

Agora vamos explorar a imutabilidade de valores, a noção de que em nossos programas utilizamos apenas valores que não podem ser alterados.

## Imutabilidade de Tipos Primitivos

Valores dos tipos primitivos (`number`, `string`, `boolean`, `null` e `undefined`) já são imutáveis; não há nada que você possa fazer para alterá-los:

```js
// inválido, e também não faz sentido
2 = 2.5;
```

Entretanto, O JS tem um comportamento peculiar que parece permitir a mutação desses valores de tipo primitivo: "boxing". Quando você acessa uma propriedade em certos tipos de valores primitivos -- especificamente `number`, `string`, e `boolean` -- O JS por baixo dos panos automaticamente os envolve (também conhecido como "boxes") em seus equivalentes de objeto (`Number`, `String`, e `Boolean`, respectivamente).

Considere:

```js
var x = 2;

x.length = 4;

x;              // 2
x.length;       // undefined
```

Números normalmente não tem uma propriedade `length` disponível, então a configuração `x.length = 4` tenta adicionar uma nova propriedade, mas falha silenciosamente (ou é ignorada/descartada, dependendo do seu ponto de vista); `x` continua a armazenar o simples número primitivo `2`.

Mas o fato de o JS permitir que a instrução `x.length = 4` seja executada pode parecer preocupante, se não por outro motivo além da confusão que pode causar nos leitores. A boa notícia é, se você usar o modo estrito (`"use strict";`), tal instrução lançará um erro.

E se você tentar alterar explicitamente a representação do objeto encapsulado como um valor?

```js
var x = new Number( 2 );

// Funciona sem problemas
x.length = 4;
```

`x` neste trecho contém uma referência a um objeto, então propriedades customizadas podem ser adicionadas e modificadas sem problemas

A imutabilidade de simples tipos primitivos como `number` provavelmente parece muito óbvio. Mas e os valores `string`? Desenvolvedores JS têm um equívoco muito comum de que strings são como arrays e que isso pode ser modificado. A sintaxe do JS até sugere que eles sejam "semelhantes a um array" com o operador de acesso `[]`. No entanto, strings também são imutáveis:

```js
var s = "hello";

s[1];               // "e"

s[1] = "E";
s.length = 10;

s;                  // "hello"
```

Apesar de poder acessar `s[1]` como se fosse um array, Strings no JS não são arrays reais. Fazendo `s[1] = "E"` e `s.length = 10` os dois falham, assim como `x.length = 4` fez antes. Em modo estrito, Essas atribuições vão falhar, porque a propriedade `1` e a propriedade `length` é somente lida no valor primitivo `string`.

Curiosamente, até mesmo o valor do objeto `String` encapsulada agirá(principlamente) imutável, pois vai gerar erros no modo estrito se vocêalterar as propriedades existentes:

```js
"use strict";

var s = new String( "hello" );

s[1] = "E";         // erro
s.length = 10;      // erro

s[42] = "?";        // OK

s;                  // "hello"
```

## Valor a Valor
Detalharemos essa ideia ao longo do capítulo, mas apenas para começar com um entendimento claro em mente: a imutabilidade dos valores não significa que não possamos permitir que os valores mudem ao longo do nosso programa. Um programa sem mudança de estado não é muito interessante! Também não significa que nossas variáveis ​​não possam conter valores diferentes. Todos esses são equívocos comuns sobre a imutabilidade do valor.

Valores imutáveis significa que *quando* nos precisamos mudar o estado em nosso programa, devemos criar e rastrear um novo valor ao invés de alterar um valor existente.

Por exemplo:

```js
function adicionaValor(arr) {
    var novoArray = [ ...arr, 4 ];
    return novoArray;
}

adicionaValor( [1,2,3] );    // [1,2,3,4]
```

Observe que não mudamos o array `arr` que faz referência, mas em vez disso criei um novo array (`novoArray`) que contém os valores existentes mais o novo valor `4`.

Analise `adicionaValor(..)` com base no que discutimos no [Capítulo 5](ch5.md) sobre causas/efeitos colaterias. É puro? Possui transparência referencial? Dado o mesmo array, ela sempre produzirá o mesmo output? Está livre de causas e efeitos colaterais? **Yes.**

Imagine que o array `[1,2,3]` representa uma sequência de dados de algumas operações anteriores e armazenamos em alguma variável. Esse é o nosso estado atual. Se quisermos programar qual é o próximo estado da nossa aplicação, chamamos `adicionaValor(..)`. Mas se quisermos que esse ato do próximo estado seja direto e explícito. Portanto, a operação `adicionaValor(..)` recebe um input direto, retorna um output direto, e evita o efeito colateral de alterar o array original que `arr` faz referência.

Isso significa que podemos calcular o novo estado de `[1,2,3,4]` e estar totalmente no controle dessa transição de estados. Nenhuma outra parte do nosso programa pode nos trazer uma transição inesperada para esse estado antecipadamente, ou para outro estado inteiramente, como `[1,2,3,5]`. Ao sermos disciplinados em relação aos nossos valores e tratá-los como imutáveis, reduzimos dastricamente a superfície, tornando os nossos programas mais fáceis de ler, raciocionar e, em última analise, confiável.

O array `arr` que faz referência é atualmente mutável. Escolhemos não alterar isso, então praticamos o espirito do valor imutável.

Podemos usar a estratégia copy-instead-of-mutate para objetos, também. 

Considere:

```js
function updateLastLogin(user) {
    var newUserRecord = Object.assign( {}, user );
    newUserRecord.lastLogin = Date.now();
    return newUserRecord;
}

var user = {
    // ..
};

user = updateLastLogin( user );
```

### Não local

Non-primitive values are held by reference, and when passed as arguments, it's the reference that's copied, not the value itself.

If you have an object or array in one part of the program, and pass it to a function that resides in another part of the program, that function can now affect the value via this reference copy, mutating it in possibly unexpected ways.

In other words, if passed as arguments, non-primitive values become non-local. Potentially the entire program has to be considered to understand whether such a value will be changed or not.

Consider:

```js
var arr = [1,2,3];

foo( arr );

console.log( arr[0] );
```

Ostensibly, you're expecting `arr[0]` to still be the value `1`. But is it? You don't know, because `foo(..)` *might* mutate the array using the reference copy you pass to it.

We already saw a trick in the previous chapter to avoid such a surprise:

```js
var arr = [1,2,3];

foo( [...arr] );         // ha! a copy!

console.log( arr[0] );      // 1
```

In a little bit, we'll see another strategy for protecting ourselves from a value being mutated out from underneath us unexpectedly.

## Reassignment

How would you describe what a "constant" is? Think about that for a moment before you move on to the next paragraph.

<p align="center">
    * * * *
</p>

Some of you may have conjured descriptions like, "a value that can't change", "a variable that can't be changed", or something similar. These are all approximately in the neighborhood, but not quite at the right house. The precise definition we should use for a constant is: a variable that cannot be reassigned.

This nitpicking is really important, because it clarifies that a constant actually has nothing to do with the value, except to say that whatever value a constant holds, that variable cannot be reassigned any other value. But it says nothing about the nature of the value itself.

Consider:

```js
var x = 2;
```

Like we discussed earlier, the value `2` is an unchangeable (immutable) primitive. If I change that code to:

```js
const x = 2;
```

The presence of the `const` keyword, known familiarly as a "constant declaration", actually does nothing at all to change the nature of `2`; it's already unchangeable, and it always will be.

It's true that this later line will fail with an error:

```js
// try to change `x`, fingers crossed!
x = 3;      // Error!
```

But again, we're not changing anything about the value. We're attempting to reassign the variable `x`. The values involved are almost incidental.

To prove that `const` has nothing to do with the nature of the value, consider:

```js
const x = [ 2 ];
```

Is the array a constant? **No.** `x` is a constant because it cannot be reassigned. But this later line is totally OK:

```js
x[0] = 3;
```

Why? Because the array is still totally mutable, even though `x` is a constant.

The confusion around `const` and "constant" only dealing with assignments and not value semantics is a long and dirty story. It seems a high degree of developers in just about every language that has a `const` stumble over the same sorts of confusions. Java in fact deprecated `const` and introduced a new keyword `final` at least in part to separate itself from the confusion over "constant" semantics.

Setting aside the confusion detractions, what importance does `const` hold for the FPer, if not to have anything to do with creating an immutable value?

### Intent

The use of `const` tells the reader of your code that *that* variable will not be reassigned. As a signal of intent, `const` is often highly lauded as a welcome addition to JavaScript and a universal improvement in code readability.

In my opinion, this is mostly hype; there's not much substance to these claims. I see only the mildest of faint benefit in signaling your intent in this way. And when you match that up against decades of precedent around confusion about it implying value immutability, I don't think `const` comes close to carrying its own weight.

To back up my assertion, let's consider scope. `const` creates a block scoped variable, meaning that variable only exists in that one localized block:

```js
// lots of code

{
    const x = 2;

    // a few lines of code
}

// lots of code
```

Typically, blocks are considered best designed to be only a few lines long. If you have blocks of more than say 10 lines, most developers will advise you to refactor. So `const x = 2` only applies to those next nine lines of code at most.

No other part of the program can ever affect the assignment of `x`. **Period.**

My claim is that program has basically the same magnitude of readability as this one:

```js
// lots of code

{
    let x = 2;

    // a few lines of code
}

// lots of code
```

If you look at the next few lines of code after `let x = 2;`, you'll be able to easily tell that `x` is in fact *not* reassigned. That to me is a **much stronger signal** -- actually not reassigning it! -- than the use of some confusable `const` declaration to say "won't reassign it".

Moreover, let's consider what this code is likely to communicate to a reader at first glance:

```js
const magicNums = [1,2,3,4];
```

Isn't it at least possible (probable?) that the reader of your code will assume (wrongly) that your intent is to never mutate the array? That seems like a reasonable inference to me. Imagine their confusion if later you do in fact allow the array value referenced by `magicNums` to be mutated. Might that surprise them?

Worse, what if you intentionally mutate `magicNums` in some way that turns out to not be obvious to the reader? Subsequently in the code, they see a usage of `magicNums` and assume (again, wrongly) that it's still `[1,2,3,4]` because they read your intent as, "not gonna change this".

I think you should use `var` or `let` for declaring variables to hold values that you intend to mutate. I think that actually is a **much clearer signal** of your intent than using `const`.

But the troubles with `const` don't stop there. Remember we asserted at the top of the chapter that to treat values as immutable means that when our state needs to change, we have to create a new value instead of mutating it? What are you going to do with that new array once you've created it? If you declared your reference to it using `const`, you can't reassign it.

```js
const magicNums = [1,2,3,4];

// later:
magicNums = magicNums.concat( 42 );  // oops, can't reassign!
```

So... what next?

In this light, I see `const` as actually making our efforts to adhere to FP harder, not easier. My conclusion: `const` is not all that useful. It creates unnecessary confusion and restricts us in inconvenient ways. I only use `const` for simple constants like:

```js
const PI = 3.141592;
```

The value `3.141592` is already immutable, and I'm clearly signaling, "this `PI` will always be used as stand-in placeholder for this literal value." To me, that's what `const` is good for. And to be frank, I don't use many of those kinds of declarations in my typical coding.

I've written and seen a lot of JavaScript, and I just think it's an imagined problem that very many of our bugs come from accidental reassignment.

One of the reasons FPers so highly favor `const` and avoid reassignment is because of equational reasoning. Though this topic is more related to other languages than JS and goes beyond what we'll get into here, it is a valid point. However, I prefer the pragmatic view over the more academic one.

For example, I've found measured use of variable reassignment can be useful in simplifying the description of intermediate states of computation. When a value goes through multiple type coercions or other transformations, I don't generally want to come up with new variable names for each representation:

```js
var a = "420";

// later

a = Number( a );

// later

a = [ a ];
```

If after changing from `"420"` to `420`, the original `"420"` value is no longer needed, then I think it's more readable to reassign `a` rather than come up with a new variable name like `aNum`.

The thing we really should worry more about is not whether our variables get reassigned, but **whether our values get mutated**. Why? Because values are portable; lexical assignments are not. You can pass an array to a function, and it can be changed without you realizing it. But a reassignment will never be unexpectedly caused by some other part of your program.

### It's Freezing in Here

There's a cheap and simple way to turn a mutable object/array/function into an "immutable value" (of sorts):

```js
var x = Object.freeze( [2] );
```

The `Object.freeze(..)` utility goes through all the properties/indices of an object/array and marks them as read-only, so they cannot be reassigned. It's sorta like declaring properties with a `const`, actually! `Object.freeze(..)` also marks the properties as non-reconfigurable, and it marks the object/array itself as non-extensible (no new properties can be added). In effect, it makes the top level of the object immutable.

Top level only, though. Be careful!

```js
var x = Object.freeze( [ 2, 3, [4, 5] ] );

// not allowed:
x[0] = 42;

// oops, still allowed:
x[2][0] = 42;
```

`Object.freeze(..)` provides shallow, naive immutability. You'll have to walk the entire object/array structure manually and apply `Object.freeze(..)` to each sub-object/array if you want a deeply immutable value.

But contrasted with `const` which can confuse you into thinking you're getting an immutable value when you aren't, `Object.freeze(..)` *actually* gives you an immutable value.

Recall the protection example from earlier:

```js
var arr = Object.freeze( [1,2,3] );

foo( arr );

console.log( arr[0] );          // 1
```

Now `arr[0]` is quite reliably `1`.

This is so important because it makes reasoning about our code much easier when we know we can trust that a value doesn't change when passed somewhere that we do not see or control.

## Performance

Whenever we start creating new values (arrays, objects, etc.) instead of mutating existing ones, the obvious next question is: what does that mean for performance?

If we have to reallocate a new array each time we need to add to it, that's not only churning CPU time and consuming extra memory; the old values (if no longer referenced) are also being garbage collected. That's even more CPU burn.

Is that an acceptable trade-off? It depends. No discussion or optimization of code performance should happen **without context.**

If you have a single state change that happens once (or even a couple of times) in the whole life of the program, throwing away an old array/object for a new one is almost certainly not a concern. The churn we're talking about will be so small -- probably mere microseconds at most -- as to have no practical effect on the performance of your application. Compared to the minutes or hours you will save not having to track down and fix a bug related to unexpected value mutation, there's not even a contest here.

Then again, if such an operation is going to occur frequently, or specifically happen in a *critical path* of your application, then performance -- consider both performance and memory! -- is a totally valid concern.

Think about a specialized data structure that's like an array, but that you want to be able to make changes to and have each change behave implicitly as if the result was a new array. How could you accomplish this without actually creating a new array each time? Such a special array data structure could store the original value and then track each change made as a delta from the previous version.

Internally, it might be like a linked-list tree of object references where each node in the tree represents a mutation of the original value. Actually, this is conceptually similar to how **Git** version control works.

<p align="center">
    <img src="images/fig18.png" width="33%">
</p>

In this conceptual illustration, an original array `[3,6,1,0]` first has the mutation of value `4` assigned to position `0` (resulting in `[4,6,1,0]`), then `1` is assigned to position `3` (now `[4,6,1,1]`), finally `2` is assigned to position `4` (result: `[4,6,1,1,2]`). The key idea is that at each mutation, only the change from the previous version is recorded, not a duplication of the entire original data structure. This approach is much more efficient in both memory and CPU performance, in general.

Imagine using this hypothetical specialized array data structure like this:

```js
var state = specialArray( 4, 6, 1, 1 );

var newState = state.set( 4, 2 );

state === newState;                 // false

state.get( 2 );                     // 1
state.get( 4 );                     // undefined

newState.get( 2 );                  // 1
newState.get( 4 );                  // 2

newState.slice( 2, 5 );             // [1,1,2]
```

The `specialArray(..)` data structure would internally keep track of each mutation operation (like `set(..)`) as a *diff*, so it won't have to reallocate memory for the original values (`4`, `6`, `1`, and `1`) just to add the `2` value to the end of the list. But importantly, `state` and `newState` point at different versions (or views) of the array value, so **the value immutability semantic is preserved.**

Inventing your own performance-optimized data structures is an interesting challenge. But pragmatically, you should probably use a library that already does this well. One great option is [Immutable.js](http://facebook.github.io/immutable-js), which provides a variety of data structures, including `List` (like array) and `Map` (like object).

Consider the previous `specialArray` example but using `Immutable.List`:

```js
var state = Immutable.List.of( 4, 6, 1, 1 );

var newState = state.set( 4, 2 );

state === newState;                 // false

state.get( 2 );                     // 1
state.get( 4 );                     // undefined

newState.get( 2 );                  // 1
newState.get( 4 );                  // 2

newState.toArray().slice( 2, 5 );   // [1,1,2]
```

A powerful library like Immutable.js employs sophisticated performance optimizations. Handling all the details and corner-cases manually without such a library would be quite difficult.

When changes to a value are few or infrequent and performance is less of a concern, I'd recommend the lighter-weight solution, sticking with built-in `Object.freeze(..)` as discussed earlier.

## Treatment

What if we receive a value to our function and we're not sure if it's mutable or immutable? Is it ever OK to just go ahead and try to mutate it? **No.** As we asserted at the beginning of this chapter, we should treat all received values as immutable -- to avoid side effects and remain pure -- regardless of whether they are or not.

Recall this example from earlier:

```js
function updateLastLogin(user) {
    var newUserRecord = Object.assign( {}, user );
    newUserRecord.lastLogin = Date.now();
    return newUserRecord;
}
```

This implementation treats `user` as a value that should not be mutated; whether it *is* immutable or not is irrelevant to reading this part of the code. Contrast that with this implementation:

```js
function updateLastLogin(user) {
    user.lastLogin = Date.now();
    return user;
}
```

That version is a lot easier to write, and even performs better. But not only does this approach make `updateLastLogin(..)` impure, it also mutates a value in a way that makes both the reading of this code, as well as the places it's used, more complicated.

**We should treat `user` as immutable**, always, because at this point of reading the code we do not know where the value comes from, or what potential issues we may cause if we mutate it.

Nice examples of this approach can be seen in various built-in methods of the JS array, such as `concat(..)` and `slice(..)`:

```js
var arr = [1,2,3,4,5];

var arr2 = arr.concat( 6 );

arr;                    // [1,2,3,4,5]
arr2;                   // [1,2,3,4,5,6]

var arr3 = arr2.slice( 1 );

arr2;                   // [1,2,3,4,5,6]
arr3;                   // [2,3,4,5,6]
```

Other array prototype methods that treat the value instance as immutable and return a new array instead of mutating: `map(..)` and `filter(..)`. The `reduce(..)`/`reduceRight(..)` utilities also avoid mutating the instance, though they don't by default return a new array.

Unfortunately, for historical reasons, quite a few other array methods are impure mutators of their instance: `splice(..)`, `pop(..)`, `push(..)`, `shift(..)`, `unshift(..)`, `reverse(..)`, `sort(..)`, and `fill(..)`.

It should not be seen as *forbidden* to use these kinds of utilities, as some claim. For reasons such as performance optimization, sometimes you will want to use them. But you should never use such a method on an array value that is not already local to the function you're working in, to avoid creating a side effect on some other remote part of the code.

<a name="hiddenmutation"></a>

Recall one of the implementations of [`compose(..)` from Chapter 4](ch4.md/#user-content-generalcompose):

```js
function compose(...fns) {
    return function composed(result){
        // copy the array of functions
        var list = [...fns];

        while (list.length > 0) {
            // take the last function off the end of the list
            // and execute it
            result = list.pop()( result );
        }

        return result;
    };
}
```

The `...fns` gather parameter is making a new local array from the passed-in arguments, so it's not an array that we could create an outside side effect on. It would be reasonable then to assume that it's safe for us to mutate it locally. But the subtle gotcha here is that the inner `composed(..)` which closes over `fns` is not "local" in this sense.

Consider this different version which doesn't make a copy:

```js
function compose(...fns) {
    return function composed(result){
        while (fns.length > 0) {
            // take the last function off the end of the list
            // and execute it
            result = fns.pop()( result );
        }

        return result;
    };
}

var f = compose( x => x / 3, x => x + 1, x => x * 2 );

f( 4 );     // 3

f( 4 );     // 4 <-- uh oh!
```

The second usage of `f(..)` here wasn't correct, since we mutated that `fns` during the first call, which affected any subsequent uses. Depending on the circumstances, making a copy of an array like `list = [...fns]` may or may not be necessary. But I think it's safest to assume you need it -- even if only for readability sake! -- unless you can prove you don't, rather than the other way around.

Be disciplined and always treat *received values* as immutable, whether they are or not. That effort will improve the readability and trustability of your code.

## Summary

Value immutability is not about unchanging values. It's about creating and tracking new values as the state of the program changes, rather than mutating existing values. This approach leads to more confidence in reading the code, because we limit the places where our state can change in ways we don't readily see or expect.

`const` declarations (constants) are commonly mistaken for their ability to signal intent and enforce immutability. In reality, `const` has basically nothing to do with value immutability, and its usage will likely create more confusion than it solves. Instead, `Object.freeze(..)` provides a nice built-in way of setting shallow value immutability on an array or object. In many cases, this will be sufficient.

For performance-sensitive parts of the program, or in cases where changes happen frequently, creating a new array or object (especially if it contains lots of data) is undesirable, for both processing and memory concerns. In these cases, using immutable data structures from a library like **Immutable.js** is probably the best idea.

The importance of value immutability on code readability is less in the inability to change a value, and more in the discipline to treat a value as immutable.
