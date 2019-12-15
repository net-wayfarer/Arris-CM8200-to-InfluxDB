
# Arris CM8200 to InfluxDB

This is a python script to webscrape the Arris CM8200 web interface and place data into InfluxDB for graphing in Grafana. Its intended use is on the NBN MTM HFC network, specifically the CM8200B. It will however work with other Arris modems that use the same UI and others with some modification.

This assumes that Grafana and InfluxDB are already installed and working and that you have a basic understanding of both.
This also assumes that the Arris modem is accessible from within the end users LAN (with or without NAT rules etc).

Just a note, if there is a flap or dropout the Arris NTD will do a soft reboot thus blocking access to the WebUI requiring a factory reset using the reset pin for ~5 seconds. Unfortunately this is just how it is from the NBN overlords.

As an UBNT user, the below should gain you access assuming NBN has not disabled access. Has been tested on Edgerouter and USG products.
Users with the more recent ASUS models appear to be able to natively access the NTD.
```bash
risbo@Edge:~$ configure
[edit]
risbo@Edge#

set interfaces pseudo-ethernet peth0 address 192.168.0.2/24
set interfaces pseudo-ethernet peth0 description 'Modem Access'
set interfaces pseudo-ethernet peth0 link eth0
commit
set service nat rule 5000 description 'masquerade for HFC Modem'
set service nat rule 5000 outbound-interface peth0
set service nat rule 5000 type masquerade
commit
save
```


For [OpenWRT](https://openwrt.org) users, the following below should work for your case
```bash
uci set network.CM_ACCESS=interface
uci set network.CM_ACCESS.proto='static'
uci set network.CM_ACCESS.ifname='eth0.2'
uci set network.CM_ACCESS.ipaddr='192.168.0.2'
uci set network.CM_ACCESS.netmask='255.255.255.0'
uci commit
/etc/init.d/network restart
```
Remember to replace `CM_ACCESS` with the name of interface you wish to name it as, `eth0.2` with the actual interface of your "WAN" which should begin with the words `eth` not `wan` for example. Alternatively, there is a [LuCI/webUI guide](https://simplebeian.wordpress.com/2014/03/12/accessing-your-modem-from-openwrt-router/), should you prefer to go through that route.

![Grafana Overview](https://raw.githubusercontent.com/risb0r/Arris-CM8200-to-InfluxDB/master/images/overview.png)


## Installation

Clone to machine. I chose to personally run this from the `/opt/` directory.
```bash
sudo git clone https://github.com/risb0r/Arris-CM8200-to-InfluxDB.git arris_stats
```

Install python3, python3pip and python3-lxml as required.
```bash
sudo apt install python3 python3-pip python3-lxml
```
Use the pip package manager pip3 to install necessary requirements.
Note: Requirements may also have to be installed as `root` depending on your setup.
```bash
cd arris_stats
pip3 install -r requirements.txt
```

Setup influx with a database
```bash
$ influx
> CREATE DATABASE cm8200b_stats
```
Ensure that the database was created
```bash
> show databases
name: databases
name
----
cm8200b_stats <------
```

Adjust cm8200_stats.py - Host, Port, Database, Username and Password as neccesary.
```python
# Change settings below to your influxdb - database needs to be created or existing db
# creates 5 tables - downlink, uplink, fw_ver, uptime, event_log
# Second argument = default value if environment variable is not set.
influxip = os.environ.get("INFLUXDB_HOST", "127.0.0.1")
influxport = int(os.environ.get("INFLUXDB_HOST_PORT", "8086"))
influxdb = os.environ.get("INFLUXDB_DATABASE", "cm8200b_stats")
influxid = os.environ.get("INFLUXDB_USERNAME", "admin")
influxpass = os.environ.get("INFLUXDB_PASSWORD", "")

# cm8200b URL - Leave this unless your NTD URL is http://192.168.100.1
ntd_url = os.environ.get("NTD_URL", "http://192.168.0.1")
```

## Usage
### Running the webscraper

Standalone (once off)
```bash
/usr/bin/python3 /opt/arris_stats/cm8200b_stats.py
```

![cm8200_stats.py Output](https://raw.githubusercontent.com/risb0r/Arris-CM8200-to-InfluxDB/master/images/output.png)

As cron
```bash
sudo crontab -e
```
Place the below into crontab. Ctrl + X to exit.
This will run the script every 300 seconds.
```bash
# m h  dom mon dow   command

*/5 * * * * /usr/bin/python3 /opt/arris_stats/cm8200b_stats.py
```

### Setting up Grafana

Setup the data source as below

![Datasource Overview](https://raw.githubusercontent.com/risb0r/Arris-CM8200-to-InfluxDB/master/images/datasource.png)


Import the .json

If the images are out of wack check the grafana.ini file for the following config change.
```bash
$ sudo nano /etc/grafana/grafana.ini

[panels]
# If set to true Grafana will allow script tags in text panels. Not recommended as it enable XSS vulnerabilities.
disable_sanitize_html = true
```
## To Do List        

Auto scrape [Whirlpool](https://whirlpool.net.au/wiki/cmts-upgrades) and plug in the CMTS info from the wiki rather than just filling out a text box. Personally, I can't be bothered or care too much for something that will go mostly unchanged.

# Thanks
Thanks to [Andy Fraley](https://github.com/andrewfraley/arris_cable_modem_stats) for the initial starting point and grafana json.
Thanks to Luckst0r for the current python base code and doing some testing along with those who have made various contributions.
There are a few of us lurking on the [AussieBroadband Unofficial Discord](https://forums.whirlpool.net.au/archive/2713195) if ther are any questions or in need of setup assitance.

Thanks to the team at [Aussie Broadband](https://www.aussiebroadband.com.au/) for providing dope internet on a government cockup.

CMTS info is a static item and is available on [Whirlpool](https://whirlpool.net.au/wiki/cmts-upgrades) thanks to [Roger Ramband](https://forums.whirlpool.net.au/user/117375) for making this data readily available. This data is only relevant to those connected via the NBN in Australia. Can be skipped, removed, deleted, whatever for others.
