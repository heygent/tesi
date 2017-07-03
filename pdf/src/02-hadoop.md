# Hadoop

Nell'ambito dei Big Data, Hadoop è il perno centrale su cui è basato un
*ecosistema* di tool e tecnologie, tant'è che spesso il termine Hadoop viene
utilizzato per riferirsi all'intero ecosistema di tool e tecnologie construiti
attorno a questo.

La documentazione ufficiale[@hadoop-doc-main] lo descrive come:

> ...un framework che abilita l'elaborazione distribuita di grandi dataset
in cluster di computer utilizzando semplici modelli di programmazione.
[Hadoop] è progettato per essere scalato da server singoli a migliaia di
macchine, dove ognuna di queste offre computazione e storage locale. Invece di
affidarsi all'hardware per fornire un'alta affidabilità, [Hadoop] è progettato
per rilevare e gestire i fallimenti [delle computazioni] a livello applicativo,
mettendo a disposizione un servizio ad alta affidiabilità su cluster di
computer proni al fallimento.

In questa definizione sono racchiusi dei punti molti importanti:

 *  **Semplici modelli di programmazione**

    Hadoop raggiunge molti dei suoi obiettivi fornendo un'interfaccia di
    livello molto alto al programmatore, in modo di potersi assumere la
    responsabilità di concetti complessi e necessari all'efficienza nella
    computazione distribuita, ma che hanno poco a che fare con il problema da
    risolvere in sé (ad esempio, la sincronizzazione di task paralleli e lo
    scambio dei dati tra nodi del sistema distribuito). Questo modello **pone
    dei limiti alla libertà del programmatore**, che deve adeguare la codifica
    della risoluzione del problema al modello di programmazione fornito.

 *  **Computazione e storage locale**

    L'ottimizzazione più importante che Hadoop fornisce rispetto
    all'elaborazione dei dati è il risultato dell'unione di due concetti:
    **distribuzione dello storage** e **distribuzione della computazione**.
    
    Entrambi sono importanti a prescindere dell'uso particolare che
    ne fa Hadoop: la distribuzione dello storage permette di combinare lo
    spazio fornito da più dispositivi e di farne uso tramite un'unica
    interfaccia logica, e di replicare i dati in modo da poter tollerare guasti
    nei dispositivi. La distribuzione della computazione permette di
    aumentare il grado di parallelizazione nell'esecuzione dei programmi.

    Hadoop unisce i due concetti utilizzando cluster di macchine che hanno sia
    lo scopo di mantenere lo storage, che quello di elaborare i dati.
    Quando Hadoop esegue un lavoro, **quante più possibili delle computazioni
    richieste vengono eseguite nei nodi che contengono i dati da elaborare**.
    Questo permette di ridurre la latenza di rete, minimizzando la quantità
    di dati che devono essere scambiati tra i nodi del cluster. Il meccanismo è
    trasparente all'utente, a cui basta persitere i dati da elaborare nel
    cluster per usifruirne. Questo principio viene definito **data locality**.

 *  **Scalabilità**

    |||

 *  **Hardware non necessariamente affidabile**

    I cluster di macchine che eseguono Hadoop non hanno particolari requisiti
    di affidabilità rispetto ad hardware consumer. Il framework è progettato
    per tenere in conto dell'alta probabilità di fallimento dell'hardware, e
    per attenuarne le conseguenze, sia dal punto di vista dello storage e della
    potenziale perdita di dati, che da quello della perdita di risultati
    intermedi e parziali nel corso dell'esecuzione di lavori computazionalmente
    costosi. In questo modo l'utente è sgravato dal compito generalmente
    difficile di gestire fallimenti parziali nel corso delle computazioni.

Hadoop è composto da diversi moduli:

* **HDFS**, un filesystem distribuito ad alta affidabilità, che fornisce
  replicazione automatica all'interno dei cluster e accesso ad alto throughput
  ai dati

* **YARN**, un framework per la schedulazione di lavori e per la gestione delle
  risorse all'interno del cluster

* **MapReduce**, un framework e un modello di programmazione fornito da Hadoop
  per la scrittura di programmi paralleli che processano grandi dataset.

## Installazione e Configurazione

Ogni versione di Hadoop viene distribuita in tarball, una con i sorgenti, da
cui si può eseguire una build manuale, e una binaria, che può essere estratta e
utilizzata così com'è. Per un approccio più strutturato, sono disponibili
repository che forniscono versioni pacchettizzate di Hadoop, come il PPA
per Ubuntu[@hadoop-ppa] e i pacchetti AUR per Arch Linux[@hadoop-aur].

Ci sono anche distribuzioni di immagini virtuali Linux create appositamente con
lo scopo di fornire un ambiente preconfigurato con Hadoop e vari componenti del
suo ecosistema. I due ambienti più utilizzati di questo tipo sono Cloudera
QuickStart e HortonWorks Sandbox, disponibili per VirtualBox, VMWare e Docker.
Gli esempi di questo documento sono eseguiti prevalentemente da Arch Linux e
dalla versione Docker di HortonWorks Sandbox.

Hadoop è configurabile tramite file XML, che si trovano rispetto alla cartella
d'installazione in `etc/hadoop`. Ogni componente di Hadoop (HDFS, MapReduce,
Yarn) ha un file di configurazione apposito che contiene impostazioni relative
al componente stesso, mentre un altro file di configurazione contiene proprietà
comuni a tutti i componenti.

+-----------------+-----------------+-----------------+-------------------+
| Comuni          | HDFS            | YARN            | MapReduce         |
+=================+=================+=================+===================+
| `core-site.xml` | `hdfs-site.xml` | `yarn-site.xml` | `mapred-site.xml` |
+-----------------+-----------------+-----------------+-------------------+

: Nomi dei file di configurazione per i componenti di Hadoop

```{#lst:hadoop-conf-example .xml}
<?xml version="1.0"?>
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://namenode/</value>
    </property>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>yarn.resourcemanager.address</name>
        <value>resourcemanager:8032</value>
    </property>
</configuration>
```

: Esempio di file di configurazione personalizzato di Hadoop.

È anche possibile selezionare un'altra cartella da cui prendere i file di
configurazione, impostandola come valore della variabile d'ambiente
`HADOOP_CONF_DIR`. Un approccio comune alla modifica dei file di configurazione
consiste nel copiare il contenuto di `etc/hadoop` in un'altra posizione,
specificare questa in `HADOOP_CONF_DIR` e fare le modifiche nella nuova
cartella. In questo modo si evita di modificare l'albero d'installazione di
Hadoop.

Per molti degli eseguibili inclusi in Hadoop, è anche possibile specificare un
file che contiene ulteriori opzioni di configurazione, che possono
sovrascrivere quelle in `HADOOP_CONF_DIR` tramite lo switch `-conf`. 

### Esecuzione di software in Hadoop

I programmi che sfruttano il runtime di Hadoop sono generalmente sviluppati in
Java (o in un linguaggio che ha come target di compilazione la JVM), e vengono
avviati tramite l'eseguibile `hadoop`. L'eseguibile richiede che siano
specificati il classpath del programma, e una classe contente un metodo `main`
che si desidera eseguire (analogo all'entry point dei programmi Java).

Il classpath può essere specificato tramite la variabile d'ambiente
`HADOOP_CLASSPATH`, che può essere il percorso di una directory o di un file
`jar`. La classe con il metodo `main` da invocare viene messa tra i parametri
del comando `hadoop`, seguita dagli argomenti che si vogliono passare in
`args[]`.

(@) Volendo eseguire il seguente programma in Hadoop:

    ```java
    public class SayHello {
        public static void main(String args[]) {
            System.out.println("Hello " + args[0] + "!");
        }
    }
    ```

    Lo si può compilare e pacchettizzare in un file `jar`, per poi utilizzare i
    seguenti comandi:

    ```sh

    $~ export HADOOP_CLASSPATH=say_hello.jar 
    $~ hadoop SayHello Josh

    Hello Josh!
    ```

(@) In alternativa, si può eseguire il comando `hadoop jar`, e specificare il
    file `jar` direttamente nei suoi argomenti:

    ```sh

    $~ hadoop jar say_hello.js SayHello Josh

    Hello Josh!
    ```

In generale, i programmi eseguiti in Hadoop fanno uso della sua libreria
client. La libreria fornisce accesso al package `org.apache.hadoop`, che
contiene le API necessarie per interagire con Hadoop. Non è necessario che la
libreria client si trovi nel classpath finale, in quanto il runtime di Hadoop
fornisce le classi della libreria a runtime.

Per gestire le dipendenze e la pacchettizzazione dei programmi per Hadoop è
pratico utilizzare un tool di gestione delle build. Negli esempi in questo
documento si utilizza Maven a questo scopo, che permette di specificare
le proprietà di un progetto, tra cui le sue dipendenze, in un file XML chiamato
POM (Project Object Model). A partire dal POM, Maven è in grado di scaricare
automaticamente le dipendenze del progetto, e di pacchettizzarle correttamente
negli artefatti `jar` a seconda della configurazione fornita.

```{#lst:pom_example .xml .numberLines}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="..." >
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>say_hello</artifactId>
    <version>1.0</version>

    <dependencies>

        <!-- Libreria client di Hadoop -->
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.8.1</version>
            <scope>provided</scope>
        </dependency>

    </dependencies>
</project>
```

: Un esempio semplificato di un POM per il programma SayHello.

Maven è in grado di gestire correttamente la dipendenza della libreria client
di Hadoop, attraverso un meccanismo chiamato *dependency scope*. Per ogni
dipendenza è possibile specificare una proprietà *scope*, che indica in che
modo la dipendenza debba essere gestita a tempo di build (in particolar modo,
se debba essere inclusa nel classpath). Se non specificato, lo scope è
impostato a `compile`, che indica che la dipendenza è resa disponibile nel
classpath dell'artefatto. Per gestire correttamente la dipendenza dalla
libreria client di Hadoop, è opportuno impostare lo scope della dipendenza a
`provided`, che indica che le classi della libreria sono fornite dal container
in cui è eseguito il programma.

## HDFS

HDFS è un filesystem distribuito che permette l'accesso ad alto throughput ai
dati. HDFS è scritto in Java, e viene eseguito nello userspace. Lo storage dei
dati passa per il filesystem del sistema che lo esegue. 

I dati contenuti in HDFS sono organizzati in unità logiche chiamate *blocchi*,
come è comune nei filesystem. I blocchi di un singolo file possono essere
distribuiti all'interno di più macchine all'interno del cluster, permettendo di
avere file più grandi della capacità di storage di ogni singola macchina nel
cluster. Rispetto ai filesystem comuni la dimensione di un blocco è molto più
grande, 128 MB di default. La ragione per cui HDFS utilizza blocchi così grandi
è minimizzare il costo delle operazioni di seek, dato il fatto che se i file
sono composti da meno blocchi, si rende necessario trovare l'inizio di un
blocco un minor numero di volte. Questo approccio riduce anche la
frammentazione dei dati, rendendo più probabile che questi vengano scritti
contiguamente all'interno della macchina[^1].

[^1]: Non è possibile essere certi della contiguità dei dati, perché HDFS non è
un'astrazione diretta sulla scrittura del disco, ma sul filesystem del sistema
operativo che lo esegue. Per cui la frammentazione effettiva dipende da come i
dati vengono organizzati dal filesystem sottostante.

Il blocco, inoltre, è un'astrazione che si presta bene alla replicazione dei
dati nel filesystem all'interno del cluster: per replicare i dati, come si
vedrà, si mette uno stesso blocco all'interno di più macchine nel cluster.

HDFS è basato sulla specifica POSIX, ma non la implementa in modo rigido:
tralasciare alcuni requisiti di conformità alla specifica permette ad HDFS di
ottenere prestazioni e affidabilità migliori, come verrà descritto in seguito.

### Principi architetturali

![Schema di funzionamento dell'architettura di HDFS](img/hdfsarchitecture.png)

La documentazione di Hadoop descrive i seguenti come i principi architetturali
alla base della progettazione di HDFS:

 *  **Fallimento hardware come regola invece che come eccezione**  
    
    Un sistema che esegue HDFS è composto da molti componenti, con probabilità
    di fallimento non triviale. Sulla base di questo principio, HDFS da' per
    scontato che **ci sia sempre un numero di componenti non funzionanti**, e
    si pone di rilevare errori e guasti e di fornire un recupero rapido e
    automatico da questi.

    Il meccanismo principale con cui HDFS raggiunge questo obiettivo è la
    replicazione: in un cluster, ogni blocco di cui un file è composto è
    replicato in più macchine (3 di default). Se un blocco non è disponibile in
    una macchina, o se non supera i controlli di integrità, una sua copia può
    essere letta da un'altra macchina in modo trasparente per il client.

    Il numero di repliche per ogni blocco è configurabile, e ci sono più
    criteri con cui viene deciso in quali macchine il blocco viene replicato,
    principalmente orientati al risparmio di banda di rete.

 *  **Modello di coerenza semplice**

    Per semplificare l'architettura generale, HDFS fa delle assunzioni
    specifiche sul tipo di dati che vengono salvati in HDFS e pone dei limiti
    su come l'utente possa lavorare sui file. In particolare, **non è possibile
    modificare arbitrariamente file già esistenti**, e le modifiche devono
    limitarsi a operazioni di troncamento e di aggiunta a fine file. Queste
    supposizioni permettono di semplificare il modello di coerenza, perché i
    blocchi di dati, una volta scritti, possono essere considerati immutabili,
    evitando una considerevole quantità di problemi in un ambiente dove i
    blocchi di dati sono replicati in più posti:

    - Per ogni modifica a un blocco di dati, bisognerebbe verificare quali
      altre macchine contengono il blocco, e rieseguire la modifica (o
      rireplicare il blocco modificato) in ognuna di queste.
    
    - Queste modifiche dovrebbero essere fatte in modo atomico, o richieste di
      lettura su una determinata replica di un blocco invece che in un'altra
      potrebbe portare a risultati inconsistenti o non aggiornati.

    Le limitazioni che Hadoop impone sono ragionevoli per lo use-case per cui
    HDFS è progettato, caratterizzato da grandi dataset che vengono copiati nel
    filesystem e letti in blocco.
    
 *  **Dataset di grandi dimensioni**

    I filesystem distribuiti sono generalmente necessari per aumentare la
    capacità di storage disponibile oltre quella di una singola macchina. La
    distribuzione di HDFS, assieme alla grande dimensione dei blocchi

 *  **Accesso in streaming**
    
    HDFS predilige l'accesso ai dati in streaming, per permettere ai lavori
    batch di essere eseguiti con grande efficienza. Questo approccio va a
    discapito del tempo di latenza della lettura dei file, ma permette di avere
    un throughput in lettura molto vicino ai tempi di lettura del disco.

 *  **Portabilità su piattaforme software e hardware eterogenee**
    
    HDFS è scritto in Java, ed è portabile in tutti i sistemi che ne supportano
    il runtime.

L'architettura di HDFS è di tipo master/slave, dove un nodo centrale,
chiamato **NameNode**, gestisce i metadati e la struttura del filesystem, mentre i
nodi slave, chiamati **DataNode**, contengono i blocchi di cui file sono composti.
Tipicamente, viene eseguita un'istanza del software del DataNode per macchina
del cluster, e una macchina dedicata esegue il NameNode.

I *client* del filesystem interagiscono sia con il NameNode che con i DataNode
per l'accesso ai file. La comunicazione tra il client e i nodi avviene tramite
socket TCP ed è coordinata dal NameNode, che fornisce ai client tutte le
informazioni sul filesystem e su quali nodi contengono i DataBlock dei file
richiesti.


### Comunicare con HDFS

Hadoop fornisce tool e librerie che possono agire da client nei confronti di
HDFS. Il tool più diretto è la CLI, accessibile nelle macchine in cui è
installato Hadoop tramite il comando `hadoop fs`.

```sh
% hadoop fs -help
Usage: hadoop fs [generic options]
	[-appendToFile <localsrc> ... <dst>]
	[-cat [-ignoreCrc] <src> ...]
	[-checksum <src> ...]
	[-chgrp [-R] GROUP PATH...]
	[-chmod [-R] <MODE[,MODE]... | OCTALMODE> PATH...]
	[-chown [-R] [OWNER][:[GROUP]] PATH...]
	[-copyFromLocal [-f] [-p] [-l] [-d] <localsrc> ... <dst>]
	[-copyToLocal [-f] [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
	[-count [-q] [-h] [-v] [-t [<storage type>]] [-u] [-x] <path> ...]
	[-cp [-f] [-p | -p[topax]] [-d] <src> ... <dst>]
...
```

La CLI fornisce alcuni comandi comuni nei sistemi POSIX, come `cp`, `rm`, `mv`,
`ls` e `chown`, e altri che riguardano specificamente HDFS, come
`copyFromLocal` e `copyToLocal`, utili a trasferire dati tra la macchina su cui
si opera e il filesystem. 

I comandi richiedono l'URI che identifica l'entità su cui si vuole operare. Per
riferirsi a una risorsa all'interno di un'istanza di HDFS, si usa l'URI del
namenode, con schema `hdfs`[^2], e con il path corrispondente al percorso della
risorsa nel filesystem. Ad esempio, è possibile creare una cartella `foo`
all'interno della radice del filesystem con il seguente comando:

```sh
hadoop fs -mkdir hdfs://localhost:8020/foo
```

Per diminuire la verbosità dei comandi è possibile utilizzare percorsi
relativi, e specificare l'opzione `dfs.defaultFS` nella configurazione di
Hadoop all'URI del filesystem ai cui i percorsi relativi si riferiscono.
In questo modo, si può accorciare l'esempio precedente a:

```sh
hadoop fs -mkdir foo
```

[^2]: Hadoop è abbastanza generale da poter lavorare con diversi filesystem,
con lo schema definisce il protocollo di comunicazione, che non deve essere
necessariamente `hdfs`. Ad esempio, un URI con schema `file` si riferisce al
filesystem locale, e le operazioni eseguite su URI che utilizzano questo schema
vengono effettuate sulla macchina dove viene eseguito il comando. Questo
approccio può essere adatto nella fase di testing dei programmi, ma nella
maggior parte dei casi è comunque desiderabile lavorare su un filesystem
distribuito adeguato alla gestione dei Big Data, e un'alternativa ad HDFS degna
di nota è MapR-FS[@mapr-fs].

Ad esempio, data la seguente cartella:

```sh
[root@sandbox example_data]# ls
example1.txt  example2.txt  example3.txt
```

Si possono copiare i file dalla cartella locale della macchina al filesystem
distribuito con il seguente comando:

```sh
[root@sandbox example_data]# hadoop fs -copyFromLocal example*.txt /example
```

Per verificare che l'operazione sia andata a buon fine, si può ottenere un
listing della cartella in cui si sono trasferiti i file con il comando `ls`:

```sh
[root@sandbox example_data]# hadoop fs -ls /example
Found 3 items
-rw-r--r--   1 root hdfs         70 2017-06-30 03:58 /example/example1.txt
-rw-r--r--   1 root hdfs         39 2017-06-30 03:58 /example/example2.txt
-rw-r--r--   1 root hdfs         43 2017-06-30 03:58 /example/example3.txt
```

Il listing è molto simile a quello ottenibile su sistemi Unix. Una differenza
importante è la seconda colonna, che non mostra il numero di hard link al file
nel filesystem^[HDFS correntemente non supporta link nel filesystem.], ma il
numero di repliche che HDFS ha a disposizione del file, in questo caso una per
file. Il numero di repliche fatte da HDFS può essere impostato settando il
fattore di replicazione di default, che per Hadoop in modalità distribuita è 3
di default. Si può anche cambiare il numero di repliche disponibili per
determinati file, utilizzando il comando `hdfs dfs`:

```sh
[root@sandbox ~]# hdfs dfs -setrep 2 /example/example1.txt
Replication 2 set: /example/example1.txt
[root@sandbox ~]# hadoop fs -ls /example
Found 3 items
-rw-r--r--   2 root hdfs         70 2017-06-30 03:58 /example/example1.txt
-rw-r--r--   1 root hdfs         39 2017-06-30 03:58 /example/example2.txt
-rw-r--r--   1 root hdfs         43 2017-06-30 03:58 /example/example3.txt
```

HDFS è anche accessibile tramite *HDFS Web Interface*, un tool che fornisce
informazioni sullo stato generale del filesystem e sul suo contenuto. Ci sono
anche tool di amministrazione di cluster Hadoop che offrono GUI web più
avanzate di quella fornita di default da HDFS. Due esempi sono Cloudera Manager
e Apache Ambari, che offrono un file manager lato web con cui è possibile
interagire in modo più semplice, permettendo anche a utenti in ambito meno
tecnico di lavorare con il filesystem.

![Screenshot del file manager HDFS incluso in Ambari](img/ambari_hdfs.png)

Un'altra interfaccia importante ad HDFS è l'API `FileSystem` di Hadoop, che
permette un accesso programmatico da linguaggi per JVM a tutte le funzioni del
filesystem. L'API è generale, in modo che possa essere utilizzata con
filesystem diversi da HDFS. 

Per linguaggi che non supportano interfacce Java, esiste un'implementazione in
C chiamata `libhdfs`, che si appoggia sulla Java Native Interface per esporre
l'API di Hadoop.

Esistono poi progetti che permettono il montaggio di HDFS in un filesystem
locale. Alcune di queste implementazioni sono basate su FUSE, mentre altre su
NFS Gateway. Questo metodo di accesso permette l'utilizzo di utilità native del
sistema in uso in HDFS.

### NameNode

Il NameNode è il riferimento centrale per i metadati del filesystem nel
cluster, il che vuol dire che se il NameNode non è disponibile il filesystem
non è accessibile. Questo rende il NameNode un *single point of failure* del
sistema, e per questa ragione HDFS mette a disposizione dei meccanismi per
attenutare l'indisponibilità del sistema in caso di non reperibilità del
NameNode, e per assicurare che lo stato del filesystem possa essere recuperato
a partire dal NameNode.

Il NameNode è anche il nodo a cui i client si connettono alla lettura del file.
La connessione ha il solo scopo di fornire le informazioni sui DataNode che
contengono i dati effettivi del file. I dati di un file non passano mai per il
NameNode.

Tuttavia, il NameNode non salva persistentemente le informazioni sulle
posizioni dei blocchi, che vengono invece mantenute dai DataNode. Perché il
NameNode possa avere in memoria le informazioni sui file necessarie per essere
operativo, questo deve ricevere le liste dei blocchi in possesso dei DataNode,
in messaggi chiamati **block report**. Non è necessario che il DataNode conosca
la posizione di tutti i blocchi sin dall'inizio, ma basta che per ogni blocco
conosca la posizione di un numero minimo di repliche, determinato da un'opzione
chiamata `dfs.replication.min.replicas`, di default 1.

Questa procedura avviene quando il NameNode si trova in uno stato chiamato
**safe mode**.

#### *Namespace image* ed *edit log*

Le informazioni sui metadati del sistema vengono salvate nello storage del
NameNode in due posti, la _**namespace image**_ e l'_**edit log**_.
La *namespace image* è uno snapshot dell'intera struttura del filesystem,
mentre l'*edit log* è un elenco di operazioni eseguite nel filesystem a partire
dalla *namespace image*. Partendo dalla *namespace image* e applicando le
operazioni registrate nell'*edit log*, è possibile risalire allo stato attuale
del filesystem. Il NameNode ha una rappresentazione dello stato del filesystem
anche nella memoria centrale, che viene utilizzata per servire le richieste di
lettura.

Quando HDFS riceve una richiesta che richiede la modifica dei metadati, il
NameNode esegue le seguenti operazioni:

#. registra la transazione nell'*edit log*
#. aggiorna la rappresentazione del filesystem in memoria
#. passa all'operazione successiva.

La ragione per cui i cambiamenti dei metadati vengono registrati nell'*edit
log* invece che nella *namespace image* è la velocità di scrittura: scrivere
ogni cambiamento del filesystem mano a mano che avviene nell'immagine sarebbe
lento, dato che questa può avere dimensioni nell'ordine dei gigabyte. Il
NameNode esegue un *merge* dell'*edit log* e della *namespace image* a ogni
suo avvio, portando lo stato attuale dell'immagine al pari di quello del
filesystem.

Dato che la dimensione dell'*edit log* può diventare notevole, è utile eseguire
l'operazione di *merge* al raggiungimento di una soglia di dimensione del log.
Questa operazione è computazionalmente costosa, e se fosse eseguita dal
NameNode potrebbe interferire con la sua operazione di routine.

Per evitare interruzioni nel NameNode, il compito di eseguire periodicamente il
*merge* dell'*edit log* è affidato a un'altra entità, il **Secondary
NameNode**. Il Secondary NameNode viene solitamente eseguito su una macchina
differente, dato che richiede un'unità di elaborazione potente e almeno la
stessa memoria del NameNode per eseguire l'operazione di merge.

#### Avvio del NameNode e *Safe Mode*

Prima di essere operativo, il NameNode deve eseguire alcune operazioni di
startup, tra cui attendere di aver ricevuto i block report dai DataNode in modo
da conoscere le posizioni dei blocchi. Durante queste operazioni, il NameNode
si trova in uno stato chiamato *safe mode*, in cui sono permesse unicamente
operazioni che accedono ai metadati del filesystem, e tentativi di lettura e
scrittura di file falliscono. Prima di poter permettere l'accesso completo, il
NameNode ha bisogno di ricevere le informazioni sui blocchi da parte dei
DataNode.

Per ricapitolare, al suo avvio, il NameNode effettua il merge della *namespace
image* con l'*edit log*. Al termine dell'operazione, il risultato del merge
viene salvato come la nuova *namespace image*. Il Secondary NameNode non viene
coinvolto in questo primo merge.

Prima di uscire dalla safe mode, il NameNode attende di avere abbastanza
informazioni da poter accedere a un numero minimo di repliche di ogni blocco. A
questo punto il NameNode esce dalla safe mode.

Si possono utilizzare dei comandi per verificare lo stato, attivare e
disattivare la safe mode.

```sh
bash-4.1$ hdfs dfsadmin -safemode get
Safe mode is OFF
bash-4.1$ hdfs dfsadmin -safemode enter
Safe mode is ON
bash-4.1$ hdfs dfsadmin -safemode leave
Safe mode is OFF
```

![Lo stato dello startup di un'istanza di HDFS, mostrata da HDFS Web
Interface.](img/hdfs-web-startup.png)

### Processo di lettura di file in HDFS

![Diagramma delle operazioni eseguite nella lettura di un file in
HDFS[@hadoop-guide-hdfs-file-read]](img/hdfs-file-read.png)

Per avere un quadro completo del funzionamento di HDFS, è utile osservare come
avvenga il processo di lettura di un file. In questa sezione si prende in esame
un programma di esempio che utilizza le API `FileSystem` di Hadoop per
reimplementare una versione semplificata del comando `cat`, per poi esaminare
come le operazioni specificate nel programma vengano effettivamente portate a
termine in un'istanza di HDFS.


```{#lst:hdfs-cat .java .numberLines}
import java.io.InputStream;
import java.net.URI;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;

public class MyCat {

    public static void main(String args[]) throws Exception {

        String source = args[0];
        Configuration conf = new Configuration();

        try(
            FileSystem sourcefs = FileSystem.get(URI.create(source), conf);
            InputStream in = sourcefs.open(new Path(source))
        ) {
            IOUtils.copyBytes(in, System.out, 4096, false);
        }
    }
}
```

: Programma di esempio che reimplementa il comando `cat`.


La reimplementazione del programma `cat` utilizza il primo parametro della
linea di comando per ricevere l'URI del file che si vuole stampare nello
standard output. L'URI deve contenere il percorso di rete del filesystem HDFS,
ed essere quindi del formato `hdfs://[indirizzo o hostname del namenode]/[path
del file]`.
Di seguito vengono spiegati i passi eseguiti dal programma. Quando non
qualificato, l'identificativo `hadoop` si riferisce al package Java
`org.apache.hadoop`.

#. Si crea un oggetto `hadoop.conf.Configuration`. Gli oggetti
`Configuration` forniscono l'accesso ai parametri di configurazione di Hadoop
(impostati in file XML, come descritto in [Installazione e Configurazione]).

#. Si ottiene un riferimento `sourcefs` a un `hadoop.fs.FileSystem` (dichiarato
come interfaccia Java),
che fornisce le API che verranno usate per leggere e manipolare il filesystem.
Il riferimento viene ottenuto tramite il metodo statico `FileSystem.get(URI
source, Configuration conf)`, che richiede un URI che possa essere utilizzato
per risalire a quale filesystem si vuole accedere. Un overload di
`FileSystem.get` permette di specificare solo l'oggetto `Configuration`,
e ottiene le informazioni sul filesystem da aprire dalla proprietà di
configurazione `dfs.defaultFS`.

Nel caso di un URI con schema HDFS, l'istanza concreta di `FileSystem` che
viene restituita da `FileSystem.get` è di tipo `DistributedFileSystem`.

#. Si apre il file il lettura, chiamando `sourcefs.open(Path file)`. Il
metodo restituisce un oggetto di tipo `hadoop.fs.FSDataInputStream`, una
sottoclasse di `java.io.InputStream` che supporta anche l'accesso a
punti arbitrari del file. In questo case l'oggetto è utilizzato per leggere
il file sequenzialmente, e il suo riferimento viene salvato nella variabile
`InputStream in`.

Dietro le quinte, `FSDataInputStream` utilizza chiamate a procedure remote sul
namenode per ottenere le posizioni dei primi blocchi del file. Per ogni blocco,
il namenode restituisce gli indirizzi dei datanode che lo contengono, ordinati
in base alla prossimità del client. Se il client stesso è uno dei datanode che
contiene un blocco da leggere, il blocco viene letto localmente.


#. Si copiano i dati dallo stream `in` a `System.out`, di fatto stampando i
dati nella console. Questa operazione è eseguita tramite il metodo
`hadoop.io.IOUtils.copyBytes(InputStream in, OutputStream out, int bufSize, bool
closeStream)`. Il metodo copia i dati da uno stream d'ingresso a uno d'uscita,
e non ha funzioni specifiche rispetto ad Hadoop, ma viene fornito per la
mancanza di un meccanismo simile in Java.

#. Lo stream e l'oggetto `FileSystem` vengono chiusi. L'operazione avviene
implicitamente utilizzando il costrutto try-with-resources di Java.

L'esecuzione del programma dà il seguente output:

```sh
$~ hadoop MyCat hdfs://sandbox.hortonworks.com:8020/example/example1.txt
This is the first example file
```

Le astrazioni
