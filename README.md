# Mini SIEM with Elastic Stack

A small SIEM lab: Linux SSH authentication logs are collected with Filebeat, stored in Elasticsearch, and visualized in a Kibana dashboard.

## Architecture

    auth.log (Kali VM) -> Filebeat (system module) -> Elasticsearch -> Kibana

## Components

- Elasticsearch 8.15.3 (single node, 1 GB heap)
- Kibana 8.15.3
- Filebeat 8.15.3 with the system module (auth fileset)
- rsyslog and OpenSSH server on the Kali VM (source of auth.log)

The Elastic components run in Docker Compose. Everything runs inside one VM.

## Dashboard

![Dashboard](docs/screenshots/dashboard.png)

Panels:

- Failed logins over time
- Top usernames
- Total failed logins

## Setup

Elasticsearch needs a higher memory-map limit:

    sudo sysctl -w vm.max_map_count=262144

Start the stack:

    docker compose up -d

Open Kibana at http://localhost:5601 and create a data view with the pattern `filebeat-*` and timestamp field `@timestamp`.

## Generating test data

Failed SSH logins were generated against the local SSH service only (127.0.0.1):

    for u in admin root test oracle guest; do
      sshpass -p wrongpass ssh -o StrictHostKeyChecking=no -o PubkeyAuthentication=no -o ConnectTimeout=5 $u@127.0.0.1 true 2>/dev/null
    done

Dashboard filter used:

    event.dataset : "system.auth" and event.outcome : "failure"

## Finding: data quality issue

Some usernames were stored with a leading space (for example " admin" and "admin" counted as two values), which split the counts in charts. I confirmed it with a terms aggregation in Elasticsearch. I worked around it with a Kibana runtime field, `user_clean`, which applies `trim()` to `user.name`. I did not inspect the ingest pipeline itself, so the root cause is a hypothesis: different sshd message formats being parsed differently.

## Limitations

- Lab setup only: Elastic security is disabled and ports are bound to localhost. Do not expose it to a network.
- One Linux host. The source IP is always 127.0.0.1 because the attempts were generated locally.
- No alerting rules yet.
- Windows logs are not included yet.

## Next steps

- Add Windows Security events (4625, 4624) with Winlogbeat or Elastic Agent
- Add a brute-force detection rule
- Export the dashboard as a saved object

## Disclaimer

For educational purposes only.
