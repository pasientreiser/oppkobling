# Kople til Sonic MQ med Java
Denne artikkelen beskriver hvordan en kan kople til Sonic MQ med Java-biblioteker

Som første steg bør en kontakte Pasientreiser for å få en kopi av gjeldende versjon av Sonic sine Java-bilblioteker og dokumentasjon. I den pakken følger det med eksempler, API og mye annen informasjon.
 
## Sette opp en kopling

```java
QueueConnectionFactory factory = new progress.message.jclient.QueueConnectionFactory(); // Instansier et factoryobjekt
factory.setFaultTolerant(true); // Denne må settes til true om en skal ha en fault tolerant kopling som klient. Default er false.
factory.setConnectionURLs("tcp://localhost:2506,tcp://localhost:2516"); // Kommaseparert liste over brokere å forsøke å kople seg til.
QueueConnection connection = factory.createQueueConnection (username, password); // Opprett kopling, spesifiser brukernavn og passord.
```
Koden over må kjøres i en try blokk og unntak (javax.jms.JMSException) må fanges og håndteres.

## Sette opp meldingsprodusent

```java
javax.jms.Session sendSession = connection.createSession(false,javax.jms.Session.AUTO_ACKNOWLEDGE); // Opprett session. Dette statement sier "false" til om hvorvidt det er en transacted session og setter acknowledge mode til auto.
javax.jms.Queue sendQueue = sendSession.createQueue ("minKø"); // Send meldinger mot "minKø".
javax.jms.MessageProducer sender = sendSession.createProducer(sendQueue); // Instansier en produsent av meldinger.
```
 
## Sende melding

```java
javax.jms.TextMessage msg = sendSession.createTextMessage(); // Her opprettes en melding av type tekstmelding, se dok. for andre varianter
msg.setBooleanProperty(progress.message.jclient.Constants.PRESERVE_UNDELIVERED, true); // Ved feil skal meldingen legges på DLQ.
msg.setBooleanProperty(progress.message.jclient.Constants.NOTIFY_UNDELIVERED, true); // Logg ved feil av levering av melding.
msg.setText("Her legges teksten i meldingen");
sender.send( msg, javax.jms.DeliveryMode.PERSISTENT, javax.jms.Message.DEFAULT_PRIORITY, 0); //// PERSISTENT meldingen garanteres levert av Sonic. Normal prioritet og uendelig TTL.
```
 
## Sette opp meldingsmottaker

```java
javax.jms.Session receiveSession = connection.createSession(false,javax.jms.Session.CLIENT_ACKNOWLEDGE); // Opprett session. Dette statement sier "false" til om hvorvidt det er en transacted session og setter acknowledge mode til CLIENT_ACKNOWLEDGE.
javax.jms.Queue receiveQueue = receiveSession.createQueue ("minKø"); // Opprett lesesesjon mot køen "minKø".
javax.jms.MessageConsumer qReceiver = receiveSession.createConsumer(receiveQueue); // Opprett meldingsmottaker qReceiver blir da produsenten.
qReceiver.setMessageListener(this); // Start lytteren til køen - behandling av meldinger implementeres i onMessage.
connection.start(); // Start lytteren.
```

###Ang. acknowledgement modes

Det finnes fire forskjellige acknowledgement modes:

- **AUTO_ACKNOWLEDGE** - Sesjonen styrer selv ack, ved en feil kan den siste leverte meldingen, i enkelte tilfeller, leveres på nytt.
- **CLIENT_ACKNOWLEDGE** - Klienten må selv styre ack. Ved en feil kan alle meldinger som ikke er ack'et leveres på nytt. En ack vil ved denne settingen "ack'e" samtlige meldinger som ikke var "ack'et"
- **DUPS_OK_ACKNOWLEDGE** - Sesjonen styrer selv ack, ved en feil kan flere av de siste leverte meldinger, i enkelte tilfeller, leveres på nytt. 
- **SINGLE_MESSAGE_ACKNOWLEDGE** - Klienten styrer selv ack. En ack vil i dette tilfellet kun "ack'e" sist mottatte melding. Ved en feil vil alle meldinger som ikke har fått "ack" bli levert på nytt.

ACK-mode ignoreres når en sender meldinger. Det gjelder kun mottak, dvs. at denne settingen ignoreres dersom sesjonen kun sender meldinger.
 
### Kodeeksempel på mottak av en tekstmelding.

Klassen som definerer metoden skal implementere javax.jms.MessageListener. Meldingsinnholdet parset som tekst og skrevet til output:

```java
public void onMessage(Message message) {
    TextMessage textMessage = (javax.jms.TextMessage) message;
    try {
        String text = textMessage.getText();
        System.out.println(text);
        message.acknowledge();
    } catch (JMSException e) {
        e.printStackTrace();
    }
}
```
