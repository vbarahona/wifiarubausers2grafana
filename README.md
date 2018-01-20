# wifiarubausers2grafana

Steps and configuration to create a complete wifi users dashboard in GRAFANA from ARUBA wireless switches infraestructure


# Getting Starting

## Prerequisites
The infraestruture needed is:
- Aruba wireless switch (obiously) ;-)
- InfluxDB
- Telegraf
- Grafana

I am asuming that your Aruba wireless switch is configured to anwer SNMP queries and you have a InfluxDB, Telegraf and Grafana instaled and configured propertly . I will not cover information about instalation and basic configuration.

## InfluxDB
We will use the database "telegraf" and will store all the data in the measuremnt "wifi_users". This will be created automaticaly by Telegraf (next step), but first we will need to create some "retention policies" and "continuous queries". Lets connect to influxdb and use telegraf database
```
$ influx
Connected to http://localhost:8086 version 1.4.2
InfluxDB shell version: 1.4.2
> use database telegraf
```
### Retention Policies
We will have 2 retention policies (RP). The "1day" will store the complete table data for a period of 1d. Its posible to make all the graphs from that table but very expensive for the database. That's the reason to create a second RP named "1year" where we will agregate the data for the long suport graphs. Feel free to create a longer policy if you need store a bigger data range.
```
> create retention policy "1year" on "telegraf" duration 365d replication 1
> create retention policy "1day" on "telegraf" duration 1d replication 1
```
### Continuous Queries
Now create the 5 continuous queries that will aggregate the "live" 1day user in a easier-to-work-and-store 1year data
```
CREATE CONTINUOUS QUERY devices_cq_5m ON telegraf BEGIN SELECT count(UserName) AS "Users" INTO telegraf."1year".wifi_users FROM telegraf."1day".wifi_users GROUP BY time(5m), DeviceType END
CREATE CONTINUOUS QUERY auth_cq_5m ON telegraf BEGIN SELECT count(UserName) AS "Users" INTO telegraf."1year".wifi_users FROM telegraf."1day".wifi_users GROUP BY time(5m), SubAuthenticationMethod END
CREATE CONTINUOUS QUERY encryption_cq_5m ON telegraf BEGIN SELECT count(UserName) AS "Users" INTO telegraf."1year".wifi_users FROM telegraf."1day".wifi_users GROUP BY time(5m), EncryptionMethod END
CREATE CONTINUOUS QUERY radio_cq_5m ON telegraf BEGIN SELECT count(UserName) AS "Users" INTO telegraf."1year".wifi_users FROM telegraf."1day".wifi_users GROUP BY time(5m), HTMode, PhyType END
CREATE CONTINUOUS QUERY cq_5m ON telegraf BEGIN SELECT count(UserName) AS "Users" INTO telegraf."1year".wifi_users FROM telegraf."1day".wifi_users GROUP BY time(5m), UserRole END
```
## Telegraf
We will colect data using Telegraf with the SNMP plugin, but first we have to create an output with tagexclude ant tagpass for sending the data to database "telegraf" with the right RP of 1day. Just download the [wifi-users.conf](https://github.com/vbarahona/wifiarubausers2grafana/raw/master/wifi-users.conf), copy the file in /etc/telegraf/telegraf.d/ and restart telegraf
```
sudo service telegraf restart
```
After some seconds, its a good idea to check if everything is working
```
sudo service telegraf status
```
## Grafana
