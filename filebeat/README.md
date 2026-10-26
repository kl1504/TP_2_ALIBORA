# Filebeat

Le binaire `filebeat-8.11.0-amd64.deb` n'est pas versionné (trop lourd pour git).
Télécharge-le au build :

```bash
curl -L -o filebeat.deb https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.11.0-amd64.deb
```

Ou utilise l'image Docker officielle plutôt que le paquet .deb.
