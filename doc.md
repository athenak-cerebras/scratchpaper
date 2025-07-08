# Using Telegraf, InfluxDB, and Grafana to visualize metrics on asic machines

We use a combination of Telegraf, InfluxDB, and Grafana to visualize metrics. Telegraf collects the data, InfluxDB stores the data, and Grafana visualizes the data.

## Setting Up
Set up script:
```
#!/bin/bash
set -euo pipefail

# === CONFIGURABLE DISK DEVICE ===
DISK_DEVICE=${DISK_DEVICE:-/dev/sdb}
MOUNT_LABEL="spare"
MOUNT_POINT="/spare"
INFLUX_DIR="$MOUNT_POINT/influxdb"
INFLUX_BIND="/var/lib/influxdb"

echo "==> Using disk: $DISK_DEVICE"

# === Check disk exists ===
if [ ! -b "$DISK_DEVICE" ]; then
  echo "❌ Error: Disk $DISK_DEVICE does not exist."
  exit 1
fi

# === Check if disk already has data ===
if blkid "$DISK_DEVICE" > /dev/null 2>&1; then
  FS_TYPE=$(blkid -s TYPE -o value "$DISK_DEVICE")
  echo "⚠️ Warning: $DISK_DEVICE already has a filesystem: $FS_TYPE"

  read -rp "Are you sure you want to format $DISK_DEVICE? This will erase all data. [y/N] " CONFIRM
  if [[ ! "$CONFIRM" =~ ^[Yy]$ ]]; then
    echo "Aborting."
    exit 1
  fi
fi

# === Install InfluxDB 2.x if needed ===
if ! rpm -q influxdb2 > /dev/null 2>&1; then
  echo "==> Installing InfluxDB 2.x..."
  cat > /etc/yum.repos.d/influxdb.repo <<'EOF'
[influxdb]
async = 1
baseurl = https://repos.influxdata.com/rhel/$releasever/$basearch/stable
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdata-archive_compat.key
name = InfluxDB Repository - RHEL $releasever
EOF

  dnf install -y influxdb2
  systemctl enable influxdb
else
  echo "==> InfluxDB 2.x already installed."
fi

# === Format the disk with XFS ===
echo "==> Formatting $DISK_DEVICE with XFS..."
mkfs.xfs -f "$DISK_DEVICE" -L "$MOUNT_LABEL"

# === Mount to /spare ===
echo "==> Mounting $DISK_DEVICE to $MOUNT_POINT..."
mkdir -p "$MOUNT_POINT"
grep -q "LABEL=$MOUNT_LABEL $MOUNT_POINT" /etc/fstab || echo "LABEL=$MOUNT_LABEL $MOUNT_POINT xfs defaults 0 0" >> /etc/fstab
mountpoint -q "$MOUNT_POINT" || mount "$MOUNT_POINT"

# === Stop InfluxDB ===
echo "==> Stopping InfluxDB..."
systemctl stop influxdb || true

# === Move InfluxDB data if it exists ===
if [ -d "$INFLUX_BIND" ] && [ ! -L "$INFLUX_BIND" ] && [ ! -d "$INFLUX_DIR" ]; then
  echo "==> Moving existing InfluxDB data to $INFLUX_DIR..."
  mv "$INFLUX_BIND" "$INFLUX_DIR"
fi

mkdir -p "$INFLUX_DIR"

# === Bind mount /var/lib/influxdb to /spare/influxdb ===
mkdir -p "$INFLUX_BIND"
grep -q "$INFLUX_DIR $INFLUX_BIND none bind" /etc/fstab || echo "$INFLUX_DIR $INFLUX_BIND none bind 0 0" >> /etc/fstab
mountpoint -q "$INFLUX_BIND" || mount "$INFLUX_BIND"

# === Fix permissions ===
chown -R influxdb:influxdb "$INFLUX_DIR"

# === Start InfluxDB ===
echo "==> Starting InfluxDB..."
systemctl start influxdb

# === Verify status ===
echo "==> Verifying..."
systemctl is-active --quiet influxdb && echo "✅ InfluxDB is running"
df -h "$INFLUX_BIND"
```
Telegraf, InfluxDB, and Grafana are all running on asic-admin. To make sure telegraf is sending to the database, you must make sure the telegraf config file is updated
```
[[outputs.influxdb_v2]] 
  urls = ["http://localhost:8086"] 
  token = "your-influxdb-token" 
  organization = "your-org" 
  bucket = "asic"
```
You can access the influxDB UI at http://asic-admin:8086/. For asic-admin, the organization name is "cerebras" and the bucket is "asic". To get the token, go to the influxDB UI, click the button that looks like an arrow pointing up on the left panel, and click API tokens. You can only see tokens once after you generate them so it is best to save them somewhere. 
Once you are done setting up, you can restart telegraf:
```
sudo /bin/systemctl restart telegraf
```
You can look at the telegraf status with:
```
sudo /bin/systemctl status telegraf 
```
## Telegraf 
Telegraf will run scripts on the asic machines and send the data to influxDB. The scripts collect the metrics and write them in telegraf line protocol to make it easy for influxDB to store and read.

<img src=https://github.com/user-attachments/assets/98008802-d613-4497-a3d0-278b461d1f14 height="150">

Both tag sets and field sets have key-value pairs. Tag key-value pairs have string keys and string values. Field key-value pairs have string keys and float, integer, string, or Boolean values. Each line of telegraf line protcol has a timestamp. If you do not provide the timestamp, telegraf will add it for you.

Telegraf should currently be running several scripts to collect metrics:
### du-report.py
Collects the recursive and direct directory disk usage as well as user disk usage. Also generates a report file summarizing disk usage.
### lmstat.py
Collects license usage. 
### run_squeue.sh
Collects running and pending job information

You can tell telegraf what scripts to run and send to the influxDB database in the telegraf config file. 
In the influx.exec section, add your command, data format, and interval like so.
```
[[inputs.exec]] 
  commands = ["/scratch/disk.sh --telegraf"] 
  timeout = "10s" 
  data_format = "influx" 
  interval = "1m" 
```
## InfluxDB
InfluxDB is a time series database designed to store time series data for fields such as monitoring and recording metrics. You can query data in the flux query language in the data explorer page of the influxDB UI.
The data in influxDB is stored in locations called Buckets. For our asic machine data, we are storing it in a bucket called "asic". 
## Grafana
We use grafana to visualuze the data in influxDB. To connect influxDB to grafana, go to "add new connection" on the left panel. Search for "influxDB" and click on it, and then click on "Add new data source". Select InfluxQL for the query language. Under HTTP and URL, add the influxDB URL. For asic-admin, it is http://asic-admin:8086. Turn off "Basic Auth" and add an HTTP header. For the header write "Authorization" and for the value write "Token \<your-token\>". Make sure you write out the "Token" and the space before pasting your actual token. Fill in the rest of the influxDB details like Database (the bucket name), Username, and Password. When you are finished, you can click "Save & Test" and it shoud say "datasource is working" like so.

<img width="1153" alt="Screenshot 2025-07-08 at 12 18 46 PM" src="https://github.com/user-attachments/assets/4d96b1f1-cf18-4250-b2dd-2fda0a2b7181" height="500">



