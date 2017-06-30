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

Per diminuire la verbosità dei comandi, Hadoop può essere configurato per
riferirsi a un filesystem di default quando riceve comandi che usano un URI
relativo, accorciando l'esempio precedente a:

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


Alcuni tool di amministrazione di cluster Hadoop offrono GUI web con cui è
possibile interfacciarsi in HDFS. Alcuni esempi sono Cloudera Manager e Apache
Ambari, che offrono un file manager lato web con cui è possibile interagire in
modo più semplice, permettendo anche a utenti meno esperti nel campo di
lavorare con il filesystem.

![Screenshot del file manager HDFS incluso in Ambari](img/ambari_hdfs.png)

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

![Schema di funzionamento dell'architettura di HDFS](img/hdfsarchitecture.png)


## MapReduce

Per dare un'idea concreta di come funziona la programmazione in un cluster
Hadoop

MapReduce è il primo importante modello di programmazione a cui Hadoop fa
riferimento per l'esecuzione di applicazioni distribuite. Hadoop è stato
scritto e pensato per l'esecuzione di lavori MapReduce, e nelle prime versioni
era il solo modello di programmazione disponibile.

I lavori MapReduce

