###### Appunti tratti da "Java, The Complete Reference, Ninth Edition" di Herbert Schildt, e altri ottimi articoli [[1]](https://www.javacodegeeks.com/2015/03/java-8-lambda-expressions-tutorial.html?utm_content=bufferb1cd0&utm_medium=social&utm_source=twitter.com&utm_campaign=buffer), [[2]](http://www.mokabyte.it/2015/09/java8lambda/) 
# 1. Introduzione


Le *Lambda Expressions* sono state introdotto con la JDK 8 e hanno portato un
significativo cambiamento per due principali ragioni

1. Hanno aggiunto nuovi elementi sintattici al linguaggio
2. Il compilatore è in grado di ottimizzare il codice per ottenere tutto il vantaggio della
computazione parallela dei moderni processori multicore

In aggiunta, con l'introduzione delle lambda expressions, il linguaggio è cambiato sotto altri
aspetti in quanto, come in una reazione a catena, sono stati introdotti i **metodi default per le interfacce**,
e gli oggetti `Stream` per avvantaggiarsi nelle operationi concatenate.

>In breve, le espressioni lambda **permettono di passare come argomento ad un metodo, comportamenti e funzioni e non solo
>oggetti e valori** come nelle versioni precedenti a cui gli sviluppatori erano abituati. Un esempio su tutti, è la 
>possibilità di evitare l'uso di classi anonime per fare l'override di un singolo metodo, quando va passato ad un metodo
>un comportamento come, sempre per esempio, un *clickListener*

Per capire l'implementazione delle lambda expressions, occorre aver chiari due costrutti

1. **Le lambda expressions stesse**: sono essenzialmente dei metodi anonimi, ma non eseguiti di per se ma usati per
implementare un metodo definito altrove, in una functional interface.
2. **Le interfacce funzionali** (*functional interface*): è un' interfaccia che contiene uno e un _solo metodo astratto_.

## 1.1 Functional Interface

Come accennato in precedenza, sono interfacce con **un solo metodo astratto** al loro interno. L'enfasi va posta 
proprio sul fatto che sia uno solo, e che sia astratto in quanto con JDK8 è possibile definire all'interno di una
interfaccia anche metodi statici e i default methods ovvero metodi implementati direttamente nell'interfaccia. Rimane il
fatto che un metodo non default o statico nelle interfacce, è implicitamente astratto e non necessiti della keyword *abstract*.
Quindi per essere definita funzionale, una interfaccia basta che rispetti il requisito dell'unico metodo astratto. Esiste
una annotazione, `@FunctionalInterface`, per identificare come funzionale una interfaccia, ma è prettamente informativa
per il compilatore e non necessaria anche se raccomandata. Queste ad esempio sono tutte functional interface, riportati
anche in un [esempio eseguibile](../src/LambdaExpressions.java)

```java
@FunctionalInterface
interface MyObjWithInt {
    int getInt();
    default int getAnswer(){ 
        return 42; 
    }
}

@FunctionalInterface
interface MyObjWithParam {
    boolean getBool(double a);
    static int getAnswer( int param){
        return 42*param; 
    }
}

@FunctionalInterface
interface MyObjGenerics<T> {
    T func(T t);
}
```

 Come detto in precedenza, una espressione lambda non viene eseguita di per se, ma forma l'implementazione del metodo
 astratto dell'interfaccia funzionale target di quel tipo. Come prima conseguenza, **una espressione lambda può essere
 eseguita solo in contesti dove il tipo è definito**. 

 In questi casi, viene creata automaticamente una istanza del tipo target che implementa l'interfaccia funzionale avendo
 l'espressione lambda come implementazione del proprio metodo astratto ( vedi [esempio eseguibile](../src/LambdaExpressions.java) ).

## 1.2 I fondamenti delle lambda expressions

Nella nuova sintassi viene introdotto un nuovo opertatore nel linguaggio, l'*operatore lambda*, o anche chiamato
*operatore freccia* indicato con `->` e che si legge "diventa" o "va a". <br/>
Divide l'espressione lambda in due parti:
- A sinistra vengono specificati i parametri, se necessari
- A destra, il corpo dell'espressione, che specifica l'azione da compiere.

```java
// qualche esempio di sintassi di lambda expressions
() -> Math.random()
(n) -> (n % 2) == 0
(a,b) -> a + b
```

Quando il parametro è uno solo, le parentesi possono anche essere omesse, ma
per omogeneità di espressione con gli altri casi ( più parametri o nessun parametro) è
consigliabile mantenerle in ogni caso.

Naturalmente le espressioni lambda possono contenere più che una istruzione come
negli esempi precedenti, ed in questo caso si parla di * blocchi lambda* e sono racchiusi
tra parentesi graffe. In tali blocchi si possono dichiarare variabili, usare loops e 
tutto quello che genericamente si può fare all'interno di un metodo. 
L'unica accortezza, è che va esplicitamente usato **return** per ritornare un valore,
esattamente come in un metodo non void dopotutto ( e come detto, le lambda expressions
ritornano sempre un valore ).
In aggiunta, visto che le lambda expressions corrispondono a classi anonime, 
è possibile utilizzare al loro interno anche variabili non appartenenti al blocco, basta siano definite *final*. 

```java
MyFuncInterf fattoriale = (n) ->  {
    int result = 1;
    for(int i=1; i <= n; i++)
        result = i * result;
    return result;
};
```

Se un blocco lambda sembra eccessivamente lungo, e compromette quindi la leggibilità del
codice, lo si può assegnare ad una variabile del tipo interfaccia funzionale compatibile
con il tipo ritornato dall'espressione stessa.

# 1.3 Functional interface predefinite

Prima di avventurarmi in un esempio concreto in cui le lambda expressions trovano
utilizzo, due parole su alcune nuove funzioni di JDK8 utili al caso.
È stato infatti introdotto un nuovo package **`java.util.function`** dove sono state
definite delle interfacce generiche adattabili a quasi qualunque occasione. Qui, le
più comuni, la cui lettura aiuta la comprensione del successivo esempio

| nome                                                         | descrizione                                                  |
| ------------------------------------------------------------ | :----------------------------------------------------------- |
| [UnaryOperator&lt;T&gt;](https://docs.oracle.com/javase/9/docs/api/java/util/function/UnaryOperator.html) | Applica una operazione unaria su un oggetto di tipo T e restituisce il risultato, sempre di tipo T. Il suo metodo astratto è chiamato **apply()** |
| [BinaryOperator&lt;T&gt;](https://docs.oracle.com/javase/9/docs/api/java/util/function/BinaryOperator.html) | Applica una operazione su due oggetti di tipo T il cui risultato è sempre di tipo T. Il suo metodo astratto è chiamato **apply()** |
| [Consumer&lt;T&gt;](https://docs.oracle.com/javase/9/docs/api/java/util/function/Consumer.html) | Applica un operatione su un oggetto di tipo T ed il suo metodo astratto si chiama **accept()** |
| [Supplier&lt;T&gt;](https://docs.oracle.com/javase/9/docs/api/java/util/function/Supplier.html) | Restituisce un oggetto di tipo T, il suo metodo si chiama **get()** |
| [Function&lt;T, R&gt;](https://docs.oracle.com/javase/9/docs/api/java/util/function/Function.html) | Applica una operazione su un oggetto di tipo T e restituisce il risultato come un oggetto di tipo R. Il suo metodo è **apply()** |
| [Predicate&lt;T&gt;](https://docs.oracle.com/javase/9/docs/api/java/util/function/Predicate.html) | Determina se un oggetto di tipo T soddisfa qualche vincolo. Restituisce un valore booleano che indica il risultato. Il suo metodo è **test()** |

Con queste ultime funzioni, la teoria sulle functional interface, e la grammatica
delle lambda expression posso descrivere un caso reale di utilizzo.

# 2. Applicazione

Per semplicità, immaginiamo di avere una classe che deve eseguire una serie di 
operazioni

```java
public class Calcolatrice {

    public static int somma(int a, int b) {
        return a + b;
    }
    public static int sottrai(int a, int b) {
        return a - b;
    }
    public static int moltiplica(int a, int b) {
        return a * b;
    }
	
}
```

Ora, se in futuro devo implementare ulteriori operazioni, oppure le operazioni da eseguire
sono definite altrove e non prevedibili la cosa si fa ingestibile.<br/>
Attraverso le functional interface predefinite, possiamo risolvere in una maniera più favorevole
definendo la classe Calcolatrice con un unico metodo che prende come parametri i due elementi
su cui eseguire l'operazione e l'operazione stessa, definita attraverso la functional interface `BinaryOperator`.
Avendo una interfaccia funzionale come parametro, il metodo è quindi in grado di accettare come
valore una espressione lambda, che andrà quindi ad implementare l'operazione stessa.

```java
public class Calcolatrice {

    public static int calcola(int a, int b, BinaryOperator operazione) {
        return operazione.apply(a, b);
    }

}
```

Sempre nell'[esempio eseguibile](../src/LambdaExpressions.java) a questo punto basta definire l'oggetto 
`BinaryOperator` con una lambda per avere la flessibilità richiesta. 

```java
int result;
result = LambdaExpressions.calcola(5,2, (x,y) -> x + y);
System.out.println(result);
```

# 3. Method References e Constructor References

Ci sono ancora un paio di features legate alle lambda expressions, i method
references e i constructor references

- `ClassName::methodName`, method reference: fornisce un modo per riferirsi ad un
metodo senza eseguirlo. È in relazione con le lambda expression poiché richiede un
tipo che sia compatibile con una interfaccia funzionale; quando il metodo viene
valutato, viene creata una istanza dell'interfaccia. Vale sia per metodi statici, che
per metodi di istanza.
- `ClassName::new`, constructor reference: in maniera simile per quanto avviene con
i metodi, ci si può riferire ad un costruttore. Questo riferimento può essere
assegnato ad una interfaccia funzionale di tipo compatibile al costruttore.