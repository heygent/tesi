\clearpage

# Introduzione {-}

Negli ultimi decenni, i Big Data hanno preso piede in modo impetuoso in una
grande varietà di ambiti. Settori come medicina, finanza, business analytics e
marketing sfruttano i Big Data per guidare lo sviluppo, utilizzando tecnologie
che riescono a ricavare valore da grandi dataset in tempi eccezionalmente brevi
rispetto al passato.

L'innovazione che rende possibili questi risultati è stata guidata dal software
molto più che dall'hardware. Nuove idee sulla computazione distribuita
e sull'organizzazione dei suoi processi hanno permesso un notevole progresso
nell'efficienza di elaborazione di grandi quantità di dati.

Uno dei fattori più importanti ad aver dato slancio a questo fenomeno è stato
lo sviluppo di Hadoop, un framework open source progettato per la computazione
batch di dataset di grandi dimensioni. Utilizzando un'architettura ben
congeniata, Hadoop ha permesso l'analisi in tempi molto rapidi di interi
dataset di dimensioni nell'ordine dei terabyte, fornendo una capacità di
sfruttamento di questi, e conseguentemente un valore, molto più alti.

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
software che lo circonda, in ambito aziendale e scientifico. L'adozione di
Hadoop, secondo un sondaggio fatto a maggio 2015[@hadoop-adoption-survey], si
aggira al 26% delle imprese negli Stati Uniti, e si prevede che il mercato
attorno ad Hadoop sorpasserà i 16 miliardi di dollari nel 2020
[@hadoop-market-analysis].

Tutto questo accade in un'ottica in cui la produzione di informazioni aumenta
ad una scala senza precedenti: secondo uno studio di IDC[@digital-univ], la
quantità di informazioni nell'"Universo Digitale" ammontava a 4.4 TB nel 2014,
e la sua dimensione stimata nel 2020 è di 44 TB. Data la presenza di questa
vasta quantità di informazioni, il loro sfruttamento efficace può essere fonte
di grandi opportunità. 

In questo documento si analizzano le varie tecniche che sono a disposizione per
l'utilizzo effettivo dei Big Data, come queste differiscono tra di loro, e
quali strumenti le mettono a disposizione. Nella prima parte si analizzano i
vari tipi di paradigmi di processing e di come differiscono tra loro, e le
architetture software basate su di questi. Nella seconda parte si affronta
Hadoop, il framework per la computazione distribuita più popolare per l'analisi
di Big Data. Nella terza parte e quarta parte si osservano i paradigmi di
elaborazione *batch* e *stream*, e due tool che li mettono a disposizione,
MapReduce e Spark, e degli esempi pratici per illustrare il loro fuzionamento.

La gestione di sistemi per l'elaborazione di Big Data richiede una
configurazione accurata per ottenere affidabilità e fault-tolerance. Pur
sottolineando che l'importanza di questi aspetti non è da sottovalutare, questa
tesi si concentrerà più sul modello computazionale e sulle interfacce di
programmazione che gli strumenti offrono.

