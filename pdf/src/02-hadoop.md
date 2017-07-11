# Hadoop

Hadoop è una piattaforma software utilizzata per lo storage e la computazione
distribuita di dataset di grandi dimensioni. Hadoop viene eseguito in *cluster*
di computer, che vengono coordinati dalla piattaforma per fornire delle API in
grado di astrarre una parte importante della complessità insita nei sistemi
distribuiti. Hadoop fornisce delle interfacce per l'elaborazione diretta
dei dati da parte degli utenti, e delle primitive di livello più basso che
consentono l'implementazione di altri framework basati sulla sua
infrastruttura di base. Grazie a quest'ultima caratteristica, Hadoop è
diventato un perno centrale nell'ambito dei Big Data, su cui si è costruito un
ecosistema di tool e tecnologie strettamente integrazione.

La documentazione ufficiale[@hadoop-doc-main] lo descrive come:

> ...un framework che abilita l'elaborazione distribuita di grandi dataset
in cluster di computer utilizzando semplici modelli di programmazione.
[Hadoop] è progettato per essere scalato da server singoli a migliaia di
macchine, dove ognuna di queste offre computazione e storage locale. Invece di
affidarsi all'hardware per fornire un'alta affidabilità, [Hadoop] è progettato
per rilevare e gestire i fallimenti [delle computazioni] a livello applicativo,
mettendo a disposizione un servizio ad alta affidiabilità su cluster di
computer proni al fallimento.

In questa definizione sono racchiusi dei punti importanti:

 *  **Semplici modelli di programmazione**

    Hadoop raggiunge molti dei suoi obiettivi fornendo un'interfaccia di
    alto livello al programmatore, in modo di potersi assumere la
    responsabilità di molti concetti complessi e necessari alla correttezza e
    all'efficienza della computazione distribuita, ma che hanno poco a che fare
    con il problema da risolvere in sé (come la sincronizzazione di task
    paralleli e lo scambio dei dati tra nodi del sistema distribuito). 

    Il framwork fornisce un modello di programmazione distribuita rivolto agli
    utenti, chiamato MapReduce, e ne esistono molti altri creati da terze
    parti. 

 *  **Computazione e storage locale**

    L'ottimizzazione più importante che Hadoop fornisce rispetto
    all'elaborazione dei dati è il risultato dell'unione di due concetti:
    **distribuzione dello storage** e **distribuzione della computazione**.
    
    Entrambi sono importanti a prescindere dell'uso particolare che
    ne fa Hadoop: la distribuzione dello storage permette di combinare lo
    spazio fornito da più dispositivi e di farne uso tramite un'unica
    interfaccia logica, e di replicare i dati in modo da poter tollerare guasti
    nei dispositivi. La distribuzione della computazione permette di
    aumentare il grado di parallelismo nell'esecuzione dei programmi.

    Hadoop unisce i due concetti utilizzando cluster di macchine che hanno sia
    lo scopo di mantenere lo storage, che quello di elaborare i dati.
    Quando Hadoop esegue un lavoro, **quante più possibili delle computazioni
    richieste vengono eseguite nei nodi che già contengono i dati da
    elaborare**. Questo permette di ridurre la latenza di rete, minimizzando la
    quantità di dati che devono essere scambiati tra i nodi del cluster. Il
    meccanismo è trasparente all'utente, a cui basta persitere i dati da
    elaborare nel cluster e utilizzare un framework basato su Hadoop per
    usifruirne. Questo principio viene definito **data locality**.

 *  **Rack awareness**

    Nel contesto di Hadoop, *rack awareness* si riferisce a delle
    ottimizzazioni sull'utilizzo di banda di rete e sull'affidabilità che
    Hadoop fa basandosi sulla struttura del cluster. 

    \begin{figure}
    \def\svgwidth{\linewidth}
    \input{img/hadoop_topology.pdf_tex}
    \label{fig:hadoop-topology}
    \caption{Topologia di rete tipica di un cluster Hadoop.}
    \end{figure}

    Quando configurato per essere *rack aware*, Hadoop considera il cluster
    come un insieme di *rack* che contengono i nodi del cluster. Tutti i nodi
    di un rack sono connessi a uno switch di rete (o dispositivo equivalente),
    e tutti gli switch sono a loro volta connessi a uno switch centrale.

    A partire da questa struttura si può fare un'assunzione importante: la
    comunicazione tra nodi in uno stesso rack è meno onerosa in termini di
    banda rispetto alla comunicazione tra nodi in rack diversi, perché la
    comunicazione può essere commutata tramite un solo switch.

    Quando possibile, Hadoop utilizza questo principio per minimizzare l'uso di
    banda tra nodi del cluster. Come si vedrà, i vari componenti di Hadoop
    fanno uso della configurazione di rete per ottimizzazare dell'uso della
    rete e per ottenere una migliore fault-tolerance.

 *  **Scalabilità**

    Hadoop è in grado di scalare linearmente in termini di velocità di
    computazione e storage, ed è in grado di sostenere cluster composti da un
    gran numero di macchine. Yahoo riporta di eseguire un cluster Hadoop
    composto da circa 4500 nodi, utilizzato per il sistema pubblicitario e di
    ricerca[@yahoo].

 *  **Hardware non necessariamente affidabile**

    I cluster di macchine che eseguono Hadoop non hanno particolari requisiti
    di affidabilità. Il framework è progettato per tenere in conto dell'alta
    probabilità di fallimento dell'hardware, e per attenuarne le conseguenze,
    sia dal punto di vista dello storage e della potenziale perdita di dati,
    che da quello della perdita di risultati intermedi e parziali nel corso
    dell'esecuzione di lavori computazionalmente costosi. In questo modo
    l'utente è sgravato dal compito generalmente difficile di gestire
    fallimenti parziali nel corso delle computazioni.

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
cui si può eseguire una build manuale, e una binaria. Per un approccio più
strutturato, sono disponibili repository che forniscono versioni pacchettizzate
di Hadoop, come il PPA per Ubuntu[@hadoop-ppa] e i pacchetti AUR per Arch
Linux[@hadoop-aur].

Ci sono anche distribuzioni di immagini virtuali Linux create appositamente con
lo scopo di fornire un ambiente preconfigurato di prototipazione con Hadoop e
vari componenti del suo ecosistema. I due ambienti più utilizzati di questo
tipo sono Cloudera QuickStart e HortonWorks Sandbox, disponibili per
VirtualBox, VMWare e Docker. Gli esempi di questo documento sono eseguiti
prevalentemente da Arch Linux e dalla versione Docker di HortonWorks
Sandbox[@hortonworks-sandbox].

Hadoop è configurabile tramite file XML, che si trovano rispetto alla cartella
d'installazione in `etc/hadoop`. Ogni componente di Hadoop (HDFS, MapReduce,
Yarn) ha un file di configurazione apposito che contiene le sue impostazioni,
e un file di configurazione globale per il cluster contiene proprietà comuni a
tutti i componenti.

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
        <value>hdfs://NameNode/</value>
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

È possibile far selezionare ad Hadoop una cartella diversa da `etc/hadoop` da
cui leggere i file di configurazione, impostandola come valore della variabile
d'ambiente `HADOOP_CONF_DIR`. Un approccio comune alla modifica dei file di
configurazione consiste nel copiare il contenuto di `etc/hadoop` in un'altra
posizione, specificare questa in `HADOOP_CONF_DIR` e fare le modifiche nella
nuova cartella. In questo modo si evita di modificare l'albero d'installazione
di Hadoop.

Per molti degli eseguibili inclusi in Hadoop, è anche possibile specificare un
file che contiene ulteriori opzioni di configurazione, che possono
sovrascrivere quelle in `HADOOP_CONF_DIR` tramite lo switch `-conf`. 

### Esecuzione di software in Hadoop

I programmi che sfruttano il runtime di Hadoop sono generalmente sviluppati in
Java (o in un linguaggio che ha come target di compilazione la JVM), e vengono
avviati tramite l'eseguibile `hadoop`. L'eseguibile richiede che siano
specificati il classpath del programma, e il nome di una classe contente un
metodo `main` che si desidera eseguire (un entry point dei programmi Java).

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
dati, scritto in Java ed eseguito nello userspace. HDFS è stato studiato e
progettato per fornire un sistema di storage distribuito che permetta
l'efficiente elaborazione batch di grandi dataset, e che sia resiliente al
fallimento delle singole macchine del cluster.

I dati contenuti in HDFS sono organizzati, a livello di storage, in unità
logiche chiamate *blocchi*, nel senso comune del termine nel dominio dei
filesystem. I blocchi di un singolo file possono essere distribuiti all'interno
di più macchine all'interno del cluster, permettendo di avere file più grandi
della capacità di storage di ogni singola macchina nel cluster. Rispetto ai
filesystem comuni la dimensione di un blocco è molto più grande, 128 MB di
default. La ragione per cui HDFS utilizza blocchi così grandi è minimizzare il
costo delle operazioni di seek, dato il fatto che se i file sono composti da
meno blocchi, si rende necessario trovare l'inizio di un blocco un minor numero
di volte. Questo approccio riduce anche la frammentazione dei dati, rendendo
più probabile che questi vengano scritti contiguamente all'interno della
macchina[^1].

[^1]: Non è possibile essere certi della contiguità dei dati, perché HDFS non è
un'astrazione diretta sulla scrittura del disco, ma sul filesystem del sistema
operativo che lo esegue. La frammentazione effettiva dipende da come i dati
vengono organizzati dal filesystem del sistema operativo.

HDFS è basato sulla specifica POSIX, e ha quindi una struttura gerarchica.
L'utente può strutturare i dati salvati in directory, e impostare permessi di
accesso in file e cartelle. Tuttavia, l'adesione a POSIX non è rigida, e alcune
operazioni non sono rese possibili, come la modifica dei file in punti
arbitrari. Queste restrizioni permettono ad HDFS di implementare
efficientemente funzioni specifiche del suo dominio (come il batch processing),
e di semplificare la sua architettura.

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
      potrebbe portare a risultati inconsistenti.

    Le limitazioni che Hadoop impone sono ragionevoli per lo use-case per cui
    HDFS è progettato, caratterizzato da grandi dataset che vengono copiati nel
    filesystem e letti in blocco. Il modello del filesystem di Hadoop è
    definito **write once, read many**.
    
 *  **Dataset di grandi dimensioni**

    I filesystem distribuiti sono generalmente necessari per aumentare la
    capacità di storage disponibile oltre quella di una singola macchina. La
    distribuzione di HDFS, assieme alla grande dimensione dei blocchi, offre un
    supporto privilegiato ai file molto grandi rispetto a quelli piccoli.

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

### Replicazione e fault-tolerance

Il blocco è un'astrazione che si presta bene alla replicazione dei dati nel
filesystem all'interno del cluster: per replicare i dati, HDFS persiste ogni
blocco all'interno di più macchine nel cluster. HDFS utilizza le informazioni
sulla configurazione di rete del cluster per decidere il posizionamento delle
repliche di ogni blocco: considerando che i tempi di latenza di rete sono più
bassi tra nodi in uno stesso rack, HDFS salva due copie del blocco in due nodi
che condividono il rack. In questo modo, nell'eventualità in cui una delle
copie del blocco non fosse disponibile o avesse problemi d'integrità, una sua
replica può essere recuperata in un nodo che si trova all'interno del rack,
minimizzando l'overhead di rete.

Per aumentare la fault-tolerance, HDFS salva un'ulteriore copia del blocco al
di fuori del rack in cui ha memorizzato le prime due. Questa operazione
salvaguardia l'accesso al blocco in caso di fallimento dello switch di rete del
rack che contiene le prime due copie, che renderebbe inaccessibili tutte le
macchine contenenti il blocco.

Il numero di repliche create da HDFS per ogni blocco è definito *replication
factor*, ed è configurabile tramite l'opzione `dfs.replication`. Quando il
numero di repliche di un certo file scende sotto la soglia di questa proprietà
(eventualità che accade in caso di fallimento dei nodi) HDFS riesegue
trasparentemente la replicazione dei blocchi per raggiungere la soglia definita
nella configurazione.

Inoltre, l'integrità dei blocchi è verificata trasparentemente da HDFS alla
loro lettura e scrittura, utilizzando checksum CRC-32.

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
NameNode, con schema `hdfs`[^2], e con il path corrispondente al percorso della
risorsa nel filesystem. Ad esempio, è possibile creare una cartella `foo`
all'interno della radice del filesystem con il seguente comando:

```sh
hadoop fs -mkdir hdfs://localhost:8020/foo
```

Per diminuire la verbosità dei comandi è possibile utilizzare percorsi
relativi, specificando nell'opzione `dfs.defaultFS` della configurazione del
cluster l'URI del filesystem ai cui i percorsi relativi si riferiscono.
Gli URI riferiti a istanze di HDFS hanno schema `hdfs://`, seguito
dall'indirizzo IP o dell'hostname della macchina che esegue il NameNode.
Specificando l'URI, si può accorciare l'esempio precedente a:

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
nel filesystem^[Non è necessario mostrare i link dei file, perché HDFS
correntemente non li supporta.], ma il numero di repliche che HDFS ha a
disposizione del file, in questo caso una per file. Il numero di repliche fatte
da HDFS può essere impostato settando il fattore di replicazione di default,
che per Hadoop in modalità distribuita è 3 di default. Si può anche cambiare il
numero di repliche disponibili per determinati file, utilizzando il comando
`hdfs dfs`:

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
interagire in modo più semplice, permettendo anche a utenti non tecnicamente
esperti di lavorare con il filesystem.

![Screenshot del file manager HDFS incluso in Ambari](img/ambari_hdfs.png)

Un altro importante modo di interfacciarsi ad HDFS è l'API `FileSystem` di
Hadoop, che permette un accesso programmatico da linguaggi per JVM alle
funzioni del filesystem. 

Per linguaggi che non supportano interfacce Java, esiste un'implementazione in
C chiamata `libhdfs`, che si appoggia sulla Java Native Interface per esporre
l'API di Hadoop.

Esistono poi progetti che permettono il montaggio di HDFS in un filesystem
locale. Alcune di queste implementazioni sono basate su FUSE, mentre altre su
NFS Gateway. Questi strumenti permettono l'utilizzo di utilità native del
sistema in uso in HDFS.

### NameNode in dettaglio

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
posizioni dei blocchi, che vengono invece mantenute dai DataNode. Prima che il
NameNode possa essere operativo, deve ricevere e salvare in memoria le
liste dei blocchi in possesso dei DataNode, in messaggi chiamati **block
report**. Non è necessario che il DataNode conosca la posizione di tutti i
blocchi sin dall'inizio, ed è sufficiente che per ogni blocco conosca la
posizione di un numero minimo di repliche, determinato dall'opzione del cluster
`dfs.replication.min.replicas`, di default 1.

Questa procedura avviene quando il NameNode si trova in uno stato chiamato
[*safe mode*](#safe-mode)

#### *Namespace image* ed *edit log*

Le informazioni sui metadati del sistema vengono salvate nello storage del
NameNode in due posti, la _**namespace image**_ e l'_**edit log**_.
La *namespace image* è uno snapshot dell'intera struttura del filesystem,
mentre l'*edit log* è un elenco di transazioni eseguite nel filesystem a
partire dallo stato registrato nella *namespace image*. Partendo dalla
*namespace image* e applicando le operazioni registrate nell'*edit log*, è
possibile risalire allo stato attuale del filesystem. Il NameNode ha una
rappresentazione dello stato del filesystem anche nella memoria centrale, che
viene utilizzata per servire le richieste di lettura.

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

#### Avvio del NameNode e *Safe Mode* {#safe-mode}

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
ed essere quindi del formato `hdfs://[indirizzo o hostname del NameNode]/[path
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

#. Si apre il file il lettura, chiamando `sourcefs.open(Path file)`. Il
metodo restituisce un oggetto di tipo `hadoop.fs.FSDataInputStream`, una
sottoclasse di `java.io.InputStream` che supporta anche l'accesso a
punti arbitrari del file. In questo case l'oggetto è utilizzato per leggere
il file sequenzialmente, e il suo riferimento viene salvato nella variabile
`InputStream in`.

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

Nel caso di un URI con schema HDFS, l'istanza concreta di `FileSystem` che
viene restituita da `FileSystem.get` è di tipo `DistributedFileSystem`, che
contiene le funzionalità necessarie a comunicare con HDFS. Con uno schema
diverso (ad esempio `file://` per filesystem locali), l'istanza concreta di
`FileSystem` cambia per gestire opportunamente lo schema richiesto (se
supportato).

Dietro le quinte, `FSDataInputStream`, restituito da `FileSystem.open(...)`,
utilizza chiamate a procedure remote sul NameNode per ottenere le posizioni dei
primi blocchi del file. Per ogni blocco, il NameNode restituisce gli indirizzi
dei datanode che lo contengono, ordinati in base alla prossimità del client. Se
il client stesso è uno dei datanode che contiene un blocco da leggere, il
blocco viene letto localmente.

Alla prima chiamata di `read()` su `FSDataInputStream`, l'oggetto si connette
al DataNode che contiene il primo blocco del file, e lo richiede (nell'esempio,
`read` viene chiamato da `IOUtils.copyBytes`). Il DataNode risponde inviando i
dati corrispondenti al blocco, fino al termine di questi. Al raggiungimento
della fine di un blocco, `DFSInputStream` termina la connessione con il
DataNode corrente e ne inizia un'altra con il più prossimo dei DataNode che
contiene il blocco successivo.

In caso di errore dovuto al fallimento di un DataNode o alla ricezione di un
blocco di dati corrotto, il client può ricevere un'altra copia del blocco dal
nodo successivo della lista di DataNode ricevuta dal NameNode.

I blocchi del file non vengono inviati tutti insieme, e il client deve
periodicamente richiedere al NameNode i dati sui blocchi successivi. Questo
passaggio avviene trasparentemente rispetto all'interfaccia, in cui
l'utilizzatore si limita a chiamare `read` su `DFSInputStream`.

Le comunicazioni di rete, in questo meccanismo, sono distribuite su tutto il
cluster. Il NameNode riceve richieste che riguardano solo i metadati dei file,
mentre il resto delle connessioni viene eseguito direttamente tra client e
DataNode. Questo approccio permette ad HDFS di evitare colli di bottiglia
dovuti a un punto di connessione ai client centralizzato, distribuendo le
comunicazioni di rete attraverso i vari nodi del cluster.

## YARN

YARN è acronimo di Yet Another Resource Negotiator, ed è l'insieme di API su
cui sono implementati framework di programmazione distribuita di livello più
alto, come MapReduce e Spark. YARN si definisce un *"negotiator"* perché è
l'entità che decide quando e come le risorse del cluster debbano essere
allocate per l'esecuzione distribuita, e che gestisce le comunicazioni
che riguardano le risorse con tutti i nodi coinvolti. Inoltre, YARN ha
l'importante ruolo di esporre un'interfaccia che permette di imporre **vincoli
di località** sulle risorse richieste dalle applicazioni, permettendo
l'implementazione di applicazioni che seguono il principio di *data locality*
di Hadoop.

I servizi di YARN sono offerti tramite *demoni* eseguiti nei nodi del cluster.
Ci sono due tipi di demoni in YARN:

* i **NodeManager**, che eseguono su richiesta i processi necessari allo
  svolgimento dell'applicazione nel cluster. L'esecuzione dei processi avviene
  attraverso *container*, che permettono di limitare le risorse utilizzate da
  ogni processo eseguito. Il NodeManager viene eseguito in ogni nodo del
  cluster che prende parte alle computazioni distribuite.

* il **ResourceManager**, di cui è eseguita un'istanza per cluster, e che
  gestisce le sue risorse. Il ResourceManager è l'entità che comunica con i
  NodeManager e che decide quali processi questi debbano eseguire e quando. 

I container in YARN possono essere rappresentativi di diverse modalità di
esecuzione di un processo. Queste sono configurabili dall'utente tramite la
proprietà `yarn.nodemanager.container-executor.class`, il cui valore identifica
una classe che stabilisce come i processi debbano essere eseguiti. Di default,
La configurazione permette l'uso di diversi container di virtualizzazione
OS-level, come lxc e Docker[@yarn-container-conf].

L'esecuzione di applicazioni distribuite in YARN è richiesta dai client al
ResourceManager. Quando il ResourceManager decide di avviare un'applicazione,
alloca un container in uno dei NodeManager e lo utilizza per invocare un
**application master**.

L'application master è specificato dalle singole applicazioni, ed ha i seguenti
ruoli[@hortonworks-yarn]:

* negoziare l'acquisizione di nuovi container con il ResourceManager nel corso
  dell'applicazione;
* utilizzare i container per eseguire i processi distribuiti di cui è
  costituita l'applicazione;
- monitorare lo stato e il progresso dell'esecuzione dei processi nei
  container.

Le richieste di container specificano CPU, memoria, e la specifica macchina
dove si desidera l'esecuzione. Tra i parametri della richiesta è anche
possibile specificare se si vuole permettere l'esecuzione in una macchina
diversa da quella richiesta, qualora non fosse disponibile.

Mano a mano che la computazione procede, l'application master può riferire al
ResourceManager di rilasciare determinate risorse. Quando il master decide di
porre termine all'applicazione, lo riferisce al ResourceManager, in modo da
permettere il rilascio del container in cui è eseguito.

Il ResourceManager è in grado di gestire più job contemporaneamente utilizzando
diverse politiche di scheduling. L'esecuzione dei job può essere richiesta da
diversi utenti ed entità che hanno accesso al cluster, e la scelta di una
politica di scheduling adeguata permette di stabilire priorità di accesso
diverse per ognuna delle entità coinvolte.

Tra gli scheduler forniti da Hadoop, i seguenti sono i più utilizzati:

* Lo scheduler **FIFO** esegue i job sequenzialmente in ordine di arrivo, e
  ogni job può potenzialmente utilizzare tutte le risorse del risorse del
  cluster.

* Il **Fair Scheduler** esegue i job concorrentemente, fornendo una parte delle
  risorse del cluster a ogni job. I job possono avere una *priorità*, ovvero un
  peso che determina la frazione di risorse che ricevono. Mano a mano che nuovi
  job arrivano, le risorse rilasciate dai job già in esecuzione vengono
  riassegnate al nuovo job per bilanciare la distribuzione delle risorse in
  base ai pesi[@yarn-fair-scheduler]. È anche possibile configurare lo
  scheduler in modo che le risorse siano distribuite in base agli utenti che
  richiedono l'esecuzione dei job. 

* Il **Capacity Scheduler** è il più adatto per condividere cluster tra
  organizzazioni. Lo scheduler viene configurato per avere diverse *code
  gerarchiche* di job, ognuna dedicata a un ente che fa uso del cluster.
  Per ogni coda è specificata una quantità minima di risorse del cluster che
  devono essere disponibili per l'uso in ogni momento, di cui lo scheduler
  garantisce la disponibilità.

Gli scheduler sono implementati in classi, e lo scheduler da istanziare viene
scelto dal ResourceManager cercando, tramite reflection Java, la classe con
il nome indicato in `yarn.resourcemanager.scheduler.class`. L'utente è libero
di implementare un proprio scheduler e di specificarne l'identificatore in
questa proprietà.

