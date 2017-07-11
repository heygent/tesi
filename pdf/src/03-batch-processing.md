# Batch Processing

Il Batch Processing è la *raison d'être* di Hadoop. Il primo paradigma di
programmazione per Hadoop, MapReduce, è stato l'unico per molte release, e
ha avuto il grande merito di astrarre la complessità della
computazione batch in ambiente distribuito in funzioni che associano chiavi e
valori a risultati, una grande semplificazione rispetto ai programmi che
gestiscono granularmente l'intricatezza degli ambienti distribuiti.

Pur essendo popolare, MapReduce è soggetto a molte limitazioni, che riguardano
soprattutto la necessità di esprimere i programmi da eseguire con un modello
che non lascia molto spazio alla rielaborazione dei risultati. Come si vedrà,
queste limitazioni sono intrinseche al fatto che i risultati intermedi vengano
salvati nello storage locale del nodo del cluster, e quelli finali in HDFS.
Questi due fattori influenzano pesantemente le prestazioni che si possono
ottenere da un algoritmo, perché vi introducono l'overhead della lettura e
scrittura nel disco, o peggio in HDFS.

YARN è stato creato proprio per questo motivo: permettere che altri modelli di
computazione diversi da MapReduce potessero essere eseguiti sfruttando HDFS. 
Le nuove versioni di MapReduce sono implementate al di sopra di YARN invece che
direttamente in Hadoop come in passato, a testimoniare l'effettiva capacità di
YARN di generalizzare i modelli di esecuzione nei cluster.

La sua alternativa più popolare, Apache Spark, ha API più espressive e
funzionali rispetto a MapReduce, ed è più performante in molti tipi di
algoritmi[@mapreduce-spark-performance]. Tramite astrazioni che offrono un
controllo più preciso sul comportamento dei risultati dell'elaborazione, Spark
trova applicazioni pratiche in vari ambiti, tra cui machine
learning[@spark-mllib], graph processing[@spark-graphx] e elaborazione
SQL[@spark-sql].

In questa sezione si esaminano MapReduce e Spark, quali sono le limitazioni di
MapReduce che hanno fatto sentire la necessità di un nuovo modello
computazionale, e alcune delle soluzioni e interfacce di programmazione offerte
da Spark. 

## MapReduce

Il modello computazionale di MapReduce è composto, nella sostanza, da due
componenti, il Mapper e il Reducer. Questi componenti sono specificati
dall'utilizzatore del framework, e possono essere descritti come due funzioni.

$$Map(K_1, V_1) \mapsto Sequence[(K_2, V_2)]$$

La funzione $Map$ è eseguita nello stadio iniziale della computazione su valori
di input esterni. L'input della funzione $Map$ è una coppia chiave-valore $K_1$
e $V_1$, i cui valori dipendono dal tipo di input letto. Ad esempio, nei file
di testo, $K_1$ rappresenta il numero di riga di un file e $V_1$ la riga di
testo corrispondente. 

A partire da ogni coppia, $Map$ elabora e restituisce una sequenza di nuove
coppie chiave-valore di tipo $K_2$ e $V_2$. Queste coppie vengono poi rielaborate
trasparentemente dal framework, che esegue due operazioni:

 #. **ordina** tutte le coppie in base alla chiave;
 #. **aggrega** le coppie che condividono la stessa chiave in una nuova coppia
    $(K_2, Sequence[V_2])$.

$$Reduce(K_2, Sequence[V_2]) \mapsto (K_2, V_3)$$

Ognuna delle coppie aggregate dal framework viene poi fornita in input alla
funzione $Reduce$, che ha quindi a disposizione una chiave $K_2$ e tutti i valori
restituiti da $Map$ che hanno la stessa chiave. $Reduce$ esegue una
computazione sui valori di input e restituisce $(K_2, V_3)$, che andrà a far
parte dell'output finale dell'applicazione assieme al risultato delle altre
invocazioni di $Reduce$, una per ogni chiave distinta restituita da $Map$.

Sintetizzando, MapReduce permette di categorizzare l'input in diverse parti e
di elaborare un risultato per ognuna di queste.

MapReduce è un paradigma *funzionale*, dato che il framework richiede di
ricevere in input le funzioni utili all'elaborazione dei dati. Per esprimere
questo tipo di paradigma in Java si ricorre a classi che incapsulano le
funzioni richieste dal framework, che vengono quindi chiamate Mapper e Reducer.

Il Mapper in un'applicazione MapReduce è una classe contenente un metodo
`void map`, che riceve in input una chiave e un valore, e un oggetto `Context`,
il cui ruolo più importante è fornire il metodo `Context.write(K, V)`, che
viene utilizzato per scrivere i valori di output del Mapper.

Le applicazioni MapReduce specificano un proprio Mapper estendendo la classe
`Mapper` nella libreria di Hadoop, e compilando i tipi dei parametri generici
opportunamente. La firma di `Mapper` è la seguente:

```java
public class Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT> extends Object
```

Le chiavi e i valori ricevuti in input dal Mapper sono derivati direttamente
dall'elemento letto in HDFS. È possibile configurare quali chiavi e valori
vengano derivati dalla sorgente e come, creando una classe che implementa
l'interfaccia `InputMapper` fornita nella libreria di Hadoop. Nella libreria,
Hadoop fornisce diversi `InputMapper` che corrispondono a comportamenti di
lettura desiderabili per diversi tipi di file e sorgenti, come file con formati
colonnari, o contenti coppie chiave-valore divise da marcatori.

I tipi ricevuti in input dal Mapper sono specificati nei parametri generici
`KEYIN` e `VALUEIN`, e devono corrispondere ai tipi che l'`InputFormat` di
riferimento restituisce. `KEYOUT` e `VALUEOUT` sono invece i tipi che il Mapper
restituisce rielaborando le chiavi e i valori in input. `map` ha la seguente
signature:

```java
protected void map(KEYIN key, VALUEIN value, Context context) 
    throws IOException, InterruptedException
```

Una volta restituiti dal Mapper, le coppie vengono date in input a una classe
`Reducer`, che ha una signature simile a quella del `Mapper`:

```java
public class Reducer<KEYIN,VALUEIN,KEYOUT,VALUEOUT> extends Object
```

Una classe che estende `Reducer` ha un metodo `reduce`, che riceve in input una
chiave, e un iterabile di tutti i valori restituiti dai Mapper che hanno quella
stessa chiave:

```java
protected void reduce(KEYIN key, Iterable<VALUEIN> values, Context context) 
    throws IOException, InterruptedException
```

I valori possono quindi essere combinati a seconda dell'esigenza dell'utente
per restituire un risultato finale.

### Esempio di un programma MapReduce {#mapred-example}

Come esempio di programma per MapReduce, si prende in considerazione l'analisi
di log di un web server. Il dataset su cui si esegue l'elaborazione è fornito
liberamente dalla NASA[@nasa-weblog], e corrisponde ai log di accesso al server
HTTP del Kennedy Space Center dal 1/07/1995 al 31/07/1995. Il log è un file di
testo con codifica ASCII, dove ogni riga corrisponde a una richiesta e contiene
le seguenti informazioni:

#. L'host che esegue la richiesta, sotto forma di hostname quando disponibile o
   indirizzo IP altrimenti;
#. Timestamp della richiesta, in formato "`WEEKDAY MONTH DAY HH:MM:SS YYYY`" e
   fuso orario, con valore fisso `-0400`;
#. La Request-Line HTTP tra virgolette;
#. Il codice HTTP di risposta;
#. La dimensione in byte della risposta.

```{#lst:log-sample}
ntp.almaden.ibm.com - - [24/Jul/1995:12:40:12 -0400] 
    "GET /history/apollo/apollo.html HTTP/1.0" 200 3260

fsd028.osc.state.nc.us - - [24/Jul/1995:12:40:12 -0400]
    "GET /shuttle/missions/missions.html HTTP/1.0" 200 8678
```

: Campione di due righe dal log da analizzare

A partire da questo log, si vuole capire quante richieste siano state ricevute
da ogni risorsa HTTP. Un possibile approccio alla risoluzione del problema è
eseguire il parsing di ogni riga del log nel Mapper utilizzando un'espressione
regolare, per estrarre l'URI dalla richiesta. Il Mapper, per ogni riga,
restituische l'URI come chiave e 1 come valore.

Dopo l'esecuzione dei Mapper, i Reducer riceveranno una coppia formata dall'URI
delle richieste come chiave, e da un iterabile di valori 1, uno per ogni
richiesta. È sufficiente sommare questi valori per ottenere il numero di
richieste finale per l'URI chiave.

```{#lst:log-mapper .java}
import java.io.IOException;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;

public class LogMapper extends Mapper<LongWritable, Text, Text, LongWritable> {

    private final static Pattern logPattern = Pattern.compile(
        ".*\"[A-Z]+ (.*) HTTP.*"
    );

    private final static LongWritable one = new LongWritable(1);

    @Override
    protected void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException {

        final String request = value.toString();
        final Matcher requestMatcher = logPattern.matcher(request);

        if(requestMatcher.matches()) {
            context.write(
                new Text(requestMatcher.group(1)),
                one
            );
        }
    }

}
```

: Implementazione del Mapper utilizzato per analizzare il file di log.

Come si può osservere da [@lst:log-mapper], i tipi utilizzati dal Mapper non
sono tipi standard Java, ma sono forniti dalla libreria. Hadoop utilizza un suo
formato di serializzazione per lo storage e per la trasmissione dei dati in
rete, diverso dalla serializzazione integrata in Java. In questo modo il
framework ha controllo preciso sulla fase di serializzazione, un fattore
importante data la crucialità in termini di efficienza che questa può avere.

Le funzionalità di serializzazione di Hadoop sono rese accessibili dagli
oggetti serializzabili tramite l'interfaccia `hadoop.io.Writable`. Le classi
`LongWritable` e `Text` sono dei wrapper sui tipi `long` e `String` che
implementano l'interfaccia `Writable`, e i valori contenuti in questi tipi
possono essere ottenuti rispettivamente con `LongWritable.get()` e
`Text.toString()`[^6].

[^6]: Le classi definite dagli utenti possono implementare a loro volta
l'interfaccia `Writable` per essere supportate come tipi di chiavi e valori nei
Mapper e nei Reducer. 

Nel Mapper, si utilizza l'espressione regolare `/.*"[A-Z]+ (.*) HTTP.*/` per
ottenere il token contenente l'URI della richiesta, e tramite `context.write`
si restituisce la coppia URI e 1.

```{#lst:log-reducer .java}

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

import java.io.IOException;

public class LogReducer extends Reducer<Text, LongWritable, Text, LongWritable> {
    @Override
    protected void reduce(Text key, Iterable<LongWritable> values, Context context)
            throws IOException, InterruptedException {

        long accumulator = 0;

        for(LongWritable value: values) {
            accumulator += value.get();
        }

        context.write(key, new LongWritable(accumulator));

    }
}
```

: Implementazione del Reducer per il programma di analisi dei log.

Il Reducer, mostrato in [@lst:log-reducer], prende in input nel suo metodo
`reduce` i valori aggregati in base alla chiave. Una volta sommati in una
variabile accumulatore, questi vengono scritti in output in una coppia
URI-accumulatore. L'insieme di tutte le coppie restituite dal Reducer
costituiscono l'output finale del programma, che vengono scritte in un file di
testo separando le chiavi dai valori con caratteri di tabulazioni, e ogni
valore di restituzione con un nuova riga.

![Diagramma di funzionamento di
MapReduce[@mapred-diagram]](img/mapreduce_diagram.png){width=75%}

Prima di poter eseguire l'applicazione, è necessario creare un esecutore,
ovvero una classe contenente un punto d'entrata `main` che utilizzi le API di
Hadoop per eseguire il programma, analogamente a come descritto in
[Esecuzione di software in Hadoop]. I lavori MapReduce sono configurati tramite
l'oggetto `hadoop.mapreduce.Job`, che richiede di specificare le classi da
utilizzare come Mapper e Reducer, assieme ai percorsi dei file da elaborare.
L'esecutore dell'analizzatore di log è mostrato in [@lst:log-executor].

```{#lst:log-executor .java}

import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class LogAnalyzer {

    public static void main(String args[]) throws Exception {
        if(args.length != 2) {
            System.err.println("Usage: LogAnalyzer <input path> <output path>");
        }

        Job job = Job.getInstance();
        job.setJarByClass(LogAnalyzer.class);
        job.setJobName("LogAnalyzer");

        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        job.setMapperClass(LogMapper.class);
        job.setReducerClass(LogReducer.class);

        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(LongWritable.class);

        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

: Esecutore dell'analizzatore di log.

L'oggetto `job` è il centro della configurazione del programma MapReduce.
Tramite questo si specificano il `jar` contenente le classi dell'applicazione,
il nome del Job, utilizzato per mostrare descrittivamente nei log e
nell'interfaccia web lo stato di completamente di questo, le classi Mapper e
Reducer e i tipi dei valori di output del Reducer. Vengono impostati anche i
path del file di input e dei file di output, utilizzando i valori ricevuti come
parametri in `args`. Il job viene effettivamente eseguito alla chiamata di
`job.waitForCompletion(bool verbose)`, che restituisce `true` quando questo va
a buon fine.

Al termine della compilazione e pacchettizzazione, il programma può essere
eseguito con il comando `hadoop`:

```sh
$ hadoop LogAnalyzer /example/NASA_access_log_Jul95 /example/LogAnalyzerOutput
```


Il metodo `job.waitForCompletion` è stato invocato con il parametro `verbose`
impostato a `true`, per cui l'esecuzione stampa in output un log sul job in
esecuzione. Lo stato di esecuzione dei job è anche consultabile tramite
un'interfaccia web fornita dal framework.

```sh
17/07/03 18:17:47 INFO Configuration.deprecation: session.id is deprecated.
    Instead, use dfs.metrics.session-id
17/07/03 18:17:47 INFO jvm.JvmMetrics: Initializing JVM Metrics with 
    processName=JobTracker, sessionId=
17/07/03 18:17:47 WARN mapreduce.JobResourceUploader: Hadoop command-line 
    option parsing not performed. Implement the Tool interface and execute
    your application with ToolRunner to remedy this.
17/07/03 18:17:48 INFO input.FileInputFormat: Total input files to process : 1
17/07/03 18:17:48 INFO mapreduce.JobSubmitter: number of splits:2
17/07/03 18:17:48 INFO mapreduce.JobSubmitter: Submitting tokens for 
    job: job_local954245035_0001
17/07/03 18:17:48 INFO mapreduce.Job: The url to track the job: http://localhost:8080/
17/07/03 18:17:48 INFO mapreduce.Job: Running job: job_local954245035_0001
17/07/03 18:17:48 INFO mapred.LocalJobRunner: OutputCommitter set in config null
...
```

Al termine dell'esecuzione, i risultati sono disponibili in HDFS nella cartella
`/example/LogAnalyzerOutput`, come specificato nei parametri d'esecuzione. I
risultati si trovano in una cartella perché possono essere composti da più
file, uno per ogni Reducer eseguito parallelamente dal framework. Questa
limitazione è dovuta ad HDFS, che restringe rigidamente l'accesso in
scrittura ai file a un solo utilizzatore. In questo caso, il job è stato
eseguito da un solo reducer, per cui i risultati si trovano in un unico file.
L'output dei reducer è salvato in file testuali con il nome `part-r-` seguito
da un numero sequenziale che identifica l'istanza del Reducer che lo ha
prodotto.

Eseguendo `ls` nella cartella di output si può effettivamente verificare la
presenza del file prodotto dal Reducer.

```sh 
$ hadoop fs -ls /example/LogAnalyzerOutput
Found 2 items
-rw-r--r--   3 heygent hdfs          0 2017-07-03 18:17 /example/.../_SUCCESS
-rw-r--r--   3 heygent hdfs     804597 2017-07-03 18:17 /example/.../part-r-0000
```

Assieme al risultato della computazione, MapReduce salva un file vuoto chiamato
`_SUCCESS`, utilizzabile per verificare programmaticamente se il job è andato a
buon fine. Consultando il file, si può osservare il risultato della
computazione eseguita.

```sh
...
/elv/DELTA/del181.gif	71
/elv/DELTA/del181s.gif	390
/elv/DELTA/deline.gif	84
/elv/DELTA/delseps.jpg	90
/elv/DELTA/delta.gif	1492
/elv/DELTA/delta.htm	267
/elv/DELTA/deprev.htm	71
/elv/DELTA/dsolids.jpg	84
/elv/DELTA/dsolidss.jpg	369
/elv/DELTA/euve.jpg	    36
/elv/DELTA/euves.jpg	357
/elv/DELTA/rosat.jpg	38
/elv/DELTA/rosats.jpg	366
/elv/DELTA/uncons.htm	163
...
```

\clearpage

### Modello di esecuzione di MapReduce

Il modello di programmazione di MapReduce è progettato per essere altamente
parallelizzabile, e in modo che sia possibile processare diverse parti
dell'input indipendentemente. Questo dato si riflette nel design del Mapper,
che riceve come input piccole porzioni del file letto, permettendo al framework
di assegnare l'elaborazione delle operazioni di Map a diversi processi
indipendenti.

MapReduce è implementato in YARN, e utilizza le sue astrazioni per
avvantaggiarsi della località dei dati, eseguendo i processi che riguardano una
certa porzione di input nei nodi che contengono i corrispondenti blocchi HDFS.
L'esecuzione dei lavori MapReduce avviene secondo i seguenti step:

 #. I file vengono partizionati da MapReduce in frammenti chiamati *split*, e
    per ognuno di questi MapReduce esegue un *map task* in un determinato nodo del
    cluster. Ogni map task può eseguire uno o più processi nel nodo in cui si
    trova, a seconda delle risorse assegnate da YARN.

    La dimensione degli split è configurabile, e non corrisponde
    necessariamente alla dimensione di un blocco HDFS, pur essendo questa
    l'opzione di default (128 MB). Con split della stessa dimensione dei blocchi,
    la maggior parte dei dati può essere processata dai nodi che contengono il
    blocco nel loro storage locale. È possibile configurare MapReduce per
    utilizzare split più grandi, ma se una parte dello split non si trova nel nodo
    in cui viene eseguito il map task, questa deve essere ricevuta tramite rete da
    un altro nodo nel cluster che la contiene, riducendo quindi la *data
    locality*.

 #. In ogni *map task*, lo split corrispondente viene diviso in più *record*, che
    corrispondono alle coppie ricevute in input dal Mapper. Il map task esegue
    il Mapper in uno o più processi del nodo in cui si trova, per poi salvare il
    loro output nello storage locale del nodo. Lo storage
    locale è più efficiente per la scrittura rispetto ad HDFS, ma non offre fault-tolerance, per cui
    in caso di fallimento del nodo che contiene i risultati di un *map task*,
    l'application master dell'applicazione MapReduce deve schedulare la sua
    riesecuzione.

    Oltre all'esecuzione dei Mapper, i nodi in questa fase ordinano la parte di
    output in loro possesso in base alla chiave. Questo permette di eseguire
    parallelamente buona parte del sorting dell'input dei Reducer.

 #. Quando non ci sono più *map task* da eseguire sull'input, l'application master
    inizia ad avviare i *reduce task*. I *reduce task* ricevono in input i
    risultati ordinati prodotti dai *map task* mano a mano che questi sono
    disponibili. Ogni *map task* può inviare coppie chiave-valore a ogni
    *reduce task*, a condizione che coppie con la stessa chiave finiscano
    sempre nello stesso reducer.
    
    Per fare questo, i nodi che eseguono *map task* dividono il loro output in
    partizioni, una per ogni Reducer. Ogni chiave delle coppie di output viene
    associata univocamente a una partizione, utilizzando la seguente
    funzione[@mapred-partition-function]:

    $$partitionId(K_i) = hash(K_i) \bmod partitionCount$$

    In questo modo, la stessa chiave è sempre associata alla stessa partizione
    in ogni nodo.

 #. I nodi che eseguono i Reducer ricevono dai nodi Mapper diversi insiemi
    ordinati di coppie chiave-valore. Questi gruppi vengono uniti tramite
    un'operazione di *merge*, analoga alla stessa operazione nel contesto del
    *mergesort*. Una volta ricevuti tutti i valori, il *reduce task* esegue i
    Reducer che computano l'output finale dell'applicazione.

## Spark

Le astrazioni fornite dal paradigma computazionale di MapReduce tolgono
dall'utente l'onere di pensare al dataset in elaborazione, astraendo
l'applicazione a una serie di elaborazioni su chiavi e valori. Questa
astrazione ha tuttavia un costo: l'utente non ha il controllo sulla gestione
del flusso dei dati, che è affidata interamente dal framework.

Il costo della semplificazione diventa evidente quando si cerca di utilizzare
MapReduce per eseguire operazioni che richiedono la rielaborazione di
risultati. Al termine di ogni job MapReduce, l'output viene salvato in HDFS, ed
è quindi necessario rileggerlo dal filesystem per poterlo utilizzare.

Di per sé, MapReduce non contiene un meccanismo che permetta la schedulazione
consecutiva di job che ricevono in input l'output di un altro job, e per
eseguire elaborazioni che richiedono più fasi è necessario utilizzare tool
esterni. Inoltre, l'overhead della lettura e scrittura in HDFS è alto, e
MapReduce non fornisce metodi per rielaborare i dati direttamente nella memoria
centrale.

Il creatore di Spark, Matei Zaharia[@rdd-conf], ha posto questo problema come
dovuto alla mancanza di *primitive efficienti per la condivisione di dati* in
MapReduce. Per come le interfacce di MapReduce sono poste, sarebbe anche
difficile crearne di nuove, data la mancanza di un'API che sia rappresentativa
del dataset invece che delle singole chiavi e valori.

Infine, la scrittura dei risultati delle computazioni in HDFS è necessaria per
fornire fault-tolerance su di questi, che andrebbero persi nel caso di un
fallimento di un nodo che mantiene i risultati nella memoria centrale. Un
sistema di elaborazione che agisca sulla memoria centrale deve necessariamente
avere un meccanismo di recupero da fault, per evitare che il fallimento di uno
dei singoli nodi coinvolti nella computazione renda necessario rieseguire
completamente l'applicazione.

Spark si propone come alternativa a MapReduce, con l'intenzione di offrire
soluzioni a questi problemi. Le soluzioni derivano da un approccio funzionale,
sfruttando strutture con semantica di immutabilibità per rappresentare i
dataset e API che utilizzano funzioni di ordine superiore per esprimere
concisamente le computazioni. L'astrazione principale del modello di Spark è il
Resilient Distributed Dataset, o RDD, che rappresenta una collezione immutabile
e distribuita di record di cui è composto un dataset o una sua rielaborazione.

Spark è scritto in Scala, e la sua esecuzione su Hadoop è gestita da YARN. YARN
non è l'unico motore di esecuzione di Spark, che può essere eseguito anche su
Apache Mesos o in modalità standalone, sia su cluster che su macchine singole.
Le API client di Spark sono canonicamente disponibili in Scala, Java, R e
Python.

Spark dispone anche di una modalità interattiva, in cui l'utente interagisce
con il framework tramite una shell REPL Scala o Python. Questa modalità
permette la prototipazione rapida di applicazioni, e abilita l'utilizzo di
paradigmi come l'**interactive data mining**, che consiste nell'eseguire
analisi sui dataset in via esploratoria, scegliendo quali operazioni
intraprendere mano a mano che si riceve il risultato delle elaborazioni
precedenti.

### RDD API

I Resilient Distributed Dataset sono degli oggetti che rappresentano un dataset
partizionato e distribuito, su cui è possibile eseguire operazioni
parallelamente. 

Gli RDD sono immutabili, e ogni computazione richiesta su di questi restituisce
un valore o un nuovo RDD. Le computazioni sono eseguite tramite metodi chiamati
sugli oggetti RDD, e si dividono in due categorie: **azioni** e
**trasformazioni**.

Le trasformazioni creano un nuovo RDD, basato su delle operazioni
deterministiche sull'RDD di origine. L'elaborazione del nuovo RDD è lazy, e
non viene eseguita finché non viene richiesta l'esecuzione di un'azione. 

Alcuni esempi di trasformazioni sono `map`, che associa a ogni valore del
dataset un nuovo valore, e `filter`, che scarta dei valori nel dataset in base
a un predicato. Spesso, per descrivere le computazioni, le trasformazioni
richiedono in input funzioni pure (prive di side-effect).

+-------------------------+-------------------------------------------+
| Trasformazione          | Risultato                                 |
+=========================+===========================================+
| `map(fun)`              | Restituisce un nuovo RDD passando         |
|                         | ogni elemento della sorgente a `fun`.     |
+-------------------------+-------------------------------------------+
| `filter(fun)`           | Restituisce un RDD formato dagli elementi |
|                         | che `fun` mappa in `true`.                |
+-------------------------+-------------------------------------------+
| `union(dataset)`        | Restituisce un RDD che contiene gli       |
|                         | elementi della sorgente uniti con quelli  |
|                         | di `dataset`.                             |
+-------------------------+-------------------------------------------+
| `intersection(dataset)` | Restitusce un RDD contente gli elementi   |
|                         | comuni alla sorgente e a `dataset`        |
+-------------------------+-------------------------------------------+
| `distinct([numTasks]))` | Restituisce un RDD contentente gli        |
|                         | elementi del dataset senza ripetizioni    |
+-------------------------+-------------------------------------------+

: Alcune trasformazioni supportate da Spark

Le azioni fanno scattare la valutazione dell'RDD, che porta quindi
all'esecuzione di tutte le trasformazioni da cui questo è derivato. Alcuni
esempi di azioni sono `foreach`, che esegue una funzione specificata
dall'utente per ogni input del dataset, `reduce`, che utilizza una funzione di
input per aggregare i valori del dataset, e `saveAsTextFile`, che permette il
salvataggio di un RDD in un file testuale.

Ogni sessione interattiva e programma Spark utilizza un oggetto `SparkContext`
per creare gli RDD iniziali. Lo `SparkContext` contiene le impostazioni
principali sul programma, come il master di esecuzione (`local`, `yarn`,
`mesos`) e l'identificativo con cui tracciare il job in esecuzione. Le sessioni
interattive forniscono lo `SparkContext` automaticamente, in una variabile
globale chiamata `sc`.

Le sessioni interattive Spark possono essere avviate tramite i comandi
`spark-shell`, che mette a disposizione una shell REPL Scala, o `pyspark`, che
ne mette a disposizione una Python. Tramite gli argomenti dell'eseguibile si
può specificare il master (di default `local`).

![Avvio di una sessione interattiva Spark.](img/spark-shell.png)

Tramite l'oggetto `sc`, si può creare un nuovo RDD a partire da diversi fonti.
Il seguente codice crea un RDD partendo da un range inclusivo Scala, analogo
allo stesso concetto in Python.

```scala
scala> val range = sc.parallelize(1 to 50)
range: org.apache.spark.rdd.RDD[Int] = 
  ParallelCollectionRDD[0] at parallelize at <console>:24
```

Per creare un RDD a partire da un percorso, `SparkContext` fornisce metodi come
`textFile(path: String)`, che permette la lettura di file di testo da storage
locali e distribuiti, avvantaggiandosi della *data locality* quando possibile,
o `hadoopRDD(job: JobConf)`, che permette l'utilizzo di qualunque `InputFormat`
Hadoop per creare il dataset.
Nel seguente esempio si crea un RDD a partire dalla versione testuale inglese
del libro *Le metamorfosi* di Franz Kafka, offerto gratuitamente dal Progetto
Gutenberg[@kafka-metamorphosis].

```scala
scala> val book = sc.textFile("/books/kafka-metamorphosis.txt")
book: org.apache.spark.rdd.RDD[String] = 
    /books/kafka-metamorphosis.txt MapPartitionsRDD[28] at textFile 
    at <console>:24
```

Una volta ottenuto l'RDD, è possibile iniziare a eseguirvi trasformazioni. È
importante tenere in conto che ogni trasformazione restituisce un nuovo RDD, di 
cui è necessario salvare un riferimento per poterlo utilizzare in seguito.
Nelle sessioni interattive Scala i risultati di tutte le espressioni valutate
nella shell sono disponibili in variabili con il nome `res` seguito da un
identificativo numerico sequenziale, utilizzabili per tenere traccia degli RDD
valutati.

```scala
scala> val words = book.flatMap(_.split(' ')).filter(_ != "")
words: org.apache.spark.rdd.RDD[String] = 
    MapPartitionsRDD[30] at filter at <console>:26
```

La trasformazione `flatMap` riceve in input una funzione[^scalafn] che
restituisce un iterabile di elementi. La funzione viene chiamata su tutti gli
elementi dell'RDD, e i valori contenuti negli iterabili restituiti sono
raggruppati nell'RDD restituito. Con la funzione `_.split(' ')` si separano le
parole in ogni riga del libro. Viene poi eseguita la trasformazione `filter`
con il predicato `_ != ""`, per scartare le stringhe vuote che possono
risultare dalla trasformazione precedente[^simple].

[^scalafn]: Scala supporta una sintassi concisa per la creazione di funzioni
anonime, definibili con espressioni che utilizzano l'identificativo `_` come
valori. A ogni utilizzo di `_` corrisponde un parametro della funzione, che
viene sostituito nella rispettiva posizione alla chiamata della funzione.

[^simple]: Per semplificare l'esempio, si ignora il casing e la punteggiatura
delle parole, che andrebbero altrimenti normalizzate per ottenere un risultato
corretto sul conteggio delle parole.

Se non vengono fatte specificazioni, l'esecuzione delle trasformazioni avviene
ogni volta che viene chiamata un'azione su di un RDD. Per evitare
ricomputazioni costose, è possibile specificare quali RDD persistere nella
memoria dei nodi, in modo che i risultati computati possano essere riutilizzati
in operazioni successive. Per richiedere al framework di salvare i valori
computati di un RDD, è sufficiente chiamare il suo metodo `persist`.

```scala
scala> words.persist()
res10: words.type = MapPartitionsRDD[30] at filter at <console>:26
```

Se il dataset da salvare è molto grande le partizioni potrebbero non entrare
completamente in memoria. Il comportamento di default di Spark in questo caso
consiste nel mettere in cache solo parte della partizione, e ricomputare la
parte restante quando viene richiesta. Spark può anche eseguire azioni
alternative, come serializzare gli oggetti in modo che occupino meno spazio o
eseguire parte del caching su disco. Nel gergo di Spark il comportamento da
attuare in questi casi è definito **livello di persistenza**, ed è
specificabile come argomento del metodo `persist`.

+------------------------+-----------------------------------------------------------------------------+
| Livello di persistenza | Effetto                                                                     |
+========================+=============================================================================+
| `MEMORY_ONLY`          |                                                                             |
|                        | Salva l'RDD sotto forma di oggetti deserializzati nella JVM. Se l'RDD non   |
|                        | entra in memoria, alcune partizioni non vengono persistite e vengono        |
|                        | ricomputate al volo ogni volta che sono richieste. (default)                |
+------------------------+-----------------------------------------------------------------------------+
| `MEMORY_AND_DISK`      |                                                                             |
|                        | Salva l'RDD sotto forma di oggetti deserializzati nella JVM. Se l'RDD non   |
|                        | entra in memoria, alcune partizioni vengono scritte su disco e lette quando |
|                        | sono richieste.                                                             |
+------------------------+-----------------------------------------------------------------------------+
| `MEMORY_ONLY_SER`      |                                                                             |
| (solo Java e Scala)    | Salva l'RDD come oggetti Java serializzati. Questa opzione è più efficiente |
|                        | in termini di spazio degli oggetti deserializzati, ma più                   |
|                        | computazionalmente intensiva.                                               |
+------------------------+-----------------------------------------------------------------------------+
| `MEMORY_AND_DISK_SR`   |                                                                             |
|                        | Come `MEMORY_ONLY_SER`, ma le partizioni che non entrano in memoria sono    |
|                        | salvate su disco invece di essere ricomputate.                              |
+------------------------+-----------------------------------------------------------------------------+
| `DISK_ONLY`            | Salva le partizioni solo su disco.                                          |
+------------------------+-----------------------------------------------------------------------------+

: Alcuni livelli di persistenza forniti da Spark.

Per avviare l'esecuzione delle trasformazioni, è necessario eseguire un'azione.
Nel seguente esempio, l'azione eseguita è `take(n: Int)`, che restituisce i
primi `n` elementi dell'RDD in un array Scala.

```scala
scala> words.take(20)
res19: Array[String] = Array(One, morning,, when, Gregor, Samsa, woke,
    from, troubled, dreams,, he, found, himself, transformed, in, his,
    bed, into, a, horrible, vermin.)
```

Il file di origine è stato diviso in parole, come specificato nelle
trasformazioni. Dato che è stato chiamato `persist` sull'RDD `words`, i valori
rielaborati si trovano ancora nella memoria dei nodi, ed è possibile
riutilizzarli semplicemente eseguendo operazioni sull'RDD.

Tramite le interfacce di Spark si può facilmente rappresentare il modello
computazionale di MapReduce. A partire da `words`, si può eseguire il conto
delle parole all'interno del libro mappando il dataset a coppie chiave-valore,
dove la chiave è la parola e il valore è 1. Per eseguire il conto, gli RDD
forniscono il metodo `reduceByKey`, che esegue la stessa operazione effettuata
dai Reducer nel modello MapReduce: aggrega i valori delle coppie con la stessa
chiave.

Diversamente da MapReduce, in `reduceByKey` la funzione che rappresnta il
Reducer non riceve un iterabile dei valori, ma un accumulatore e uno degli
elementi aggregati. Per ogni gruppo di valori aggregati a una chiave, la
funzione viene chiamata per ogni valore del gruppo, ricevendolo in input
assieme all'accumulatore. Il suo valore di restituzione viene utilizzato come
accumulatore di input per l'invocazione sul valore successivo.

$$reducer(A_i, V_i) = A_{i + 1}$$ 

Le coppie chiave-valore possono essere rappresentate con tuple Scala. La
funzione di riduzione da utilizzare in questo caso è la somma.

```scala
scala> val wordCount = words.map(w => (w, 1)).reduceByKey(_ + _)
wordCount: org.apache.spark.rdd.RDD[(String, Int)] =
    ShuffledRDD[61] at reduceByKey at <console>:28

scala> wordCount.take(20)
res21: Array[(String, Int)] = Array(
    (swishing,,1), (pitifully,1), (someone,5), (better.,2), (propped,1),
    (nonetheless,3), (bone,1), (movements.,2), (order,7), (drink,,1),
    (experience,,1), (behind,15), (Father,,1), (wasn't,5), (been,99),'
    (they,,1), (Father.,1), (introduction,,1), ("Gregor,,3), (she's,1)
    )
```

Al termine della computazione, si rende utile salvare i valori in un mezzo di
storage. Questa operazione è eseguibile tramite diverse azioni, come
`saveAsTextFile(path: String)`, che salva i risultati come testo, o
`saveAsObjectFile(path: String)`, che serializza efficientemente i valori,
permettendone un rapido accesso programmatico.

```scala
scala> wordCount.saveAsTextFile("/tmp/results")
```

Come per MapReduce, i risultati possono essere sparsi per diversi file, a
seconda di quanti task paralleli sono stati coinvolti nell'operazione di
riduzione. 

```sh
$ cd /tmp/results 
$ ls
part-00000  part-00001  _SUCCESS
$ head part-00000
(swishing,,1)
(pitifully,1)
(someone,5)
(better.,2)
(propped,1)
(nonetheless,3)
(bone,1)
(movements.,2)
(order,7)
(drink,,1)
```

#### Sviluppo ed esecuzione di un Job

Le applicazioni Spark vengono sviluppate ed eseguite in modo simile a
MapReduce. Ogni applicazione ha un punto di entrata `main`, dove viene inserito
il codice relativo all'esecuzione del job. Le API sono simili 

Riprendendo l'esempio dell'[analizzatore di log](#mapred-example), questo è
rappresentabile in maniera molto più succinta tramite le interfacce fornite da
Scala e Spark. La prima operazione da eseguire è creare un oggetto di
configurazione, come mostrato in [@lst:log-analyzer-spark], righe 9-11. In
questo caso, si specifica il nome del job come "Log Analyzer" e `yarn` come
master di esecuzione. Nella riga 13 si crea uno `SparkContext` utilizzando la
configurazione, che viene poi utilizzato per aprire un file HDFS il cui
percorso è preso in input dagli argomenti del programma.

L'RDD ottenuto viene utilizzato per mappare ogni riga di richiesta alla risorsa
corrispondente. La trasformazione `collect` riceve in input una funzione
parziale Scala, che in questo caso tenta di eseguire il match delle righe con
l'espressione regolare e di estrarre il valore corrispondente al gruppo di
cattura. Sulle righe in cui il match è valido, la funzione restituisce una
coppia URI-1. Per eseguire il calcolo finale, si utilizza l'azione
`reduceByKey`.

Prima di salvare il file, il risultato viene mappato a una stringa, la cui
formattazione permette all'output di essere un valido file TSV (Tab Separated
Values).

```{#lst:log-analyzer-spark .scala .numberLines}
import org.apache.spark._

object LogAnalyzer {

  private val logURIRegex = """.*"[A-Z]+\s(.*)\sHTTP.*""".r

  def main(args: Array[String]) {

    val conf = new SparkConf()
      .setAppName("Log Analyzer")
      .setMaster("yarn")

    val sc = new SparkContext(conf)

    val logs = sc.textFile(args(0))

    val counts = logs
      .collect {
        case logURIRegex(uri) => (uri, 1)
      }
      .reduceByKey(_ + _)

    counts.map { case (k, v) => s"$k\t$v" }.saveAsTextFile(args(1))

  }
}
```

: Analizzatore di log reimplementato in Scala e Spark.

Per essere eseguito, il file deve essere compilato e impacchettato in un file
`jar`. La richiesta di esecuzione di un job può essere fatta tramite
l'eseguibile `spark-submit`, distribuito con Spark. Come per l'eseguibile
`hadoop`, è possibile specificare gli argomenti che si vogliono passare al
programma. 

```sh
$ spark-submit sparkdemo-assembly-1.0.jar /example/NASA_access_log_Jul95
    /tmp/results
```

Una volta inviato, lo stato di un Job può essere consultato tramite
un'interfaccia web, come in MapReduce. L'interfaccia mostra la fase di
esecuzione del Job, e può fornire visualizzazioni in forma di grafo diretto
aciclico degli stage richiesti per la sua esecuzione.

![Interfaccia web di Spark, tracciamento dell'esecuzione dei
job.](img/spark-web-ui.png)

![Interfaccia web di Spark, visualizzazione DAG delle
operazioni.](img/spark-dag-view.png){#fig:dag-visualization}

I risultati possono essere quindi consultati nella cartella `/tmp/results`
dell'istanza HDFS del cluster.

```sh
[root@sandbox ~]# hadoop fs -cat /tmp/results/part-00001 | head
/cgi-bin/imagemap/countdown?112,206	2
/shuttle/missions/41-b/images/	19
/cgi-bin/imagemap/countdown?292,205	2
/cgi-bin/imagemap/countdown70?248,269	1
/history/apollo/apollo-13.apollo-13.html	1
/cgi-bin/imagemap/countdown?220,280	1
/news/sci.space.shuttle/archive/sci-space-shuttle-7-feb-1994-87.txt	1
/htbin/wais.pl?current+position	1
/cgi-bin/imagemap/countdown?105,213	2
/cgi-bin/imagemap/fr?280,27	1
```

### DataFrame API

Spark fornisce un'altra API per l'elaborazione dei dati, basata sull'astrazione
del DataFrame. Un DataFrame rappresenta dati strutturati o semistrutturati,
come documenti JSON o CSV, di cui Spark è internamente consapevole della struttura.
Utilizzando i DataFrame, Spark è in grado di eseguire ottimizzazioni e di
fornire operazioni aggiuntive all'utente. Una delle funzioni notevoli
dell'API DataFrame è la possibilità di eseguire query SQL sui dataset,
utilizzando anche funzioni di aggregazione come `sum`, `avg` e `max`.

Le API DataFrame sono disponibili nelle sessioni interattive Spark tramite un
oggetto di `SparkSession`, fornito nella variabile `spark`. In modo analogo
agli RDD, le azioni eseguibili sui DataFrame possono essere azioni e
trasformazioni.

La creazione di DataFrame è simile alla creazione degli RDD, e avviene
utilizzando l'oggetto `spark` per caricare i dati da una sorgente. I dati di
questo esempio sono forniti da Population.io, un servizio che fornisce dati
aggiornati sulla popolazione mondiale[@population-io]. Il dataset che viene
caricato è un file JSON contenente i dati sulla popolazione degli Stati Uniti
nell'anno 2017, strutturato come un array di oggetti con i seguenti campi:

* `age`: Fascia di età a cui l'oggetto si riferisce
* `females`: Numero di donne
* `males`: Numero di uomini
* `total`: Totale di donne e uomini
* `country`: Nazione di riferimento (in questo caso sempre Stati Uniti)
* `year`: Anno di riferimento (2017)

Il seguente codice carica il DataFrame in memoria:

```scala
scala> val population = spark.read.json("us_population.json")
population: org.apache.spark.sql.DataFrame = 
    [age: bigint, country: string ... 4 more fields]
```

La stringa rappresentativa del dataframe visualizzata in risposta dà qualche
indizio sulla struttura rilevata. Si può richiedere al DataFrame di
visualizzare la sua intera struttura:

```scala
scala> population.printSchema()
root
 |-- age: long (nullable = true)
 |-- country: string (nullable = true)
 |-- females: long (nullable = true)
 |-- males: long (nullable = true)
 |-- total: long (nullable = true)
 |-- year: long (nullable = true)
```

Spark ha interpretato correntamente il file, distinguendo tra i campi numerici
e stringa. Si può stampare il DataFrame utilizzando il metodo `show`:

```scala
scala> population.show(10)
+---+-------------+-------+-------+-------+----+
|age|      country|females|  males|  total|year|
+---+-------------+-------+-------+-------+----+
|  0|United States|1953000|2044000|3997000|2017|
|  1|United States|1950000|2041000|3991000|2017|
|  2|United States|1889000|1977000|3866000|2017|
|  3|United States|1918000|2006000|3925000|2017|
|  4|United States|1946000|2034000|3980000|2017|
|  5|United States|1972000|2060000|4032000|2017|
|  6|United States|1996000|2083000|4079000|2017|
|  7|United States|2018000|2104000|4123000|2017|
|  8|United States|2040000|2125000|4165000|2017|
|  9|United States|2055000|2139000|4194000|2017|
+---+-------------+-------+-------+-------+----+
only showing top 10 rows
```

I DataFrame sono implementati tramite RDD, il cui accesso è reso disponibile
agli utenti. Gli RDD dei DataFrame sono composti da oggetti di tipo `spark.sql.Row`,
che possono essere indirizzati in modo analogo agli array Scala. Il seguente
esempio utilizza l'RDD del DataFrame per ottenere la terza colonna,
corrispondente alla popolazione femminile, dei primi 10 elementi del DataFrame.

```scala
scala> population.rdd.take(10).map(_(2))
res30: Array[Any] = Array(1953000, 1950000, 1889000, 1918000,
    1946000, 1972000, 1996000, 2018000, 2040000, 2055000)
```

Un modo più idiomatico per accedere ai dati del DataFrame è utilizzare i metodi
da esso forniti. I DataFrame espongono un DSL ispirato a SQL come metodo di
accesso ai dati, che nel seguente esempio viene utilizzato per selezionare la
popolazione di adulti maschi di età compresa tra i 20 e i 30 anni.

```scala
scala> val adult_males = population.select($"males")
    .filter($"age" >= 20 && $"age" <= 30)

adult_males: org.apache.spark.sql.Dataset[org.apache.spark.sql.Row] =
    [males: bigint]

scala> adult_males.show
+-------+
|  males|
+-------+
|2175000|
|2262000|
|2344000|
|2432000|
|2479000|
|2460000|
|2400000|
|2344000|
|2280000|
|2238000|
|2235000|
+-------+
```

I numeri così ottenuti possono essere sommati per ottenere il numero totale di
adulti in questa fascia d'età.

```scala
scala> adult_males.agg(sum("males")).first.get(0)
res46: Any = 25649000
```

Oltre al DSL fornito dal DataFrame, Spark supporta l'esecuzione diretta di
query SQL specificate tramite stringhe. Le query vengono eseguite su degli
oggetti definiti view, che vengono mantenuti globalmente in memoria da Spark
fino alla fine della sessione[^hive].

Per creare una view, si può chiamare `createOrReplaceTempView(name: String)`
sul DataFrame di interesse. Una volta creata, la view è accessibile nelle query
SQL con il nome specificato in `name`. 

Le query possono essere eseguite chiamando il metodo `spark.sql(query:
String)`, che restituisce il loro risultato sotto forma di DataFrame. Le query
possono essere utilizzate per eseguire computazioni di vario tipo, utilizzando
le funzioni di aggregazione fornite. Il seguente codice esegue e mostra il
risultato di una query, che richiede la somme delle popolazioni maschile e
femminile di età compresa tra i 40 e i 60 anni.

```scala
scala> population.createOrReplaceTempView("population")

scala> spark.sql("""
     | SELECT SUM(males), SUM(females) FROM population
     | WHERE age >= 40 AND age <= 60
     | """).show
+----------+------------+
|sum(males)|sum(females)|
+----------+------------+
|  44632000|    45014000|
+----------+------------+
```


[^hive]: Spark supporta anche l'integrazione con Hive, un engine SQL progettato
per Hadoop, che può essere utilizzato per creare e mantenere tabelle
persistenti.

### Modello di esecuzione

Spark utilizza diversi componenti nella sua esecuzione[@learning-apache-spark]:

![Diagramma del modello di esecuzione di Apache
Spark[@spark-cluster].](img/spark-execution.png)

* Il **driver** è il programma principale delle applicazioni, ed è definito
  dagli utenti tramite le interfacce della libreria client di Spark. Il driver
  coordina, tramite l'oggetto `SparkContext`, i processi in esecuzione nel
  cluster, comunicando con il *cluster manager* in utilizzo.

* Il **cluster manager** è l'entità che esegue allocazioni di risorse nel
  cluster, che può essere YARN, Mesos, o il cluster manager integrato in Spark.

* I **worker node** sono processi avviati nelle macchine coinvolte nella
  computazione distribuita nel cluster e che ne gestiscono le risorse.

* Gli **executor** sono processi allocati all'interno delle macchine worker per
  eseguire i task assegnati dal driver. Le applicazioni Spark eseguono gli
  *executor* al loro avvio, e li terminano a fine computazione.

* I **task** sono le unità di lavoro eseguite dai singoli worker, che vengono
  inviati sotto forma di funzioni serializzate. Gli *executor* deserializzano i
  *task*, per poi eseguirli su partizioni di dataset.


Le computazioni sono definite tramite le funzioni passate come parametro ad
azioni e trasformazioni. Spark tiene traccia di queste tramite un grafo,
definito **lineage**. Tramite il grafo di *lineage*, Spark crea un *execution
plan*, per determinare come l'esecuzione debba essere organizzata nei nodi.
Il criterio utilizzato è di eseguire quante più computazioni possibili in
uno stesso nodo, per ridurre gli spostamenti che richiederebbero banda di rete.
Alcune operazioni, come `reduce`, richiedono necessariamente lo spostamento dei
dati in rete, dato che devono trovarsi nello stesso nodo per poter essere aggregati.
Questo step è definito *shuffle*, ed è simile all'operazione eseguita da
MapReduce per partizionare i risultati dei Mapper.

Per ogni job, Spark esegue una divisione logica sulle operazioni da eseguire,
che vengono raggruppate in un grafo aciclico, i cui nodi sono *execution
phases*. Ogni fase di esecuzione raggruppa quante più operazioni possibili, e
le fasi sono separate l'una dall'altra solo da operazioni che richiedono
l'esecuzione di uno *shuffle*. Il grafo delle fasi di esecuzione dei job è
visibile nell'interfaccia web di monitoraggio dei job, come mostrato in
[@fig:dag-visualization] per l'analizzatore di log.

Spark utilizza il grafo di lineage anche per fornire fault-tolerance:
nell'eventualità in cui un nodo contenente una partizione di un RDD
dovesse fallire, Spark può retrocedere agli RDD genitori sul grafo di lineage,
fino a trovare degli RDD candidati da cui si può ricavare la partizione non più
disponibile. Dal grafo si possono ricavare quali sono le operazioni che hanno
prodotto la partizione dell'RDD nella memoria del nodo in stato di fault, che
possono quindi essere rischedulate per riottenere la partizione persa. Gli RDD
di cui è stato eseguito il caching sono buoni candidati per ricavare la
partizione non più disponibile.

