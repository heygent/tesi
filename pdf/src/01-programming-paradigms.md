# Big Data e Paradigmi di Elaborazione

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
esigenze dell'utente e del tipo di dati con cui si ha a che fare. I modelli di
elaborazione più importanti e rappresentativi sono il *batch processing* e lo
*stream processing*.

## Batch e Streaming Processing

Il batch processing è il pattern di elaborazione generalmente più efficiente, e
consiste nell'elaborazione di un intero dataset in un'unità di lavoro, per poi
ottenere i risultati al termine di questa. 

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

Lo **stream processing**, invece, è progettato per *data in motion*, dati in
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

Un esempio di *data at rest* sono i resoconti delle vendite di un'azienda, su
cui si possono cercare pattern per identificare quali prodotti sono in trend
nelle vendite. Per *data in motion* si può considerare l'invio di dati da parte
di sensori IoT o le pubblicazioni degli utenti nei social media, che sono
continui e senza una fine determinata.

## Hadoop e modelli di elaborazione

Nella prossima sezione si discuterà di Hadoop, un progetto nato con l'intento
di affrontare la computazione batch
