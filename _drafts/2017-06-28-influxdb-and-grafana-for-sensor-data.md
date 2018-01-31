---
layout: post
title: InfluxDB and Grafana for sensor data
---

![kuva]({{ site.url }}/assets/images/IMGP7873.jpg)

[InfluxDB](https://www.influxdata.com/time-series-platform/influxdb/) is
*a custom high-performance data store written specifically for time series data*.



[Grafana](https://grafana.com/) is *[t]he open platform for beautiful
analytics and monitoring*. It can be hosted on your own server and there is also
a [cloud option](https://grafana.com/cloud/grafana) available, which is free to use
for single user with also some other limitations.

For the purposes of this blog entry and the system so far, cloud Grafana is more
than enough, and it was really easy to set up once the data is available in
the internet.

### Required hardware and software

1.  Raspberry Pi
2.  Ansible and clone of [raspberry-ansible](https://github.com/rhietala/raspberry-ansible)
    repository configured with correct IP addresses and SSH keys
3.  One or more DS18B20 or compatible temperature sensors and
    Raspberry Pi configured with 1-Wire and 1-Wire temperature sensor
    support, see
    [previous blog post]({% post_url 2016-09-04-raspberry-pi-and-onewire-temperature-sensor %})
4.  Server for InfluxDB with a public IP address

### Installing InfluxDB

The Ansible playbook `tempreader-influxdb.yml` and `hosts` files must be
configured with InfluxDB server address. The other configuration options can
be left as default. When the playbook is run, it will ask for a password that
is set for InfluxDB access.

    $ ansible-playbook -i hosts tempreader-influxdb.yml

The role in `roles/influxdb/tasks/main.yml` assumes that the server is
running Debian. It will add Influxdata repository and install InfluxDB from
there. It will also create one admin user for the database and open
its ports in firewall with ufw.

If the ansible role is more in the way than of help, InfluxDB can be installed
manually and ansible used only for handling the Raspberry Pi.

### Reading temperatures and uploading them to InfluxDB

Second step of the ansible playbook is to install a Python script to read
temperatures and send the data to InfluxDB. This is done in the role
`roles/tempreader-influxdb/tasks/main.yml`.

Like with [RRDtool]({% post_url 2016-09-20-rrdtool-as-timeseries-datastore-for-sensor-data %}),
`pi`-user's home folder and crontab should look like this:

    pi@raspberry1:~ $ ls
    readtemp-influxdb.py

    pi@raspberry1:~ $ crontab -l
    #Ansible: read all temperature sensors to influxdb
    * * * * * /usr/bin/python /home/pi/readtemp-influxdb.py

Also the temperature reading script is quite similar to RRDtool one, here
the readings are wrapped to suitable format for writing to InfluxDB:

    measurements = []

    for sensor in W1ThermSensor.get_available_sensors():
        measurements.append({
            "measurement": "temperature",
            "time": datetime.datetime.utcnow().isoformat() + 'Z',
            "tags": {
                "sensor_id": sensor.id,
                "sensor_name": sensor_name(sensor.id),
                "host": platform.node()
            },
            "fields": {
                "value": sensor.get_temperature()
            }
        })

    if measurements: client.write_points(measurements)

> Conceptually you can think of a `measurement` as an SQL table, where the
> primary index is always time. `tags` and `fields` are effectively columns
> in the table. `tags` are indexed, and `fields` are not. The difference is
> that, with InfluxDB, you can have millions of measurements, you don’t
> have to define schemas up-front, and null values aren’t stored.
> [[Writing and exploring data]](https://docs.influxdata.com/influxdb/v1.2/introduction/getting_started/#writing-and-exploring-data)

Here the measurement is *temperature* and from that measurement only
a single *value* is stored. Three fields of metadata are also added:
sensor's id, its name (if any) and the host, which is the hostname of
Raspberry Pi where this reading is done.

This way it is easy to separate temperature readings from different
sensors, and where they approximately are (probably close to the Pi).
Timestamp is read from the Raspberry, so NTP should be properly configured.

[InfluxDB client library](https://github.com/influxdata/influxdb-python)
is installed with pip and really easy to use.

### Setting up Grafana cloud

Configuring Grafana is quite straightforward: configure a data source for
the InfluxDB and then create a dashboard with graphs from that data. Data
source has to have correct address and access credentials and that's
about it.

Dashboard is the interesting part. Here is a simple configuration that
supports multiple sensors:

![Grafana metrics]({{ site.url }}/assets/images/grafana-metrics.png)
