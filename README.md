# Telegram Bot API Workshop

## Vorraussetzungen
- Node-RED
- Telegram App auf Handy/ Laptop installiert
- Normaler Telegram Account
- Telegram Nodes Palette: `node-red-contrib-telegrambot`
## Aufgabe 1: Telegram Bot erstellen
- Öffne die Telegram App (auf Handy / Laptop)
- Suche nach: `BotFather` und chat starten
- Erstelle einen neuen Bot mit `/newbot`
- Wähle einen `Anzeigenamen` und **einmaligen** `bot_username` (`bot_username`muss mit "bot" enden)
- Speichere das ausgegebene `token` für die folgenden Aufgaben

## Aufgabe 2: Bot API Testen
- Öffne die Telegram App (auf Handy / Laptop)
- Suche deinen Bot via deinem `<bot_username>` und starte einen Chat
- Danach öffne den Browser
- gebe in die URL-Leiste ein:
	- `https://api.telegram.org/bot<token>/getUpdates`
	- ersetze `<token>` mit deinem **token**
- Ausgabe sollte u.a. `"ok": true` und eine `chatId` enthalten
- Speichere die `chatId` für später
## Aufgabe 2: Einfache Telegram Nachricht senden

![[Pasted image 20251026104649.png]]

- **Inject Node**
	- der Payload kann *optional* gelöscht werden, wird aber in der Function node sowieso überschrieben
- **Function Node**
```javascript
const telegramMessage = {
	chatId: 8286143389, // gib hier deine ChatId ein
	type: "message",
	content: "Hello World!"
}

msg.payload = telegramMessage;
return msg;
```
- *optional* Debug Node
- Telegram Sender Node (Installation: siehe Vorraussetzungen)
	- Bot-Name: `<your_unique_bot_name>`
	- Bot-Token: `<your_bot_token>`

Hinweise
- stelle sicher das in der Function Node die `chatId` richtig gesetzt ist
#### Ausführen
- Deploy nicht vergessen ;)
- Inject Node auslösen
	- erwartete Nachricht in Telegram: `Hello World!`

## Aufgabe 3: Wetter Updates

![[weather_updates_example.png]]

- Der Bot besteht aus 3 Teilen:
	- **Subscribe Handler**
	- **Wetter Updates via HTTP erhalten**
	- **Nachrichten-Objekt bauen und senden**
- Wir werden den Bot in 3 Schritten aufbauen
### 3.1 Subscribe Handler

![[Pasted image 20251026113414.png]]

Ziel:
- User sendet: `/startUpdates` -> der user zu `flow.subscribers (array)`hinzugefügt
- User sendet: `/stopUpdates` -> der user wird aus `flow.subscribers (array)`entfernt

Hinweise:
- Statt Wetter Updates zu beziehen, verwenden wir einen Init Node um vorerst nur die `Subscriber` im **Debug** auszugeben und an den Chat zurückzusenden

Nodes:
- startUpdates - **Telegram Command Node**:
	- Bot-Name: `<your_unique_bot_name>`
	- Bot-Token: `<your_bot_token>`
	- Command: `/startUpdates`
- stopUpdates - **Telegram Command Node**:
	- Bot-Name: `<your_unique_bot_name>`
	- Bot-Token: `<your_bot_token>`
	- Command: `/stopUpdates`
- addSubscriber - **Function Node**
```javascript
let subscribers = flow.get("subscribers") || [];
const currentChatId = msg.payload.chatId;

// Subscriber hinzufügen
if (!subscribers.includes(currentChatId)) {
	subscribers.push(currentChatId);
}
flow.set("subscribers", subscribers);
// !! kein return msg; !!
```
- removeSubscriber - **Function Node**
```javascript
let subscribers = flow.get("subscribers") || [];
const currentChatId = msg.payload.chatId;

// Chat-ID aus Liste filtern (löscht sie, falls vorhanden)
subscribers = subscribers.filter(id => id !== currentChatId);

flow.set("subscribers", subscribers);
// !! kein return msg; !!
```
- showSubscribers - **Function Node**
```javascript
const subscribers = flow.get("subscribers") || [];
const messageContent = "subscribed ChatIds: " + subscribers;

const telegramMessage = {
	chatId: subscribers,
	type: "message",
	content: messageContent
};

msg.payload = telegramMessage
return msg;
```
- Telegram Sender - **Telegram Sender Node**
	- Bot-Name: `<your_unique_bot_name>`
	- Bot-Token: `<your_bot_token>`
- Init - **Inject Node**
	- fürs erste als ``manuell``belassen

#### Ausführen
- Deploy nicht vergessen ;)
- In Telegram App an deinen Bot `/startUpdates` senden
	- erwartete Nachricht in Telegram: `subscribed ChatIds: 12345678`

### 3.2 Wetter Updates via HTTP

![[Pasted image 20251026124859.png]]

Ziel:
- GET Request an ein API endpoint über das Internet an [Open-Meteo.com](https://open-meteo.com/)
- Response Json parsen und in Javascript Object umwandeln
- Payload ausgeben und Ausgabe prüfen

Nodes:
- Init - **Inject Node**
	- fürs erste als `manuell` belassen
- httpGetWeather - **HTTP Request Node**
	- URL: `https://api.open-meteo.com/v1/forecast?latitude=52.52&longitude=13.41&current_weather=true`
	- Method: GET
- json - **JSON Node**
- debug - **Debug Node**
#### Ausführen
- Deploy nicht vergessen ;)
- Inject Node auslösen
- erwarteter Debug Output: Json Objekt mit Wetter-Daten

### 3.3 SubFlows verbinden und Wetter Nachricht verfassen

Ergebnis aus Aufgabe 3.2:
![[Pasted image 20251026124859.png]]

Ergebnis aus Aufgabe 3.1:
![[Pasted image 20251026113414.png]]

Ziel Bot System:
![[weather_updates_example.png]]
- Anstatt die `subscribed ChatIds` zu senden, sollen relevante Wetter Daten gesendet werden

Benötigte Änderungen
- verbinde die Subflows aus `Aufgabe 3.2` und `Aufgabe 3.1` so, wie im Bild `Ziel Bot System` dargestellt
- Ändere den `Inject Node` vor dem `http Node` so, dass er im Intervall alle 5 Sekunden auslöst
- ersetze die Funktion im Funktion node `show subscribers` mit dem Folgenden:
```javascript
const subscribers = flow.get("subscribers") || [];
const weather = msg.payload.current_weather;

if (!weather) {
node.warn("Keine Wetterdaten gefunden");
return null;
}

// Schönen Text für Telegram zusammenbauen
const messageText =
`🌤️ Wetter-Update für Berlin\n\n` +
`🌡️ Temperatur: ${weather.temperature} °C\n` +
`💨 Wind: ${weather.windspeed} km/h aus ${weather.winddirection}°\n` +
`🕒 Zeit: ${weather.time.replace('T', ' ')}`;

const telegramMessage = {
chatId: subscribers,
type: "message",
content: messageText
};

msg.payload = telegramMessage
return msg;
```
#### Ausführen
- Deploy nicht vergessen ;)
- In Telegram App an deinen Bot `/startUpdates` senden
	- Erwartung: alle 5 Sekunden Nachricht in Telegram  mit Wetter Updates
- In Telegram App an deinen Bot `/stopUpdates` senden
	- Erwartung: keine weiteren Updates
