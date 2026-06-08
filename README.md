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
Change your ports and environment variables to suit your needs.

# Configuring Promtail

First we need to tell Promtail to grab those rsyslog-collector logs. This is done in the promtail-config.yml file.
Create a scrape config to grab the logs from the rsyslog container. Make sure your folder mappings are correct.

# Configuring Loki

I don´t think I changed anything specifically here, but I have included the loki-config.yml file just in case.

# Configuring Grafana

I will not go into detail on how to get Grafana up and running here. Plenty of guides to find on how to do that online.
The only thing you need to do is configure Loki as a data source. If all your port mappings and IP addresses are correct and you have no firewall or ACLs configured blocking this, it should be straightforward.

# Configuring Omada

Now it is time to enable syslog in Omada. Go to your Site, Logs, Settings, Advanced and enable Remote Logging.

<img width="665" height="438" alt="image" src="https://github.com/user-attachments/assets/0f8203f0-7a09-42d8-888f-8d5a7f7fa919" />

Enter the IP address of your rsyslog-collector container (or the IP address of the host machine, depending on your configuration) and the rsyslog port you configured.

If you now go to your Network Config, Security, ACL, you should see a Log toggle:

<img width="1466" height="642" alt="image" src="https://github.com/user-attachments/assets/f2ec94c9-72fe-4f64-a774-05efe338ab86" />

# ACL tips and tricks

Here are a few things I have discovered by checking the forums and trial and error:

- If you want to block external internet access into your network, blocking access to "IPGroup_Any" is not sufficient. You also will want to block access to the Gateway Management Page. From my understanding: Blocking the Gateway Management Page blocks access to your WAN IP address and traffic will not even be able to enter your ingress interface. Blocking IPGroup_Any will make the traffic enter your ingress interface, and only then will it be blocked it it hits an ACL.
-  Omada has an internal GEOIP Database as well which you can use to create Location Groups. I do not know where Omada gets its Geo-IP data. I tried to search the forums but no answers. What I do know is that it differs from the MaxMind GeoIP database. If you block, for example, France, you will still see France pop up in your Grafana dasbhoard because Omada and MaxMind classify it differently.
-  If you go to your Global Settings, Security, you will also be able to block certain countries from access. If you block countries in this page, blocks will not be logged unless you have IDS/IPS enabled. This is not an option for me, as it completely tanks my performance on the ER-707M2. However, even if IDS/IPS is disabled, the countries you have blocked on the Threat Management Map will still be blocked. Only, there will be no logging whatsoever. As these attempts also do not make it to your ACLs.
