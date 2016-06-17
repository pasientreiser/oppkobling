# Kople til Sonic MQ med .Net

Denne artikkelen beskriver hvordan en kan kople til Sonic MQ med .net-biblioteker (C#).

Som første steg bør en kontakte Pasientreiser for å få en kopi av gjeldende versjon av Sonic sine .net-bilblioteker og dokumentasjon. I den pakken følger det med eksempler, API og mye annen informasjon.
 
## Sette opp en kopling
For å sette opp en vanlig connection, som ikke er fault tolerant bruker man Sonic.Jms.ConnectionFactory. For å sette opp fault tolerant connections må man bruke Sonic.Jms.Ext.ConnectionFactory. Eksemplene her går utifra at en skal sette opp fault tolerant connections. Fremgangsmåten er dog relativt lik foruten nevnte forskjell. 

```c#
var factory = new Sonic.Jms.Cf.Impl.ConnectionFactory();
factory.setFaultTolerant(true); // Sett kopling til å være fault tolerant.
factory.setConnectionURLs("tcp://localhost:2516,tcp://localhost:2519"); // Legg inn URL'ene til aktuelle noder. Legg primærnoden først i listen, da den forsøkes koples opp til først.
connection = (Sonic.Jms.Ext.Connection)factory.createConnection (username, password); // Brukernavn og passord å kople opp med. Ved en ok kopling returneres et Connection-objekt.
```

Koden over må kjøres i en try blokk og unntak (Sonic.Jms.JMSException) må fanges og håndteres. Når en har et connection-objekt må en lage sessions for les eller skriv til kø.

## Sette opp meldingsprodusent

```c#
Sonic.Jms.Session sendSession = connection.createSession(false, Sonic.Jms.SessionMode.AUTO_ACKNOWLEDGE); // Opprett session. Dette statement sier "false" til om hvorvidt det er en transacted session og setter acknowledge mode til auto.
Sonic.Jms.Queue sendQueue = sendSession.createQueue("minKø"); // Opprett sendesesjon mot køen "minKø".
Sonic.Jms.MessageProducer sender = sendSession.createProducer(sendQueue); // Opprett meldingsprodusent, sender blir da produsenten.
```

## Sende melding

```c#
Sonic.Jms.TextMessage msg = sendSession.createTextMessage(); // Her opprettes en melding av type tekstmelding, se dok. for andre varianter.
msg.setBooleanProperty(Sonic.Jms.Ext.Constants.PRESERVE_UNDELIVERED, true); // Ved feil skal meldingen legges på DLQ.
msg.setBooleanProperty(Sonic.Jms.Ext.Constants.NOTIFY_UNDELIVERED, true); // Logg ved feil av levering av melding.
msg.setText("Her legges teksten i meldingen");
sender.send(msg, Sonic.Jms.DeliveryMode.PERSISTENT, Sonic.Jms.DefaultMessageProperties.DEFAULT_PRIORITY, 0); // PERSISTENT meldingen garanteres levert av Sonic. Normal prioritet og uendelig TTL.
```

## Sette opp meldingsmottaker

```c#
receiveSession = connection.createSession(false, Sonic.Jms.SessionMode.CLIENT_ACKNOWLEDGE); // Opprett session. Dette statement sier "false" til om hvorvidt det er en transacted session og setter acknowledge mode til auto.
Sonic.Jms.Queue receiveQueue = receiveSession.createQueue("minKø"); // Opprett lesesesjon mot køen "minKø".
Sonic.Jms.MessageConsumer qReceiver = receiveSession.createConsumer(receiveQueue); // Opprett meldingsmottaker qReceiver blir da produsenten.
qReceiver.setMessageListener(this); // Sett lytteren til køen - behandling av meldinger implementeres i onMessage.
connection.start(); // Start koplingen.
```

### Ang. acknowledgement modes
Det finnes fire forskjellige acknowledgement modes:
- **AUTO_ACKNOWLEDGE** - Sesjonen styrer selv ack, ved en feil kan den siste leverte meldingen, i enkelte tilfeller, leveres på nytt.
- **CLIENT_ACKNOWLEDGE** - Klienten må selv styre ack. Ved en feil kan alle meldinger som ikke er ack'et leveres på nytt. En ack vil ved denne settingen "ack'e" samtlige meldinger som ikke var "ack'et"
- **DUPS_OK_ACKNOWLEDGE** - Sesjonen styrer selv ack, ved en feil kan flere av de siste leverte meldinger, i enkelte tilfeller, leveres på nytt. 
- **SINGLE_MESSAGE_ACKNOWLEDGE** - Klienten styrer selv ack. En ack vil i dette tilfellet kun "ack'e" sist mottatte melding. Ved en feil vil alle meldinger som ikke har fått "ack" bli levert på nytt.

ACK-mode ignoreres når en sender meldinger. Det gjelder kun mottak, dvs. at denne settingen ignoreres dersom sesjonen kun sender meldinger.
 
### Kodeeksempel på mottak av en tekstmelding.

Meldingsinnholdet parset som tekst og skrevet til output:

```c#
public virtual void onMessage(Sonic.Jms.Message aMessage) {
    try {
        Sonic.Jms.TextMessage textMessage = (Sonic.Jms.TextMessage) aMessage;
        String msgText = textMessage.getText();
        Console.Out.WriteLine(msgText);
        aMessage.acknowledge();
    }
    catch (Exception e) {
        Console.Error.WriteLine(e);
    }
}
```
