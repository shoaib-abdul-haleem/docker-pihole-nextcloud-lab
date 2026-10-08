# Fehlersuche / Troubleshooting

Sprache wählen / Choose a language

<details open name="language">
<summary><b>Deutsch</b></summary>

Hier steht, wie die Container miteinander reden, und was ich bei typischen Problemen prüfe.

## So reden die Container miteinander

1. Compose legt für das Projekt ein eigenes Netzwerk an. Es heißt wie der Ordner, hier `docker-pihole-nextcloud-lab_default`. Alle drei Container hängen darin.
2. In diesem Netzwerk gibt es einen eingebauten DNS-Dienst von Docker (im Container die Adresse `127.0.0.11`). Er macht aus dem Namen eines Dienstes die Adresse des Containers. Darum erreicht Nextcloud die Datenbank unter dem Namen `db`. Die IP-Adresse des Containers darf sich ändern.
3. `ports: "8080:80"` bedeutet: Port 8080 auf der VM wird an Port 80 im Container weitergeleitet (VM-Port:Container-Port). Das braucht nur, wer von außen zugreifen will, zum Beispiel der Browser. Zwischen Containern ist es nicht nötig. Darum hat die Datenbank keinen `ports`-Eintrag.
4. `depends_on` mit `healthcheck` sorgt dafür, dass Nextcloud erst startet, wenn die Datenbank bereit ist. Ohne das startet Nextcloud manchmal zu früh und findet die Datenbank nicht.
5. Volumes liegen außerhalb des Containers. Wird ein Container gelöscht und neu angelegt, bleiben die Daten.

Selbst ansehen:

```bash
docker compose ps
docker network ls
docker network inspect docker-pihole-nextcloud-lab_default
docker exec nextcloud getent hosts db
```

`docker network inspect` zeigt die drei Container mit ihren Adressen im Netzwerk. `getent hosts db` zeigt die Adresse, die Docker für den Namen `db` liefert.

So sieht es bei mir aus:

```text
$ docker exec nextcloud getent hosts db
172.18.0.3      db

$ docker exec nextcloud cat /etc/resolv.conf
nameserver 127.0.0.11
search .
options edns0 trust-ad ndots:0

$ docker network inspect docker-pihole-nextcloud-lab_default --format '{{range .Containers}}{{.Name}}  {{.IPv4Address}}{{"\n"}}{{end}}'
nextcloud  172.18.0.4/16
pihole  172.18.0.2/16
nextcloud_db  172.18.0.3/16
```

Der Name `db` ist der Dienstname aus der Compose-Datei. Er zeigt auf `172.18.0.3`, und das ist der Container `nextcloud_db`. Der Container hat also zwei Namen: den Dienstnamen `db` und den Containernamen `nextcloud_db`. Nextcloud benutzt den Dienstnamen.

## Typische Probleme

### 1. Pi-hole startet nicht: „address already in use“ auf Port 53

Ubuntu startet `systemd-resolved`, und der belegt `127.0.0.53:53`. Ein Container, der Port 53 auf allen Adressen haben will, stößt dagegen. Darum steht in der Compose-Datei `${HOST_IP}:53:53`, also die Adresse der VM. Wer den Port belegt, zeigt `sudo ss -tulpn | grep :53`.

Kommt nach einem Neustart „cannot assign requested address“, hat die VM eine andere Adresse bekommen als in `.env`. Mit `ip -br a` nachsehen, `HOST_IP` anpassen und `docker compose up -d` wiederholen.

### 2. Port 80 oder 443 ist schon belegt

Auf meiner VM läuft noch Nginx aus dem Webserver-Projekt auf 80 und 443. Darum liegt Pi-hole auf 8081 und Nextcloud auf 8080. Mit `sudo ss -tlnp` sehe ich, welche Ports schon belegt sind.

### 3. Nextcloud: „Zugriff über eine nicht vertrauenswürdige Domain“

Nextcloud lässt nur Adressen zu, die in `trusted_domains` stehen. Die Variable `NEXTCLOUD_TRUSTED_DOMAINS` gilt nur bei der ersten Einrichtung. Später füge ich eine Adresse so hinzu:

```bash
docker exec -u www-data nextcloud php occ config:system:get trusted_domains
docker exec -u www-data nextcloud php occ config:system:set trusted_domains 2 --value=192.168.56.50
```

Der erste Befehl zeigt die vorhandenen Einträge. Beim zweiten nehme ich die nächste freie Nummer.

### 4. Nextcloud kommt nicht an die Datenbank

Zuerst `docker compose ps` ansehen. Die Datenbank sollte `healthy` sein. Sonst helfen `docker compose logs db` und `docker compose logs nextcloud`.

Ein häufiger Grund: Ich habe ein Passwort in `.env` geändert, nachdem die Datenbank schon einmal gestartet war. MariaDB übernimmt die Passwörter nur beim allerersten Start, wenn das Volume noch leer ist. In einem reinen Test-Setup lösche ich alles und fange neu an:

```bash
docker compose down -v
docker compose up -d
```

Achtung: `-v` löscht auch die Daten.

### 5. „permission denied“ bei `docker`-Befehlen

Die Meldung lautet „permission denied while trying to connect to the docker API at unix:///var/run/docker.sock“. Bei mir kam sie in einem Terminal der VM-Oberfläche, während `docker` in der SSH-Sitzung funktionierte. Der Grund ist vermutlich, dass die Gruppe `docker` erst nach einer neuen Anmeldung gilt und das Terminal noch zur alten Sitzung gehörte. Wer nicht in der Gruppe ist, fügt sich mit `sudo usermod -aG docker $USER` hinzu. Danach ab- und wieder anmelden, oder im aktuellen Terminal `newgrp docker` eingeben. Bis dahin hilft `sudo docker ...`.

### 6. Pi-hole antwortet nicht oder blockt nichts

Antwortet Pi-hole nicht auf Anfragen von Windows, prüfe ich: Läuft der Container (`docker compose ps`)? Ist Port 53 für TCP und UDP veröffentlicht? Steht `FTLCONF_dns_listeningMode: all` in der Compose-Datei? Ohne diese Einstellung antwortet Pi-hole in einem Container nur auf Anfragen aus dem eigenen Netz.

Wird `doubleclick.net` nicht geblockt, ist meist die Blockliste noch nicht geladen. Dann lade ich sie neu und sehe in die Logs:

```bash
docker exec pihole pihole -g
docker compose logs pihole
```

Pi-hole wirkt nur auf Geräte, die ihn als DNS-Server benutzen. In `nslookup` gebe ich ihn darum ausdrücklich an.

### 7. Pi-hole-Passwort vergessen

```bash
docker exec -it pihole pihole setpassword
```

### 8. Docker und UFW

Docker legt eigene Netzwerkregeln an. Ein Port, den ich mit `ports:` veröffentliche, kann deshalb auch erreichbar sein, wenn die UFW-Firewall ihn nicht erlaubt. Das ist bekanntes Verhalten von Docker, und ich habe es auf meiner VM gesehen: `sudo ufw status` zeigt UFW als aktiv, erlaubt sind nur 22, 80 und 443. Trotzdem konnte ich von Windows Nextcloud auf Port 8080 und Pi-hole auf Port 8081 öffnen, und DNS auf Port 53 hat geantwortet.

Gegenmaßnahmen: nur Ports veröffentlichen, die wirklich nötig sind, oder einen Port an eine Adresse binden, zum Beispiel `127.0.0.1:8080:80`. Für dieses Lab ist das in Ordnung.

### 9. Das Herunterladen der Images hängt

Beim ersten `docker compose up -d` blieb bei mir der Download von Nextcloud lange bei `511MB / 515.6MB` stehen, obwohl die anderen beiden Images schon fertig waren. Die Zeit lief weiter, die Zahl änderte sich nicht.

Das habe ich gemacht:

```bash
# 1. Mit Ctrl+C abbrechen und prüfen, ob die VM Docker Hub erreicht
curl -sI https://registry-1.docker.io/v2/ | head -1

# 2. Das Image einzeln laden
docker pull nextcloud:apache

# 3. Danach normal starten
docker compose up -d
```

Als Antwort auf den `curl`-Befehl ist `401` in Ordnung. Es heißt, die Verbindung klappt und Docker Hub verlangt nur eine Anmeldung, die für öffentliche Images nicht nötig ist. Docker behält die Teile (Layer), die schon fertig sind, und lädt nur den Rest nach. Nach dem einzelnen Download startete `docker compose up -d` in wenigen Sekunden. Die Ursache war vermutlich eine hängende Verbindung der VM. Sicher weiß ich es nicht.

### 10. „no configuration file provided: not found“

`docker compose ps` meldet das, wenn ich nicht im Projektordner bin. Compose sucht die `docker-compose.yml` im aktuellen Ordner. Bei mir war ich im Home-Ordner. Mit `cd ~/docker-pihole-nextcloud-lab` in den Projektordner wechseln, dann klappt es. Die Befehle `docker exec` und `docker pull` brauchen den Ordner nicht.

## Nützliche Befehle

| Befehl | Wofür |
|---|---|
| `docker compose up -d` | alles starten (oder nach einer Änderung neu anlegen) |
| `docker compose ps` | Zustand der Container ansehen |
| `docker compose logs -f nextcloud` | Logs live mitlesen |
| `docker exec -it nextcloud bash` | eine Shell im Container öffnen |
| `docker compose pull` und `docker compose up -d` | Images aktualisieren |
| `docker compose down` | stoppen und Container löschen, Daten bleiben |
| `docker compose down -v` | zusätzlich alle Daten löschen |

</details>

<details name="language">
<summary><b>English</b></summary>

This page explains how the containers talk to each other and what I check for typical problems.

## How the containers talk to each other

1. Compose creates its own network for the project. It is named after the folder, here `docker-pihole-nextcloud-lab_default`. All three containers are in it.
2. This network has a built-in DNS service from Docker (the address `127.0.0.11` inside a container). It turns the name of a service into the address of the container. That is why Nextcloud reaches the database under the name `db`. The container's IP address is allowed to change.
3. `ports: "8080:80"` means: port 8080 on the VM is forwarded to port 80 inside the container (VM port:container port). Only someone who wants access from outside needs this, for example the browser. Containers do not need it to talk to each other. That is why the database has no `ports` entry.
4. `depends_on` with a `healthcheck` makes Nextcloud wait until the database is ready. Without it, Nextcloud sometimes starts too early and cannot find the database.
5. Volumes live outside the container. If a container is deleted and created again, the data stays.

See it for yourself:

```bash
docker compose ps
docker network ls
docker network inspect docker-pihole-nextcloud-lab_default
docker exec nextcloud getent hosts db
```

`docker network inspect` lists the three containers with their addresses in the network. `getent hosts db` shows the address Docker returns for the name `db`.

This is what it looks like on my VM:

```text
$ docker exec nextcloud getent hosts db
172.18.0.3      db

$ docker exec nextcloud cat /etc/resolv.conf
nameserver 127.0.0.11
search .
options edns0 trust-ad ndots:0

$ docker network inspect docker-pihole-nextcloud-lab_default --format '{{range .Containers}}{{.Name}}  {{.IPv4Address}}{{"\n"}}{{end}}'
nextcloud  172.18.0.4/16
pihole  172.18.0.2/16
nextcloud_db  172.18.0.3/16
```

The name `db` is the service name from the compose file. It points to `172.18.0.3`, which is the container `nextcloud_db`. So the container has two names: the service name `db` and the container name `nextcloud_db`. Nextcloud uses the service name.

## Typical problems

### 1. Pi-hole does not start: "address already in use" on port 53

Ubuntu runs `systemd-resolved`, which uses `127.0.0.53:53`. A container that wants port 53 on all addresses runs into it. That is why the compose file says `${HOST_IP}:53:53`, which is the address of the VM. `sudo ss -tulpn | grep :53` shows who uses the port.

If you get "cannot assign requested address" after a restart, the VM has a different address than the one in `.env`. Check with `ip -br a`, fix `HOST_IP` and run `docker compose up -d` again.

### 2. Port 80 or 443 is already taken

On my VM, Nginx from the web server project still runs on 80 and 443. That is why Pi-hole uses 8081 and Nextcloud uses 8080. `sudo ss -tlnp` shows which ports are already in use.

### 3. Nextcloud: "Access through untrusted domain"

Nextcloud only accepts addresses listed in `trusted_domains`. The variable `NEXTCLOUD_TRUSTED_DOMAINS` is only used during the first setup. Later I add an address like this:

```bash
docker exec -u www-data nextcloud php occ config:system:get trusted_domains
docker exec -u www-data nextcloud php occ config:system:set trusted_domains 2 --value=192.168.56.50
```

The first command lists the existing entries. For the second I use the next free number.

### 4. Nextcloud cannot reach the database

First I look at `docker compose ps`. The database should be `healthy`. Otherwise `docker compose logs db` and `docker compose logs nextcloud` help.

A common reason: I changed a password in `.env` after the database had already started once. MariaDB only takes the passwords on the very first start, when the volume is still empty. In a pure test setup I delete everything and start again:

```bash
docker compose down -v
docker compose up -d
```

Careful: `-v` deletes the data as well.

### 5. "permission denied" with `docker` commands

The message reads "permission denied while trying to connect to the docker API at unix:///var/run/docker.sock". For me it showed up in a terminal on the VM's desktop, while `docker` worked in the SSH session. The reason is probably that the `docker` group only counts after a new login, and that terminal still belonged to the old session. If you are not in the group yet, add yourself with `sudo usermod -aG docker $USER`. Then log out and back in, or type `newgrp docker` in the current terminal. Until then `sudo docker ...` works.

### 6. Pi-hole does not answer or does not block anything

If Pi-hole does not answer queries from Windows, I check: Is the container running (`docker compose ps`)? Is port 53 published for TCP and UDP? Is `FTLCONF_dns_listeningMode: all` in the compose file? Without this setting, Pi-hole in a container only answers queries from its own network.

If `doubleclick.net` is not blocked, the blocklist is usually not loaded yet. Then I reload it and look at the logs:

```bash
docker exec pihole pihole -g
docker compose logs pihole
```

Pi-hole only affects devices that use it as their DNS server. That is why I name it explicitly in `nslookup`.

### 7. Forgot the Pi-hole password

```bash
docker exec -it pihole pihole setpassword
```

### 8. Docker and UFW

Docker adds its own network rules. A port that I publish with `ports:` can therefore be reachable even if the UFW firewall does not allow it. This is known Docker behavior, and I saw it on my VM: `sudo ufw status` shows UFW as active, with only 22, 80 and 443 allowed. Still, I could open Nextcloud on port 8080 and Pi-hole on port 8081 from Windows, and DNS on port 53 answered.

Countermeasures: publish only the ports that are really needed, or bind a port to one address, for example `127.0.0.1:8080:80`. For this lab it is fine.

### 9. The image download hangs

On the first `docker compose up -d`, the Nextcloud download stayed at `511MB / 515.6MB` for a long time, although the other two images were already done. The timer kept running, but the number did not change.

This is what I did:

```bash
# 1. Stop it with Ctrl+C and check that the VM can reach Docker Hub
curl -sI https://registry-1.docker.io/v2/ | head -1

# 2. Download the image on its own
docker pull nextcloud:apache

# 3. Then start normally
docker compose up -d
```

For the `curl` command, `401` is fine. It means the connection works and Docker Hub only asks for a login, which public images do not need. Docker keeps the parts (layers) that are already finished and only downloads the rest. After the single download, `docker compose up -d` started in a few seconds. The cause was probably a stuck connection of the VM. I do not know for sure.

### 10. "no configuration file provided: not found"

`docker compose ps` shows this when I am not in the project folder. Compose looks for `docker-compose.yml` in the current folder. For me, I was in the home folder. Change into the project folder with `cd ~/docker-pihole-nextcloud-lab` and it works. The commands `docker exec` and `docker pull` do not need the folder.

## Useful commands

| Command | Purpose |
|---|---|
| `docker compose up -d` | start everything (or create again after a change) |
| `docker compose ps` | see the state of the containers |
| `docker compose logs -f nextcloud` | follow the logs live |
| `docker exec -it nextcloud bash` | open a shell inside the container |
| `docker compose pull` and `docker compose up -d` | update the images |
| `docker compose down` | stop and delete the containers, data stays |
| `docker compose down -v` | also delete all data |

</details>
