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
funzionali rispetto a MapReduce, e prestazioni molto più elevate in molti
algoritmi[@mapreduce-spark-performance]. Tramite astrazioni che offrono un
controllo più preciso sul comportamento dei risultati dell'elaborazione, Spark
trova moltissime applicazioni pratiche sia negli ambiti , tra cui il machine
learning

In questa sezione si esaminano MapReduce e Spark, quali sono le limitazioni di
MapReduce che hanno richiesto la necessità di un nuovo modello computazionale,
e quale soluzioni sono offerte da Spark.

## MapReduce

Il modello computazionale di MapReduce è composto, nella sostanza, da due
componenti, che sono intuitivamente il Mapper e il Reducer. 

Il Mapper è una classe contenente una funzione `map`, che riceve in input una
coppia composta da chiave e valore, e che restituisce a sua volta zero, uno, o
più coppie di chiavi e valori[^4]. Le chiavi e i valori ricevuti in input dal
Mapper sono derivati direttamente dall'elemento letto in HDFS. Nel caso dei
file di testo, ad esempio, la chiave è un intero che rappresenta la riga del
file letto, e il valore è la riga di testo. È possibile configurare quali
chiavi e valori vengano derivati dalla sorgente e come, creando una classe che
implementa l'interfaccia `InputMapper` fornita nella libreria di Hadoop. 

Le applicazioni MapReduce specificano un proprio Mapper estendendo la classe
`Mapper` nella libreria di Hadoop, e specificando i tipi dei parametri
generici opportunamente. La firma di `Mapper` è la seguente:

```java
public class Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT> extends Object
```

I tipi di `KEYIN` e `VALUEIN` sono gli input della funzione `map` del Mapper, e
devono corrispondere ai tipi che l'`InputFormat` di riferimento restituisce.
`KEYOUT` e `VALUEOUT` sono invece i tipi che il Mapper restituisce rielaborando
le chiavi e i valori in input. `map` ha la seguente signature:

```java
protected void map(KEYIN key, VALUEIN value, Context context) 
    throws IOException, InterruptedException
```

Una volta restituiti dal Mapper, le coppie vengono date in input a una classe
`Reducer`, che ha una signature simile:

```java
public class Reducer<KEYIN,VALUEIN,KEYOUT,VALUEOUT>
```

Una classe che estende `Reducer` ha un metodo `reduce`, che diversamente dal
metodo `map` riceve in input una chiave, e un iterabile di tutti i valori che
hanno quella stessa chiave:

```java
protected void reduce(KEYIN key, Iterable<VALUEIN> values, Context context) 
    throws IOException, InterruptedException
```

Nella fase di reduce, quindi, i valori sono **aggregati** in base alla chiave.
I valori a questo punto possono essere combinati a seconda dell'esigenza
dell'utente per restituire un risultato finale.

[^4]: Dato che Java non permette la restituzione di valori multipli da una
funzione, per "restituire" i valori si usa il metodo `Context.write`
dell'oggetto `Context` ricevuto in input da `map` e `reduce`. È comunque
intuitivo pensare ai Mapper e ai Reducer come entità che eseguono associazioni
da valore a valore.


[^5]: MapReduce permette l'elaborazione di diversi tipi di file, tra cui testo,
file colonnari come ORC, e tabelle di strumenti come HBase o Hive. A seconda
della sorgente di input, i valori di `KEYIN` e `VALUEIN` cambiano per fornire
dati quanto più possibilmente significativi rispetto alla sorgente.

### Esempio di un programma MapReduce

Come esempio di programma per MapReduce, si prende in considerazione l'analisi
di log di un web server. Il dataset su cui si esegue l'elaborazione è fornito
liberamente dalla NASA[@nasa-weblog], e corrisponde ai log di accesso al server HTTP del NASA
Kennedy Space Center dal 1/07/1995 al 31/07/1995. Il log è un file di testo
con codifica ASCII, dove ogni riga corrisponde a una richiesta e ognuna di
queste contiene le seguenti informazioni:

#. L'host che esegue la richiesta, sottoforma di hostname quando disponibile o
   indirizzo IP altrimenti
#. Timestamp della richiesta, in formato "`WEEKDAY MONTH DAY HH:MM:SS YYYY`" e
   fuso orario, con valore fisso `-0400`.
#. La Request-Line HTTP tra virgolette
#. Il codice HTTP di risposta
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

```{#lst:log-mapper .java .numberLines}
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
rete, le cui funzionalità sono accessibili tramite l'interfaccia
`hadoop.io.Writable`. Le classi `LongWritable` e `Text` sono dei wrapper sui
tipi `long` e `String` che forniscono i metodi richiesti dall'interfaccia di
serializzazione, e i valori contenuti in questi tipi possono essere ottenuti
rispettivamente con `LongWritable.get()` e `Text.toString()`.

[^6]: Le classi definite dagli utenti possono implementare a loro volta
l'interfaccia `Writable` per essere supportate come tipi di chiavi e valori nei
Mapper e nei Reducer. 

Per il resto, le operazioni del Mapper sono intuitive: si utilizza
l'espressione regolare per ottenere il token contenente l'URI della richiesta,
e tramite `context.write` il Mapper invia la coppia URI e 1.

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
URI-accumulatore. L'insieme di tutti i valori restituiti dal Reducer
costituiscono l'output finale del programma.

Prima di poter eseguire l'applicazione, è necessario creare un esecutore,
ovvero una classe contenente un punto d'entrata `main` che utilizzi le API di
Hadoop per eseguire il programma, analogamente a come descritto in
[Esecuzione di software in Hadoop]. I lavori MapReduce sono configurati tramite
l'oggetto `hadoop.mapreduce.Job`, che richiede di specificare le classi da
utilizzare come Mapper e Reducer, assieme ai percorsi dei file da elaborare.
L'esecutore dell'analizzatore di log è mostrato in [@lst:log-executor].

```{#lst:log-executor .numberLines .java}
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

Il metodo `job.waitForCompletion` è stato invocato con il parametro `verbose`
impostato a `true`, per cui l'esecuzione stampa in output un log sul job in
esecuzione. È anche possibile verificare lo stato di esecuzione dei job tramite
un'interfaccia web fornita dal framework.

Al termine dell'esecuzione, i risultati sono disponibili in HDFS nella cartella
`/example/LogAnalyzerOutput`, come specificato nei parametri d'esecuzione. I
risultati si trovano in una cartella perché questi sono composti da più file,
uno per ogni Reducer eseguito parallelamente dal framework. In questo caso, il
job è stato eseguito da un solo reducer, per cui i risultati si trovano in un
unico file. È possibile scegliere la quanitità di Reducer da eseguire
parallelamente nel framework, mentre i Mapper, come si vedrà, sono stabiliti
in base all'input.

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



### Astrazioni su MapReduce

## Spark

