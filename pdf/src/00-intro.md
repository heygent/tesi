\clearpage

# Introduzione

Negli ultimi anni, i Big Data hanno preso piede in modo impetuoso in una grande
varietà di ambiti. In parte per questa ragione, il fenomeno ha avuto un enorme
impatto: settori come medicina, finanza, business analytics, istruzione e ...
possono sfruttare le capacità analitiche avanzate nei Big Data per guidare lo
sviluppo e l'innovazione in modi semplicemente non possibili prima.

La ragione per cui 

L'innovazione che ha reso possibili questi risultati è guidata dal software
molto più che dall'hardware. Ci sono stati dei grandi cambiamenti nel
modo di pensare alla computazione e all'organizzazione dei suoi processi, che
hanno portato a risultati notevoli primariamente nell'efficienza di
elaborazione di grandi quantità di dati.

Il cambiamento è dovuto principalmente ad Hadoop, un framework open source il
cui design è orientato alla computazione batch di dataset di grandi dimensioni.
Utilizzando un'architettura ben congeniata, Hadoop ha permesso 

Una delle conseguenze più importanti di Hadoop è stata una democratizzazione
delle possibilità di analisi dei dati:

* Hadoop è sotto licenza Apache, permettendo a chiunque di utilizzarlo a scopi
  commerciali e non;
* Hadoop non richiede hardware costoso ad alta affidabilità, e incoraggia
  l'adozione di macchine più generiche e prone al fallimento per il suo uso,
  che possono essere ottenute a costi inferiori;
* Il design di Hadoop permette la sua esecuzione in cluster di macchine
  eterogenee nel software e nell'hardware, che possono essere acquisite da
  diversi rivenditori, un altro fattore che permette l'abbattimento dei costi;
* I vari modelli di programmazione in Hadoop hanno in comune l'astrazione della
  computazione distribuita e dei problemi intricati che questa comporta,
  abbassando la barriere di entrata in termini di conoscenze e lavoro richiesti
  per creare programmi che necessitano di un altro grado di parallelismo.

Questi fattori hanno spinto a una vasta adozione di Hadoop e dell'ecosistema
software che lo circonda, in ambito aziendale e scientifico. 
L'adozione di Hadoop, secondo un sondaggio fatto a maggio
2015[@hadoop-adoption-survey], si aggira al
26% delle imprese, e si prevede che il mercato di Hadoop attorno ad Hadoop
sorpasserà i 16 miliardi di dollari nel 2020 [@hadoop-market-analysis].

http://www.techrepublic.com/article/the-secret-ingredients-for-making-hadoop-an-enterprise-tool/

## Definizione di Big Data

Per Big Data si intendono collezioni di dati non gestibili da tecnologie
"tradizionali". La ragione per cui queste collezioni non sono gestibili è
rilevabile in tre fattori

* Il **volume** della collezione;
* La **varietà**, intesa come la varietà di *fonti* e di *possibili
  strutturazioni* dell'informazione;
* La **velocità** dell'informazione, intesa come la velocità di produzione di
  nuova informazione.

Ognuno dei punti di questo modello deriva da esigenze relativamente recenti (a
volte per la tipologia, a volte per la scala di necessità), in particolare:

* Il volume delle collezioni dei dati è aumentato esponenzialmente in tempi
  recenti, con l'avvento dei Social Media, dell'IOT, e degli smartphone
  muniti di molti sensori diversi. Generalizzando, i fattori che hanno portato
  a un grande incremento del volume dei data set sono un aumento della
  generazione automatica di dati da parte di dispositivi (sensori a basso costo
  e smartphone), in opposizione all'inserimento manuale dei dati da parte di
  operatori, e di un grande incremento dei contenuti prodotti dagli utenti
  rispetto al passato.

* La varietà delle collezioni di dati è aumentata, perché ci sono più fonti
  rispetto che in passato da cui è desiderabile attingere dati, e molte fonti
  forniscono dati che non sono strutturati uniformemente rispetto alle altre.
  Le fonti possono differire in struttura, o possono essere non strutturate
  affatto, come nel caso dei documenti JSON o del linguaggio naturale.
  Una struttura uniforme è una condizione necessaria per l'elaborazione
  corretta dei dati, e a volte può non essere triviale giungere a questa
  condizione. Ci sono molti casi in cui le fonti di dati possono avere
  informazioni non corrette che richiedono di essere filtrate, o in cui è 
  necessario applicare strategie difensive nei confronti dei dati ricevuti, per
  la possibilità che questi siano mal filtrati o provengano da una fonte non
  sicure.

* Si possono fare le stesse considerazioni fatte per il volume dei dati per
  quanto riguarda la velocità. I flussi di dati vengono generati dai
  dispositivi e dagli utenti, che li producono a velocità molto maggiori
  rispetto a degli operatori.

La definizione di Big Data fornita parla di collezioni di dati non
gestibili da tecnologie tradizionali. Definite le caratteristiche di queste
collezioni, le domande consequenziali a questa definizione sono, *quali sono
le tecnologie tradizionali*, e *perché non sono adeguate?*

## RDBMS

I database relazionali sono stati in grado di gestire consistentemente e
affidabilmente 



