# Schlicht – Umbrel Community App Store

Eigener App Store für umbrelOS. Enthält aktuell eine App:

| App | ID | Zweck |
|---|---|---|
| NetBird (Kernel) | `schlicht-netbird` | NetBird-Client mit echtem `wt0`-Interface und Routen auf dem Host |

## Warum nicht die offizielle NetBird-App?

Die offizielle App nutzt das Image `netbird:*-rootless`. Das läuft als UID 1000 im
**Netstack-Modus** (`NB_USE_NETSTACK_MODE=true`): Es gibt kein `wt0`-Interface und keine
Routen im Host. Andere Peers erreichen den Umbrel, aber der Umbrel selbst kann keine
NetBird-Netzwerke, Routen oder Exit-Nodes nutzen. Außerdem gibt es keine Möglichkeit,
Management-URL oder Setup-Key einzutragen.

`schlicht-netbird` nutzt das normale Image als root mit `/dev/net/tun`, Host-Netzwerk und
Kernel-WireGuard (falls vorhanden). Pfade für Config und Socket sind dieselben wie in der
offiziellen App, deshalb lässt sich deren `config.json` direkt übernehmen.

| Einstellung | Wert | Grund |
|---|---|---|
| `NB_CONFIG` | `/var/lib/netbird/config.json` | gleiche Datei wie die offizielle App |
| `NB_DAEMON_ADDR` | `unix:///var/lib/netbird/netbird.sock` | Socket wird mit dem Status-Container geteilt |
| `NB_DISABLE_DNS` | `true` | NetBird-DNS wirkt im Container nicht auf den Host, siehe unten |
| `NB_LOG_FILE` | `console` | Logs über `docker logs` bzw. „Troubleshoot“ |

Die Weboberfläche zeigt den Live-Status (`netbird status --json`, alle 10 Sekunden).

## App Store in Umbrel hinzufügen

1. Das Repo muss öffentlich sein, damit Umbrel es ohne Zugangsdaten klonen kann.
2. In Umbrel: **App Store** → Menü „…“ oben rechts → **Community App Stores** →
   `https://github.com/schlichtercode/umbrel-community-netbird` einfügen → **Add**.
3. Im neuen Store „Schlicht“ erscheint **NetBird (Kernel)**.

## Umzug von der offiziellen NetBird-App

Beide Apps dürfen **nicht gleichzeitig** laufen (beide wollen `wt0` und Port 51820).
Die Registrierung (Peer-Key, IP, Name) bleibt erhalten, wenn `config.json` übernommen wird.

```sh
# 1. Registrierung sichern (das Home-Verzeichnis übersteht Neustarts)
sudo cp -a ~/umbrel/app-data/netbird/data/app_data/config.json ~/netbird-config-backup.json
```

2. Offizielle **NetBird**-App in Umbrel deinstallieren. Dabei wird ihr Datenordner gelöscht,
   deshalb vorher Schritt 1.
3. **NetBird (Kernel)** aus dem Store „Schlicht“ installieren. Die App startet ohne
   Registrierung und wartet auf einen SSO-Login – den Link **nicht** öffnen.

```sh
# 4. App stoppen, frisch angelegte Dateien beiseitelegen, Backup einspielen
sudo docker stop schlicht-netbird_app_1
cd ~/umbrel/app-data/schlicht-netbird/data
sudo mkdir -p fresh-leftovers
sudo find app_data -mindepth 1 -maxdepth 1 ! -name .gitkeep -exec mv -t fresh-leftovers {} +
sudo cp -a ~/netbird-config-backup.json app_data/config.json
```

5. App in Umbrel neu starten (Rechtsklick → **Restart**) und prüfen:

```sh
sudo docker exec schlicht-netbird_app_1 netbird status
ip -br link | grep wt
```

Erwartet: gleiche NetBird-IP wie vorher, `Management: Connected`, `Interface type: Kernel`
und ein `wt0`-Interface. Danach können `~/netbird-config-backup.json` und
`fresh-leftovers` gelöscht werden.

Praktischer Alias für `~/.bashrc`:

```sh
alias netbird='sudo docker exec -it schlicht-netbird_app_1 netbird'
```

## Neues Gerät mit Setup-Key registrieren

Vor oder nach der Installation eine `setup.env` im Datenordner anlegen und die App neu starten:

```sh
sudo tee ~/umbrel/app-data/schlicht-netbird/data/app_data/setup.env >/dev/null <<'EOF'
NB_MANAGEMENT_URL=https://vpn.example.com
NB_SETUP_KEY=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
EOF
sudo chmod 600 ~/umbrel/app-data/schlicht-netbird/data/app_data/setup.env
```

Für NetBird Cloud `NB_MANAGEMENT_URL` weglassen. Nach erfolgreicher Registrierung die
Datei wieder löschen – die Zugangsdaten stehen dann in `config.json`.

Ohne `setup.env` wartet NetBird auf einen SSO-Login. Der Link steht in den App-Logs
(Rechtsklick auf die App → **Troubleshoot**).

## NetBird-Version aktualisieren

1. Digest der neuen Version holen:

   ```sh
   V=0.81.4
   TOKEN=$(curl -s 'https://ghcr.io/token?scope=repository:netbirdio/netbird:pull' | sed -E 's/.*"token":"([^"]+)".*/\1/')
   curl -sI -H "Authorization: Bearer $TOKEN" \
     -H 'Accept: application/vnd.oci.image.index.v1+json' \
     "https://ghcr.io/v2/netbirdio/netbird/manifests/$V" | grep -i docker-content-digest
   ```

2. In `schlicht-netbird/docker-compose.yml` Tag und Digest bei **beiden** Services
   (`app` und `status`) ersetzen.
3. In `schlicht-netbird/umbrel-app.yml` `version` und `releaseNotes` anpassen.
4. Committen und pushen. Umbrel zeigt das Update im Store an.

## Hinweise

- **DNS:** Mit Host-Netzwerk hat der Container trotzdem eine eigene `/etc/resolv.conf`
  und keinen Zugriff auf den D-Bus des Hosts. NetBird-DNS würde also nur im Container
  wirken, deshalb ist es abgeschaltet. Peers per IP ansprechen.
- **Exit-Node:** Wählbar mit `netbird networks select <ID>`. Das betrifft den gesamten
  Traffic des Umbrel; das lokale Netz (SSH, Weboberfläche) bleibt direkt erreichbar.
- **Zurück zur offiziellen App:** Diese App deinstallieren, offizielle App installieren
  und `config.json` analog übernehmen. Danach einmal
  `sudo chown -R 1000:1000 ~/umbrel/app-data/netbird/data/app_data`, weil die offizielle
  App nicht als root läuft.
