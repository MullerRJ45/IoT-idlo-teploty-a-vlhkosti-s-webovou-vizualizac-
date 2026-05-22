# IoT Temperature and Humidity Sensor with Web Visualization

## 1. Installing Raspberry Pi OS
First, you'll need to prepare an SD card or another supported storage medium. Then download the Raspberry Pi Imager app from the official website:
[Official Raspberry Pi Website](https://www.raspberrypi.com/software/)
Once you open the program, select your Raspberry Pi model, the operating system version, and the storage medium you want to write the system to. You can also preconfigure things like your username, password, or Wi-Fi connection ahead of time.

Once everything is set, start writing the system to the medium. When it's done, insert the SD card into the Raspberry Pi and power it on. From there, just follow the initial setup of Raspberry Pi OS.

After the system boots and the desktop appears, it's a good idea to update everything first:

```bash
sudo apt update && sudo apt full-upgrade -y
```

You'll also want to enable SSH so you can manage the Raspberry Pi remotely:
```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

## 2. Connecting the Temperature and Humidity Sensor to GPIO

First, you need to enable the I2C bus in the Raspberry Pi settings using this command:
```bash
sudo raspi-config nonint do_i2c 0
```

I used the TH3 temperature and humidity sensor, which contains a Sensirion SHT31 smart sensor. It communicates via a multi-core copper cable with a silicone jacket, using the I2C bus.

The Raspberry Pi sends a command to the sensor, and the sensor responds with data that looks like this:
```
66 7A 93 5C A1 7F
```

The I2C bus only uses 2 wires: SDA and SCL.

SCL controls the timing `|‾|`, generating high/low pulses that determine when SDA should be read. SDA carries the actual bits (0/1). On each SCL clock pulse, SDA holds either a 0 or 1, and the device either pulls it LOW or leaves it HIGH. This is how the Raspberry Pi reads and processes data from the sensor over the I2C bus.

The GPIO wiring uses the following pins:

- 1 — 3.3V
- 3 — gpio2 - SDA
- 5 — gpio3 - SCL
- 9 — GND
- 17 — 3v3
- 25 — GND
- 27 — DNC
- 28 — DNC
![Wiring](image1.png)
I found the sensor's address using:
```bash
sudo i2cdetect -y 1
```

I used Gemini to generate the code and i just gave it the correct wiring and the sensor address (0x44). The code confirmed that the sensor and Raspberry Pi are communicating correctly.

If you're getting an error, doublecheck the sensor address and try again.

## 3. Data Collection

Working Python code for reading sensor data every 10 seconds:

```python
import time
from smbus2 import SMBus

I2C_ADDRESS = 0x44

def read_sht3x():
    with SMBus(1) as bus:
        bus.write_i2c_block_data(I2C_ADDRESS, 0x2C, [0x06])
        time.sleep(0.05)
        data = bus.read_i2c_block_data(I2C_ADDRESS, 0x00, 6)
        raw_temp = (data[0] << 8) + data[1]
        temperature = -45 + (175 * raw_temp / 65535.0)
        raw_humidity = (data[3] << 8) + data[4]
        humidity = 100 * raw_humidity / 65535.0
        return temperature, humidity

print("Starting SHT3x sensor readings...")
print("---------------------------------")

try:
    while True:
        temp, hum = read_sht3x()
        print(f"Temperature: {temp:.2f} °C | Humidity: {hum:.2f} %")
        time.sleep(10)
except KeyboardInterrupt:
    print("\nMeasurement stopped by user.")
except Exception as e:
    print(f"\nCommunication error: {e}")
```

## 4. Database

Out of InfluxDB and PostgreSQL, I went with InfluxDB.

I successfully connected InfluxDB to the Raspberry Pi and the sensor data is readable in it. Installation was done with the following commands:

```bash
curl -fsSL https://repos.influxdata.com/influxdata-archive_compat.key | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg

echo "deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg] https://repos.influxdata.com/debian stable main" | sudo tee /etc/apt/sources.list.d/influxdata.list

sudo apt update && sudo apt install influxdb2 -y
```

You'll also need the following libraries:

```bash
pip install influxdb-client
pip install Flask
pip install smbus2 raspi-sht31
```

Proposed data structure:

| InfluxDB Term | Name in Code | Meaning | Reason |
|---|---|---|---|
| Bucket | humidity_temperature | Project name | Isolated space for all sensor data |
| Measurement | climate_sensors | Like a SQL table name | Groups all climate data from the sensor |
| Tag | Location = Raspberry Pi | Descriptive info | InfluxDB uses tags to filter data |
| Field | Temperature | 22.5 decimal number | Used to draw graphs based on value |
| Field | Humidity | 45.2 decimal number | Used to draw graphs based on value |
| Timestamp | Auto-generated | time | Exact time when the measurement occurred |

Start InfluxDB with `sudo systemctl start influxdb` and verify it's running with `sudo systemctl status influxdb`. If the service is active, log into the web interface at `http://localhost:8086` that's for when you're accessing it directly from the Raspberry Pi. If you're accessing it from another device, make sure both are on the same network. You can find the Raspberry Pi's IP address by running `hostname -I` on it, then enter `http://(YourIP):8086` in your browser.

Then, in the terminal, start Python with `python3` and paste in the following code:

```python
import time
from smbus2 import SMBus
import influxdb_client
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS

token = "your influxdb token"
org = "your org"
url = "http://127.0.0.1:8086"
bucket = "humidity_temperature"

write_client = InfluxDBClient(url=url, token=token, org=org)
write_api = write_client.write_api(write_options=SYNCHRONOUS)

I2C_ADDRESS = 0x44

def read_sht3x():
    with SMBus(1) as bus:
        bus.write_i2c_block_data(I2C_ADDRESS, 0x2C, [0x06])
        time.sleep(0.05)
        data = bus.read_i2c_block_data(I2C_ADDRESS, 0x00, 6)
        raw_temp = (data[0] << 8) + data[1]
        temperature = -45 + (175 * raw_temp / 65535.0)
        raw_humidity = (data[3] << 8) + data[4]
        humidity = 100 * raw_humidity / 65535.0
        return temperature, humidity

print(f"Starting SHT3x sensor logging to bucket '{bucket}'...")
print("-------------------------------------------------------")

try:
    while True:
        temp, hum = read_sht3x()
        print(f"Measured -> Temperature: {temp:.2f} °C | Humidity: {hum:.2f} %")
        
        point = Point("climate_sensors") \
            .tag("location", "raspberry_pi") \
            .field("temperature", temp) \
            .field("humidity", hum)
        
        try:
            write_api.write(bucket=bucket, org=org, record=point)
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
Screenshot of the InfluxDB graph:

![InfluxDB graph screenshot](image2.png)

## 5. Web Application and Visualization

I saved the web app's backend in a folder called `my_dashboard` and the frontend in a subfolder called `templates`.

For the web app to work, you need the InfluxDB script running in the background. Then open a new terminal and run:

```bash
cd ~/my_dashboard
python3 app.py
```

The terminal will show the address where the page is running. Once you open it, it should look like this:
![](image3.png)

Here's the frontend code:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Weather Station</title>
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
    <h1>Raspberry Pi Weather Station</h1>
    <div class="dashboard">
        <div class="card">
            <h2>Temperature</h2>
            <div class="value"><span id="temp">--</span><span class="unit"> °C</span></div>
        </div>
        <div class="card">
            <h2>Humidity</h2>
            <div class="value"><span id="hum">--</span><span class="unit"> %</span></div>
        </div>
    </div>
    <div class="chart-container">
        <canvas id="historyChart"></canvas>
    </div>
    <div class="status">Last updated: <span id="time">--:--:--</span></div>
    <script>
        let myChart;

        function createChart(times, temperatures, humidities) {
            const ctx = document.getElementById('historyChart').getContext('2d');
            myChart = new Chart(ctx, {
                type: 'line',
                data: {
                    labels: times,
                    datasets: [
                        { label: 'Temperature (°C)', data: temperatures, borderColor: '#ff6384', backgroundColor: 'rgba(255, 99, 132, 0.1)', yAxisID: 'yTemp', tension: 0.2, pointRadius: 1 },
                        { label: 'Humidity (%)', data: humidities, borderColor: '#36a2eb', backgroundColor: 'rgba(54, 162, 235, 0.1)', yAxisID: 'yHum', tension: 0.2, pointRadius: 1 }
                    ]
                },
                options: {
                    responsive: true,
                    animation: false,
                    scales: {
                        yTemp: { type: 'linear', position: 'left', title: { display: true, text: 'Temperature (°C)', color: '#ff6384' } },
                        yHum: { type: 'linear', position: 'right', title: { display: true, text: 'Humidity (%)', color: '#36a2eb' }, grid: { drawOnChartArea: false } }
                    }
                }
            });
        }

        async function updateCurrentData() {
            try {
                const response = await fetch('/api/data');
                const data = await response.json();
                document.getElementById('temp').innerText = data.temperature;
                document.getElementById('hum').innerText = data.humidity;
                document.getElementById('time').innerText = new Date().toLocaleTimeString('en-US');
            } catch (e) { console.error(e); }
        }

        async function updateChart() {
            try {
                const response = await fetch('/api/history');
                const data = await response.json();
                if (!myChart) {
                    createChart(data.times, data.temperatures, data.humidities);
                } else {
                    myChart.data.labels = data.times;
                    myChart.data.datasets[0].data = data.temperatures;
                    myChart.data.datasets[1].data = data.humidities;
                    myChart.update();
                }
            } catch (e) { console.error(e); }
        }

        updateCurrentData();
        updateChart();

        setInterval(updateCurrentData, 3000);
        setInterval(updateChart, 10000);
    </script>
</body>
</html>
```

And here's the backend code:

```python
from flask import Flask, render_template, jsonify
from influxdb_client import InfluxDBClient

app = Flask(__name__)

TOKEN = "PASTE YOUR INFLUXDB TOKEN HERE"
ORG = "YOUR ORG"
URL = "http://127.0.0.1:8086"
BUCKET = "humidity_temperature"

client = InfluxDBClient(url=URL, token=TOKEN, org=ORG)
query_api = client.query_api()

def get_current_data():
    query = f'''
    from(bucket: "{BUCKET}")
      |> range(start: -10m)
      |> filter(fn: (r) => r["_measurement"] == "climate_sensors")
      |> last()
    '''
    try:
        tables = query_api.query(query, org=ORG)
        data = {"temperature": "--", "humidity": "--"}
        for table in tables:
            for record in table.records:
                if record.get_field() == "temperature":
                    data["temperature"] = round(record.get_value(), 2)
                elif record.get_field() == "humidity":
                    data["humidity"] = round(record.get_value(), 2)
        return data
    except Exception:
        return {"temperature": "Error", "humidity": "Error"}

def get_history():
    query = f'''
    from(bucket: "{BUCKET}")
      |> range(start: -24h)
      |> filter(fn: (r) => r["_measurement"] == "climate_sensors")
      |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
      |> sort(columns: ["_time"])
      |> tail(n: 100)
    '''
    try:
        tables = query_api.query(query, org=ORG)
        history = {"times": [], "temperatures": [], "humidities": []}
        for table in tables:
            for record in table.records:
                time = record.get_time().strftime('%H:%M:%S')
                t = record.values.get("temperature")
                h = record.values.get("humidity")
                history["times"].append(time)
                history["temperatures"].append(round(t, 1) if t is not None else None)
                history["humidities"].append(round(h, 1) if h is not None else None)
        return history
    except Exception as e:
        print(f"Backend error: {e}")
        return {"times": [], "temperatures": [], "humidities": []}

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/api/data')
def api_data():
    return jsonify(get_current_data())

@app.route('/api/history')
def api_history():
    return jsonify(get_history())

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

## 6. Automation — Autostarting Apps on Boot

In the terminal, open the `logger.py` file with:

```bash
nano ~/my_dashboard/logger.py
```

And paste in the following code:

```python
import time
from smbus2 import SMBus
import influxdb_client
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS

token = "YOUR TOKEN HERE"
org = "your org"
url = "http://127.0.0.1:8086"
bucket = "humidity_temperature"

write_client = InfluxDBClient(url=url, token=token, org=org)
write_api = write_client.write_api(write_options=SYNCHRONOUS)

I2C_ADDRESS = 0x44

def read_sht3x():
    with SMBus(1) as bus:
        bus.write_i2c_block_data(I2C_ADDRESS, 0x2C, [0x06])
        time.sleep(0.05)
        data = bus.read_i2c_block_data(I2C_ADDRESS, 0x00, 6)
        raw_temp = (data[0] << 8) + data[1]
        temperature = -45 + (175 * raw_temp / 65535.0)
        raw_humidity = (data[3] << 8) + data[4]
        humidity = 100 * raw_humidity / 65535.0
        return temperature, humidity

print(f"Starting SHT3x sensor logging to bucket '{bucket}'...")
print("-------------------------------------------------------")

try:
    while True:
        temp, hum = read_sht3x()
        print(f"Measured -> Temperature: {temp:.2f} °C | Humidity: {hum:.2f} %")
        point = Point("climate_sensors") \
            .tag("location", "raspberry_pi") \
            .field("temperature", temp) \
            .field("humidity", hum)
        try:
            write_api.write(bucket=bucket, org=org, record=point)
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

Save and close the file with `Ctrl + O`, `Enter`, and `Ctrl + X`.

Now create the service file in the terminal:

```bash
sudo nano /etc/systemd/system/meteologger.service
```

And paste in the following configuration:

```ini
[Unit]
Description=Weather Station - SHT3x Sensor Logger
After=network.target influxdb.service

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/my_dashboard
ExecStart=/usr/bin/python3 /home/pi/my_dashboard/logger.py
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

And paste in this configuration:

```ini
[Unit]
Description=Weather Station - Flask Web Dashboard
After=network.target influxdb.service

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/my_dashboard
ExecStart=/usr/bin/python3 /home/pi/my_dashboard/app.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Save and close the file.

Now the last step — run the following commands in the terminal:

```bash
sudo systemctl daemon-reload
sudo systemctl enable meteologger.service
sudo systemctl enable meteoweb.service
sudo systemctl start meteologger.service
sudo systemctl start meteoweb.service
```

That's it, good luck!
