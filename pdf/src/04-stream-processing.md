# Stream Processing

Ci sono molte differenze semantiche da considerare tra il batch processing e lo
stream processing. Dato che gli stream non hanno necessariamente un limite di
dimensione, o un termine della computazione, i tool orientati allo stream
processing devono fornire astrazioni che tengano in conto della dimensione
temporale dei flussi, in modo da fornire una misura diversa dal volume dei
dati.  In particolare, è necessario che le API permettano di stabilire
degli intervalli per l'esecuzione e la raccolta dei risultati, o che forniscano
un meccanismo di reazione a fronte di eventi di ricezione. Inoltre, è utile che
i flussi mettano a disposizione meccanismi per il mantenimento di stato.

Gli approcci alla stream computation principali sono due: (near) **real-time**
e **microbatching**. L'approccio real-time esegue le computazioni su ogni
record in input non appena questo è disponibile, mentre il microbatching
raccoglie un numero di input in un buffer in un certo intervallo di tempo, che
vengono poi processati in gruppo. Il primo approccio favorisce una latenza
minore, mentre il secondo un throughput più alto.

In questa sezione si osserva Spark Streaming, un modulo di Apache Spark
dedicato allo stream processing orientato al microbatching, che riutilizza
molte delle sue strutture e dei suoi concetti, permettendo di riapplicare le
conoscenze già acquisite sull'elaborazione batch allo stream processing.

## Spark Streaming

Spark Streaming è un'estensione di Spark che permette di lavorare con stream,
che possono essere ottenuti tramite diverse fonti (socket, sistemi di
ingestione come Kafka, HDFS...). I dati ricevuti in uno stream sono
raccolti in piccoli batch in un arco di tempo specificato dall'utente, per poi
essere rielaborati utilizzando l'engine di esecuzione di Spark.

L'astrazione utilizzata sui flussi è chiamata DStream, che sta per Discretized
Stream. I DStream permettono l'esecuzione di azioni e trasformazioni come per
gli RDD, aggiungendo alcune operazioni particolari dedicate agli stream.
Internamente, i DStream sono rappresentati come sequenze di RDD. Gli RDD
interni corrispondenti ai microbatch sono resi accessibili all'utente, per
eseguire trasformazioni con le API specifiche degli RDD.

I risultati dell'elaborazione possono essere salvati in diversi mezzi, come
database, HDFS e filesystem locali, o dashboard per analytics in real-time.
Come per gli RDD, il formato degli output può essere specificato, tuttavia la
struttura è differente: per ogni microbatch elaborato, Spark Streaming crea una
nuova cartella con i risultati. Le cartelle con i risultati vengono nominate
con un prefisso e un suffisso specificati dall'utente, e un timestamp in
formato epoca UNIX che indica il momento della computazione.

Gli stream possono essere creati con l'analogo dello `SparkContext` per Spark
Streaming, ovvero uno `StreamingContext`. Nell'istanziazione di uno
StreamingContext, si specifica un oggetto `SparkConf`, dello stesso tipo
utilizzato per gli `SparkContext`, e un intervallo di batch, utilizzato per
stabilire ogni quanto tempo i dati raccolti debbano essere elaborati.

```scala
val conf = new SparkConf()
   .setMaster(args(0))
   .setAppName("Some Spark Streaming job")

val ssc = new StreamingContext(conf, Seconds(1))
```

Le fonti da cui è possibile ricavare DStream si dividono in due categorie:

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
Streaming che utilizza un flusso di tweet come input. Nella riga 13 la
trasformazione `flatMap` viene utilizzata per mappare ogni tweet al paese da
cui è stato inviato, e scartare i tweet di cui non si conosce la provenienza.
Il valore è restituito in una coppia composta dal paese e il valore 1. Con
`reduceByKey`, si aggregano le coppie per paese e si sommano i valore delle
coppie.

`tranform`, rr. 15, è un metodo che riceve in input una funzione, che come parametro
ottiene un RDD rappresentativo del microbatch. La funzione viene utilizzata per
accedere al metodo `sortBy` dell'RDD, che ne ordina i valori in base a una
funzione che restituisce una chiave di comparazione. La chiave utilizzata è il
numero di tweet, e l'ordinamento viene specificato come decrescente. In questo
modo, il risultato finale è ordinato in base al numero di tweet, creando una
classifica dei paesi che hanno inviato più tweet. Infine, si richiede al
framework di stampare nello standard output i primi cinque risultati.

Per avviare l'elaborazione in Spark Streaming, sono necessarie due chiamate
finali a metodi dello `StreamingContext`: `ssc.start()`, che avvia la
computazione, e `ssc.awaitTermination()`, una chiamata a funzione bloccante che
fa sì che il programma non termini fino al termine del lavoro. La fine del
lavoro può essere segnalata al framework chiamando il metodo `ssc.stop()`. Dato
che nel programma questo metodo non viene chiamato, il job resta in esecuzione
fino a quando non viene terminato dall'utente.


```{#lst:spark-streaming-tweet-location .numberLines .scala}
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
[@lst:spark-streaming-tweet-location-output]. Per ogni output, il framework
stampa automaticamente il momento di elaborazione sotto forma di UNIX epoch.

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
di mantenere uno stato arbitrario su una serie di chiavi. Il metodo riflette
un meccanismo analogo a `reduceByKey`, dove coppie chiave-valore vengono
raggruppate in base alla chiave e un'operazione specificata dall'utente aggrega
i valori di ogni gruppo. In `updateStateByKey`, l'operazione coinvolge anche
uno stato, che rappresenta il risultato dell'operazione di aggregazione
precedente. Lo stato può essere unito al risultato dell'operazione di
aggregazione corrente per ottenere un nuovo stato, che verrà a sua volta
passato nell'operazione di aggregazione successiva.

L'operazione di aggregazione viene specificata come una funzione, che prende in
input una sequenza di valori del microbatch aggregati in base alla chiave, e
lo stato precedente. Riprendendo l'esempio della classifica del
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
computazioni sugli input ricevuti entro un certo lasso di tempo. Il concetto è
mostrato visivamente in [@fig:sliding-window].

![Schema rappresentante l'operazione su finestre di Spark
Streaming[@spark-sliding-window]](img/streaming-window.png){#fig:sliding-window}

I *windowed DStream* vengono creati a partire da un DStream tramite una
trasformazione, e hanno due parametri fondamentali:

* la *window duration*, che rappresenta l'arco di tempo antecedente al
  microbatch di cui si vogliono considerare gli input;

* la *sliding duration*, che indica ogni quanto tempo la finestra si sposta in
  avanti.

Entrambi questi parametri devono essere multipli dell'intervallo di microbatch.
A ogni intervallo di tempo di durata *sliding duration*, i DStream emettono i
valori di input ricevuti nel periodo di tempo che va dall'attimo che antecede
il momento attuale di un tempo uguale alla *window duration*, fino al momento
attuale. I valori ottenuti possono poi essere rielaborati come un normale
DStream. In questo modo, si può ottenere una gestione dei tempi
dell'elaborazione più flessibile rispetto alla sola configurazione
dell'intervallo di batch, permettendo l'interleaving degli intervalli di
elaborazione con quelli di ricezione dell'input.

Il contatore di tweet può essere riadattato facilmente a questo paradigma. Per
creare un *windowed DStream* a partire da un DStream, la trasformazione più
generale è `window(windowDuration: Duration, slidingDuration: Duration)`.
Esistono metodi più specifici per fare uso delle operazioni su finestre, come
`reduceByKeyAndWindow`, che esegue un'operazione di `reduce` nell'arco di una
finestra. Il metodo `reduceByKeyAndWindow` si adatta bene allo use case del
contatore, e può essere utilizzato per fare una classifica dei paesi che hanno
inviato più tweet nell'arco dell'ultimo minuto.

```scala
tweets
  .flatMap(t => Try { (t.getPlace.getCountry, 1) }.toOption)
  .reduceByKeyAndWindow(_ + _, Minutes(1))
  .transform(_.sortBy(_._2, ascending = false))
  .print(5)
```

```{#lst:sliding-window-output}
-------------------------------------------
Time: 1499737107000 ms
-------------------------------------------
(United States,21)
(Brasil,9)
(Argentina,2)
(Peru,1)
(Nigeria,1)
...

-------------------------------------------
Time: 1499737108000 ms
-------------------------------------------
(United States,21)
(Brasil,9)
(Argentina,2)
(Japan,2)
(Peru,1)
...

-------------------------------------------
Time: 1499737109000 ms
-------------------------------------------
(United States,22)
(Brasil,9)
(Argentina,2)
(Japan,2)
(Peru,1)
...
```

: Campione dell'output del programma di classifica dei paesi che hanno inviato
più tweet, configurato con una sliding window di un minuto.
