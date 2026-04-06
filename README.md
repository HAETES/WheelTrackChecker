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

Da ein Swisstrac andere physikalische Eigenschaften hat als ein Smoov One, arbeitet die App mit speicherbaren Profilen (via Browser LocalStorage). Vor dem Scan wählt man einfach das Gerät aus, mit dem man heute unterwegs ist:

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

## 7. Beispiel-Konfiguration: Profil "Handbike (Muskelkraft)"

In diesem Abschnitt wird die physikalische Logik beschrieben, nach der eine Route im Handbike-Profil analysiert wird. Jedes Segment wird individuell auf Basis der folgenden Parameter geprüft und ggf. markiert:

### 7.1 Die "Dynamische Steigungs-Matrix"
Das Herzstück des Profils koppelt die erlaubte Steigung an den jeweiligen Untergrund, um Traktionsverlust am Antriebsrad zu vermeiden:
* **Asphalt / Beton (paved):** Bis 5 % unproblematisch (Dauerlast), bis 7 % kurzzeitig okay, **über 8 % Sperrung (🔴)**.
* **Wassergebundene Decke (compacted):** Maximal **2,5 %** (Gefahr des Eingrabens).
* **Feinsplitt (fine_gravel):** Maximal **1,0 %** (Vermeidung von Traktionsverlust/Rutschen).
* **Naturwege / Erde / Gras:** Maximal **2,0 % - 4,0 %** (je nach Bodenfestigkeit).
* **Sand:** Wird komplett gemieden (**0 %** Steigung erlaubt / unpassierbar).

### 7.2 Untergründe & Präferenzen
* **Pflastersteine (cobblestone):** Werden akzeptiert, um mittlere Umwege zu vermeiden (Straf-Faktor ca. 2.0), aber wegen der Vibrationen als Warnung (**🟡**) markiert.
* **Landschafts-Bonus:** Bevorzugung von Wegen durch Wälder und entlang von Gewässern bei der Analyse von Alternativvorschlägen.

### 7.3 Hindernisse & Barrieren ("Showstopper")
Strikte physikalische Grenzen, die sofort einen **roten Wegpunkt (🛑)** auslösen:
* **Treppen & Stufen (steps):** Komplett gesperrt.
* **Drängelgitter (kissing_gate):** Gesperrt (Wenderadius/Länge des Geräts zu groß).
* **Bordsteinkanten (kerb):** Bis 6 cm okay; 6 bis 10 cm Warnung; über 10 cm Sperrung.
* **Breite:** Generelles Minimum **80 cm**; Engstellen (z. B. Poller) absolut minimum **65 cm**.

### 7.4 Straßen-Logik & City-Modus
* **Bundes- & Landstraßen:** Außerorts ohne Radweg strikt gesperrt (🔴). Innerorts nur als letzte Option (🟡).
* **Fußgänger-Bonus:** Bevorzugung von Fußgängerzonen und Gehwegen gegenüber dem Autoverkehr.
* **Abbiege-Verhalten:** Bevorzugung flüssiger Kurven gegenüber häufigem 90-Grad-Abbiegen (Wendekreis-Optimierung).

### 7.5 Mathematische Grundlage für den Check
Die Berechnung der Steigung $G$ erfolgt für jedes Segment zwischen zwei GPS-Punkten:

$$G = \frac{h_2 - h_1}{d} \cdot 100$$

*(G = Steigung in %, h = Höhe, d = Distanz)*

### 7.6 Daten-Integration (Metadaten)
Der Analyzer nutzt die **OpenRouteService (ORS) API**, um die GPX-Rohdaten mit präzisen Metadaten (Surface, Steepness, Smoothness) anzureichern.

### 7.7 Oberflächen-Qualität & Querneigung
* **Oberflächen-Qualität (Smoothness):** Auswertung des `smoothness`-Tags. Asphaltwege im schlechten Zustand (`bad`, `very_bad`) werden automatisch wie Schotter behandelt (Limit 2,5 %).
* **Querneigung (Camber):** Wo Daten zur Seitenneigung vorliegen, werden Werte über **2-3 %** als Warnung markiert, um einseitige Ermüdung und Kippgefahr zu vermeiden.

## 8. Service-Infrastruktur & Wheelmap-Integration

Über die Analyse von Gefahrenstellen hinaus fungiert der **WheelTrackChecker** als digitaler Service-Guide. Er nutzt die Barrierefreiheits-Daten von OpenStreetMap (identisch mit der Wheelmap-Datenbank), um hilfreiche Infrastruktur entlang der Route zu finden.

### 8.1 Der "Korridor-Scan" (POI-Suche)
Die App scannt nicht wahllos, sondern legt einen virtuellen Suchkorridor um die geplante Route:
* **Suchradius:** Standardmäßig **200 Meter** links und rechts der Wegstrecke.
* **Logik:** Es werden nur Orte markiert, die ohne große Umwege oder zusätzliche, ungeprüfte Steigungen vom Handbike aus erreichbar sind.

### 8.2 Das Ampel-System für Barrierefreiheit (`wheelchair`)
Gefundene Points of Interest (POIs) werden basierend auf ihrem Barrierefreiheits-Status in die GPX-Datei injiziert:
* **♿ Grün (Vollständig):** Tag `wheelchair=yes`. Der Ort ist stufenlos zugänglich, alle Räume sind barrierefrei erreichbar.
* **⚠️ Gelb (Eingeschränkt):** Tag `wheelchair=limited`. Der Eingang ist meist stufenlos, aber es gibt Einschränkungen (z. B. eine kleine Schwelle oder das WC ist nicht rollstuhlgerecht).
* **❌ Ignoriert:** Orte mit `wheelchair=no` werden standardmäßig nicht als Service-Punkt angezeigt, um Fehlplanungen zu vermeiden.

### 8.3 Priorisierte Service-Kategorien
Die App filtert gezielt nach Infrastruktur, die für die Tourenplanung mit dem Handbike essenziell ist:
* **Sanitäranlagen (`amenity=toilets`):** Explizite Suche nach `toilets:wheelchair=yes`.
* **Gastronomie (`amenity=cafe|restaurant`):** Pausenorte mit gesichertem Zugang.
* **Parkmöglichkeiten (`amenity=parking`):** Kennzeichnung von Behindertenparkplätzen (`parking:condition=wheelchair`) für Start- und Zielpunkte.
* **Medizinische Notfälle:** Apotheken und Krankenhäuser mit gesichertem Rollstuhlzugang.

### 8.4 Wegpunkt-Injektion & Info-Tags
Jeder gefundene Service-Punkt wird als XML-Wegpunkt (`<wpt>`) in die GPX-Datei geschrieben. Dabei werden, sofern in OSM vorhanden, Details in die Beschreibung übernommen (z. B. Türbreiten oder vorhandene Rampen), die in der Navi-App als Info-Fenster oder Sprachansage erscheinen.

## 9. Web-App & Plattform-Flexibilität

Um den WheelTrackChecker ohne technische Hürden nutzbar zu machen, ist die Umsetzung als Web-App vorgesehen. Nutzer wählen ihre GPX-Datei im Browser aus, die Verarbeitung erfolgt vollständig lokal auf dem Endgerät (client-seitig), und die "geimpfte" Datei wird sofort wieder zum lokalen Download bereitgestellt. Es findet kein Upload der sensiblen Routendaten auf einen Server statt. Dies ermöglicht die Nutzung auf Android, iOS und Desktop-Systemen gleichermaßen bei maximaler Datensicherheit.

## 10. Intelligente Umleitung & Notfall-Integration (Zukunftsidee)
Das Tool entwickelt sich vom passiven Warner zum aktiven Assistenten:
* **Alternative-Sourcing:** Erkennt das System ein kritisches Hindernis (🔴), sucht es über die ORS-API automatisch nach einer barrierefreien Umfahrung für diesen spezifischen Abschnitt und schlägt diese grafisch vor.
* **Google Maps Fallback:** Für den Fall, dass spezialisierte Navi-Apps nicht zur Verfügung stehen, wird in kritische Wegpunkte ein direkter Google-Maps-Navigationslink injiziert. So kann per einfachem Klick eine Standard-Navigation zum nächsten sicheren Punkt gestartet werden.

## 11. Ad-hoc-Suche: "Nächste barrierefreie Toilette/Café"

Zusätzlich zum Vorab-Scan der gesamten Route bietet der **WheelTrackChecker** eine Echtzeit-Suchfunktion für unvorhergesehene Pausen oder Notfälle.

### 11.1 Die "Quick-Find"-Logik

Per Knopfdruck (oder Klick auf den aktuellen GPS-Standort) startet die App eine Radialsuche im unmittelbaren Umkreis. Da die App lokal im Browser arbeitet, sendet das Endgerät hierfür eine direkte API-Abfrage mit den aktuellen Koordinaten an die Kartendienste, ohne Umweg über eigene Server.
* **Priorisierung:** Es werden ausschließlich Orte mit dem Status `wheelchair=yes` (🟢) oder `wheelchair=limited` (⚠️) angezeigt.
* **Distanz-Check:** Die Ergebnisse werden nach der tatsächlichen Wegstrecke (nicht Luftlinie!) sortiert, um sicherzustellen, dass keine unüberwindbaren Hindernisse zwischen dem Nutzer und dem Ziel liegen.

### 11.2 Notfall-Kategorien (Emergency POIs)
Die Suche ist auf die kritischsten Bedürfnisse optimiert:
1. **WC-Notfall:** Suche nach der nächsten `toilets:wheelchair=yes`.
2. **Energie/Wetter:** Suche nach dem nächsten barrierefreien Café/Restaurant zum Aufwärmen oder Akkuladen.
3. **Technik:** Suche nach dem nächsten Ort mit Werkzeug oder Unterstellmöglichkeit.

### 11.3 Interaktiver Rettungslink
Zu jedem gefundenen Notfall-Punkt wird sofort ein **Google Maps Navigations-Link** generiert. Da die App "Man-in-the-Middle" arbeitet, kann dieser Punkt sofort an die bevorzugte Navigations-App übergeben werden, um die aktuelle Route für den Abstecher kurzzeitig zu verlassen.

## 12. Rescue-Modus: "Ich stecke fest – Finde einen Ausweg"

Der Rescue-Modus ist eine Ad-hoc-Funktion für Situationen, in denen die aktuelle Route unpassierbar wird (z. B. durch Baustellen, Blockaden oder Fehlplanungen).

### 12.1 Die "Back-to-Track"-Logik
Per Knopfdruck analysiert die App die Umgebung ausgehend vom aktuellen GPS-Standort, um den Nutzer sicher zurück auf die geplante Route oder zum Ziel zu führen.
* **Sperr-Radius:** Der unpassierbare Punkt wird mit einem virtuellen Sperr-Radius (z. B. 50m) belegt, damit der Algorithmus nicht versucht, den Nutzer wieder dorthin zurückzuführen.
* **Barrierefreier Korridor:** Die App sucht im Umkreis nach dem nächstgelegenen Segment der Original-Route, das die Kriterien des Geräte-Profils (Asphalt, < 8 % Steigung, min. 80 cm Breite) erfüllt.

### 12.2 Geführte Umgehung (Re-Routing)
Anstatt nur eine Linie anzuzeigen, erstellt die App eine temporäre "Mini-Route":
1. **Ausgangspunkt:** Aktuelle GPS-Position.
2. **Ziel:** Der nächste sichere "Einstiegspunkt" auf der Original-Route.
3. **Profil-Treue:** Auch diese Notfall-Umleitung wird sofort gegen die Steigungs-Matrix und Hindernis-Datenbank des Handbike-Profils geprüft.

### 12.3 Visueller Ausweg (Google Maps Overlay)
Falls die Navi-App (Komoot/OsmAnd) Schwierigkeiten hat, die Umleitung schnell zu verarbeiten, generiert das Tool einen **"Rescue-Link"**. Dieser öffnet eine Google Maps Ansicht, in der die barrierefreie Umleitung als Overlay über der Standardkarte liegt, um eine sofortige Orientierung per Sichtnavigation zu ermöglichen.

## 13. KI-Integration: Vom statischen Filter zum lernenden Assistenten

Der WheelTrackChecker nutzt KI-gestützte Ansätze, um Datenungenauigkeiten auszugleichen und die Sicherheit zu erhöhen:

* **Predictive Path Analysis:** Einsatz von Machine Learning, um die individuelle Erschöpfung des Fahrers zu prognostizieren. Steigungen werden dynamisch bewertet: Was am Anfang der Tour "Grün" ist, kann gegen Ende der Tour als "Gelb" (Warnung) markiert werden.
* **Visual Hazard Detection:** (Zukunftsidee) Integration von Bilderkennung, um via Street-View-Daten Bordsteinhöhen und Oberflächenqualitäten zu verifizieren, die in den OSM-Basisdaten fehlen.
* **Natural Language Rescue:** Ein KI-Sprachinterface ermöglicht im "Rescue-Modus" eine intuitive Kommunikation. Der Nutzer kann Hindernisse per Sprache beschreiben, woraufhin die KI unter Einhaltung aller Profil-Limits eine alternative Route generiert.

## 14. Visuelle Verifikation (Street View Integration)

Um die Abhängigkeit von rein numerischen Daten (Steigung in %) zu verringern, integriert der **WheelTrackChecker** eine visuelle Kontrollinstanz:

* **Automated Link Injection:** Jeder Warn-Wegpunkt (🔴/🟡) erhält automatisch einen generierten Google Street View Link für die exakten Koordinaten des Hindernisses.
* **On-the-Fly Check:** Der Nutzer kann direkt aus der Navigations-App heraus (z. B. Komoot) das reale Straßenbild öffnen, um die Beschaffenheit der Steigung oder die Höhe eines Bordsteins visuell zu prüfen.
* **Fall-Back:** Sollten keine Street-View-Panoramen vorhanden sein (z. B. auf schmalen Weinwegen), führt der Link alternativ zur hochauflösenden Satellitenansicht von Google Maps, um die Wegbreite und Vegetation einschätzen zu können.

* ## 15. Real-World-Visualisierung (Google Photo & Static View)

Um Unsicherheiten bei der Wegbeschaffenheit zu minimieren, nutzt der **WheelTrackChecker** die visuelle Datenbank von Google:

* **Static Preview:** Bei kritischen Steigungen (🔴) wird automatisch ein statisches Vorschaubild der GPS-Koordinate angefordert. Dies ermöglicht eine sofortige Beurteilung der Oberflächenbeschaffenheit (z. B. loses Geröll vs. glatter Asphalt).
* **Crowdsourced Accessibility Photos:** Bei Service-Points (WCs, Cafés) ruft die App gezielt Nutzerfotos ab, die unter dem Label "Barrierefreiheit" hochgeladen wurden. So lässt sich die Steilheit einer Rampe oder die Breite einer Toilettentür bereits vor der Ankunft visuell verifizieren.
* **Deep Link Metadata:** Jeder Wegpunkt enthält Metadaten, die bei Klick direkt die Google-Maps-Galerie des Ortes öffnen, um eine 360-Grad-Ansicht der Umgebung zu ermöglichen.

## 16. Autarkie, API-Lizenzen und Sicherheit

**16.1 Autarkie & API-Key-Souveränität (Fallback)**
Um den Betrieb des WheelTrackCheckers unabhängig von zentralen Server-Kosten zu gestalten, nutzt die App das "Bring Your Own Key"-Prinzip. Nutzer können in ihren Profileinstellungen ihren eigenen, kostenlosen API-Key (z. B. von OpenRouteService) hinterlegen. Dieser Key wird sicher im lokalen `LocalStorage` des Browsers abgelegt. Dadurch bleibt das Tool ein rein persönliches Werkzeug, das dezentral und ohne laufende Infrastrukturkosten funktioniert.

**16.2 Ehrenamt-Boni & Lizenzen für gemeinnützige Projekte**
Gerade im Bereich Barrierefreiheit (Accessibility) stehen die Türen bei vielen Anbietern weit offen. Hier ist die Übersicht für die wichtigsten Dienste:

* **OpenRouteService (ORS):** Das ist der wichtigste Partner für die Routenanalyse. ORS wird vom Heidelberg Institute for Geoinformation Technology (HeiGIT) betrieben – das ist also ein universitäres, forschungsnahes Projekt direkt aus dem DACH-Raum und keine kalte US-Firma. 
  * *Standard (Kostenlos):* Jeder Nutzer bekommt standardmäßig schon ein ziemlich großzügiges Limit (oft rund 2.000 Anfragen pro Tag).
  * *Der Ehrenamt-Bonus:* HeiGIT hat ein Herz für Projekte, die der Gesellschaft nützlich sind. Wenn das Projekt fertig ist, kann man sie einfach anschreiben, das Konzept erklären und nach einem „Collaborative / Non-Profit API Key“ fragen. Sehr oft heben sie das Limit für solche Projekte kostenlos massiv an.

* **Wheelmap / Sozialhelden (OSM):** Hinter Wheelmap steckt der Berliner Verein Sozialhelden e.V. Die Daten basieren auf OpenStreetMap (sind also ohnehin frei). Wenn spezifische Wheelmap-APIs genutzt werden sollen, sind die Entwickler dort extrem hilfsbereit gegenüber anderen Open-Source-Entwicklern im Bereich Inklusion.

* **Google Maps Platform (Street View & Photos):** Google ist ein kommerzieller Riese, hat aber zwei Auffangnetze:
  * *Die 200-Dollar-Freigrenze:* Jeder Entwickler bekommt jeden Monat 200 US-Dollar Guthaben geschenkt. Damit lassen sich zigtausende Street-View-Bilder abrufen, bevor auch nur ein Cent berechnet wird. Für ein Projekt, das in Neckenmarkt startet und organisch wächst, reicht das oft monatelang.
  * *Google for Nonprofits:* Wenn das Projekt später vielleicht über einen Verein läuft (dafür braucht man einen offiziellen NGO/Vereins-Status), gibt es riesige Gratis-Kontingente.

**16.3 ⚠️ Die "Frontend-Falle" (Wichtig für das lokale Konzept)**
Wenn ein „Master-Key“ (also ein eigener, erhöhter API-Key) für alle Nutzer in der Web-App hinterlegt wird, gibt es ein technisches Problem: Da die App komplett lokal im Browser läuft, steht dieser Schlüssel als reiner Text im JavaScript-Code. Jeder, der im Browser **F12** drückt, könnte den Schlüssel kopieren und für seine eigenen Projekte missbrauchen.

* **Die Lösung (Domain Restriction):** Es muss bei ORS oder Google im Dashboard zwingend eingestellt werden, dass dieser API-Key nur funktioniert, wenn die Anfrage von der eigenen Webseite kommt.
* **Erlaubte Herkunft (Referrer):** `https://wheeltrackchecker.app/*`

Wenn jemand den Key klaut und von seinem PC aus nutzt, wird die Anfrage sofort blockiert.

# Kapitel 17: Die Mercator-Evolution
## Von unbrauchbaren Schätzungen zur präzisen digitalen Vermessung

### 1. Rückblick und Fehleranalyse: Warum das alte System (ORS) scheiterte
Unsere bisherigen Tests mit der OpenRouteService (ORS) Elevation-API zeigten deutliche Mängel, die für die Planung barrierefreier Routen kritisch sind:

* **Die 1-Meter-Falle:** ORS basiert auf groben SRTM-Radardaten. Die vertikale Auflösung liegt oft nur bei vollen Metern. Wenn die Route auf einer kurzen Distanz (z. B. 50 cm) von 235 m auf 236 m springt, berechnet der Algorithmus eine Steigung von **200 %**. Dies erzeugt ein massives "Rauschen" in den Daten.
* **Die Glättungs-Lüge:** Um diese Zacken zu verbergen, glättet ORS das Profil so stark, dass reale, kurze Rampen von 7 % oder mehr (die für Rollstuhlfahrer unüberwindbar sein können) einfach verschwinden und als "0 % (eben)" gemeldet werden.
* **Ergebnis:** Die Daten waren für eine verlässliche Sicherheitsbewertung unbrauchbar.

---

### 2. Das neue Konzept: Die Mercator-Logik
Anstatt fertige, ungenaue Höhenangaben abzufragen, nutzen wir nun die **Web-Mercator-Projektion (EPSG:3857)**, um direkt in die Rohdaten von Mapbox (Terrain-RGB) zu schauen.

* **Das Prinzip:** Die Welt wird als Mosaik aus quadratischen Bildern (**Kacheln**) betrachtet. Jedes Bild besteht aus 256x256 Pixeln.
* **Die Brücke:** Da GPS-Koordinaten auf einer Kugel liegen, die Kacheln aber flach sind, nutzen wir Mercator als mathematischen Übersetzer.
* **Pixel-Genauigkeit:** Wir berechnen für jede GPS-Koordinate exakt, in welcher Kachel und an welcher Pixel-Position sie liegt.

#### Mathematische Grundlage der Umrechnung:
Um den Kachel-Index $x$ und $y$ bei einem Zoom-Level $z$ zu finden:

$$x = \lfloor \frac{lon + 180}{360} \cdot 2^z \rfloor$$

$$y = \lfloor (1 - \frac{\ln(\tan(lat \cdot \frac{\pi}{180}) + \frac{1}{\cos(lat \cdot \frac{\pi}{180})})}{\pi}) \cdot \frac{1}{2} \cdot 2^z \rfloor$$

---

### 3. Zoom-Level 14 bis 16: Das digitale Mikroskop
Die Genauigkeit wird über den Zoom gesteuert. Wir haben uns für ein einstellbares Modell entschieden, um auf unterschiedliche Anforderungen reagieren zu können:

| Zoom | Pixel-Breite (in AT) | Anwendung |
| :--- | :--- | :--- |
| **14** | ~ 6,5 Meter | Grobe Analyse langer Überlandstrecken. |
| **15** | **~ 3,2 Meter** | **Standard-Modus.** Ideal, um 7%-Rampen zu identifizieren. |
| **16** | ~ 1,6 Meter | Detail-Modus für komplexe urbane Kreuzungen. |

---

### 4. Effizienz durch intelligentes Caching
Da eine Route oft hunderte Punkte hat, die geografisch nah beieinander liegen, wäre das einzelne Abfragen jedes Punktes ineffizient und teuer.

* **Bitmap-Caching:** Das Skript identifiziert vorab alle benötigten Kacheln für die gesamte Route.
* **Einmaliger Download:** Jede Kachel (PNG) wird nur ein einziges Mal geladen.
* **ImageData-Speicher:** Die Pixelwerte werden im Arbeitsspeicher des Browsers abgelegt. Weitere Berechnungen erfolgen in Millisekunden lokal auf dem Gerät, ohne erneuten Netzwerkzugriff.

---

### 5. Persistenz: LocalStorage-Konzept
Um die Benutzererfahrung stabil zu halten, werden Einstellungen dauerhaft im Browser gespeichert:

* **API-Keys:** Sowohl der ORS-Key (für die Wegführung) als auch der Mapbox-Token (für die Höhe) bleiben gespeichert.
* **Präferenzen:** Der zuletzt genutzte Zoom-Level wird beim nächsten Start automatisch wiederhergestellt.
* **Sicherheits-Zähler:** Ein lokaler Zähler überwacht die Anzahl der Kachel-Abfage, um das Mapbox-Freikontingent (750.000 Tiles/Monat) im Blick zu behalten.

---

### 6. Fazit: Warum dieser Weg "brauchbar" ist
Durch den Wechsel auf die Mercator-Bitmap-Analyse erreichen wir eine vertikale Präzision von **0,1 Metern (10 Zentimeter)**. 

Im Gegensatz zu den 1-Meter-Sprüngen von ORS erlaubt uns dies eine sanfte Berechnung der realen Neigung. In Kombination mit einem **10-Meter-Glättungsfenster** eliminieren wir das mathematische Rauschen und erhalten ein ehrliches Höhenprofil, das die "versteckten" Steigungen zeigt, auf die es beim Rollstuhl-Routing ankommt.
