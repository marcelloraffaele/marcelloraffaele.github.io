---
layout: post
title: Creare un semplice Bot Telegram in PHP [ITA]
author: rmarcello
date: 2020-06-27 00:00:00 +0000
description: Come creare un semplice bot Telegram in PHP, versione italiana
image: assets/images/telegram-bot-php/telegram-bot-php.png # Add image post (optional)
categories: [telegram, php, bot, 5min]
comments: false
---

Spesso realizzare un Bot può sembrare un impresa difficile e costosa.
In quest'articolo parlerò di come è possibile creare un Bot Telegram con funzionalità molto semplici utilizzando un hosting PHP che supporta HTTPS. E' possibile trovare un hosting di questo tipo spendendo veramente poco.

### Cosa è un Bot Telegram
Telegram definisce un Bot come un' applicazione di terze parti che vengono eseguite all'interno di telegram. L'utente può interagire con i bot inviando messaggi, comandi e altri tipi di richieste. L'interazione tra l'applicazione e Telegram avviene utilizzando richieste HTTPS e il formato è utilizzato è definito dalla Bot API di Telegram (<a href="https://core.telegram.org/bots/api">https://core.telegram.org/bots/api</a>).

Ovviamente la complessità del Bot dipende dalla complessità della sua implementazione e può andare dalla semplice risposta automatica a programmi complessi che utilizzano l'intelligenza artificiale.
Lo scopo di quest'articolo è quello di raggiungere in breve tempo un risultato accettabile con un bot che implementa le API di base per ricevere i messaggi degli utenti e inviare risposte.

Telegram permette di interagire con le proprie API in due modalità:
<ul>
<li><b>webhook</b>: Telegram invoca il nostro BOT ogni qualvolta ci sarà un nuovo "evento". Per questa modalità è indispensabile HTTPS.</li>
<li><b>long polling</b>: Il nostro BOT chiede periodicamente a Telegram se ci sono nuovi eventi da gestire.</li>
</ul>
In questo articolo utilizzeremo la modalità <b>"webhook"</b>, andremo a definire una pagina php che servirà le richieste provenienti da Telegram. Nell'immagine che segue vediamo il tipico scenario che segue l'invio di un messaggio da parte dell'utente che da smartphone scrive qualcosa al bot. L'app di telegram comunica ai propri server che un determinato utente ha inviato un messaggio a sua volta inoltra la richiesta al nostro servizio che dovrà gestire l'evento. A questo punto il servizio php elabora la risposta le la inoltra al server Telegram che la inoltra all'applicazione dello smartphone.

![telegram-php-diagram]({{site.baseurl}}/assets/images/telegram-bot-php/telegram-php-diagram.png)

Passiamo alla pratica e creiamo il nostro bot.

### Come creare un nuovo Bot

Telegram permette di creare nuovi Bot utilizzando un Bot speciale chiamato "BotFather". Per questo motivo è indispensabile avere un account Telegram. L'account ci tornerà utile anche per fare qualche test e verificare il funzionamento del nostro Bot.

Accediamo a Telegram da smartphone o da web (<a href="https://web.telegram.org/">https://web.telegram.org/</a>) e seguiamo questi passi:
<ol>
<li>creiamo una nuova conversazione con l'utente BotFather oppure clicca qui: <a href="https://t.me/botfather">Botfather</a></li>
<li>digitiamo il comando <b>/newbot</b></li>
<li>Inseriamo il nome del nostro bot, io sceglierò <b>Botzellette</b></li>
<li>Inseriamo lo username del bot che deve terminare per "bot", io sceglierò <b>Botzellette_bot</b>, voi sceglietene un altro perchè non possono esistere più bot con lo stesso username.</li>
</ol>

A questo punto, BotFather genera un token che ci sarà utile più avanti per utilizzare le API.
<b>NOTA: custodite il token al sicuro perchè permette a chiunque di controllare il vostro BOT</b>.

![telegram-bot-botfather]({{site.baseurl}}/assets/images/telegram-bot-php/telegram-bot-botfather.png)

A questo punto abbiamo un token e possiamo definire il servizio PHP.

### Sviluppo
A questo punto possiamo cominciare a scrivere codice, come avrete capito dal nome del bot, vedremo come creare un bot che risponde a richieste dell'utente con alcune barzellette. Per semplicità, l'applicazione non applica logiche complesse, ne fa accessi a database ne implementa intelligenze artificiali. L'esempio che andremo a scrivere ascolterà due comandi:
<ul>
<li>/menu: per mostrare il menu</li>
<li>/next: per richiedere una barzelletta in modo randomico da un insieme predefinito</li>
</ul>

{% highlight php %}
<?php
//configurazione: sostituire qui il vostro token
$botToken = "1234567890:ABCDEFGHILMNOPQRSTUVZABCDEFGHABCDEF";
$botAPI = "https://api.telegram.org/bot" . $botToken;

//estrazione dati della richiesta
$update = file_get_contents('php://input');
//trasformazione json -> array associativo
$update = json_decode($update, TRUE);
                        
if (!$update) {
    exit;
}
//estrazione del messaggio e degli altri campi che si trovano nella richiesta
$message = isset($update['message']) ? $update['message'] : "";
$messageId = isset($message['message_id']) ? $message['message_id'] : "";                       
$date = isset($message['date']) ? $message['date'] : "";
$chatId = isset($message['chat']['id']) ? $message['chat']['id'] : "";
$username = isset($message['chat']['username']) ? $message['chat']['username'] : "";
$firstname = isset($message['chat']['first_name']) ? $message['chat']['first_name'] : "";
$lastname = isset($message['chat']['last_name']) ? $message['chat']['last_name'] : "";

//possibilità di gestire i diversi tipi di richiesta
// messaggio di testo
if (isset($message['text'])) {
	//estrazione del testo del messaggio
	$messageText = isset($message['text']) ? $message['text'] : "";
    
	//implementazione della logica in base al comando inviato
	//se comando /menu o /start
	if (strpos($messageText, "/menu") === 0 || strpos($messageText, "/start") === 0) {
        sendTextMessage($botAPI, $chatId, "Ciao ". $firstname );
        sendTextMessage($botAPI, $chatId, "/menu or /start per visualizzare il menu");
        sendTextMessage($botAPI, $chatId, "/next per dare una nuova barzelletta" );
    }
	//se comando /next
	else if (strpos($messageText, "/next") === 0) {
        sendTextMessage($botAPI, $chatId, nextBarzelletta() );
    }
	//altrimenti
	else {
		sendTextMessage($botAPI, $chatId, "Scusa non ho capito, vuoi sentire una barzelletta? /next" );
	}
	
}
//se un altro tipo di messaggio
else {
    $response = "Scusa ma questo tipo messaggio non è gestito";
    sendTextMessage($botAPI, $chatId, $response);
}

//Funzione di utilità che invia messaggi di testo
function sendTextMessage($api, $chatId, $text) {
    $url = $api . '/sendMessage?chat_id=' . $chatId . '&text=' . urlencode($text) ;
    file_get_contents($url);
}

//funzione di utilità che crea una barzelletta
function nextBarzelletta() {
    $barzellette = array(
        "Cosa dice uno spaventapasseri bugiardo? Dice le balle di fieno!",
         "Lo sai perché nei ristoranti genovesi non possono vedere il conto? Perché è buio pesto! ",
         "Se il cane ringhia, la ringhiera abbaia?!",
         "Cosa dice uno spaventapasseri bugiardo? Dice le balle di fieno!",
         "Ma se una mucca si trova a bratislava, è una slo-vacca?",
         "Come si dice uno scontro tra due carrelli?..... scontrino!"
    );
    return $barzellette[rand(0,count($barzellette)-1)];
}

?>
{% endhighlight %}


A questo punto non ci resta che pubblicare la pagina php e prenderne il path compreso https. Nel mio caso sarà: <b>https://www.rmarcello.it/Botzellette/index.php</b>.

### Impostare il Webhook

Come abbiamo detto nei passaggi precedenti la "connessione" tra i server Telegram e il servizio che abbiam implementato è il Webhook. Bisogna dire alla API di tegram che può gestire tutti i messaggi destinati al nostro bot tramite il nostro link.
Apriamo un semplice browser al link:

https://api.telegram.org/bot<yourtoken>/setwebhook?url=https://yourdomain.com/yourbot.php

se otterrete:
{% highlight json %}
{"ok":true,"result":true,"description":"Webhook was set"}
{% endhighlight %}

allora possiamo provare il nostro bot, accediamo a <a href="t.me/Botzellette_bot">t.me/Botzellette_bot</a>.
 
### Test
Per testare il nostro bot, basta aprire una finestra di chat e inviare messaggi:
![telegram-bot-botBotzelle]({{site.baseurl}}/assets/images/telegram-bot-php/telegram-bot-Botzelle.png)

Oppure guarda questo:
<iframe width="560" height="315" src="https://www.youtube.com/embed/m5EgaGtsIzo" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

### Conclusioni
L'articolo è molto semplice ed ha come scopo quello di mostrare come è possibile utilizzare un semplice dominio PHP per realizzare un BOT Telegram sempre online e senza spendere un capitale per hosting complessi.

L'esempio trattato è molto "casareccio" e implementa manualmente le API base di Telegram. Per la realizzazione di logiche complesse vi consiglio di utilizzare qualche libreria che vi semplificherà la vita sulla gestione degli eventi, immagini, accessi a database e molto altro.