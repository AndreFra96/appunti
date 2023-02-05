# Architettura degli Elaboratori II

## Hazard e Forwarding

### Hazard

Situazione che si presenta quando un'istruzione non può essere eseguita nel ciclo di clock immediatamente successivo a quello dell'istruzione precedente

#### Cause strutturali

Due o più istruzioni in pipeline necessitano lo stesso componente, ho bisogno di duplicare i componenti:

- 3 ALU
- Divisione MM:  
  La **memoria istruzioni** e la **memoria dati** sono due cache che vengono mappate in punti diversi della MM, la mem istruzioni nel primo segmento (da 0 a 256 MB) e la memoria dati nel segmento data (fino a 2 GB).  
  In questo modo possiamo utilizzare la memoria istruzioni nella fase di IF e la memoria dati nella fase di MEM.

#### Cause di dati

Devo eseguire un'istruzione in cui almeno uno degli operandi è il risultato di un'operazione precedente.

Il dato all'interno dei registri viene aggiornato durante la fase di WB, a questo punto però le tre istruzioni successive (quelle che si trovano in MEM, EXE, ID) hanno già caricato il dato sbagliato e andranno a causare degli errori.

> Anche l'istruzione in fase di ID legge il dato sbagliato perchè (essendo i registri dei FLIP-FLOP) all'interno di un ciclo di clock la prima parte del ciclo è dedicata alla lettura dal register file mentre la seconda parte alla scrittura, quello che succede è quindi che nella prima fase di clock l'istruzione che si trova in ID legge dal register file il dato sbagliato e nella seconda fase del ciclo viene scritto il dato giusto nel registro.

> Approfondimento - Perche i registri sono FLIP-FLOP?  
> I registri sono dei FLIP-FLOP per evitare di eseguire operazioni in loop nel caso in cui il registro target sia lo stesso di uno dei due operandi, con un flip-flop ho il nuovo dato in uscita al ciclo di clock successivo e non immediatamente. Questo era un problema nelle CPU senza pipeline, in quelle con pipeline ho il registro di transizione ID/EXE che mi permette di non andare in loop potrei quindi usare dei Latch D.

Nell'esempio seguente quando l'istruzione di add si trova in fase di WB le tre istruzioni successive hanno letto un dato errato:

```
add $t0 $t1 $t2  #WB
sub $s0 $t3 $t0  #MEM ($t0 errato)
or  $s1 $t0 $t2  #EXE ($t0 errato)
and $s2 $t0 $t2  #ID  ($t0 errato)
sw  $s6 100($t0) #IF  ($t0 corretto)
```

##### Soluzione 1 - aspettare (stallo)

La prima soluzione consiste nell'aspettare 3 cicli a caricare l'istruzione successiva, in questo modo il dato corretto sarà presente nel registro nel momento dell'esecuzione.

Per ottenere questo risultato è necessario che il compilatore inserisca 3 istruzioni che "non fanno nulla", queste operazioni vengono chiamate **nop** (not operation).

Risolve il problema ma spreco 3 cicli di clock, troppo frequente perchè sia accettabile

##### Soluzione 2 - prevenire (esecuzione fuori ordine)

La seconda soluzione consiste nel riordinare le istruzioni in modo tale da sostituire alle 3 nop proposte nella soluzione precedente alcune istruzioni che dovrebbero essere in ogni caso eseguite (istruzioni precedenti o successive che non hanno registri in comune).

Nelle CPU più semplici è il compilatore a occuparsi di riordinare le istruzioni mentre in quelle più complesse è la CPU stessa a farlo.

Questa soluzione ci permette di non sprecare 3 cicli di clock ma non è sempre possibile trovare queste istruzioni da "spostare" quindi non è sempre applicabile

##### Soluzione 3 - modificare

###### Modifica 1 - Register File

Utilizziamo Latch D anzi che Flip-Flop per il register file.

Il register file (come spiegato nell'approfondimento sopra) utilizza flip-flop anzi che Latch D poichè nelle CPU senza pipeline si poteva verificare un problema di loop, per questo motivo anche l'istruzione che si trova in fase di ID legge il dato precedente e non quello appena scritto nel registro.  
Nelle CPU con pipeline però, in uscita dal RF, i dati passano ai registri di transizione che sono loro stessi dei flip-flop, possiamo quindi sostituire ai flip-flop del RF dei Latch D senza pericolo che si verifichino dei loop.

Questa modifica ci permette di leggere in fase di ID il dato corretto nello stesso ciclo di clock in cui viene scritto riducendo da 3 a 2 il numero di cicli di "attesa" necessari.

###### Modifica 2 - Feed Forwarding

I dati corretti sono disponibili nella pipeline prima che vengano scritti nel RF, in particolare:

- i risultati delle istruzioni di tipo aritmetico/logico sono disponibili alla fine della fase di EXE
- i risultati delle operazioni di load sono disponibili alla fine della fase di MEM

Possiamo quindi <u>identificare le criticità</u> e <u>correggerle</u> propagando all'indietro i dati richiesti.

<u>Caso 1</u>

Hazard al passo successivo su rt

```
add $t0 $t1 $t2
sub $s0 $t3 $t0
```

L'istruzione di sub ha bisogno del risultato dell'addizione fra `$t1` e `$t2`

Il dato necessario in questo caso viene prodotto alla fine della fase di EXE in uscita dalla ALU e sarà necessario all'istruzione di sub quando quest'ultima sarà in fase di EXE.

Correzione:

Quando l'istruzione di add è in fase di MEM retropropaghiamo il risultato dell'operazione alla fase di EXE dove a quel punto si troverà la sub e tramite un multiplexer in ingresso alla ALU inseriamo una scelta fra il dato (sbagliato) letto nella fase precedente e il dato (giusto) retropropagato dalla fase di MEM.

Identificazione:

La <u>condizione da verificare per attivare il multiplexer</u> è la seguente:

```
IF (EXE/MEM.rd = ID/EXE.rt AND EXE/MEM.RegWrite = 1)
THEN utilizza il dato retropropagato
ELSE utilizza il dato classico
```

Che in linguaggio naturale corrisponde a:

"Se il registro di destinazione del'istruzione che si trova in fase di MEM è lo stesso registro del registro target dell'istruzione che si trova in fase di EXE e l'istruzione in fase di MEM è un'istruzione che deve scrivere nel register file, allora si è verificato un hazard e utilizzo il dato retropropagato, altrimenti utilizzo il dato classico"

> Nelle CPU fornite dal prof:
>
> > Il bus che porta il contenuto del registro rd (EXE/MEM.rd) è l'ultimo bus in basso in uscita dal registro di transizione EXE/MEM  
> > Il bus che porta il contenuto del registro rt (ID/EXE.rt) è il primo dei due bus in basso in uscita dal registro di transizione ID/EXE che finiscono nel mux  
> > Il bus che porta l'informazione "l'istruzione deve scrivere nel RF" è quello in alto in uscita dal registro di transizione EXE/MEM nella sezione dedicata ai segnali di controllo di WB

Il circuito che si occupa di controllare questa condizione viene chiamata **forwarding unit** ed è composta da un comparatore che confronta l'indirizzo di ID/EXE.rt con quello di EXE/MEM.rd e poi passa il risultato del confronto in and con il segnale di controllo EXE/MEM.RegWrite.

<u> Caso 2 </u>

Hazard al passo successivo su rs e rt

```
add $t0 $t1 $t2
sub $s0 $t0 $t0
```

L'istruzione di sub ha bisogno del risultato dell'addizione fra `$t1` e `$t2` per entrambi gli operandi

Questo caso viene gestito in maniera praticamente identica al caso 1, le modifiche rispetto a quel caso sono:

- Inserimento di un mux anche nel primo ingresso della ALU
- Passaggio del registro ID/EXE.rs alla forwarding unit
- Modifica della **forwarding unit** aggiungendo un'unità di controllo parallela che fa lo stesso controllo del primo caso ma con il registro rs al posto del registro rt restituendo due segnali di controllo uno dei quali va al primo mux e uno va al secondo.  
   nuovo controllo:

  ```
  IF (EXE/MEM.rd = ID/EXE.rt AND EXE/MEM.RegWrite = 1)
  THEN utilizza il dato retropropagato per l'ingresso A della ALU
  ELSE utilizza il dato classico per l'ingresso A della ALU

  IF (EXE/MEM.rd = ID/EXE.rs AND EXE/MEM.RegWrite = 1)
  THEN utilizza il dato retropropagato per l'ingresso B della ALU
  ELSE utilizza il dato classico per l'ingresso B della ALU
  ```

<u> Caso 3 </u>

Hazard al secondo passo successivo su rt

```
add $t0 $t1 $t2
sub $s0 $s0 $t3
or $s1 $t1 $t0
```

L'istruzione or ha bisogno del risultato dell'addizione fra `$t1` e `$t2`

Il dato necessario in questo caso si trova nella fase di WB insieme all'istruzione add e sarà necessario all'istruzione or che si trova in fase di EXE.

Correzione:

Quando l'istruzione di add è in fase di MEM retropropaghiamo l'uscita del multiplexer allo stadio di EXE dove a quel punto si troverà la or e tramite un multiplexer in ingresso alla ALU inseriamo una scelta fra il dato (sbagliato) letto nella fase precedente e il dato (giusto) retropropagato dalla fase di MEM.

Identificazione:

La <u>condizione da verificare per attivare il multiplexer</u> è la seguente:

```
IF (MEM/WB.rd = ID/EXE.rt AND MEM/WB.RegWrite = 1)
THEN utilizza il dato retropropagato
ELSE utilizza il dato classico
```

Che in linguaggio naturale corrisponde a:

"Se il registro di destinazione del'istruzione che si trova in fase di EB è lo stesso registro del registro target dell'istruzione che si trova in fase di EXE e l'istruzione in fase di WB è un'istruzione che deve scrivere nel register file, allora si è verificato un hazard e utilizzo il dato retropropagato, altrimenti utilizzo il dato classico"

> Nelle CPU fornite dal prof:
>
> > Il bus che porta il contenuto del registro rd (MEM/WB.rd) è l'ultimo bus in basso in uscita dal registro di transizione MEM/WB  
> > Il bus che porta il contenuto del registro rt (ID/EXE.rt) è il primo dei due bus in basso in uscita dal registro di transizione ID/EXE che finiscono nel mux  
> > Il bus che porta l'informazione "l'istruzione deve scrivere nel RF" è quello in alto in uscita dal registro di transizione MEM/WB nella sezione dedicata ai segnali di controllo di WB

<u> Caso particolare: </u>

In questo caso il contenuto del registro `$t0` necessario per la or deve essere retropropagato da MEM o WB?

```
add $t0 $t1 $t2
sub $t0 $t0 $t3
or  $s1 $t0 $t2
```

Deve essere retropropagato dalla fase più vicina a quella dov'è eseguito, ovvero la fase di MEM perchè l'istruzione che si trova in quella fase potrebbe avere a sua molta modificato il contenuto del registro (come avviene proprio in questo esempio)

<u> Hazard sui dati con branch </u>

```
sub $t2 $s4 $s3
add $t2 $s4 $s3
beq $t2 $s6 7
```

Come vedremo più avanti il confronto fra $t2 e $s6, in seguito alle modifiche alla CPU per migliorare la gestione degli hazard sul controllo, viene effettuato nella fase di ID.

Dipendenza fra sub e beq:

La dipendenza fra la sub e beq può essere gestita tramite una modifica alla forwarding unit, abbiamo infatti bisogno di \$t2 quando l'istruzione beq si trova in fase di ID per effettuare il confronto, a quel punto l'istruzione sub si troverà in fase di MEM e possiamo retropropagarle il dato in in uscita dal registro MEM/WB.

Dipendenza fra sub e add:

<span style="color:red; font-weight: bold">Non lo dice??? come si fa?? uno stallo???</span>

###### Generazione segnali MUX

L'unità di forwarding si occupa di generare i segnali di controllo necessari per i multiplexer in ingresso alla ALU:

| ID/EXE.rs = EXE/MEM.rd & EXE/MEM.RegWrite | ID/EXE.rs = MEM/WB.rd & MEM/WB.RegWrite | MUX A |
| ----------------------------------------- | --------------------------------------- | ----- |
| 0                                         | 0                                       | 00    |
| 0                                         | 1                                       | 10    |
| 1                                         | 0                                       | 01    |
| 1                                         | 1                                       | 01    |

Il multiplexer utilizza il segnale per selezionare gli ingressi della ALU:

| Controllo MUX | Registro sorgente | Funzione                                                                                      |
| ------------- | ----------------- | --------------------------------------------------------------------------------------------- |
| PropagaA=00   | ID/EXE            | Il primo operando della ALU proviene da RF                                                    |
| PropagaA=01   | EXE/MEM           | Il primo operando della ALU è propagato dal risultato della ALU per l'istruzione precendete   |
| PropagaA=10   | MEM/WB            | Il primo operando della ALU è propagato dalla memoria o da un'altra istruzione precedente     |
| PropagaB=00   | ID/EXE            | Il secondo operando della ALU proviene dal Register File                                      |
| PropagaB=01   | EXE/MEM           | Il secondo operando della ALU è propagato dal risultato della ALU per l'istruzione precedente |
| PropagaB=10   | MEM/WB            | Il secondo operando della ALU è propagato dalla memoria o da un'altra istruzione precedente   |

##### STALL ON LOAD

Cosa succede se l'istruzione che causa un hazard sui dati è una lw?

```
lw $s2 40($s3)
add $t2 $s2 $s5
or $t3 $s6 $s2
```

###### Problema

In questo caso il dato che deve essere scritto nel registro \$s2 dalla lw è disponibile alla fine della fase di MEM, a differenza dei casi visti in precedenza dove era disponibile alla fine della fase di EXE.

Utilizzando il classico circuito di forwarding quello che succede è che quando l'istruzione di lw è in fase di MEM viene retropropagato all'istruzione precedente l'indirizzo di memoria alla quale deve accedere l'istruzione anzi che il dato stesso.

###### Soluzione

Non possiamo fare altro che introdurre uno stallo per l'istruzione immediatamente successiva alla lw, in questo modo quando la lw sarà in fase di WB ci sarà una `nop` in fase di MEM e l'istruzione di add sarà in fase di EXE. Tramite il circuito classico di retropropagazione viene passato il dato richiesto dalla fase di WB a quella di EXE.

###### Applicazione

Il compilatore può utilizzare ad esempio il riordinamento del codice per prevenire queste situazioni ma non sempre è possibile, è necessario quindi che la cpu sappia gestire queste situazioni.

1. Rilevazione

"Se l'istruzione corrente è una lw è uno dei registri operando dell'istruzione successiva è uguale al registro rt della lw, allora occorre inserire una nop"

Quando e come capiamo se l'istruzione è una lw?
Posso accorgermene in fase di ID, utilizzo il segnale di controllo MemRead che è utilizzato da tutte le istruzioni che leggono dalla memoria (in questo modo gestisco tutte e non solo la lw)

```
IF ID/EXE.MemRead AND (IF/ID.rt == ID/EXE.rt OR IF/ID.rs == ID/EXE.rs)
THEN "inserisci una nop"
```

2. Soluzione

Al ciclo di clock successivo devo fare due cose:

1.  Bloccare le due istruzioni successive alla lw  
    ovvero fare in modo che al prossimo ciclo di clock rimangano ferme nella stessa fase. Per ottenere questo risultato devo impostare a 0 il segnale di scrittura del program counter e del registro di transizione IF/ID.

    > devo aggiungere un segnale esplicito perchè di default i registri di transizione hanno come segnale di Write direttamente il clock, quindi aggiungo un segnale esplicito che viene generato dalla condizione descritta sopra che va in and con il clock prima di entrare nel program counter e in IF/ID (la hazard detection unit è quella che fa questo controllo e invia il segnale ai registri)

2.  Depotenziare l'istruzione successiva  
    Al ciclo di clock seguente l'istruzione successiva alla lw (che a questo punto sarà in MEM) verrà rieseguità in fase di ID ma se non facciamo niente verrà eseguita anche in EXE con il dato sbagliato il che può causare problemi.
    Quello che facciamo per gestire questa situazione è "depotenziare" l'istruzione che passa in fase di EXE mettendo a 0 tutti i suoi segnali di controllo, in questo modo qualsiasi istruzione sia non andrà non andrà a modificare lo stato della pipeline (ho inserito una Bolla).
    > per azzerare i segnali di controllo utilizzo un multiplexer nella fase di ID fra l'unità di controllo è un segnale di 0 (la hazard detection unit invia anche questo segnale)

#### Cause di controllo

Riguarda i salti condizionati, il controllo per i salti condizionati avviene in fase di EXE e il salto avviene al ciclo di clock successivo, ovvero quando la branch si trova in fase di MEM.  
Nel frattempo però ho già caricato le 3 istruzioni successive nelle 3 fasi precedenti.

Come per le cause di dati posso utilizzare diverse strategie per gestire gli hazard sul controllo, in particolare:

##### Soluzione 1 - aspettare (stallo)

Aspettiamo a caricare le 3 istruzioni successive alla branch nel program counter, fino a quando l'istruzione non si trova in fase di MEM. A quel punto sapremo se scrivere nel program counter PC + 4 o l'indirizzo di salto.

Per ottenere questo risultato è sufficiente che il compilatore inserisca 3 nop dopo ogni istruzione di branch.  
Questo ci permette di risolvere il problema ma vuol dire sprecare 3 cicli di clock per ogni istruzione di branch, dato che è un'istruzione molto frequente questa non può essere considerata un'opzione valida.

##### Soluzione 2 - prevenire (branch delay slots)

Riorganizzare le istruzioni (lo fa il compilatore) in modo tale da inserire dopo la branch 3 istruzioni che dovrebbero essere eseguire in ogni caso.

##### Soluzione 3 - modificare

Per minimizzare le conseguenze di un eventuale hazard la cosa migliore da fare è rendersi conto prima possibile di quello a cui si sta andando incontro. In questo modo possiamo scartare meno istruzioni e perdere di conseguenza meno cicli di clock.

Per le istruzioni di branch abbiamo tutte le informazioni necessarie per renderci conto di stare andando incontro ad un hazard già in fase di ID, dobbiamo fare però alcune modifiche per gestirlo:

1. **Identificazione** (Anticipazione della valutazione)
   Per identificare l'hazard sul controllo in fase di ID è sufficiente inserire un comparatore che controlli se il contenuto del registro source è uguale a quello di destinazione e leggere in uscita dall'unità di controllo se l'istruzione è un'istruzione di branch.  
   Se entrambe le condizioni sono verificate allora il salto deve essere preso:

   ```
   IF (\*rs == \*rt) AND branch
   THEN "hazard"
   ```

   Il circuito che effettua questo controllo è composto da un comparatore che compara il valore dei due bus in uscita dal RF dove si trovano i valori di rs e rt, dopo di che il segnale in uscita dal comparatore va in AND con il segnale di branch in uscita dal'unità di controllo. Il segnale finale è il segnale di hazard, se è 1 significa che il salto doveva essere effettuato.

2. **Gestione** (Anticipazione del calcolo dell'indirizzo)  
   A questo punto ci siamo resi conto di dover effettuare il salto ma come lo facciamo?  
   Spostiamo in ID il circuito necessario per calcolare l'indirizzo di salto.

   > Il circuito necessario per il calcolo dell'indirizzo di salto è composto da uno shifter che si occupa di fare lo shift a sx di 2 bit (che sarebbe la moltiplicazione per 4) dell'offset (bit dallo 0 al 15 dell'istruzione), e un sommatore che calcoli PC + 4 + offset per ottenere l'indirizzo di salto completo.

3. **Flush** dell'istruzione in IF

   Scartiamo l'istruzione attualmente in IF, non è così immediato perchè non ho segnali di controllo a questo punto perchè non ho ancora decodificato l'istruzione.

   Posso creare una nop, scrivendo in IF/ID l'istruzione composta da tutti 0, questa istruzione corrisponde a `sll $zero $zero 0` ovvero lo shift di zero posizioni del registro \zero con salvataggio nel registro $zero. (operazione che non fa nulla, possiamo utilizzarla come nop).

   Devo scrivere quindi tutti zeri nel registro IF/ID e per farlo posso utilizzare la porta di reset presente nei FLIP-FLOP di cui è composto il registro, quindi <u>il Flush del registro IF/ID si ottiene tramite l'invio del segnale reset ai flip-flop presenti nel registro</u>

##### Soluzione 4 - scartare istruzioni

Posso supporre che il salto non venga effettuato, se effettivamente non viene effettuato non devo fare nient'altro. Altrimenti se il salto viene preso (me ne accorgo quando la branch è in fase di MEM) devo scartare le 3 istruzioni che ho già caricato nella pipeline per errore e che devo buttare via, questa operazione di "buttare via" si chiama flush (svuotamento brusco)

##### Soluzione 5 - prevedere (predire)

Possiamo provare a prevedere se l'istruzione di salto deve essere presa o meno, questo viene fatto tramite alcuni meccanismi di predizioni appositi (della CPU) che cercano di predire su base statistica se il salto deve essere effettuato o meno.  
Se la predizione è corretta posso proseguire con l'esecuzione, altrimenti se la predizione si rivela sbagliata (anche il questo caso me ne accorgo in fase di MEM devo eseguire il **roll-back** (ripristino dello stato della CPU al momento in cui viene eseguita la branch)

Per eseguire il roll back bisogna scartare le 3 istruzioni che sono state caricate per errore e scrivere nel program counter l'indirizzo dell'istruzione corretta da eseguire.
Per scartare le 3 istruzioni attualmente in IF, ID e EXE si utilizza un meccanismo detto <u>Flush</u> (un modo per fare il flush è impostare a 0 i segnali di controllo delle istruzioni, ma non è l'unico)

> Differenza fra flush e stallo:  
> Sono entrambe operazioni che fanno perdere un ciclo di clock alla CPU
> ma nel caso del flush il contenuto dei registri viene "buttato via" mentre
> nel caso dello stallo l'esecuzione dell'istruzione viene messa "in pausa" per un ciclo
> per poi essere ripresa a quello successivo

###### Come funziona la CPU con predizione? (UNITA' DI SPECULAZIONE)

Nella CPU con predizione il codice compilato prima di essere inviato alla CPU passa per un'unità di speculazione che si occupa di riordinare il codice.

CODICE COMPILATO => SPECULAZIONE => CODICE RIODINATO => CPU

L'unità di speculazione per effettuare la predizione sull'indirizzo di salto utilizza il **branch prediction buffer**.

Branch prediction buffer:  
Il branch prediction buffer è appunto un buffer di memoria utilizzato per la predizione dei salti, ogni elemento di memoria all'interno del branch prediction buffer contiene un certo numero di bit che sono un riferimento all'indirizzo della branch (non l'indirizzo di salto eh, proprio l'indirizzo dell'istruzione branch) e un bit che indica se l'ultima volta il salto era stato preso o meno.

<u>Problemi del BPB ad 1 bit</u>:  
prendiamo un classico ciclo for:

```
start: 0x400 beq  $t0 $t1 salta
       0x404 add  $s0 $s1 $s2
       0x408 sub  $s3 $s4 $s5
       0x40C addi $t0 $t0 1
       0x410 start
salta: 0x414 and  $t2 $t3 $t4
```

Vediamo cosa c'è nel BPB:

1. Prima iterazione  
   Alla prima iterazione il branch prediction buffer sarà composto in questo modo:  
   [ 0x400 | 0 ]  
   L'unità di speculazione prevede che il salto non deve essere effettuato e non lo effettua, la predizione si rivela **corretta**

2. Iterazioni intermedie  
   Durante le iterazioni successive il branch prediction buffer non viene modificato:  
    [ 0x400 | 0 ]  
   L'unità di speculazione prevede che il salto non deve essere effettuato e non lo effettua, la predizione si rivela **corretta**

3. Ultima iterazione  
   All'ultima iterazione il branch prediction buffer è ancora uguale:  
    [ 0x400 | 0 ]  
   L'unità di speculazione prevede ancora che il salto non deve essere effettuato ma questa volta la predizione si rivela **sbagliata** (necessario flush dell'istruzione successiva e caricare nel pc l'indirizzo dell'istruzione corretta)  
   Il branch prediction buffer viene modificato:  
    [ 0x400 | 1 ]

4. Prima iterazione successiva  
    A questo punto il ciclo è terminato ma il valore del BPB è rimasto salvato, la prossima volta che viene eseguita la stessa istruzione il branch prediction buffer sarà ancora:  
   [ 0x400 | 1 ]

   L'unità di speculazione prevede che il salto deve essere effettuato ma la predizione si rivela **sbagliata** (necessario flush dell'istruzione successiva e caricare nel pc l'indirizzo dell'istruzione corretta)  
   Il branch prediction buffer viene modificato:  
   [ 0x400 | 0 ]

Con il branch prediction buffer ad 1 bit abbiamo quindi <u>2 errori per ogni ciclo</u>.

Posso migliorare questo risultato utilizzando una macchina a stati finiti che commuti la predizione solo quando si verificano 2 errori consecutivi. (BPB a 2 bit)
