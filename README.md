# Grafana-Omada-Dashboard
Grafana Dashboard for displaying TP-Link Omada ACL logs

<img width="3066" height="1239" alt="Screenshot_20260607_210118" src="https://github.com/user-attachments/assets/90dc4ad6-5b14-4d6e-b655-09c4a104becc" />

# Why?

I am quite fond of the Omada ecosystem, but one thing that has always bothered me is that there is no way of visualizing your Firewall/ACL logs. Recently I found out that, when you enable an external syslog server, this enables a Log option on your ACLs. Turns out that every single Allow or Block action is very neatly logged to your external syslog server. In order to actually visualize these logs, I am using a Loki/Grafana stack to turn the raw syslog into neatly organized tables and graphs.

# What do you need?

- A syslog server. I am using rsyslog-collector, as this seemed to be by far the easiest one to set up. It just ingests syslog messages and dumps them into a local log file.
- Promtail. This is the service that will read and parse the log file, and is able to enrich it with GeoIP information.
- Loki. This is the database that will store the ingested Promtail logs.
- Grafana. The web UI for displaying the logs.
- Optional, if you want GeoIP info: The MaxMindInc/geoipudpate container to grab the geo database and store it locally.

In my setup I will be using Docker Compose.

# Compose file

See the docker-compose.yaml file. This contains all the required containers mentioned above.

