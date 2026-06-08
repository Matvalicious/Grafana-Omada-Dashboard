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

- If you want to block external internet access into your network, blocking access to "IPGroup_Any" is not sufficient. You also will want to block access to the Gateway Management Page. From my understanding: Blocking the Gateway Management Page blocks access to your WAN IP address and traffic will not even be able to enter your ingress interface. Blocking IPGroup_Any will make the traffic enter your ingress interface, and only then will it be blocked if it hits an ACL.
-  Omada has an internal GEOIP Database as well which you can use to create Location Groups. I do not know where Omada gets its Geo-IP data. I tried to search the forums but no answers. What I do know is that it differs from the MaxMind GeoIP database. If you block, for example, France, you will still see France pop up in your Grafana dasbhoard because Omada and MaxMind classify it differently.
-  If you go to your Global Settings, Security, you will also be able to block certain countries from access. If you block countries in this page, blocks will not be logged unless you have IDS/IPS enabled. This is not an option for me, as it completely tanks my performance on the ER-707M2. However, even if IDS/IPS is disabled, the countries you have blocked on the Threat Management Map will still be blocked. Only, there will be no logging whatsoever. As these attempts also do not make it to your ACLs.

# Configuring Grafana

Open Grafana, go to Explore, select Loki as your datasource and choose the "job" and "ryslogs" as Label filters. If you Run query, this should now give you the sylogs from your Omada:

<img width="1720" height="1353" alt="image" src="https://github.com/user-attachments/assets/a89227e0-814e-4410-b8d1-00bbf3a8f72a" />

This will give ALL the syslog messages. If you specifically want the ACL logs, you can filter on "[ACL]GATEWAY".

The ACL log messages all follow the same pattern:

``` X.X.X.X MODEL: TIMESTAMP firewall<5>: [ACL]GATEWAY=XX-XX-XX-XX-XX-XX,DESC=xxxxxxxxxx,ACTION=Block,SRC_MAC=xx:xx:xx:xx:xx:xx,DST_MAC=xx:xx:xx:xx:xx:xx,SRC=X.X.X.X,DST=X.X.X.X,PROTO=TCP,Sport=xxxxx,Dport=xxxxx ```


``` Gateway IP, Gateway Model, Timestamp, logtype<severity>, [ACL]GATEWAY=Gateway MAC address, ACL ID, Action, Source MAC, Destination MAC, Source IP, Destination IP, Protocol, Source Port, Destination Port ```

You will see that the syslog message does not say which ACL is being hit, it only gives an ACL ID in the "DESC" field. To figure out which ACL ID belongs to which ACL, we can turn to syslog again.

Go to your ACL and turn off logging, then turn it back on. Go back to Grafana and now filter the messsage on Line Contains "entryId". You should see a syslog message containing a lot of information, including

``` (...) \"entryId\":xxxxxxxxxxx,\"siteId\":\"yyyyyyyyyyyyyyyyyyyyyy\",\"name\":\"ACL_NAME\" (...) ``` 

In this log line, you can see exaclty the ID of the ACL you changed, and the name of the ACL. Do this for all your ACLs and write them down somewhere.

# Creating a dasboard

To create a table showing my ACL logs, I used the following query:

```
{job="ryslogs"} 
|= `[ACL]GATEWAY` 
|~ `(?i)${search_filter:raw}` 
|~ `(?i)${action_filter:raw}` 
|~ `(?i)${acl_filter:raw}` 
| geoip_country_name =~ "(?i).*${country_filter:raw}.*"
| regexp `ACTION=(?P<Action>[^,]+)` 
| regexp `SRC=(?P<Source>[^,]+)` 
| regexp `DST=(?P<Destination>[^,]+)` 
| regexp `Sport=(?P<Source_Port>\d+)` 
| regexp `Dport=(?P<Destination_Port>\d+)` 
| regexp `PROTO=(?P<Protocol>[^,]+)` 
| regexp `DESC=(?P<Description>[^,]+)`
| label_format Country="-", City="-", Continent="-"
| label_format Country=geoip_country_name, City=geoip_city_name, Continent=geoip_continent_name
```
Then applied an Extract fields, and Organize fields by name Transoformation in order to hide the labels I do not want to see.

On the right hand side, look for Value Mappings and create a value mapping for each of your ACL IDs to their respective Names

<img width="1720" height="1353" alt="image" src="https://github.com/user-attachments/assets/d501693b-c715-4ec3-b46b-147b6267c8eb" />

This will give you a nicely formatted table of your logs. Tweak to your liking.

For the Geomap dashboard, use the following query

```
sum by (geoip_location_latitude, geoip_location_longitude, geoip_country_name, geoip_city_name, geoip_country_code) (
  count_over_time(
    {job="ryslogs"} 
    |= `[ACL]GATEWAY` 
    |~ `(?i)${search_filter:raw}` 
    |~ `(?i)${action_filter:raw}` 
    |~ `(?i)${acl_filter:raw}` 
    | geoip_country_name =~ "(?i).*${country_filter:raw}.*" 
    [$__auto]
  )
) or vector(0)
```

With the Transformations Reduce (Series to Rows) and Organize fields by name. Change the location Mode to Coords and fill in the latitude and longitude data fields. Configure the radius, blur, etc, to your likings

<img width="1720" height="1353" alt="image" src="https://github.com/user-attachments/assets/fdaf35ee-2a65-4553-bc86-d367fb591059" />

For the top sources/top destinations pie chart:

```
topk(10, sum by(Source) (
  count_over_time(
    {job="ryslogs"} 
    |= `[ACL]GATEWAY` 
    |~ `(?i)${search_filter:raw}` 
    |~ `(?i)${action_filter:raw}` 
    |~ `(?i)${acl_filter:raw}` 
    | geoip_country_name =~ "(?i).*${country_filter:raw}.*"
    | regexp `SRC=(?P<Source>[^,]+)` 
    [$__range]
  )
))
```

Here no transformations are needed.

# Filtering the dashboard

You may notice this filter in all my queries:

```
    |~ `(?i)${search_filter:raw}` 
    |~ `(?i)${action_filter:raw}` 
    |~ `(?i)${acl_filter:raw}`
    | geoip_country_name =~ "(?i).*${country_filter:raw}.*"
```

This is coming from the variables defined on top, and allows me to filter the table, map, and pie charts all at once.

<img width="706" height="97" alt="image" src="https://github.com/user-attachments/assets/0139e3dd-98db-4edc-8aab-0321bb7f2c53" />

These are just textbox and custom filters. Manually populated with the ACL ID to Name mappings for the ACL filter:

<img width="1360" height="986" alt="image" src="https://github.com/user-attachments/assets/0d78d82e-9f9b-4306-a9f0-7d27c8a0c9a4" />

To make the IP addresses clickable and have them automatically applied to the top filters, you need to use Data Links. For example, on the Table view, create a Datalink for the Soruce field with URL

```
/d/dashboard_ID/dashboard_Name?var-search_filter=${__value.text}&var-action_filter=${action_filter}&var-acl_filter=${acl_filter}&var-country_filter=${country_filter}&${__url.timeRange}
```

This will grab the IP you clicked, and enter in the search_filter textbox on top, while leaving the other filters active. You can do the same for the other fields.

# Whats next?

I may still integrate a line chart showing a graph of all the ACLs and the number of times they are being hit over time.
