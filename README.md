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

## 7. Spezifische Routing-Logik & Anforderungen

In diesem Abschnitt wird die physikalische Logik beschrieben, nach der die Route analysiert und bewertet wird. Jedes Segment wird individuell auf Basis der folgenden Parameter geprüft:

### 7.1 Die "Dynamische Steigungs-Matrix"
Das Herzstück des Profils koppelt die erlaubte Steigung an den jeweiligen Untergrund, um Traktionsverlust zu vermeiden:
* **Asphalt / Beton (paved):** Bis 5 % unproblematisch (Dauerlast), bis 7 % kurzzeitig okay, **über 8 % Sperrung**.
* **Wassergebundene Decke (compacted):** Maximal **2,5 %** (Gefahr des Eingrabens).
* **Feinsplitt (fine_gravel):** Maximal **1,0 %** (Vermeidung von Traktionsverlust/Rutschen).
* **Naturwege / Erde / Gras:** Maximal **2,0 % - 4,0 %** (je nach Bodenfestigkeit).
* **Sand:** Wird komplett gemieden (**0 %** Steigung erlaubt / unpassierbar).

### 7.2 Untergründe & Präferenzen
* **Pflastersteine / Kopfsteinpflaster (cobblestone):** Werden akzeptiert, um mittlere Umwege zu vermeiden (Straf-Faktor ca. 2.0 bis 2.2), aber wegen der Vibrationen nicht bevorzugt.
* **Landschafts-Bonus:** Starke Bevorzugung von Wegen durch Wälder und entlang von Flüssen/Gewässern.

### 7.3 Hindernisse & Barrieren ("Showstopper")
Strikte physikalische Grenzen für das Handbike:
* **Treppen & Stufen (steps):** Komplett gesperrt (Faktor 1.000.000).
* **Drängelgitter (kissing_gate):** Komplett gesperrt (Wenderadius/Länge des Geräts zu groß).
* **Drehkreuze & Stile:** Komplett gesperrt.
* **Bordsteinkanten (kerb):** Bis 6 cm kein Problem; 6 bis 10 cm mit Strafzeit; über 10 cm gesperrt.
* **Breite:** Generelles Minimum **80 cm**; Engstellen (z. B. Poller) absolut minimum **65 cm**.

### 7.4 Straßen-Logik & City-Modus
Spezielle Regeln für den Mischverkehr:
* **Bundes- & Landstraßen:** Außerorts strikt gesperrt (außer bei begleitendem Radweg). Innerorts nur als letzte Option.
* **Fußgänger-Bonus:** Bevorzugung von Fußgängerzonen und Gehwegen (rechtliche Gleichstellung als Fußgänger).
* **Radweg-Vorrang:** Existiert ein Radweg neben einem Gehweg, gewinnt der Radweg für flüssigeres Fahren.
* **Abbiege-Verhalten:** Bevorzugung flüssiger Kurven gegenüber häufigem 90-Grad-Abbiegen.

### 7.5 Mathematische Grundlage für den Check
Die Analyse der GPX-Daten erfolgt über die Gradienten-Formel für jedes Wegsegment:

$$G = \frac{h_2 - h_1}{d} \cdot 100$$

*(G = Steigung in %, h = Höhe, d = Distanz)*

### 7.6 Daten-Integration (Metadaten)
Das Tool nutzt die **OpenRouteService (ORS) API**, um die GPX-Rohdaten mit präzisen Metadaten (Surface, Steepness, OSM-IDs) anzureichern und abzugleichen.

### 7.7 Oberflächen-Qualität & Querneigung
* **Oberflächen-Qualität (Smoothness):** Auswertung des `smoothness`-Tags. Asphaltwege im schlechten Zustand (`bad`, `very_bad`) werden wie Schotter behandelt, um Erschütterungen zu minimieren.
* **Querneigung (Camber):** Wo Daten zur Seitenneigung vorliegen, werden Werte über **2-3 %** bestraft. Dies verhindert einseitige Ermüdung durch Gegensteuern und minimiert die Kippgefahr.
