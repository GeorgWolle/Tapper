# Tapper
Tap your tempo

## Android-APK aus GitHub Actions

Der Workflow **Android APK** baut bei jedem Push eine signierte, installierbare
APK mit Bubblewrap. Unter **Actions → Android APK → Workflow-Lauf → Artifacts**
kannst du `app-release-signed.apk` direkt herunterladen. Die Downloads bleiben
30 Tage verfügbar. Über **Run workflow** ist auch ein manueller Build möglich.
Auf Android musst du die Installation aus dieser Downloadquelle erlauben.

### Website aktivieren

1. Unter **Settings → Pages → Build and deployment → Source** die Option
   **GitHub Actions** auswählen.
2. Auf den Standardbranch pushen oder dort den Workflow manuell starten.
   Der Workflow veröffentlicht ausschließlich die Website-Dateien auf Pages.
3. Die Website ist dann unter <https://georgwolle.github.io/Tapper/> erreichbar.

APK-Build und Pages-Deployment laufen unabhängig: Auch bei noch nicht
konfiguriertem Pages ist der APK-Download möglich, aber die App funktioniert erst,
wenn die Website erreichbar ist. Pushes auf andere Branches bauen ebenfalls APKs,
veröffentlichen aber keine Website. Diese APKs öffnen weiterhin die veröffentlichte
Website, nicht den jeweiligen Branchstand.

Bubblewrap erstellt eine **Trusted Web Activity**, keine Offline-Verpackung von
`index.html`. Die App benötigt einen kompatiblen Browser (z. B. Chrome) und eine
erreichbare HTTPS-Website. Auch das Tailwind-Styling wird derzeit aus dem Internet
geladen. Für eine andere Hosting-Adresse kannst du unter **Settings → Secrets and
variables → Actions → Variables** die Variable `ANDROID_WEB_URL` setzen
(vollständige HTTPS-Startadresse, bei einem Verzeichnis mit abschließendem `/`).
Das ändert das APK-Ziel, nicht das Pages-Deployment.

### Dauerhafte Signatur für App-Updates

Ohne Secrets erzeugt jeder Build einen neuen temporären Schlüssel. Die APK lässt
sich installieren, aber vor einem späteren Build musst du die vorherige App
deinstallieren. Für Updates ohne Deinstallation einen eigenen Schlüssel **einmal
lokal** erzeugen und sicher außerhalb von GitHub sichern:

```bash
export BUBBLEWRAP_KEYSTORE_PASSWORD="$(openssl rand -hex 32)"
export BUBBLEWRAP_KEY_PASSWORD="$BUBBLEWRAP_KEYSTORE_PASSWORD"
keytool -genkeypair -noprompt -storetype JKS \
  -keystore android.keystore -alias android \
  -keyalg RSA -keysize 2048 -validity 10000 -dname "CN=Tapper" \
  -storepass:env BUBBLEWRAP_KEYSTORE_PASSWORD \
  -keypass:env BUBBLEWRAP_KEY_PASSWORD
base64 < android.keystore | tr -d '\n'
```

Unter **Settings → Secrets and variables → Actions → Secrets** alle drei Secrets setzen:

- `ANDROID_KEYSTORE_BASE64`: die Base64-Ausgabe des Keystores.
- `ANDROID_KEYSTORE_PASSWORD`: der Wert von `BUBBLEWRAP_KEYSTORE_PASSWORD`.
- `ANDROID_KEY_PASSWORD`: der Wert von `BUBBLEWRAP_KEY_PASSWORD`.

Passwörter dürfen nur Buchstaben, Ziffern und `._=-` enthalten, damit Bubblewrap sie
sicher an seine Build-Werkzeuge übergibt. Die oben erzeugten Hex-Passwörter passen.
Den Keystore und die Passwörter **nicht committen oder als Artifact hochladen**.
Nur vertrauenswürdige Personen dürfen Branches und Workflows ändern, die auf diese
Signatur-Secrets zugreifen.

### Browserleiste entfernen

Ohne verifizierte Website-Zuordnung öffnet der Browser die App mit einer
Browserleiste. Der Build liefert zusätzlich das öffentliche Artifact
`tapper-assetlinks` mit `assetlinks.json`. Für eine vertrauenswürdige Vollbild-App:

1. Einen dauerhaften Signaturschlüssel konfigurieren.
2. Die erzeugte Datei unter
   `https://georgwolle.github.io/.well-known/assetlinks.json` veröffentlichen,
   ohne Weiterleitung und mit dem Content-Type `application/json`.

Bei GitHub Project Pages reicht `/Tapper/.well-known/assetlinks.json` **nicht**.
Die Datei muss im separaten Root-Pages-Repository `GeorgWolle.github.io` liegen
(bei Jekyll `.well-known` explizit einschließen oder `.nojekyll` verwenden).
Alternativ eine eigene Domain verwenden und die Datei dort im Domain-Root
bereitstellen. Dieser Workflow verändert kein anderes Repository.

### Build-Werkzeuge

Bubblewrap ist über `package-lock.json` fixiert und wird mit `npm ci` installiert.
Der Workflow nutzt Node.js 22, Java 17 und Android SDK 36 mit Build Tools 36.1.0.
`npm run build:android` erzeugt das Android-Projekt und baut APK und AAB; lokal
setzt das eine konfigurierte Bubblewrap-Umgebung, erreichbare `iconUrl` und einen
Keystore samt `BUBBLEWRAP_KEYSTORE_PASSWORD` und `BUBBLEWRAP_KEY_PASSWORD` voraus.
Die Versionsnummer der CI-APK steigt mit der Workflow-Laufnummer.

`npm audit` meldet derzeit ungepatchte transitive Bubblewrap-Abhängigkeiten
(ZIP-Entpacken, Dateityperkennung und UUID-Hilfsfunktionen). Der CI-Pfad umgeht
Bubblewraps ZIP-Installer durch vorkonfigurierte Java-/SDK-Pfade, verarbeitet
ausschließlich das lokale Repository-Icon und verwendet keine Google-Play-
Uploadfunktionen. Keine fremden Archive oder Bilder über Bubblewrap verarbeiten;
die Abhängigkeiten bei zukünftigen Bubblewrap-Updates erneut prüfen.
