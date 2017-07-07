# Batch Processing

Il Batch Processing è la *raison d'être* di Hadoop. Il primo paradigma di
programmazione per Hadoop, MapReduce, è stato l'unico per molte release, e
ha avuto il grande merito di astrarre la complessità della
computazione batch in ambiente distribuito in funzioni che associano chiavi e
valori a risultati, una grande semplificazione rispetto ai programmi che
gestiscono granularmente l'intricatezza di ambienti distribuiti.

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
direttamente in Hadoop, a testimoniare l'effettiva capacità di YARN di
generalizzare i modelli di esecuzione nei cluster.

La sua alternativa più popolare, Apache Spark, ha API più espressive e
funzionali rispetto a MapReduce, ed è più performante in molti tipi di
algoritmi[@mapreduce-spark-performance]. Tramite astrazioni che offrono un
controllo più preciso sul comportamento dei risultati dell'elaborazione, Spark
trova applicazioni pratiche in vari ambiti, tra cui machine
learning[@spark-mllib], graph processing[@spark-graphx] e elaborazione
SQL[@spark-sql].

In questa sezione si esaminano MapReduce e Spark, quali sono le limitazioni di
MapReduce che hanno fatto sentire la necessità di un nuovo modello
computazionale, e quali sono le soluzioni offerte da Spark. Si accenneranno
anche ad alcune astrazioni fatte al di sopra di MapReduce, come Pig e Hive, che
forniscono dei modelli computazionali che vengono tradotti in job MapReduce. 

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
restituiti da $Map$ che hanno la stessa chiave $K_2$. $Reduce$ esegue una
computazione sui valori di input e restituisce $(K_2, V_3)$, che andrà a far
parte dell'output finale dell'applicazione assieme al risultato delle altre
invocazioni di $Reduce$, una per ogni chiave distinta restituita da $Map$.

Sintetizzando, MapReduce permette di categorizzare l'input in diverse parti e
di elaborare un risultato per ognuna di queste.

MapReduce è quindi un paradigma *funzionale*, dato che il framework richiede di
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

Nella fase di reduce, quindi, i valori sono aggregati in base alla chiave e
resi disponibili tramite l'interfaccia `Iterable` di Java.
I valori a questo punto possono essere combinati a seconda dell'esigenza
dell'utente per restituire un risultato finale.

### Esempio di un programma MapReduce

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
importante data la crucialità in termini di efficienza di elaborazione che
questa può avere. Le funzionalità di serializzazione di Hadoop sono rese
accessibili dagli oggetti serializzabili tramite l'interfaccia
`hadoop.io.Writable`. Le classi `LongWritable` e `Text` sono dei wrapper sui
tipi `long` e `String` che implementano l'interfaccia `Writable`, e i valori
contenuti in questi tipi possono essere ottenuti rispettivamente con
`LongWritable.get()` e `Text.toString()`[^6].

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
risultati si trovano in una cartella perché questi sono composti da più file,
uno per ogni Reducer eseguito parallelamente dal framework. In questo caso, il
job è stato eseguito da un solo reducer, per cui i risultati si trovano in un
unico file. 

Eseguendo `ls` nella cartella di output si può effettivamente verificare la
presenza del file prodotto dal Reducer.

```sh 
$ hadoop fs -ls /example/LogAnalyzerOutput
Found 2 items
-rw-r--r--   3 heygent hdfs          0 2017-07-03 18:17 /example/.../_SUCCESS
-rw-r--r--   3 heygent hdfs     804597 2017-07-03 18:17 /example/.../part-r-0000
```

Assieme al risultato della computazione, MapReduce salva un file vuoto chiamato
`_SUCCESS`, di cui si può verificare la presenza in HDFS per capire se il job è
andato a buon fine.
Consultando il file, si può osservare il risultato della computazione eseguita.

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

### Efficienza di MapReduce

Il modello di programmazione di MapReduce è progettato per essere altamente
parallelizzabile e in modo che sia possibile processare diverse parti
dell'input indipendentemente. Questo dato si riflette nel design del Mapper,
che riceve come input piccole porzioni del file letto, permettendo al framework
di assegnare l'elaborazione delle operazioni di Map a diversi processi
indipendenti.

MapReduce è implementato in YARN, e utilizza le sue astrazioni per
avvantaggiarsi della località dei dati, eseguendo i processi che riguardano una
certa porzione di input nei nodi che contengono i corrispondenti blocchi HDFS.

I file vengono partizionati da MapReduce in frammenti chiamati *split*, e per
ognuno di questi MapReduce esegue un *map task* in un determinato nodo del
cluster. Ogni map task può eseguire uno o più processi nel nodo in cui si
trova, a seconda delle risorse assegnate da YARN.

La dimensione degli split è configurabile, e non corrisponde necessariamente
alla dimensione di un blocco HDFS, pur essendo questa l'opzione di default. Con
split della stessa dimensione dei blocchi, la maggior parte dei dati può essere
processata dai nodi che contengono il blocco nel loro storage locale. È
possibile configurare MapReduce per utilizzare split più grandi, ma se una
parte dello split non si trova nel nodo in cui viene eseguito il map task,
questa deve essere ricevuta tramite rete da un altro nodo nel cluster che la
contiene, con conseguente overhead.

In ogni *map task*, lo split corrispondente viene diviso in più *record*, che
corrispondono alle coppie ricevute in input dal Mapper[^7]. Il map task esegue
il Mapper in uno o più processi del nodo in cui si trova, per poi salvare il
loro output nello storage locale del nodo che esegue il map task. 

Una volta terminati i map task, il framework esegue i *reduce task*. Prima di
eseguire l'operazione di reduce, 

## Spark

Le astrazioni fornite dal paradigma computazionale di MapReduce tolgono
dall'utente l'onere di pensare al dataset in elaborazione, astraendo
l'applicazione a una serie di elaborazioni su chiavi e valori. Questa
astrazione ha tuttavia un costo: l'utente non ha il controllo sulla gestione
del flusso dei dati, che è gestita interamente dal framework.

Il costo della semplificazione diventa evidente quando si cerca di utilizzare
MapReduce per eseguire operazioni che richiedono la rielaborazione di
risultati. Al termine di ogni job MapReduce, questi vengono salvati in HDFS, ed
è quindi necessario rileggerli dal filesystem per poterli riutilizzare.

Di per sé, MapReduce non contiene un meccanismo che permetta la schedulazione
consecutiva di job che ricevono in input l'output di un altro job, e per
eseguire elaborazioni che richiedono più fasi è necessario utilizzare tool
esterni.

Inoltre, l'overhead della lettura e scrittura in HDFS è alto, e MapReduce non
fornisce metodi per rielaborare i dati nella memoria centrale prima della
scrittura in HDFS.

Il creatore di Spark, Matei Zaharia[@rdd-conf], ha posto questo problema come
dovuto alla mancanza di *primitive efficienti per la condivisione di dati* in
MapReduce. Per come le interfacce di MapReduce sono poste, sarebbe anche
difficile crearne di nuove, data la mancanza di un'API che sia rappresentativa
del dataset invece che delle singole chiavi e valori.

Infine, la scrittura dei risultati delle computazioni in HDFS è necessaria per
fornire fault-tolerance sui risultati delle computazioni, che andrebbero persi
nel caso di un fallimento di un nodo che mantiene i risultati nella memoria
centrale. Per poter avere

Spark si propone come alternativa a MapReduce, con l'intenzione di dare una
soluzione a questi problemi. Le soluzioni derivano da un approccio funzionale
alla computazione, sfruttando strutture dati immutabili per rappresentare i
dataset e API che utilizzano funzioni di ordine superiore per esprimere
concisamente le computazioni. L'astrazione principale del modello di Spark è
il Resilient Distributed Dataset, o RDD, che rappresenta una collezione
immutabile di record di cui è composto un dataset distribuito o una sua
rielaborazione.

Spark è scritto in Scala, e la sua esecuzione su Hadoop è gestita da YARN.
YARN non è l'unico motore di esecuzione di Spark, che può essere eseguito anche
su Apache Mesos o in modalità standalone, su cluster Spark dedicati.
Le API client di Spark sono disponibili in Scala, Java e Python.

Spark dispone anche di una modalità interattiva, in cui l'utente interagisce
con il framework tramite una shell REPL Scala o Python. Questa modalità
permette la prototipazione rapida di applicazioni, e abilita l'utilizzo di
paradigmi come l'**interactive data mining**, che consiste nell'eseguire
analisi sui dataset in via esploratoria, scegliendo quali operazioni
intraprendere mano a mano che si riceve il risultato delle elaborazioni
precedenti.

### Interfaccia di Spark

I Resilient Distributed Dataset sono degli oggetti che rappresentano un dataset
distribuito. Gli RDD possono essere creati a partire da HDFS, sfruttando
la data locality per la lettura, o da un qualunque elemento iterabile.
La creazione degli RDD è eseguita da un oggetto SparkContext, che 

(@)

L'API di Spark consiste di un oggetto `SparkSession`, 

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
