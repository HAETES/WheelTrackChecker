# WheelTrackChecker (GPX-Wegpunkt-Injektion)

## 1. Die Grundidee (Der "Man-in-the-Middle")
Die App ist kein eigenes Navigationssystem und zeichnet keine eigenen Landkarten. Sie ist ein reines **Prüf- und Filter-Werkzeug**. Sie nimmt eine geplante Route als Standard-GPX-Datei entgegen, analysiert sie metergenau auf die physikalischen Grenzen von Rollstühlen und Zuggeräten, stempelt Warn- und Info-Wegpunkte direkt in die Datei und übergibt sie zurück an eine Navi-App (z. B. Komoot).

## 2. Der praktische Workflow (Der Alltag)
1. **Planen:** Eine Tour wird ganz normal in Komoot oder einem anderen Planer erstellt.
2. **Teilen:** Export der GPX-Datei über die "Teilen"-Funktion an die WheelTrackChecker-App.
3. **Der Scan:** Die App schickt die Koordinaten an die OpenRouteService (ORS) API und lädt genaue Untergrund-, Höhen- und Hindernisdaten herunter.
4. **Die Injektion:** Die App schreibt XML-Wegpunkte (`<wpt>`) an die kritischen Stellen in der lokalen GPX-Datei.
5. **Re-Import:** Die markierte Datei wird in der Navi-App geöffnet. Die Route ist nun mit Warnschildern oder Service-Infos gespickt und kann bei Bedarf umgeplant werden.

## 3. Die intelligente Analyse-Logik
Das System verknüpft Steigung dynamisch mit dem Untergrund. Die exakten Grenzwerte hängen vom gewählten Geräte-Profil ab (siehe Punkt 5). Es gibt generell drei Markierungs-Stufen:

* **🟢 Grün (Unsichtbar / Freie Fahrt):** Alles innerhalb der Komfortzone des Geräts.
* **🟡 Gelb (Warnung - ⚠️):** Grenzbereich erreicht. (Beispiel: "Kraftlimit naht" oder "Traktionsverlust möglich").
* **🔴 Rot (STOPP - 🛑):** Physikalisches Limit überschritten oder absolutes Hindernis (Treppen: `highway=steps`, Drängelgitter: `barrier=kissing_gate`, Durchfahrtsbreiten-Limit unterschritten).

## 4. Die Service-Erweiterung (Wheelmap)
Die App sucht über OSM/Wheelmap-Daten im Umkreis der Route nach barrierefreier Infrastruktur. 
Findet sie ein rollstuhlgerechtes WC (`toilets:wheelchair=yes`) oder ein zugängliches Café, setzt sie einen **grünen Info-Wegpunkt (♿)** in die GPX-Datei als Empfehlung für Pausen.

## 5. Volle Kontrolle (Dynamische Geräte-Profile)
Da ein Swisstrac andere physikalische Eigenschaften hat als ein Smoov One, arbeitet die App mit **speicherbaren Profilen** (via Android `DataStore`). Vor dem Scan wählt man einfach das Gerät aus, mit dem man heute unterwegs ist:

* **Profil A: Handbike (Muskelkraft)**
  * *Fokus:* Traktionsverlust auf Schotter, Kraftlimit bei Steigungen, große Fahrzeugbreite.
  * *Limits:* Asphalt max. 8 %, Schotter max. 2,5 %, Breite min. 90 cm.
* **Profil B: Smoov One / e-Fix (Heckantrieb & kleine Lenkräder)**
  * *Fokus:* Steigungen sind durch den Motor leichter, aber kleine Vorderräder bleiben an Kanten hängen; Kippgefahr nach hinten.
  * *Limits:* Höhere Steigungstoleranz (z. B. 12 %), dafür harte rote Warnungen bei Kopfsteinpflaster (`cobblestone`) oder Bordsteinen über 3 cm (`kerb=raised`).
* **Profil C: Swisstrac (Front-Zuggerät)**
  * *Fokus:* Sehr hohe Zugkraft und Geländegängigkeit, meistert Schotter hervorragend, aber Gesamtlänge und Breite des Gespanns sind kritisch.
  * *Limits:* Schotter bis 8 % erlaubt, aber extrem strenge Prüfung bei engen Kurvenradien oder Drängelgittern.
* **Profil D: E-Scooter / Rollstuhl**
  * *Fokus:* Bodenfreiheit und Reichweite.

Zusätzlich zu den Profilen gibt es eine **dynamische Blacklist**: Ein Textfeld für neue OSM-Tags (z. B. Furten: `ford=yes`). Beim nächsten Scan wird dieses Hindernis sofort für alle Profile rot markiert.

## 6. Die Daten-Philosophie (Read-Only)
Die App arbeitet rein **lesend**. Sie holt sich das Wissen aus OpenStreetMap, verändert aber ausschließlich die lokale GPX-Datei auf dem Handy. Fehlende Hindernisse in der echten Weltkarte werden nicht über diese App, sondern über spezialisierte Tools (z. B. StreetComplete) in die OSM-Datenbank eingetragen.
