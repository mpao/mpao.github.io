---
layout: default
title: Kotlin
nav_order: 2
---

# Coroutines
{: .no_toc }

Questi sono appunti personali presi dai numerosi articoli che si trovano in rete. Tento di riassumere e mettere in ordine alcuni concetti partendo dagli articoli più autorevoli:

- [Coroutines in Kotlin 1.3 explained](https://antonioleiva.com/coroutines/)
- [Kotlin Coroutines patterns & anti-patterns](https://proandroiddev.com/kotlin-coroutines-patterns-anti-patterns-f9d12984c68e)
- [Kotlin Coroutines](<https://kotlinlang.org/docs/reference/coroutines/coroutines-guide.html>)
- [Coroutines Official docs](<https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/index.html>)
- [Coroutines Repository](https://github.com/Kotlin/kotlinx.coroutines)

Vediamo che ne viene fuori...


## Indice
{: .no_toc .text-delta }

1. TOC
{:toc}



## 1. La basi

Le coroutines non sono parte del linguaggio e nemmeno della libreria standard; fanno parte di una libreria separata che contiene tutto quel che serve.

```groovy
dependencies {
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.3.0-M1'
}
```

e se l'applicazione è per Android

```groovy
dependencies {
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.3.0-M1'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.3.0-M1'
}
```

### 1.1 Il problema

Si può evitare di usare le coroutines ? Sì. E allora perchè esistono e le usiamo ? 

Le coroutines nascono per semplificare il lavoro e la lettura del codice nella gestione dei threads e dei tasks asincroni. Il paradigma classico per la gestione di threads si basa sull'esecuzione di una operazione il cui risultato deve essere ritornato su un thread differente attraverso una `callback`. Oltre al maggior codice da scrivere, la minor chiarezza, le callbacks soffrono di quello che prende il nome di `callbacks hell`, ovvero quando viene lanciato un nuovo thread all'interno di una callback, che necessita quindi di una nuova callback e così via, fino a rendere il codice arduo da leggere e comprendere. Se non hai capito quello che sto descrivendo, è esattamente quello che succede in un callback hell :-)

E le coroutines ? Le coroutines permettono di scrivere codice asincrono, usando uno **stile sincrono**, in altre parole il valore di ritorno di una funzione è il risultato di una chiamata asincrona.

```kotlin
fun main() {
    GlobalScope.launch { // launch a new coroutine in background and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello,") // main thread continues while coroutine is delayed
    Thread.sleep(2000L) // block main thread for 2s to keep JVM alive until delay() ends
}
```

Oltre alla qualità del codice, le coroutines portano con sé un altro grosso vantaggio: sono **molto più efficienti** dei normali threads poichè più di una coroutine può girare sullo stesso thread o usare un pool per ottimizzare l'esecuzione.

Lo stesso codice può essere riscritto utilizzando solo coroutines, senza l'utilizzo di `Thread` utilizzando la **suspend function** `delay()`
all'interno il costruttore di coroutine `runBlocking` 

```kotlin
fun main() { 
    GlobalScope.launch { // launch a new coroutine in background and continue
        delay(1000L)
        println("World!")
    }
    println("Hello,") // main thread continues here immediately
    runBlocking {     // but this expression blocks the main thread
        delay(2000L)  // ... while we delay for 2 seconds to keep JVM alive
    } 
}
```

o riscritta in maniera più idiomatica

```kotlin
fun main() = runBlocking<Unit> { // start main coroutine
    GlobalScope.launch { // launch a new coroutine in background and continue
        delay(1000L)
        println("World!")
    }
    println("Hello,") // main coroutine continues here immediately
    delay(2000L)      // delaying for 2 seconds to keep JVM alive
}
```

### 1.2 Job

Ritardare il flusso con `delay` non è un valido approccio da utilizzare in produzione, vale solo per dei semplici test dimostrativi. Il modo corretto è attendere, in maniera non bloccante, che il task in background abbia completato l'esecuzione e per farlo si utilizza [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html).

```kotlin
val job = GlobalScope.launch { // launch a new coroutine and keep a reference to its Job
    delay(1000L) // simulate long time task
    println("World!")
}
println("Hello,")
job.join() // wait until child coroutine completes
```

*un Job è un task eseguito in background con un proprio ciclo di vita che si conclude con la completa esecuzione o se viene interrotto esplicitamente*

La [documentazione](<https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html>) a riguardo è davvero esaustiva.

### 1.3 GlobalScope ? Nope

[Smetti di usarlo](<https://proandroiddev.com/kotlin-coroutines-patterns-anti-patterns-f9d12984c68e#4fb8>). Global scope è usato per la durata di vita dell´intera applicazione ed i task eseguiti non possono essere abortiti. Invece di utilizzare `GlobalScope` per avviare delle coroutines, si possono creare degli specifici `scope` per le operazioni. Negli esempi precedenti, ho utilizzato `runBlocking` come costruttore di una coroutine, ma **ogni costruttore crea una istanza di `CoroutineScope` per il blocco di codice racchiuso in esso. **

All'interno di questo *scope* posso evitare `join` perché la coroutine esterna **non si concluderà fino a che tutte le coroutines interne non saranno terminate**: quindi il flusso per il codice che segue è il seguente

- parte *runBlocking*
- parte *launch*
- scrivo *Hello*
- termina *launch*
- termina *runBlocking* con il risultato "Hello World"

```kotlin
fun main() = runBlocking { // this: CoroutineScope
    launch { // launch a new coroutine in the scope of runBlocking
        delay(1000L) // simulate long time task
        println("World!")
    }
    println("Hello,")
}
```



### 1.4 Scope Builder

Come detto precedentemente, ogni costruttore crea un `CoroutineScope`, ma è anche possibile dichiarare un proprio scope usando il costruttore `coroutineScope`, che a differenza di `runBlocking` **, non blocca il thread corrente in attesa dei propri figli**. Nell'esempio, *runBlocking* lancia #2 poi lo *scope* con #1 e #3 ma aspetta che tale scope abbia concluso il suo ciclo per lanciare #4. *coroutineScope* al contrario, dopo aver lanciato #1, non blocca l'esecuzione di #2, pertanto l'output sarà 1,2,3,4 dovuto a *delay* usato come simulatore di impiego del tempo per i tasks

```kotlin
fun main() = runBlocking { // this: CoroutineScope
    launch { 
        delay(1200L)
        println("2")
    }
    coroutineScope { // Creates a coroutine scope
        launch {
            delay(1500L) 
            println("3")
        }
        delay(1000L)
        // This line will be printed before the nested launch
        println("1") 
    }
    // This line is not printed until the nested launch completes
    println("4") 
}
```





## 2. Suspending functions

Prendiamo ora il blocco di codice all'interno di *launch{ }* per creare una funzione a parte. 

```kotlin
suspend fun doWork(time: Long = 0L, message: String){
    delay(time)
    println(message)
}
```

Nulla di strano se non per la keyword `suspend`. Suspend è un modificatore che rende la funzione utilizzabile solo all'interno di una coroutine o all'interno di un'altra `suspend function`. Una *suspend function* è una funzione con anteposta la keyword *suspend* che ha l'abilità di bloccare l'esecuzione di una coroutine; una volta terminata ritorna il proprio valore che può essere utilizzato all'interno della coroutine. `delay`, ad esempio, è una suspend function.

### 2.1 Funzioni composte

Supponiamo di dover compiere due lunghe operazioni prima di poter calcolare il risultato finale di una operazione

```kotlin
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

- `Uso sequenziale`: eseguo la prima funzione, ne aspetto il valore di ritorno per prendere decisioni o altro e successivamente lancio la seconda funzione

  ```kotlin
  val time = measureTimeMillis {
      val one = doSomethingUsefulOne()
      val two = doSomethingUsefulTwo()
      println("The answer is ${one + two}")
  }
  println("Completed in $time ms")
  ```

  

- `Uso concorrente`: non mi interessa la sequenzialità delle operazioni, voglio solo andare più veloce per finire prima il lavoro

  ```kotlin
  val time = measureTimeMillis {
      val one = async { doSomethingUsefulOne() }
      val two = async { doSomethingUsefulTwo() }
      println("The answer is ${one.await() + two.await()}")
  }
  println("Completed in $time ms")
  ```

### 2.2 async

[async](<https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html>) permette di svolgere compiti in maniera parallela quindi. Ma che cos'è ? Concettualmente `async` è proprio come `launch`. Fa partire una coroutine che lavora in maniera concorrente con le altre coroutines. La differenza meglio sottolinearla

> - **launch** ritorna un [Job](<https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html>), che non porta con se nessun valore di ritorno
> - **async** ritorna un [Deferred](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html), un *future* non bloccante che rappresenta la promessa di fornire un valore in là nel tempo. È possibile utilizzare il metodo `await()` su un `deferred` per ottenere tale valore. *Deferred* è sottoclasse di *Job*, quindi può essere cancellato e fermato se necessario.

L'esecuzione di *async* è doppiamente più veloce poichè i due tasks vengono eseguiti simultaneamente utilizzando quindi la metà del tempo.

Async ha un'opzione `lazy` in cui il processo può partire solo quando è necessario il suo valore di ritorno o quando viene fatto esplicitamente partire con `start()`. Se ometto *start* il task diventa **sequenziale** in quanto *await()* fa partire la coroutine e ne attende il risultato

```kotlin
fun main() = runBlocking { // this: CoroutineScope
    val time = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        one.start()
        //two.start()
        println("The answer is ${one.await()} e ${two.await()}")
    }
    println("Completed in $time ms")
}
```

### 2.3 Concorrenza strutturata

Dall'esempio precedente estraiamo la funzione

```kotlin
suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}
```

In questo modo, se qualcosa va storto all'interno della funzione, tutte le coroutines lanciate in quello *scope* vengono cancellate, mantenendo la concorrenza. Questo è il modo corretto di gestire le coroutines concorrenti e le eventuali eccezioni, che verranno propagate su tutta la gerarchia di coroutines

```kotlin
fun main() = runBlocking<Unit> {
    try {
        failedConcurrentSum()
    } catch(e: ArithmeticException) {
        println("Computation failed with ArithmeticException")
    }
}

suspend fun failedConcurrentSum(): Int = coroutineScope {
    val one = async<Int> { 
        try {
            delay(Long.MAX_VALUE) // Emulates very long computation
            42 // this will never be compute
        } finally {
            println("First child was cancelled")
        }
    }
    val two = async<Int> { 
        println("Second child throws an exception")
        throw ArithmeticException()
    }
    one.await() + two.await()
}
```





## 3. Interruzioni e timeout

Torniamo all'esecuzione sincrona per un momento.

### 3.1 Cancellare l'esecuzione di una coroutine

Se un `job` sta impiegando troppo per essere completato, o non abbiamo più bisogno che venga a termine è possibile fermare il task e lavorare con gli eventuali risultati parziali. Si utilizza una combo dei metodi `cancel()` e `join()` di *Job* o eventualmente la sincrasi `cancelAndJoin()`

```kotlin
val job = launch {
    repeat(1000) { i ->
            println("job: I'm sleeping $i ...")
        delay(500L)
    }
}
delay(1300L) // delay a bit
println("main: I'm tired of waiting!")
job.cancel() // cancels the job
job.join() // waits for job's completion 
println("main: Now I can quit.")
```

Tutte le funzioni contenute in `kotlinx.coroutines` sono cancellabili e se fermate, sollevano una `CancellationException` che deve essere intercettata. Attenzione però: se per esempio, una coroutine sta eseguendo un loop senza controllo sulla cancellazione, verrà portata ugualmente a termine!

```kotlin
val startTime = System.currentTimeMillis()
val job = launch(Dispatchers.Default) {
    var nextPrintTime = startTime
    var i = 0
    //while (i < 5) { // computation loop, just wastes CPU
    while (isActive) { // cancellable computation loop
        // print a message twice a second
        if (System.currentTimeMillis() >= nextPrintTime) {
            println("job: I'm sleeping ${i++} ...")
            nextPrintTime += 500L
        }
    }
}
delay(1300L) // delay a bit
println("main: I'm tired of waiting!")
job.cancelAndJoin() // cancels the job and waits for its completion
println("main: Now I can quit.")
```

### 3.2 Timeout

Invece di fare tutto a manina, esiste già una funzione atta allo scopo: `withTimeout()` che solleva una `TimeoutCancellationException` allo scadere del tempo, cancellando l'esecuzione della coroutine.

```kotlin
withTimeout(1300L) {
    repeat(1000) { i ->
            println("I'm sleeping $i ...")
        delay(500L)
    }
}
```

La funzione `withTimeoutOrNull` invece, ritorna `null` invece che sollevare l'eccezione.

### 3.3 Gestire le eccezioni

La [CancellationException](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-cancellation-exception/index.html) vista prima è un caso particolare poichè viene ignorata dal meccanismo delle coroutines in quanto è una legittima conclusione del processo. Ma gli altri casi ?

Sostanzialmente si dividono in due tipologie, a seconda del *builder*:

- `launch` e `actor` propagano automaticamente l'eccezione
- `async` e `produce` espongono all'utente l'eccezione, lasciandone il compito di intercettarla

```kotlin
fun main() = runBlocking {
    val job = GlobalScope.launch {
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException() // Will be printed to the console by Thread.defaultUncaughtExceptionHandler
    }
    job.join()
    println("Joined failed job")
    val deferred = GlobalScope.async {
        println("Throwing exception from async")
        throw ArithmeticException() // Nothing is printed, relying on user to call await
    }
    try {
        deferred.await()
        println("Unreached")
    } catch (e: ArithmeticException) {
        println("Caught ArithmeticException")
    }
}
```





## 4. Context e Dispatchers

Il `CoroutineContext`, definito nella libreria standard di Kotlin è il contesto in cui vengono eseguite le coroutines. Tale contesto è formato da vari elementi, tra cui i principali sono i `Job` e i `Dispatchers`

### 4.1 Dispatchers e threads

Dispatchers credo sia l'ennesima parola anglofona intraducibile, ma il concetto che ci sta dietro è quello *di colui che evade la pratica*, *l'esecutore del compito*, un "disbrigatore" insomma.

Il `dispatcher` è quell'elemento del `context` che determina su quale thread la coroutine deve essere eseguita. Tutti i `builders` accettano opzionalmente un parametro [CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) che può essere usato per specificare esplicitamente il tipo di *dispatcher* per la coroutine.

|                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [Default](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html) | `val Default: CoroutineDispatcher`The default [CoroutineDispatcher](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html) that is used by all standard builders like [launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html), [async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html), etc if neither a dispatcher nor any other [ContinuationInterceptor](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation-interceptor/index.html) is specified in their context. |
| [IO](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-i-o.html) | `val IO: CoroutineDispatcher`The [CoroutineDispatcher](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html) that is designed for offloading blocking IO tasks to a shared pool of threads. |
| [Main](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-main.html) | `val Main: MainCoroutineDispatcher`A coroutine dispatcher that is confined to the Main thread operating with UI objects. Usually such dispatchers are single-threaded. |
| [Unconfined](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html) | `val Unconfined: CoroutineDispatcher`A coroutine dispatcher that is not confined to any specific thread. It executes the initial continuation of a coroutine in the current call-frame and lets the coroutine resume in whatever thread that is used by the corresponding suspending function, without mandating any specific threading policy. Nested coroutines launched in this dispatcher form an event-loop to avoid stack overflows. |

Quando ad esempio si utilizza `launch` all'interno di `runBlocking`, questo eredita il contesto - e quindi il dispatcher - dal `CoroutineScope` da cui è stato lanciato, quindi dalla coroutine *runBlocking* che gira sul `main` thread.

### 4.2 Context





## Appendice: recap e terminologia

- `suspend function`: are functions that can stop the execution of a coroutine at any point and then get the control back to the coroutine once the result is ready and the function has finished doing its work.
- [`GlobalScope`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html): A global [CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html) not bound to any job. Global scope is used to launch top-level coroutines which are operating on the whole application lifetime and are not cancelled prematurely. Another use of the global scope is operators running in [Dispatchers.Unconfined](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html), which don’t have any job associated with them.
- [`CoroutineScope`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html): defines a scope for new coroutines. Every coroutine builder is an extension on CoroutineScope and inherits its [coroutineContext](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/coroutine-context.html) to automatically propagate both context elements and cancellation.
- [`async`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html), [`launch`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html), e altri `builders`: vedi CoroutineScope
- [`Job`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html): A background job. Conceptually, a job is a cancellable thing with a life-cycle that culminates in its completion.
- [`Deferred<T>`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html): Deferred value is a non-blocking cancellable future — it is a [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html) with a result. It is created with the [async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) coroutine builder or via the constructor of [CompletableDeferred](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-completable-deferred/index.html) class. It is in [active](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/is-active.html) state while the value is being computed.
- [`SupervisorJob`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-supervisor-job.html): Creates a *supervisor* job object in an active state. Children of a supervisor job can fail independently of each other. A failure or cancellation of a child does not cause the supervisor job to fail and does not affect its other children, so a supervisor can implement a custom policy for handling failures of its children
- [`CoroutineContext`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/index.html): persistent context for the coroutine. It is an indexed set of [Element](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/-element/index.html) instances. An indexed set is a mix between a set and a map. Every element in this set has a unique [Key](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/-key.html).
- [`Dispatchers`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/index.html): Base class that shall be extended by all coroutine dispatcher implementations. The following standard implementations are provided by `kotlinx.coroutines` as properties on Dispatchers objects:
  - [Dispatchers.Default](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html) – is used by all standard builder if no dispatcher nor any other [ContinuationInterceptor](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation-interceptor/index.html) is specified in their context. It uses a common pool of shared background threads. This is an appropriate choice for compute-intensive coroutines that consume CPU resources.
  - [Dispatchers.IO](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-i-o.html) – uses a shared pool of on-demand created threads and is designed for offloading of IO-intensive *blocking* operations (like file I/O and blocking socket I/O).
  - [Dispatchers.Unconfined](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html) – starts coroutine execution in the current call-frame until the first suspension. On first suspension the coroutine builder function returns. The coroutine resumes in whatever thread that is used by the corresponding suspending function, without confining it to any specific thread or pool.**Unconfined dispatcher should not be normally used in code**.
  - Private thread pools can be created with [newSingleThreadContext](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-single-thread-context.html) and [newFixedThreadPoolContext](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-fixed-thread-pool-context.html).
  - An arbitrary [Executor](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html) can be converted to dispatcher with [asCoroutineDispatcher](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/java.util.concurrent.-executor/as-coroutine-dispatcher.html) extension function.



