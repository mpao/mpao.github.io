---
layout: default
title: Fibonacci
parent: Kotlin
nav_order: 20
---

# Fibonacci
{: .no_toc }

## Indice
{: .no_toc .text-delta }

1. TOC
{:toc}

Questi appunti derivano dall'articolo su Kousenit.org [Fibonacci in Kotlin](https://kousenit.org/2019/11/26/fibonacci-in-kotlin/), che a sua volta proviene da Twitter, ma nell'articolo originale ci sono già elencate tutte le fonti, quindi passiamo oltre.

La **successione di Fibonacci** probabilemente è la più conosciuta, ma in ogni caso ecco la definizione da [Wikipedia](https://it.wikipedia.org/wiki/Successione_di_Fibonacci)

>La **successione di Fibonacci** (detta anche **successione aurea**), indicata con ![F_{n}](https://wikimedia.org/api/rest_v1/media/math/render/svg/76cdf519c21deec43f984815e57e15d2dd3575d7) o con ![Fib(n)](https://wikimedia.org/api/rest_v1/media/math/render/svg/26e544e72ba25d203f7ceea1c5783d6a35b49311), in matematica indica una successione di numeri interi in cui ciascun numero è la somma dei due precedenti, eccetto i primi due che sono, per definizione: ![{\displaystyle F_{0}=0}](https://wikimedia.org/api/rest_v1/media/math/render/svg/58ebe8b2d5551fb272cd4258940fe1e492592d02) e ![F_{1}=1](https://wikimedia.org/api/rest_v1/media/math/render/svg/c374ba08c140de90c6cbb4c9b9fcd26e3f99ef56).

>Questa successione è definita ricorsivamente secondo la seguente regola:
>
>![ F_{0}=0](https://wikimedia.org/api/rest_v1/media/math/render/svg/50f0540e5bf18821f31581e56a08d6bb276f1041)

>![F_{1}=1](https://wikimedia.org/api/rest_v1/media/math/render/svg/1d2b4efdae465c4699bd64aaf19b99dcb26eb4a6)

>![F_{n}=F_{{n-1}}+F_{{n-2}}](https://wikimedia.org/api/rest_v1/media/math/render/svg/4fa6d281e7a54e08aeffeef7458ddc0884333686) (per ogni n>1)
>
>Gli elementi ![F_{n}](https://wikimedia.org/api/rest_v1/media/math/render/svg/76cdf519c21deec43f984815e57e15d2dd3575d7) sono anche detti numeri di Fibonacci. I primi termini della successione di Fibonacci, che prende il nome dal matematico pisano del XIII secolo Leonardo Fibonacci, sono: *0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, ...*



### Implementazione ricorsiva semplice

Stando alla definizione è facile definire la funzione di Fibonacci come 

```kotlin
fun fibonacci(n: Int): Int = when(n){
    0 -> 0
    1 -> 1
    else -> fibonacci(n-1) + fibonacci(n-2)
}
```

dove `n` rappresenta il numero di elementi della successione. È piuttosto semplice da leggere: 

> se n è 0 restituisci 0,  se n è 1 restituisci 1, altrimenti esegui ricorsivamente la somma tra n-1 e n-2

Il problema con questa implementazione sta nel fatto che i risultati intermedi verranno ricalcolati ogni volta ed il lavoro per questo calcolo aumenta in maniera esponenziale al crescere di `n`.

[Decompilando](https://medium.com/@mydogtom/tip-how-to-show-java-equivalent-for-kotlin-code-f7c81d76fa8) il bytecode si ottiene qualcosa di simile, che ci servirà a breve

```java
public static final int fibonacci(int n) {
      int var10000;
      switch(n) {
      case 0:
         var10000 = 0;
         break;
      case 1:
         var10000 = 1;
         break;
      default:
         var10000 = fib1(n - 1) + fib1(n - 2);
      }
      return var10000;
}
```



 ### Tail Call

Cominciamo dalla definizione, sempre da [Wikipedia](https://en.wikipedia.org/wiki/Tail_call)

> a **tail call** is a subroutine call performed as the final action of a procedure. If a tail call might lead to the same subroutine being called again later in the call chain, the subroutine is said to be **tail-recursive**, which is a special case of recursion.

...e Kotlin supporta questo tipo di funzione: questo permette di poter scrivere in maniera ricorsiva, algoritmi che invece andrebbero scritti usando *cicli*. Quando una funzione è marcata con la keyword `tailrec`, il compilatore ottimizzerà la ricorsione utilizzando "sotto il cofano" una versione a cicli più veloce ed efficiente come si legge nella [documentazione](https://kotlinlang.org/docs/reference/functions.html#tail-recursive-functions) a riguardo.

La funzione di Fibonacci diventa quindi

```kotlin
tailrec fun fibonacci(n: Int): Int
```

ma c'è qualcosa che **non funziona** ! Lint ci riempie di warning `a function is marked as tail-recursive but no tail calls are found`. Il codice decompilato dopo l'aggiunta della keyword `tailrec` è praticamente identico. Ma una *tail call* è per definizione "una routine eseguita come azione finale della procedura". Ma che significa ?

In una **normale ricorsione**, vengono eseguite prima tutte le chiamate ed infine calcolato il risultato finale con i valori ottenuti da ogni singola chiamata. Va da sé quindi che la funzione non sia l'ultima operazione eseguita.

**In una tail call, il risultato finale viene computato ad ogni passaggio e viene quindi passato come parametro alla prossima chiamata ricorsiva**.

Riassumendo, per ottenere i vantaggi di una tail call la funzione deve

1. chiamare sé stessa come ultima operazione ( e già era così )
2. calcolare il risultato parziale durante il processo

Occorre quindi rivedere l'implementazione della funzione `fibonacci()` per ottenere i vantaggi di una tail call. Prima di tutto ci serve un parametro aggiuntivo che faccia da **accumulatore** ( tieni bene a mente il termine, perché sarà ben chiaro nell'ultima parte di questi appunti ) in cui salvare i risultati parziali della ricorsione. Ecco come si trasforma quindi

```kotlin
@JvmOverloads
tailrec fun fibonacci(n: Int, a: Int = 0, b: Int = 1): Int = when (n) {
        0 -> a
        1 -> b
        else -> fibonacci(n - 1, b, a + b)
}
```

dove i parametri `n`, `a`, `b` rappresentano rispettivamente

* n = l'ennesimo numero di Fibonacci cercato
* b = è il numero precedente
* a = è il numero precedente a b

Nella chiamata ricorsiva, il successivo valore di `b` è `a+b` ovvero il valore precedente più il valore attuale, che di fatto definisce un **accumulatore** per definizione.

In questa nuova funzione, i parametri `a` e `b` vengono definiti con un valore di default in modo che la funzione possa essere richiamata come `fibonacci(10)` senza fornire gli inutili ( per l'utente ) altri due parametri. Java però non supporta i *default parameters*, ed ecco quindi spiegato l'utilizzo dell'annotazione `@JmvOverloads`, che ha lo scopo di far generare al compilatore il codice per fare l'overloading della funzione.

Questa volta nessun warning e, se decompilata, la funzione ha l'implementazione corretta di una tail call

```java
   public static final int fibonacci(int n, int a, int b) {
      while(true) {
         int var10000;
         switch(n) {
         case 0:
            var10000 = a;
            break;
         case 1:
            var10000 = b;
            break;
         default:
            var10000 = n - 1;
            int var10001 = b;
            b += a;
            a = var10001;
            n = var10000;
            continue;
         }
         return var10000;
      }
   }
```



### Kotlin `fold` e `reduce`

Abbiamo visto fino ad ora due implementazioni, entrambe procedurali in cui la prima implementa strettamente la definizione della sequenza di Fibonacci, e la seconda che trae vantaggio da un accumulatore. 

Ecco ora un altro tipo di approccio funzionale attraverso la funzione [fold](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html) che

> Accumulates value starting with initial value and applying operation from left to right to current accumulator value and each element.

```kotlin
fun fibonacci(n: Int) =
    if(n == 0) 0 else (2 until n).fold(1 to 1) { (prev, curr), _ -> 
        curr to (prev + curr) 
    }.second
```

Tale funzione quindi, applica l'operazione passata come lambda su ogni elemento del `Iterable` su cui viene eseguita e l'ultimo valore valutato diventa il nuovo valore dell'argomento per la successiva iterazione ( un accumulatore quindi ). La differenza tra `fold` e `reduce` è che la prima prende un valore iniziale per l'accumulatore, la seconda usa il primo elemento della sequenza per inizializzare l'accumulatore.

Per capirla meglio avere sott'occhio innanzi tutto la definizione

```kotlin
inline fun <T, R> Iterable<T>.fold(
    initial: R,
    operation: (acc: R, T) -> R
): R
```

dove R è il valore iniziale dell'accumulatore, dove viene eseguita una operazione su R e T che ha come risultato un nuovo accumulatore di tipo R. Provando a riscrivere `fibonacci()` in maniera esplicita viene fuori che

```kotlin
fun fibonacci(n: Int): Int{
    val target: Iterable<Int> = (2 until n) // Iterable<T>
    val accumulatorStartValue = Pair(1,1) // initial di tipo R
    val operationr = fun(accumulator: Pair<Int, Int>, type: Int): Pair<Int, Int>{
        return accumulator.second to (accumulator.first + accumulator.second)
    } // operation con parametri R e T che ritorna R
    val finalResult: Pair<Int, Int> = target.fold(accumulatorStartValue, operation) // R
    val fibonaccisValue = finalResult.second // T
    return fibonaccisValue
}
```

In lingua corrente, partendo da 2 fino ad n-1 esegui sul valore la funzione [fold](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html), con accumulatore iniziale un Pair(1,1) e come operazione aggiorna l'accumulatore con il valore corrente sommato al precedente e restituiscimi il valore contenuto alla fine ( in un Pair, second restituisce il valore della chiave ). Utilizzare un Pair come accumulatore, dove la key è il valore precedente e il value il valore corrente è un'idea furbissima; l'unica cosa che ancora può generare perplessita è l'`Iterable` definito come `2 until n`, che altro non è che un modo per gestire i primi due casi particolari come visto in precedenza.



### Stream

Un approccio analogo si può effettuare con gli `Stream` 

```kotlin
fun fibonacci(n: Int) = Stream.iterate(
    	Pair(1, 1), { acc: Pair<Int, Int> -> (acc.second to acc.first + acc.second) }
	)
        .mapToInt { acc: Pair<Int, Int> -> acc.first }
        .limit(n.toLong())
        .max()
        .orElse(0)

```

La funzione `iterate` prende come argomenti un *seed* ed una operazione ( analogamente a `fold` ) e dal `Pair` risultato prendo il primo parametro dove è contenuto il valore dell'accumulatore, limito lo stream a `n` elementi e prendo il valore massimo. Ma gli `Stream` sono di Java, esiste qualcosa specifico per Kotlin ?



### Sequences

Estratto dalla [documentazione](https://kotlinlang.org/docs/reference/sequences.html#sequences)

> Along with collections, the Kotlin standard library contains another container type – *sequences* ([`Sequence<T>`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/-sequence/index.html)). Sequences offer the same functions as [`Iterable`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-iterable/index.html) but implement another approach to multi-step collection processing.
>
> When the processing of an `Iterable` includes multiple steps, they are executed eagerly: each processing step completes and returns its result – an intermediate collection. The following step executes on this collection. In turn, multi-step processing of sequences is executed lazily when possible: actual computing happens only when the result of the whole processing chain is requested.
>
> The order of operations execution is different as well: `Sequence` performs all the processing steps one-by-one for every single element. In turn, `Iterable` completes each step for the whole collection and then proceeds to the next step.
>
> So, the sequences let you avoid building results of intermediate steps, therefore improving the performance of the whole collection processing chain. However, the lazy nature of sequences adds some overhead which may be significant when processing smaller collections or doing simpler computations. Hence, you should consider both `Sequence` and `Iterable` and decide which one is better for your case.

La funzione [yield](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/-sequence-scope/yield.html#yield) fornisce un valore all'Iteratore e si sospende fino a quando non viene richiesto il valore successivo. L'esempio seguente su Fibonacci arriva proprio dalla documentazione di `yield`

```kotlin
fun fibonacci() = sequence {
    var terms = Pair(0, 1)

    // this sequence is infinite
    while (true) {
        yield(terms.first)
        terms = Pair(terms.second, terms.first + terms.second)
    }
}

println(fibonacci().take(10).toList()) // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

Il vantaggio rispetto agli Stream, è che le `Sequences` sono pensate per le `Coroutines`, la funzione `sequence` crea uno scope in cui `yield`, che è una `suspend function` può operare.

Questo è un approccio completamente differente al problema, ma l'implementazione sotto si basa sempre su un `Pair`.  In alcuni testi si trovano implementazioni con `array` a 2 soli valori, approccio analogo ma forse meno chiaro ed esplicito rispetto ad utilizzare un `Pair`.
