# TP_2_ALIBORA

Deux chaînes indépendantes : **métriques** (qu'est-ce qui se passe ?) et **logs** (pourquoi ça s'est passé ?).

## 1. Architecture globale

```mermaid
flowchart LR
  subgraph SRC["Sources"]
    NE["node-exporter :9100"]
    BB["blackbox-exporter :9115"]
    LOGS["/var/log/syslog, auth.log"]
  end

  subgraph MET["Chaîne métriques - docker-compose.yml"]
    P["Prometheus :9090"]
    AM["Alertmanager :9095"]
    G["Grafana :3000"]
  end

  subgraph LOG["Chaîne logs - elastic/docker-compose.yml"]
    FB["Filebeat (hôte)"]
    LS["Logstash :5044"]
    ES["Elasticsearch :9200"]
    K["Kibana :5601"]
  end

  NE -->|"scrape 15s"| P
  BB -->|"scrape /probe"| P
  P -->|"alertes Firing"| AM
  AM -->|"SMTP 587"| MAIL(["Gmail"])
  G -->|"PromQL"| P
  LOGS --> FB
  FB -->|"protocole Beats"| LS
  LS -->|"HTTP"| ES
  K -->|"requêtes"| ES
```

| Service | Port | Rôle |
| --- | --- | --- |
| Prometheus | 9090 | collecte + stockage TSDB + évaluation des règles |
| Alertmanager | 9095 (hôte) → 9093 | regroupe, route, inhibe, notifie |
| Grafana | 3000 | tableaux de bord (datasource Prometheus provisionnée) |
| Blackbox | 9115 | sondes HTTP / TCP depuis l'extérieur |
| Node exporter | 9100 | métriques de l'hôte (CPU, RAM, disque) |
| Elasticsearch / Kibana | 9200 / 5601 | stockage et exploration des logs |
| Logstash | 5044 / 9600 | entrée Beats / API de monitoring |

## 2. Cycle de vie d'une alerte

```mermaid
stateDiagram-v2
  [*] --> Inactive
  Inactive --> Pending: "expr devient vraie"
  Pending --> Inactive: "expr redevient fausse avant for"
  Pending --> Firing: "expr vraie pendant for"
  Firing --> Inactive: "expr fausse (resolved)"
  Firing --> [*]: "envoyée à Alertmanager"
```

**Point clé :** `for: 1m` évite les faux positifs. Une alerte ne part qu'après 1 minute continue de condition vraie.

## 3. Scénario InstanceDown, de bout en bout

```mermaid
sequenceDiagram
  autonumber
  participant N as node-exporter
  participant P as Prometheus
  participant A as Alertmanager
  participant M as Mail

  P->>N: scrape (toutes les 15s)
  Note over N: docker stop node-exporter
  P-xN: scrape échoué, up = 0
  Note over P: règle InstanceDown : Pending
  P->>P: up == 0 depuis 1 min : Firing
  P->>A: envoi de l'alerte
  Note over A: group_wait 10s
  A->>M: mail "OPS CRITICAL: InstanceDown"
  Note over N: docker start node-exporter
  P->>N: scrape OK, up = 1
  P->>A: alerte résolue
  A->>M: mail "resolved"
```

Délai attendu entre l'arrêt et le mail : environ 1 min 30 à 2 min (scrape + `for` + évaluation + `group_wait`).

## 4. Routage et inhibition dans Alertmanager

```mermaid
flowchart TD
  IN(["Alerte reçue"]) --> Q1{"severity=critical ET team=security ?"}
  Q1 -->|oui| R1["mail-security-critical (repeat 1h)"]
  Q1 -->|non| Q2{"severity=warning ET team=security ?"}
  Q2 -->|oui| R2["mail-security-warning (repeat 6h)"]
  Q2 -->|non| Q3{"severity=critical ET team=ops ?"}
  Q3 -->|oui| R3["mail-ops-critical (repeat 2h)"]
  Q3 -->|non| R4["default-receiver (repeat 4h)"]
  INH["InstanceDown actif"] -. "inhibe les warning de la même instance" .-> IN
```

La première route qui correspond gagne (pas de `continue`).

| Alerte | Labels | Receveur |
| --- | --- | --- |
| InstanceDown | critical / ops | mail-ops-critical |
| ProbeFailed | critical / ops | mail-ops-critical |
| SslCertExpiresSoon | warning / security | mail-security-warning |
| HighCpuLoad | warning / ops | default-receiver (aucune route warning/ops) |
| DiskSpaceHigh | warning / ops | default-receiver (idem) |

> Le receveur `mail-security-critical` n'est déclenché par aucune règle actuelle : à signaler honnêtement, ou à illustrer en ajoutant une règle `critical / security`.

## 4 bis. Comment blackbox est interrogé (relabeling)

```mermaid
flowchart LR
  T["Cible déclarée : https://google.com"] --> R1["relabel 1 : __address__ devient __param_target"]
  R1 --> R2["relabel 2 : __param_target devient label instance"]
  R2 --> R3["relabel 3 : __address__ remplacé par blackbox-exporter:9115"]
  R3 --> REQ["GET blackbox:9115/probe?module=http_2xx&target=https://google.com"]
```

Prometheus n'interroge jamais Google : il interroge **blackbox**, qui sonde la cible. C'est pourquoi `up` vaut 1 même si le site est tombé, et pourquoi `ProbeFailed` surveille `probe_success == 0`.

## 5. Pipeline des logs

```mermaid
flowchart LR
  F1["/var/log/syslog"] --> IN["Filebeat : input filestream"]
  F2["/var/log/auth.log"] --> IN
  IN -->|"output.logstash 127.0.0.1:5044"| BEATS["Logstash : input beats"]
  BEATS --> FILT["filter : mutate add_field pipeline"]
  FILT --> OUT["output elasticsearch"]
  FILT --> DBG["output stdout rubydebug"]
  OUT --> IDX[("Index filebeat-8.11.0-AAAA.MM.JJ")]
  IDX --> KIB["Kibana : Data View filebeat-*"]
```

## 6. Démarrage

```bash
# 0) Secret (non versionné)
echo -n 'MOT_DE_PASSE_APPLICATION_GMAIL' > secrets/smtp_password

# 1) Métriques
docker compose up -d

# 2) Logs (Linux : sudo sysctl -w vm.max_map_count=262144)
cp elastic/.env.example elastic/.env
docker compose -f elastic/docker-compose.yml up -d

# 3) Filebeat sur l'hôte
curl -L -o filebeat.deb https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.11.0-amd64.deb
sudo dpkg -i filebeat.deb && sudo cp filebeat/filebeat.yml /etc/filebeat/filebeat.yml
sudo systemctl enable --now filebeat
```
