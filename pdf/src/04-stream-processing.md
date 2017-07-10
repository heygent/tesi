# Stream Processing

Ci sono molte differenze semantiche da considerare tra il batch processing e lo
stream processing. I tool orientati allo stream processing devono tenere in
conto della natura fluente dei dati, e di come questo comporti necessariamente
un cambiamento di approccio alla fault tolerance. Il fault in sistemi di batch
processing può portare a una perdita dei risultati della computazione, per cui
c'è sempre la possibilità, anche se potenzialmente costosa, di rieseguire
l'elaborazione. Nello stream processing, i dati ricevuti possono essere
effimeri e non necessariamente recuperabili, per cui un fault può significare
un'effettiva perdita di informazione. Le soluzioni offerte dai tool che
adottano l'approccio dell'elaborazione stream sono spesso il riflesso della
gestione dell'affidabilità in protocolli come TCP, riutilizzando concetti come
*sliding windows* e *acknowledgements*.

Oltre alla fault tolerance, un aspetto importante da considerare è la
differenza concettuale tra *data in motion* e *data at rest*, e di come le
astrazioni che rappresentano i dati di input debbano cambiare per rispecchiare
questa differenza. Fortunatamente, alcuni paradigmi computazionali sono invece
riutilizzabili, come quello di Spark, che è abbastanza generale da poter
supportare lo stream processing con qualche aggiunta. Spark fornisce delle API
apposite per lo stream processing, parte del progetto *Spark Streaming*.

Gli approcci alla stream computation principali sono due: (near) **real-time**
e **microbatching**. Il real-time, come il nome suggerisce, esegue le
computazioni su ogni dato in input non appena questo è disponibile, mentre il
microbatching raccoglie un certo numero di input in un buffer, che vengono poi
processati in gruppo. Il primo approccio favorisce una latenza minore, mentre
il secondo un throughput più alto.

In questa sezione si osservano Spark Streaming, che adotta il microbatching, e
Apache Storm, che utilizza l'approccio real-time.


## Spark Streaming

Spark Streaming è un'estensione di Spark che permette di lavorare con stream,
che possono essere ottenuti tramite diverse fonti (socket, sistemi di
ingestione come Kafka, HDFS...). I dati ricevuti in uno stream sono
raccolti in piccoli batch in un arco di tempo specificato dall'utente, per poi
essere rielaborati utilizzando l'engine di esecuzione di Spark.

L'astrazione utilizzata sui flussi è chiamata DStream, che sta per Discretized
Stream. I DStream permettono l'esecuzione di azioni e trasformazioni come per
gli RDD, aggiungendo alcune operazioni particolari dedicate agli stream, e sono
rappresentati internamente come sequenze di RDD. Gli RDD interni corrispondenti
ai microbatch sono anche accessibili all'utente, per eseguire trasformazioni
con le API specifiche di questi.

I risultati dell'elaborazione possono essere salvati in diversi mezzi, come
database, HDFS e filesystem locali, o dashboard per analytics in real-time.
Come per gli RDD, il formato degli output può essere specificato, tuttavia la
struttura è differente: per ogni microbatch elaborato, Spark Streaming crea una
nuova cartella con i risultati. Le cartelle con i risultati vengono nominate
con un prefisso e un suffisso specificati dall'utente, e un timestamp in
formato epoca UNIX che indica il momento della computazione.

Gli stream possono essere creati con l'analogo dello `SparkContext` per Spark
Streaming, ovvero uno `StreamingContext`. Nell'istanziazione di uno
SparkContext, si specifica un oggetto `SparkConf`, dello stesso tipo utilizzato
per gli `SparkContext`, e un intervallo di batch, utilizzato per stabilire ogni
quanto tempo i dati raccolti debbano essere elaborati.

```scala
val conf = new SparkConf()
   .setMaster(args(0))
   .setAppName("Some Spark Streaming job")

val ssc = new StreamingContext(conf, Seconds(1))
```

Le fonti creabili dai DStream si dividono in due categorie:

* Le sorgenti base possono essere create direttamente dallo `StreamingContext`,
  e rappresentano primitive semplici, come socket e file di testo.
* Le sorgenti avanzate richiedono l'uso di librerie apposite, e vengono create
  tramite classi d'utilità fornite da queste. Le librerie forniscono interfacce
  di alto livello a protocolli applicativi di diverse applicazioni e servizi,
  come Kafka e Twitter.


```{#lst:dstream-creation .scala}
val lines = ssc.socketTextStream("localhost", 9999)
val tweets = TwitterUtils.createStream(ssc, None)
```

: Creazione di DStream a partire da una sorgente base, un socket, e una
sorgente avanzata, uno stream di tweet.

In [@lst:spark-streaming-tweet-location] viene mostrato un programma Spark
Streaming che utilizza un flusso di tweet come input. La trasformazione
`flatMap` viene utilizzata per mappare ogni tweet al paese da cui è stato
inviato, e scartare i tweet di cui non si conosce la provenienza. Il valore è
restituito in una coppia composta dal paese e il valore 1. Con `reduceByKey`,
si aggregano le coppie per paese sommando i rispettivi valori.

`tranform` è un metodo che riceve in input una funzione, che come parametro
ottiene un RDD rappresentativo del microbatch. La funzione viene utilizzata per
accedere al metodo `sortBy` dell'RDD, che ne ordina i valori in base a una
funzione che restituisce una chiave di comparazione. La chiave utilizzata è il
numero di tweet, e l'ordinamento è specificato come decrescente. In questo
modo, il risultato finale è ordinato in base al numero di tweet, creando una
classifica dei paesi che hanno inviato più tweet. Infine, si richiede al
framework di stampare nello standard output i primi cinque risultati.

Per avviare l'elaborazione in Spark Streamng, sono necessarie due chiamate
finali a metodi dello `StreamingContext`: `ssc.start()`, che avvia la
computazione, e `ssc.awaitTermination()`, una chiamata a funzione bloccante che
fa sì che il programma non termini fino al termine del lavoro. Il termine del
lavoro può essere segnalato al framework chiamando il metodo `ssc.stop()`. Dato
che nel programma questo metodo non viene chiamato, il job resta in esecuzione
fino a quando non viene terminato dall'utente.


```{#lst:spark-streaming-tweet-location .scala}
object Main {

  def main(args: Array[String]) {

    val conf = new SparkConf()
      .setMaster(args(0))
      .setAppName("Some Spark Streaming App")

    val ssc = new StreamingContext(conf, Seconds(60))
    val tweets = TwitterUtils.createStream(ssc, None)

    tweets
      .flatMap(t => Try { (t.getPlace.getCountry, 1) }.toOption)
      .reduceByKey(_ + _)
      .transform(_.sortBy(_._2, ascending = false))
      .print(5)

    ssc.start()
    ssc.awaitTermination()

  }
}
```

: Programma Spark Streaming che calcola, per ogni gruppo di tweet inviato
nell'arco di 60 secondi, quanti ne sono stati inviati da ogni paese.

Il programma stampa un output ogni 60 secondi, nella forma mostrata in
[@lst:spark-streaming-tweet-location-output]. Per ogni output, viene anche
stampato il momento di elaborazione sotto forma di UNIX epoch.

```{#lst:spark-streaming-tweet-location-output}
-------------------------------------------
Time: 1499720340000 ms
-------------------------------------------
(United States,4)
(España,3)
(Türkiye,2)
(Venezuela,1)
(United Arab Emirates,1)
...

-------------------------------------------
Time: 1499720400000 ms
-------------------------------------------
(United States,12)
(Brasil,8)
(United Kingdom,6)
(Australia,1)
(Kosovo,1)
...
```
: Output del programma in [@lst:spark-streaming-tweet-location].

### Operazioni Stateful

In molti contesti, è desiderabile essere in grado di mantenere uno stato
relativo al flusso, in modo che i risultati delle elaborazioni possano
riguardare periodi di tempo più estesi rispetto all'intervallo di batch. A
questo scopo, i DStream forniscono un metodo `updateStateByKey`, che permette
di mantenere uno stato arbitrario su una serie di chiavi. Il metodo rappresenta
un meccanismo analogo a `reduceByKey`, dove coppie chiave-valore vengono
raggruppate in base alla chiave e un'operazione specificata dall'utente aggrega
i valori di ogni gruppo. In `updateStateByKey`, l'operazione coinvolge anche
uno stato, che rappresenta il risultato dell'operazione di aggregazione
precedente. Lo stato può essere unito al risultato dell'operazione di
aggregazione corrente per ottenere un nuovo stato, che verrà a sua volta
passato nell'operazione di aggregazione successiva.

L'operazione di aggregazione viene specificata come una funzione, che prende in
input una sequenza di valori aggregati in base alla chiave del microbatch
corrente, e lo stato precedente. Riprendendo l'esempio della classifica del
numero di tweet per paese, `updateStateByKey` può essere utilizzato per
eseguire il conto totale dei tweet a partire dall'esecuzione del job. La
funzione di input di `updateStateByKey` è la seguente:

```scala
def updateTweetCount(newVals: Seq[Int], state: Option[Int]): Option[Int] =
  Some(state.getOrElse(0) + newVals.sum)
```

Lo stato preso in input dalla funzione è di tipo `Option`[^option], per gestire
il caso della computazione iniziale, in cui nessuno stato precedente è stato
calcolato. Il metodo `getOrElse(default: T)` restituisce il valore dello stato,
se esiste, altrimenti 0. Il risultato viene calcolato sommando lo stato
precedente (oppure 0) con la somma dei nuovi tweet.

[^option]: `Option` è una classe astratta utilizzata in Scala per incapsulare
un valore nullabile, e ha due sottoclassi concrete, `Some` e `None`, che
rappresentano rispettivamente la presenza e l'assenza di un valore.

Il nuovo elenco di operazioni sul DStream dei tweet appare come segue.

```scala
    tweets
      .flatMap(t => Try { (t.getPlace.getCountry, 1) }.toOption)
      .updateStateByKey(updateTweetCount)
      .transform(_.sortBy(_._2, ascending = false))
      .print(5)
```

`updateStateByKey` restituisce un DStream contente le coppie chiave-stato, che
può essere utilizzato per eseguire le stesse operazioni precedentemente
descritte per ottenere la classifica dei tweet. L'output del programma,
è esposto in [@lst:tweet-location-stateful].

```{#lst:tweet-location-stateful}
-------------------------------------------
Time: 1499722150000 ms
-------------------------------------------
(United States,34)
(Brasil,17)
(United Kingdom,5)
(Argentina,4)
(Brazil,3)
...

-------------------------------------------
Time: 1499722155000 ms
-------------------------------------------
(United States,35)
(Brasil,18)
(United Kingdom,5)
(Argentina,5)
(Brazil,3)
...
```

: Output consecutivi del programma di conteggio dei tweet, configurato per
l'operazione stateful con un intervallo di batch di cinque secondi.

### Operazioni su Finestre

Le operazioni su finestre sono trasformazioni che permettono di eseguire
computazioni sugli input ricevuti entro un certo lasso di tempo. L'immagine ...
mostra visivamente il concetto.


