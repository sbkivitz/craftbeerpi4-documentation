# Sensor data logging (CSV or InfluxDB)

## Moving the logs off the SD card

An SD card wears out from **writes**, and on a running CraftBeerPi almost all of
the writes are logs: `cbpi.log` plus one rotating CSV per sensor. Rotation caps
how much space they occupy — it does nothing to reduce how much is written.

Both can be relocated together. Point them at a USB stick or a mounted network
share and the wear leaves the card entirely.

**In `config.yaml`** — the easier option on a normal install:

```yaml
logs-folder-path: /mnt/usb/cbpi-logs
```

**On the command line**, which takes precedence:

```bash
cbpi -c /home/pi/config -l /mnt/usb/cbpi-logs start
```

With systemd you can edit the unit's `ExecStart`, but using `config.yaml` means
you never have to touch the unit file.

{% hint style="info" %}
This moves **everything** that is written frequently: `cbpi.log` and every
`sensor_*.log`. Your configuration stays where it is, so the system still boots
if the drive is missing — CraftBeerPi warns and carries on rather than refusing
to start.
{% endhint %}

### How much does it actually write?

Roughly one 27-byte row per sensor per reading, so what dominates is the
**sensor publish interval**, not the number of sensors:

| OneWire `Interval` | 3 sensors | Per year |
| --- | --- | --- |
| 60 s (default) | ~117 KB/day | ~43 MB |
| 10 s | ~700 KB/day | ~250 MB |
| 1 s | ~7 MB/day | ~2.5 GB |

{% hint style="warning" %}
If you do not need per-second resolution, raising the sensor `Interval` is the
single most effective thing you can do for SD card life. A mash moves slowly; a
reading every 10 or 30 seconds loses nothing you care about.
{% endhint %}

### Two servers must not share a logs folder

If you run a second instance — a test rig, a simulator, a staging copy — give it
its **own** logs folder. Two servers writing the same `sensor_*.log` files
interleave their rows into one file, and rotation then fails outright:

```
PermissionError: [WinError 32] The process cannot access the file
because it is being used by another process
```

Each instance needs its own `logs-folder-path`.

## CSV logging

The data retrieved by sensors can be logged in different ways. The default is csv logging. CSV files will be stored for each sensor and the log is rotating. This means, that up to 4 csv files with 1 Mb per file can be created per sensor. Some sensors write one data point per second. If you have various sensors, plenty of write cycle will be done per second which is not good for the health of an SD card. In addition to that dashboard charts will read from those files which may drive up the CPU usage to 100% and may cause an unresponsive Interface in particular, when you are using a screen that is directly connected to your pi.

Therefore, it is not recommended to use charts in your brewing or fermentation dashboard. I recommend to use the analytics page instead.

To reduce the write cycles to the sd card, you can move the log folder to a USB drive or a network share — see [Moving the logs off the SD card](#moving-the-logs-off-the-sd-card) below.

If you are not using the log files and charts at all, it is recommended to switch of csv logging, which can be done on the [settings page](settings.md#global-system-parameters) via the parameter CSVLOGFILES. Set this parameter to 'No'.

CraftbeerPi4 has also the possibility to forward the sensor data to an InfluxDB database. This method is way more sophisticated and recommended as it allows the user to use and display the data for instance with grafana. 

To activate this functionality, you need to adapt the [settings](settings.md#global-system-parameters) for InfluxDB (Address, Port, database name,..) and set the parameter INFLUXDB to 'Yes'. Currently, InfluxDB versions up to 1.8.X are supported for server version < 4.1.7. Newer server versions can handle InfluxDB 2.0 or larger as well.

Below is an example for the usage of Influxdb in combination with Grafana. The dashboard shows sensor data for the Kettle and two fermenters. In addition, data from the cbpi4-system plugin that monitors the CPU load, free memory and more is displayed. I am also using a SCD30 sensor to monitor the 'environmental condition' of the room (CO2, temp and rel. humidity) and display this on the same dashboard.

![Example for InfluxDB usage in combination with Grafana](../../.gitbook/assets/cbpi4-grafana.png)

## Grafana Dashboard Chart
As shown in the [dashboard](dashboard.md#item-menu) section, you can also add a grafana chart to the dashboard.

![Example of Grafana Chart in Dashboard](../../.gitbook/assets/cbpi4-grafana-chart.png)

Required Parameters are described briefly in the dashboard section. Below is an example, how to set the 'url' and 'panelID' for your chart.

![Grafana URL Example](../../.gitbook/assets/cbpi4-grafana-url-example.png)

The url for this chart on Grafana is: ```http://192.168.163.105:3000/d/m5Lx6uYnz/craftbeerpi-4?orgId=1&viewPanel=7```

You need to use the first part of the url prior to the `?` and replace the `d`with `d-solo` and enter this into the url parameter:

```url: http://192.168.163.105:3000/d-solo/m5Lx6uYnz/craftbeerpi-4```

In the `panelID` Parameter you need to enter just the number of the panel from the original url: `7`

```panelID: 7```

In the `timeframe` Parameter you need to enter the timeframe you want to see in the chart until 'now'. It has to be entered in the grafana syntax:

```now-5m``` shows the last 5 minutes.

```now-12h``` shows the last 12 hours.

```now-7d``` shows the last 7 days.

```2023-02-17 2:00``` shows data from this date until now.

In your grafana server, you need to adapt a few settings.

1. You need to allow [embedding](https://grafana.com/docs/grafana/latest/administration/configuration/#allow_embedding) of your charts.
2. You need to allow [external_access](https://grafana.com/docs/grafana/latest/administration/configuration/#external_enabled).  
3. You need to enable [anonymous authentication](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/grafana/#anonymous-authentication)

{% hint style="info" %}
Depending on your setup on your pi, you may need to adapt also the chromium settings if the charts are displayed on another device, but not on your pi screen.

In this case, you need to allow all third party cookies in chromium. You need to enter: ```chrome://settings/content```

and change the cookie setting accordingly.
{% endhint %}