\clearpage

# Introduzione

Negli ultimi decenni, i Big Data hanno preso piede in modo impetuoso in una
grande varietà di ambiti. Il fenomeno ha avuto un enorme impatto: settori come
medicina, finanza, business analytics e marketing sfruttano i Big Data per
guidare lo sviluppo in modi semplicemente non possibili prima.

L'innovazione che rende possibili questi risultati è guidata dal software
molto più che dall'hardware. Ci sono stati dei grandi cambiamenti nel
modo di pensare alla computazione e all'organizzazione dei suoi processi, che
hanno portato a risultati notevoli nell'efficienza di elaborazione di grandi
quantità di dati.

Uno dei più importanti fenomeni che hanno portato a questa spinta è stato lo
sviluppo di Hadoop, un framework open source progettato per la computazione
batch di dataset di grandi dimensioni. Utilizzando un'architettura ben
congeniata, Hadoop ha permesso l'analisi in tempi molto rapidi di interi
dataset di dimensioni nell'ordine dei terabyte, fornendo una capacità di
sfruttamento di questi, e conseguentemente un valore molto più alti.

Una delle conseguenze più importanti di Hadoop è stata una democratizzazione
delle capacità di analisi dei dati:

* Hadoop è sotto licenza Apache, permettendo a chiunque di utilizzarlo a scopi
  commerciali e non;
* Hadoop non richiede hardware costoso ad alta affidabilità, e incoraggia
  l'adozione di macchine più generiche e prone al fallimento per il suo uso,
  che possono essere ottenute a costi inferiori;
* Il design di Hadoop permette la sua esecuzione in cluster di macchine
  eterogenee nel software e nell'hardware che possono essere acquisite da
  diversi rivenditori, un altro fattore che permette l'abbattimento dei costi;
* I vari modelli di programmazione in Hadoop hanno in comune l'astrazione della
  computazione distribuita e dei problemi intricati che questa comporta,
  abbassando la barriere in entrata in termini di conoscenze e lavoro richiesti
  per creare programmi che necessitano di un altro grado di parallelismo.

Questi fattori hanno spinto a una vasta adozione di Hadoop e dell'ecosistema
software che lo circonda, in ambito aziendale e scientifico. 
L'adozione di Hadoop, secondo un sondaggio fatto a maggio
2015[@hadoop-adoption-survey], si aggira al 26% delle imprese negli Stati
Uniti, e si prevede che il mercato attorno ad Hadoop sorpasserà i 16 miliardi
di dollari nel 2020 [@hadoop-market-analysis].

Tutto questo accade in un'ottica in cui la produzione di informazioni aumenta
ad una scala senza precedenti: secondo uno studio di IDC[@digital-univ], la
quantità di informazioni nell'"Universo Digitale" ammontava a 4.4 TB nel 2014,
e la sua dimensione stimata nel 2020 è di 44 TB.

Data la presenza di questa vasta quantità di informazioni, lo sfruttamento
efficace di queste è fonte di grandi opportunità. In questo documento si
analizzano le varie tecniche che sono a disposizione per l'utilizzo effettivo
dei Big Data, come queste differiscono tra di loro, e quali strumenti le
mettono a disposizione. Si parlerà inoltre di come gli strumenti possano essere
integrati in sistemi di produzione esistenti, le possibili architetture di un
sistema di questo tipo e come ...

La gestione di sistemi per l'elaborazione di Big Data richiede una
configurazione accurata per ottenere affidabilità e fault-tolerance. Pur
sottolineando che l'importanza di questi aspetti non è da sottovalutare, questa
tesi si concentrerà più sul modello computazionale e di programmazione che gli
strumenti offrono.

## Big Data

Per Big Data si intendono collezioni di dati con caratteristiche tali da
richiedere strumenti innovativi per poterli gestire e analizzare.
Uno dei modelli tradizionali e più popolari per descrivere le
caratteristiche dei Big Data si chiama **modello delle 3V**. Il modello
identifica i Big Data come collezioni di informazione che presentano
grande abbondanza in una più delle seguenti caratteristiche:

* Il **volume** delle informazioni, che può aggirarsi dalle decine di terabyte
  per arrivare all'ordine dei petabyte;
* La **varietà**, intesa come la varietà di *fonti* e di *possibili
  strutturazioni* delle informazioni di interesse;
* La **velocità** di produzione delle informazioni di interesse.

Ognuno dei punti di questo modello deriva da esigenze che vanno ad accentuarsi
andando avanti nel tempo, in particolare:

* Il volume delle collezioni dei dati è aumentato esponenzialmente in tempi
  recenti, con l'avvento dei Social Media, dell'IOT, e degli smartphone
  muniti di molti sensori diversi. Generalizzando, i fattori che hanno portato
  a un grande incremento del volume dei data set sono un aumento della
  generazione automatica di dati da parte dei dispositivi e dei contenuti
  prodotti dagli utenti.

* L'aumento dei dispositivi e dei dati generati dagli utenti portano
  conseguentemente a un aumento delle fonti dei dati, ed essendo queste gestite
  da enti e persone diverse la struttura che le fonti presentano difficilmente
  sarà uniforme, l'una rispetto all'altra. Inoltre, l'utilizzo di dati non
  strutturati rigidamente è prevalente nelle tecnologie web (in particolare
  documenti JSON), che sono spesso un obiettivo desiderabile di analisi.

* Si possono fare le stesse considerazioni fatte per il volume dei dati per
  quanto riguarda la velocità. I flussi di dati vengono generati dai
  dispositivi e dagli utenti, che li producono a velocità molto maggiori
  rispetto agli operatori.

Per l'elaborazione di dataset con queste caratteristiche sono stati sviluppati
molti strumenti, che usano diversi pattern di elaborazione a seconda delle
esigenze dell'utente e del tipo di dati con cui si ha a che fare. Molto spesso
questi strumenti hanno come anello di collegamento Hadoop, che è molto
versatile dal punto di vista dell'integrazione con altri tool, al punto che
spesso ci si riferisce ad Hadoop come all'ecosistema dei prodotti che si
possono interfacciare con questo.

A seconda delle esigenze, è utile avere a disposizione diversi pattern di
elaborazione dei Big Data. I modelli di elaborazione più importanti e
rappresentativi sono il *batch processing* e lo *stream processing*.



## Batch e Streaming Processing

Il **batch processing** è il punto di forza di Hadoop, che è stato progettato
con questo modello di elaborazione in mente. Il batch processing è il pattern
di elaborazione generalmente più efficiente, e consiste nell'elaborazione di un
intero dataset in un'unità di lavoro, per poi ottenere i risultati al termine
di questa. 

Questo approccio è ottimale quando non c'è una necessità impendente di avere
risultati, ma in alcuni casi è necessario avere i risultati a disposizione mano
a mano che la computazione procede, e il batch processing non è adatto a questo
scopo:

* Le fasi del batch processing richiedono la schedulazione dei lavori da parte
  dell'utente, con un conseguente overhead dovuto alla schedulazione in sé o
  alla configurazione di strumenti automatizzati che se ne occupino;

* Non è possibile accedere ai risultati prima del termine del job, che può
  avere una durata eccessiva rispetto alle esigenze dell'applicazione o
  dell'utente.

Per use case in cui questi fattori sono rilevanti, lo **stream processing** si
presta come più adatto. In questo paradigma, i dati da elaborare vengono
ricevuti da stream (nella maggior parte dei casi da Internet) e vengono
processati mano a mano con il loro arrivo. I job in streaming molto spesso non
hanno un termine prestabilito, ma vengono terminati dall'utente, e i risultati
dell'elaborazione possono essere disponibili mano a mano che l'elaborazione
procede, permettendo quindi un feedback più rapido rispetto ai lavori batch.

### *Data at Rest* e *Data in Motion*

I due paradigmi si differenziano anche per il modo in cui i dati sono
disponibili. Il processing batch richiede che l'informazione sia *data at
rest*, ovvero informazioni salvate interamente in un mezzo di memoria
accessibile al programma. I dati di input in una computazione batch sono
determinati all'inizio dell'elaborazione, e non possono cambiare nel corso di
questa. Questo significa che se nuova informazione arriva nel corso di un job
batch, questa non può essere tenuta in conto nell'elaborazione finale.

Lo **streaming processing**, invece, è progettato per *data in motion*, dati in
arrivo continuo la cui quantità non è fissa a priori. È possibile utilizzare
strumenti di processing in streaming anche per *data at rest*, rappresentando
il dataset come uno stream. Questa proprietà è desiderabile, perché permette di
utilizzare le stesse applicazioni per l'elaborazione di dati in arrivo dalla
rete e quelli salvati. Come si vedrà, la Kappa architecture, ovvero una
possibile architettura software per l'elaborazione di Big Data, utilizza questa
proprietà per sfruttare un solo paradigma sia per computazioni in real-time di
dati in arrivo da stream, che per i dati salvati storicamente, permettendo di
utilizzare gli stessi tool e interfacce di programmazione per entrambi i tipi
di elaborazione e massimizzare il riutilizzo di codice.

+----------------------+-----------------+-----------------------------------+
| Caratteristiche      | Batch           | Streaming                         |
+======================+=================+===================================+
| Ottimizzazione       | Alto throughput | Bassa latenza                     |
+----------------------+-----------------+-----------------------------------+
| Tipo di informazione | *Data at rest*  | *Data in motion* e *Data at rest* |
+----------------------+-----------------+-----------------------------------+
| Accesso ai dati      | Stabilito       | Dipendente dallo stream           |
|                      | all'inizio      |                                   |
+----------------------+-----------------+-----------------------------------+
| Accesso ai risultati | Fine job        | Continuo                          |
+----------------------+-----------------+-----------------------------------+

: Differenze tra elaborazione batch e streaming

### Esempi di 
