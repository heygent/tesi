# Big Data e Paradigmi di Elaborazione

Per Big Data si intendono collezioni di dati con caratteristiche tali da
richiedere strumenti innovativi per poterli gestire e analizzare.
Uno dei modelli tradizionali e più popolari per descrivere le
caratteristiche dei Big Data si chiama **modello delle 3V**. Il modello
identifica i Big Data come collezioni di informazione che presentano
grande abbondanza in una o più delle seguenti caratteristiche:

* Il **volume** delle informazioni, che può aggirarsi dalle decine di terabyte
  per arrivare fino ai petabyte;
* La **varietà**, intesa come la varietà di *fonti* e di *possibili
  strutturazioni* delle informazioni di interesse;
* La **velocità** di produzione delle informazioni di interesse.

Ognuno dei punti di questo modello deriva da esigenze che vanno ad accentuarsi
andando avanti nel tempo, in particolare:

* Il volume delle collezioni dei dati è aumentato esponenzialmente in tempi
  recenti, con l'avvento dei Social Media, dell'IOT, e degli smartphone.
  Generalizzando, i fattori che hanno portato a un grande incremento del volume
  dei data set sono un aumento della generazione automatica di dati da parte
  dei dispositivi e dei contenuti prodotti dagli utenti.

* L'aumento dei dispositivi e dei dati generati dagli utenti portano
  conseguentemente a un aumento delle fonti, gestite da enti e persone diverse.
  Per questa ragione, le strutture dei dati ricavati difficilmente saranno
  uniformi. Inoltre, l'utilizzo di dati non strutturati rigidamente è
  prevalente nelle tecnologie consumer, business e scientifiche (come documenti
  JSON, XML e CSV), che sono spesso un obiettivo auspicabile per l'analisi.

* Si possono fare le stesse considerazioni fatte per il volume dei dati per
  quanto riguarda la velocità. I flussi di dati vengono generati dai
  dispositivi e dagli utenti, che li producono a ritmi molto più incalzanti
  rispetto agli operatori.

Per l'elaborazione di dataset con queste caratteristiche sono stati sviluppati
molti strumenti, che usano diversi pattern di elaborazione a seconda delle
esigenze dell'utente e del tipo di dati con cui si ha a che fare. I modelli di
elaborazione più importanti e rappresentativi sono il *batch processing* e lo
*stream processing*.

## Batch e Streaming Processing

Il batch processing è il pattern di elaborazione generalmente più efficiente, e
consiste nell'elaborare un intero dataset in un'unità di lavoro, per poi
ottenere i risultati al termine di questa. 

Questo approccio è ottimale quando i dati da elaborare sono disponibili a
priori, e non c'è necessità di ottenere i risultati in tempi immediati o con
bassa latenza. Tuttavia, questo approccio ha dei limiti.

* Le fasi del batch processing richiedono la schedulazione dei lavori da parte
  dell'utente, con un conseguente overhead dovuto alla schedulazione in sé o
  alla configurazione di strumenti automatizzati che se ne occupino;

* Non è possibile accedere ai risultati prima del termine del job, che può
  avere una durata eccessiva rispetto alle esigenze dell'applicazione o
  dell'utente.

Per use case in cui questi fattori sono rilevanti, lo **stream processing** si
presta come più adatto. In questo paradigma, i dati da elaborare vengono
ricevuti da *stream*, che rappresentano flussi di dati contigui provenienti da
origini non necessariamente controllate. Gli stream forniscono nuovi dati in
modo *asincrono*, e la loro elaborazione avviene a ogni nuovo evento di
ricezione. I job in streaming molto spesso non hanno un termine prestabilito,
ma vengono terminati dall'utente, e i risultati dell'elaborazione possono
essere disponibili mano a mano che l'elaborazione procede, permettendo quindi
un feedback più rapido rispetto ai lavori batch.

### *Data at Rest* e *Data in Motion*

I due paradigmi si differenziano anche per il modo in cui i dati sono
disponibili. Il processing batch richiede che l'informazione sia *data at
rest*, ovvero informazioni completamente accessibili a priori dal programma.
I dati di input in una computazione batch sono determinati al suo inizio, e non
possono cambiare durante il suo corso. Questo significa che se si rende
desiderabile dare in input una nuova informazione in un lavoro batch, l'unico
modo per farlo è rieseguire interamente il lavoro.

Lo **stream processing**, invece, è progettato per *data in motion*, dati in
arrivo continuo non necessariamente disponibili prima dell'inizio
dell'elaborazione. Esempi di data in motion possono essere rappresentati dai
dati ricevuti in un socket TCP inviati da reti di sensori IOT, o dall'ascolto
di servizi di social media.

È possibile utilizzare strumenti di processing in streaming anche per *data at
rest*, rappresentando il dataset come uno stream. Questa proprietà è
desiderabile, perché permette di utilizzare le stesse applicazioni per
elaborazioni che riguardano dataset disponibili a priori e stream di cui non si
ha completo controllo.

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

### Architetture di sistemi Big Data

I paradigmi di Batch e Stream processing presentano differenze notevoli nelle
astrazioni, nei tool e nelle API utilizzate. Il loro utilizzo è condizionat

Ad oggi, le architetture dei sistemi che sfruttano i Big Data si basano
principalmente su due modelli, la **lambda** e la **kappa** architecture.

![Diagramma della Lambda Architecture](img/lambda_architecture.png){width=75%}

La lambda architecture utilizza tre unità logiche, il **batch layer**, lo
**speed layer** e il **serving layer**. Il serving layer è un servizio o un
insieme di servizi che permettono di eseguire query sui dati elaborati dagli
altri due layer. Il batch layer esegue framework per computazioni batch, mentre
lo speed layer esegue computazioni stream. Questi due layer rendono disponibili
i risultati delle loro computazioni al serving layer per la consultazione da
parte degli utenti.

Il **batch layer** opera sui dati archiviati storicamente, e riesegue le
computazioni periodicamente per integrare i nuovi dati ricevuti. Questo layer
può eseguire velocemente computazioni sulla totalità dei dati. Lo **speed
layer** invece elabora i dati asincronamente alla loro ricezione, e offre
risultati con una bassa latenza.

Questo approccio è il più versatile, perché permette l'utilizzo di entrambi i
paradigmi e della totalità degli strumenti progettati per batch e stream
processing. Tuttavia, i layer batch e speed richiedono una gestione separata,
e il mantenimento di due basi di codice scritte con API e potenzialmente
linguaggi diversi, anche per applicazioni che eseguono le stesse funzioni.
I sistemi che implementano architetture lambda sono i più onerosi nello
sviluppo e nella manutenzione.

![Diagramma della Kappa Architecture](img/kappa_architecture.png){width=75%}

In contrapposizione, la kappa architecture non utilizza un batch layer, e la
totalità delle computazioni viene eseguita dallo speed layer. Per eseguire
elaborazioni sui dati archiviati, questi vengono rappresentati come uno stream,
che viene dato in ingestione allo speed layer. In questo modo gli strumenti e
le basi di codice possono essere unificate, semplificando l'architettura e
rendendo la gestione del sistema meno impegnativa.

Le differenze tra i due approcci sono più visibili quando si mettono a
confronto i framework di elaborazione batch e streaming per osservare le
differenze nell'uso. Come regola generale, si può definire preferibile la
lambda architecture per l'efficienza delle computazioni, superiore nei sistemi
di elaborazione batch. La lambda architecture è preferibile quando le
elaborazioni che si vogliono eseguire sui dati storici e quelli in arrivo sono
identiche o molto simili, o si vuole ottenere un sistema architetturalmente più
semplice. Spesso la scelta dipende da un tradeoff tra questi due fattori.

