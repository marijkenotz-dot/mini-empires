Mini Empires – PWA
===================
Dies ist ein kleines RTS-Browserspiel als Progressive Web App.

Schnellstart (lokal, zum Testen)
--------------------------------
1) Einfache Möglichkeit (Desktop): mit Python 3 in diesem Ordner ausführen:
   python -m http.server 8080
   Dann im Browser http://localhost:8080 öffnen.
   (Service Worker funktionieren nur über https oder localhost.)

Deployment (für Handy-Installation)
----------------------------------
1) Den Ordner auf einen HTTPS-Hoster hochladen (z. B. GitHub Pages, Netlify, Vercel).
2) Auf Android Chrome die URL öffnen → Menü → „Zum Startbildschirm hinzufügen“.
   Danach erscheint es wie eine App und läuft offline (durch Service Worker).

APK (Android-Paket) aus der PWA bauen
-------------------------------------
Variante A – PWABuilder:
- Öffne pwabuilder.com, gib die gehostete URL deiner PWA ein, folge den Schritten,
  und lade das Android-Paket (TWA/WebAPK) herunter.

Variante B – Bubblewrap/Trusted Web Activity (für Entwickler):
- Bubblewrap installieren, Projekt initialisieren mit der PWA-URL, Android-Paket bauen,
  signieren und auf ein Gerät installieren.

Hinweis zu Urheberrecht/Marken
------------------------------
Dieses Projekt ist eigenständig und verwendet keine Marken, Namen, Grafiken oder Daten
aus „Age of Empires“. Für eine große, inhaltsgleiche 1:1-Kopie bräuchte man die Rechte
des IP-Inhabers. Stattdessen kann das Spiel eigenständige Einheiten, Gebäude und eine
eigene Tech-Tree-Logik bekommen.
