# IoT Temperature and Humidity Sensor with Web Visualization

## 1. Raspberry Pi OS Installation

System update:
```bash
sudo apt update && sudo apt full-upgrade -y
```

SSH setup:
```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

## 2. Connecting the Temperature and Humidity Sensor to GPIO

First, the I2C bus needs to be enabled in the Raspberry Pi settings using the command:
```bash
sudo raspi-config nonint do_i2c 0
```

I used the TH3 temperature and humidity sensor, which contains an intelligent Sensirion SHT31 sensor. It communicates through a copper multi-core cable with a silicone sheath via the I2C bus.

The Raspberry Pi sends a command to the sensor and the sensor sends back data in the form of:
```
66 7A 93 5C A1 7F
```

The I2C bus uses only 2 wires — SDA and SCL.

SCL controls the timing `|‾|` which creates high/low pulses and determines when SDA should be read. SDA then carries the actual bits (0/1). On each SCL clock cycle, SDA carries a value of 0 or 1 — the device either pulls it LOW or leaves it HIGH. This is how the Raspberry Pi reads and processes data from the sensor via the I2C bus.

The GPIO wiring uses the following pins:

- 1 — 3.3V
- 3 — gpio2 - SDA
- 5 — gpio3 - SCL
- 9 — GND
- 17 — 3v3
- 25 — GND
- 27 — DNC
- 28 — DNC
![](image1.png)
I found the sensor address using the command:
```bash
sudo i2cdetect -y 1
```

I generated the code using Gemini and provided the correct wiring and sensor address (0x44). The code confirmed that the sensor is communicating with the Raspberry Pi correctly.

Sensor 1 is communicating.  
Sensor 2 is not communicating and is overheating.

## 3. Data Collection

Working Python code for measuring data every 10 seconds:

```python
import time
from smbus2 import SMBus

I2C_ADDRESS = 0x44

def cti_sht3x():
    with SMBus(1) as bus:
        bus.write_i2c_block_data(I2C_ADDRESS, 0x2C, [0x06])
        time.sleep(0.05)
        data = bus.read_i2c_block_data(I2C_ADDRESS, 0x00, 6)
        raw_temp = (data[0] << 8) + data[1]
        teplota = -45 + (175 * raw_temp / 65535.0)
        raw_humidity = (data[3] << 8) + data[4]
        vlhkost = 100 * raw_humidity / 65535.0
        return teplota, vlhkost

print("Starting SHT3x sensor reading...")
print("---------------------------------")

try:
    while True:
        temp, hum = cti_sht3x()
        print(f"Temperature: {temp:.2f} °C | Humidity: {hum:.2f} %")
        time.sleep(10)
except KeyboardInterrupt:
    print("\nMeasurement stopped by user.")
except Exception as e:
    print(f"\nCommunication error occurred: {e}")
```

## 4. Database

Out of InfluxDB and PostgreSQL, I chose InfluxDB.

I successfully connected InfluxDB to the Raspberry Pi and data from the sensor is readable in it. The installation was done using the following commands:

```bash
curl -fsSL https://repos.influxdata.com/influxdata-archive_compat.key | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg

echo "deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg] https://repos.influxdata.com/debian stable main" | sudo tee /etc/apt/sources.list.d/influxdata.list

sudo apt update && sudo apt install influxdb2 -y
```

The following libraries are also required:

```bash
pip install influxdb-client
pip install Flask
pip install smbus2 raspi-sht31
```

Structure design:

| InfluxDB Concept | Name in Code | Meaning | Reason |
|---|---|---|---|
| Bucket | vlhkost_teplota | Project name | Isolated space for all sensor data |
| Measurement | Klima_senzory | Like an SQL table name | Groups all climate data from the sensor |
| Tag | Location = Raspberry Pi | Descriptive information | InfluxDB uses tags to filter data |
| Field | Temperature | 22.5 decimal number | Used to draw graphs based on value |
| Field | Humidity | 45.2 decimal number | Used to draw graphs based on value |
| Timestamp | Auto-generated | time | Exact time when the measurement occurred |

We start InfluxDB with the command `sudo systemctl start influxdb` and verify it is running using `sudo systemctl status influxdb`. If the service is active, we log in to the web interface at `http://localhost:8086` — this applies when accessing directly from the Raspberry Pi.

Then in the terminal we run Python with the command `python3` and paste the following code:

```python
import time
from smbus2 import SMBus
import influxdb_client
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS

token = "your influxdb token"
org = "your org"
url = "http://127.0.0.1:8086"
bucket = "vlhkost_teplota"

write_client = InfluxDBClient(url=url, token=token, org=org)
write_api = write_client.write_api(write_options=SYNCHRONOUS)

I2C_ADDRESS = 0x44

def cti_sht3x():
    with SMBus(1) as bus:
        bus.write_i2c_block_data(I2C_ADDRESS, 0x2C, [0x06])
        time.sleep(0.05)
        data = bus.read_i2c_block_data(I2C_ADDRESS, 0x00, 6)
        raw_temp = (data[0] << 8) + data[1]
        teplota = -45 + (175 * raw_temp / 65535.0)
        raw_humidity = (data[3] << 8) + data[4]
        vlhkost = 100 * raw_humidity / 65535.0
        return teplota, vlhkost

print(f"Starting SHT3x sensor logging to bucket '{bucket}'...")
print("-------------------------------------------------------")

try:
    while True:
        temp, hum = cti_sht3x()
        print(f"Measured -> Temperature: {temp:.2f} °C | Humidity: {hum:.2f} %")
        
        bod = Point("klima_senzory") \
            .tag("umisteni", "raspberry_pi") \
            .field("teplota", temp) \
            .field("vlhkost", hum)
        
        try:
            write_api.write(bucket=bucket, org=org, record=bod)
            print("   Data successfully sent to InfluxDB")
        except Exception as e_db:
            print(f"   Failed to write to DB: {e_db}")
            
        print("-" * 55)
        time.sleep(2)

except KeyboardInterrupt:
    print("\nMeasurement stopped by user.")
except Exception as e:
    print(f"\nUnexpected error occurred: {e}")
finally:
    write_client.close()
    print("InfluxDB connection closed.")
```
Screenshot of InfluxDB graph:

![](image2.png)
## 5. Web Application and Visualization

I saved the web application backend to the `muj_dashboard` folder and the Frontend to the `templates` subfolder.

For the web application to work, the InfluxDB script needs to be running in the background. Then open a new terminal and enter:

```bash
cd ~/muj_dashboard
python3 app.py
```

The terminal will display the address where the page is running. After opening it should look like this:
![](image3.png)
Here is the frontend code:

```html
<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Meteostanice</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body { font-family: 'Segoe UI', sans-serif; background-color: #121214; color: #e1e1e6; display: flex; flex-direction: column; align-items: center; padding: 40px 20px; margin: 0; }
        h1 { color: #00adb5; }
        .dashboard { display: flex; gap: 20px; margin: 20px 0; }
        .card { background-color: #202024; border-radius: 12px; padding: 20px 40px; text-align: center; border: 1px solid #29292e; width: 150px; }
        .card h2 { margin: 0; font-size: 1.1rem; color: #8d8d99; }
        .value { font-size: 2.5rem; font-weight: bold; color: #fff; }
        .unit { font-size: 1.2rem; color: #00adb5; }
        .chart-container { background-color: #202024; border-radius: 12px; padding: 20px; width: 90%; max-width: 800px; border: 1px solid #29292e; }
        .status { margin-top: 15px; font-size: 0.85rem; color: #7c7c8a; }
    </style>
</head>
<body>
    <h1>Meteostanice Raspberry Pi</h1>
    <div class="dashboard">
        <div class="card">
            <h2>Teplota</h2>
            <div class="value"><span id="temp">--</span><span class="unit"> °C</span></div>
        </div>
        <div class="card">
            <h2>Vlhkost</h2>
            <div class="value"><span id="hum">--</span><span class="unit"> %</span></div>
        </div>
    </div>
    <div class="chart-container">
        <canvas id="historyChart"></canvas>
    </div>
    <div class="status">Poslední obnova: <span id="time">--:--:--</span></div>
    <script>
        let mujGraf;

        function vytvorGraf(casy, teploty, vlhkosti) {
            const ctx = document.getElementById('historyChart').getContext('2d');
            mujGraf = new Chart(ctx, {
                type: 'line',
                data: {
                    labels: casy,
                    datasets: [
                        { label: 'Teplota (°C)', data: teploty, borderColor: '#ff6384', backgroundColor: 'rgba(255, 99, 132, 0.1)', yAxisID: 'yTemp', tension: 0.2, pointRadius: 1 },
                        { label: 'Vlhkost (%)', data: vlhkosti, borderColor: '#36a2eb', backgroundColor: 'rgba(54, 162, 235, 0.1)', yAxisID: 'yHum', tension: 0.2, pointRadius: 1 }
                    ]
                },
                options: {
                    responsive: true,
                    animation: false,
                    scales: {
                        yTemp: { type: 'linear', position: 'left', title: { display: true, text: 'Teplota (°C)', color: '#ff6384' } },
                        yHum: { type: 'linear', position: 'right', title: { display: true, text: 'Vlhkost (%)', color: '#36a2eb' }, grid: { drawOnChartArea: false } }
                    }
                }
            });
        }

        async function aktualizujAktualniData() {
            try {
                const response = await fetch('/api/data');
                const data = await response.json();
                document.getElementById('temp').innerText = data.teplota;
                document.getElementById('hum').innerText = data.vlhkost;
                document.getElementById('time').innerText = new Date().toLocaleTimeString('cs-CZ');
            } catch (e) { console.error(e); }
        }

        async function aktualizujGraf() {
            try {
                const response = await fetch('/api/historie');
                const data = await response.json();
                if (!mujGraf) {
                    vytvorGraf(data.casy, data.teploty, data.vlhkosti);
                } else {
                    mujGraf.data.labels = data.casy;
                    mujGraf.data.datasets[0].data = data.teploty;
                    mujGraf.data.datasets[1].data = data.vlhkosti;
                    mujGraf.update();
                }
            } catch (e) { console.error(e); }
        }

        aktualizujAktualniData();
        aktualizujGraf();

        setInterval(aktualizujAktualniData, 3000);
        setInterval(aktualizujGraf, 10000);
    </script>
</body>
</html>
```

Here is the backend code:

```python
from flask import Flask, render_template, jsonify
from influxdb_client import InfluxDBClient

app = Flask(__name__)

TOKEN = "ENTER YOUR INFLUXDB TOKEN HERE"
ORG = "YOUR ORG"
URL = "http://127.0.0.1:8086"
BUCKET = "vlhkost_teplota"

client = InfluxDBClient(url=URL, token=TOKEN, org=ORG)
query_api = client.query_api()

def ziskej_aktualni_data():
    query = f'''
    from(bucket: "{BUCKET}")
      |> range(start: -10m)
      |> filter(fn: (r) => r["_measurement"] == "klima_senzory")
      |> last()
    '''
    try:
        tables = query_api.query(query, org=ORG)
        data = {"teplota": "--", "vlhkost": "--"}
        for table in tables:
            for record in table.records:
                if record.get_field() == "teplota":
                    data["teplota"] = round(record.get_value(), 2)
                elif record.get_field() == "vlhkost":
                    data["vlhkost"] = round(record.get_value(), 2)
        return data
    except Exception:
        return {"teplota": "Chyba", "vlhkost": "Chyba"}

def ziskej_historii():
    query = f'''
    from(bucket: "{BUCKET}")
      |> range(start: -24h)
      |> filter(fn: (r) => r["_measurement"] == "klima_senzory")
      |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
      |> sort(columns: ["_time"])
      |> tail(n: 100)
    '''
    try:
        tables = query_api.query(query, org=ORG)
        historie = {"casy": [], "teploty": [], "vlhkosti": []}
        for table in tables:
            for record in table.records:
                cas = record.get_time().strftime('%H:%M:%S')
                t = record.values.get("teplota")
                v = record.values.get("vlhkost")
                historie["casy"].append(cas)
                historie["teploty"].append(round(t, 1) if t is not None else None)
                historie["vlhkosti"].append(round(v, 1) if v is not None else None)
        return historie
    except Exception as e:
        print(f"Chyba na backendu: {e}")
        return {"casy": [], "teploty": [], "vlhkosti": []}

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/api/data')
def api_data():
    return jsonify(ziskej_aktualni_data())

@app.route('/api/historie')
def api_historie():
    return jsonify(ziskej_historii())

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

## 6. Automation, Autostart on Boot

In the terminal, open the `logger.py` file with the command:

```bash
nano ~/muj_dashboard/logger.py
```

And paste the following code into it:

```python
import time
from smbus2 import SMBus
import influxdb_client
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS

token = "YOUR TOKEN HERE"
org = "your org"
url = "http://127.0.0.1:8086"
bucket = "vlhkost_teplota"

write_client = InfluxDBClient(url=url, token=token, org=org)
write_api = write_client.write_api(write_options=SYNCHRONOUS)

I2C_ADDRESS = 0x44

def cti_sht3x():
    with SMBus(1) as bus:
        bus.write_i2c_block_data(I2C_ADDRESS, 0x2C, [0x06])
        time.sleep(0.05)
        data = bus.read_i2c_block_data(I2C_ADDRESS, 0x00, 6)
        raw_temp = (data[0] << 8) + data[1]
        teplota = -45 + (175 * raw_temp / 65535.0)
        raw_humidity = (data[3] << 8) + data[4]
        vlhkost = 100 * raw_humidity / 65535.0
        return teplota, vlhkost

print(f"Starting SHT3x sensor logging to bucket '{bucket}'...")
print("-------------------------------------------------------")

try:
    while True:
        temp, hum = cti_sht3x()
        print(f"Measured -> Temperature: {temp:.2f} °C | Humidity: {hum:.2f} %")
        bod = Point("klima_senzory") \
            .tag("umisteni", "raspberry_pi") \
            .field("teplota", temp) \
            .field("vlhkost", hum)
        try:
            write_api.write(bucket=bucket, org=org, record=bod)
            print("   Data successfully sent to InfluxDB")
        except Exception as e_db:
            print(f"   Failed to write to DB: {e_db}")
        print("-" * 55)
        time.sleep(2)
except KeyboardInterrupt:
    print("\nMeasurement stopped by user.")
except Exception as e:
    print(f"\nUnexpected error occurred: {e}")
finally:
    write_client.close()
    print("InfluxDB connection closed.")
```

Save and close the file using `Ctrl + O`, `Enter` and `Ctrl + X`.

Then in the terminal create the service file with the command:

```bash
sudo nano /etc/systemd/system/meteologger.service
```

And paste the following configuration into it:

```ini
[Unit]
Description=Meteostanice - Logování ze senzoru SHT3x
After=network.target influxdb.service

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/muj_dashboard
ExecStart=/usr/bin/python3 /home/pi/muj_dashboard/logger.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Save and close the file the same way as before.

Now repeat the same process for the second service file:

```bash
sudo nano /etc/systemd/system/meteoweb.service
```

And paste the following configuration into it:

```ini
[Unit]
Description=Meteostanice - Flask Web Dashboard
After=network.target influxdb.service

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/muj_dashboard
ExecStart=/usr/bin/python3 /home/pi/muj_dashboard/app.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Save and close the file.

Now the final step — enter the following into the terminal:

```bash
sudo systemctl daemon-reload
sudo systemctl enable meteologger.service
sudo systemctl enable meteoweb.service
sudo systemctl start meteologger.service
sudo systemctl start meteoweb.service
```

That's all, good luck.
