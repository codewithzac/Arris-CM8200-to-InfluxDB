
# Arris CM8200 to InfluxDB

This is a python script to webscrape the Arris CM8200 web interface and place data into InfluxDB for graphing in Grafana. Its intended use is on the NBN MTM HFC network, specifically the CM8200B. It will however work with other Arris modems that use the same UI and others with some modification.

This assumes that Grafana and InfluxDB are already installed and working and that you have a basic understanding of both.
This also assumes that the Arris modem is accessible from within the end users LAN (with or without NAT rules etc).

Just a note, if there is a flap or dropout the Arris NTD will do a soft reboot thus blocking access to the WebUI requiring a factory reset using the reset pin for ~5 seconds. Unfortunately this is just how it is from the NBN overlords.

[Click here](https://github.com/risb0r/Arris-CM8200-to-InfluxDB/blob/master/README.ROUTER.SETUP.GUIDES.md) for some end user setup guides for various consumer routers. Feel free to fork and contribute as desired.

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
# Change settings below to your influxdb - database needs to be created or existing db creates 5 tables - downlink, uplink, fw_ver, uptime, event_log
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
## Installation (Docker)

**Note:** You need to set up InfluxDB and Grafana before configuring the Docker image!

Clone to machine. Directory isn't important; `/tmp/` will work fine:

```bash
sudo git clone https://github.com/risb0r/Arris-CM8200-to-InfluxDB.git arris_stats
```

Build Docker image:

```bash
cd arris_stats
sudo docker build -t arris_stats
```

Run the image. Default settings assume Influx DB is on the same machine, but this can be overridden with environment variables:

```bash
sudo docker run --detach arris_stats:latest
```

### Environment Variables

| Variable | Notes | Default |
| --- | --- | --- |
| `_CHAP_INTERVAL` | Sets the schedule for running cm8200_stats.py.<br>Uses [Cron expressions](https://crontab.guru/examples.html). | * * * * *<br>_(i.e., every 1 minute)_ |
| `INFLUXDB_HOST` | Sets the hostname of the InfluxDB server. | 127.0.0.1 |
| `INFLUXDB_HOST_PORT` | Sets the port of the InfluxDB server. | 8086 |
| `INFLUXDB_DATABASE` | Name of the database. | cm8200b_stats |
| `INFLUXDB_USERNAME` | InfluxDB user with access to the database. | admin |
| `INFLUXDB_PASSWORD` | Password for the InfluxDB user. | _(empty)_ |

### Docker Compose

Docker Compose can also be used. Here is a simple `docker-compose.yml` example to run InfluxDB, Grafana and cm8200_stats.py in a single stack:

```
version: "3"

services:

  influxdb:
    restart: always
    image: influxdb:1.8
    ports:
      - '8086:8086'
    volumes:
      - influxdb:/var/lib/influxdb
    environment:
      - INFLUXDB_DB=cm8200b
      - INFLUXDB_HTTP_AUTH_ENABLED=true
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_USER=cm8200b
      - INFLUXDB_USER_PASSWORD=supersecretpassword

  grafana:
    image: grafana/grafana:latest
    restart: always
    ports:
      - '3000:3000'
    volumes:
      - grafana:/var/lib/grafana
    depends_on:
      - influxdb
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=extrasupersecretpassword
      - GF_INSTALL_PLUGINS=grafana-clock-panel,natel-discrete-panel,grafana-piechart-panel

  script:
    image: arris_stats:latest
    restart: always
    depends_on:
      - influxdb
    environment:
      - _CHAP_INTERVAL: "*/5 * * * *"
      - INFLUXDB_DATABASE: cm8200b
      - INFLUXDB_USERNAME: cm8200b
      - INFLUXDB_PASSWORD: supersecretpassword

volumes:
  influxdb:
  grafana:
```

## To Do List        

Auto scrape [Whirlpool](https://whirlpool.net.au/wiki/cmts-upgrades) and plug in the CMTS info from the wiki rather than just filling out a text box. Personally, I can't be bothered or care too much for something that will go mostly unchanged.

# Thanks
Thanks to [Andy Fraley](https://github.com/andrewfraley/arris_cable_modem_stats) for the initial starting point and grafana json.
Thanks to Luckst0r for the current python base code and doing some testing along with those who have made various contributions.
There are a few of us lurking on the [AussieBroadband Unofficial Discord](https://forums.whirlpool.net.au/archive/2713195) if ther are any questions or in need of setup assitance.

Thanks to the team at [Aussie Broadband](https://www.aussiebroadband.com.au/) for providing dope internet on a government cockup.

CMTS info is a static item and is available on [Whirlpool](https://whirlpool.net.au/wiki/cmts-upgrades) thanks to [Roger Ramband](https://forums.whirlpool.net.au/user/117375) for making this data readily available. This data is only relevant to those connected via the NBN in Australia. Can be skipped, removed, deleted, whatever for others.
