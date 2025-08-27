# trail01
This is an initial repo before the dry run of our hackathon project. All the changes made/ are about to be made are visible to restricted members only.
from flask import Flask, jsonify, request, render_template_string
from flask_cors import CORS
import datetime
import requests
app = Flask(__name__)
CORS(app)

# Store latest sensor readings
sensor_data = {
    "Power_W": 0.0,
    "CO_ppm": 0.0,
    "CO2_ppm": 0.0,
    "Ethanol_ppm": 0.0,
    "NH3_ppm": 0.0,
    "Toluene_ppm": 0.0,
    "Acetone_ppm": 0.0,
    "bin_fill_percent": 0.0
}

# Eco-score leaderboard (static)
eco_score = {
    "Charlie": 5,
    "Alice": 10,
    "Bob": 15
}

@app.route('/')
def dashboard():
    return render_template_string('''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Smart Green Campus Dashboard</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body { font-family: Arial; background: #f1f8f2; padding: 20px; }
        .container { display: flex; flex-wrap: wrap; gap: 20px; }
        .card { background: white; padding: 20px; border-radius: 10px; box-shadow: 0 0 10px #ccc; width: 400px; }
        .card h3 { margin-top: 0; }
        input { width: 60px; }
        button { padding: 5px 10px; margin-top: 10px; }
    </style>
</head>
<body>
    <h1>🌱 Smart Green Campus Dashboard</h1>

    <div class="container">
        <div class="card">
            <h3>Eco-Score Leaderboard</h3>
            <ul id="eco-leaderboard"></ul>
        </div>

        <div class="card">
            <h3>AI Suggestions</h3>
            <p>✅ Environment looks good!</p>
        </div>

        <div class="card">
            <h3>Predict CO2 Emissions</h3>
            <label>Energy Usage (kWh): <input type="number" id="energy" value="6"></label><br><br>
            <label>Temperature (°C): <input type="number" id="temperature" value="27"></label><br><br>
            <button onclick="predictCO2()">Predict CO2</button>
            <p><b>Predicted CO2 Emissions: </b><span id="predicted-co2">-</span> kg</p>
        </div>
    </div>

    <div class="container" style="margin-top: 40px;">
        <div class="card">
            <h3>Energy Usage (W)</h3>
            <canvas id="powerChart" width="400" height="200"></canvas>
        </div>
        <div class="card">
            <h3>CO2 Levels (ppm)</h3>
            <canvas id="co2Chart" width="400" height="200"></canvas>
        </div>
        <div class="card">
            <h3>Bin Fill (%)</h3>
            <canvas id="binChart" width="400" height="200"></canvas>
        </div>
        <div class="card">
            <h3>Carbon Footprint (kg CO2)</h3>
            <p id="carbon-footprint" style="font-size: 24px; color: green;">0.00</p>
        </div>
    </div>

<script>
let powerChart = new Chart(document.getElementById('powerChart'), {
    type: 'line',
    data: { labels: [], datasets: [{ label: 'Power (W)', data: [], borderColor: 'green', fill: false }] },
    options: { responsive: true, scales: { y: { beginAtZero: true } } }
});

let co2Chart = new Chart(document.getElementById('co2Chart'), {
    type: 'line',
    data: { labels: [], datasets: [{ label: 'CO2 (ppm)', data: [], borderColor: 'red', fill: false }] },
    options: { responsive: true, scales: { y: { beginAtZero: true } } }
});

let binChart = new Chart(document.getElementById('binChart'), {
    type: 'line',
    data: { labels: [], datasets: [{ label: 'Bin Fill (%)', data: [], borderColor: 'orange', fill: false }] },
    options: { responsive: true, scales: { y: { beginAtZero: true } } }
});

function updateDashboard() {
    fetch('/api/iot/readings')
        .then(res => res.json())
        .then(data => {
            let time = new Date().toLocaleTimeString();

            if (powerChart.data.labels.length > 20) {
                powerChart.data.labels.shift();
                co2Chart.data.labels.shift();
                binChart.data.labels.shift();
                powerChart.data.datasets[0].data.shift();
                co2Chart.data.datasets[0].data.shift();
                binChart.data.datasets[0].data.shift();
            }

            powerChart.data.labels.push(time);
            powerChart.data.datasets[0].data.push(data.Power_W);
            co2Chart.data.labels.push(time);
            co2Chart.data.datasets[0].data.push(data.CO2_ppm);
            binChart.data.labels.push(time);
            binChart.data.datasets[0].data.push(data.bin_fill_percent);

            powerChart.update();
            co2Chart.update();
            binChart.update();

            document.getElementById('carbon-footprint').innerText = (data.Power_W * 0.001).toFixed(2);
        });
}

// Leaderboard
fetch('/api/eco_score')
    .then(res => res.json())
    .then(data => {
        let ul = document.getElementById('eco-leaderboard');
        ul.innerHTML = '';
        Object.entries(data).forEach(([name, score]) => {
            let li = document.createElement('li');
            li.textContent = `${name}: ${score} pts`;
            ul.appendChild(li);
        });
    });

// CO2 Prediction
function predictCO2() {
    let energy = document.getElementById('energy').value;
    let temp = document.getElementById('temperature').value;

    fetch('/api/predict_co2', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({ energy, temperature: temp })
    }).then(res => res.json())
      .then(data => {
        document.getElementById('predicted-co2').innerText = data.predicted_co2 + ' kg';
    });
}

// Start dashboard updates
setInterval(updateDashboard, 2000);
updateDashboard();
</script>
</body>
</html>
''')

@app.route('/api/iot/readings', methods=['POST'])
def update_readings():
    global sensor_data
    data = request.get_json()
    sensor_data.update(data)
    print ("update sensor data ",sensor_data)
    return jsonify({"status": "success", "data": sensor_data})

@app.route('/api/iot/readings', methods=['GET'])
def get_readings():
    return jsonify(sensor_data)
payload = {"Power_W": 12.5, "CO2_ppm": 420, "bin_fill_percent": 25}
res = requests.post("http://127.0.0.1:5000/api/iot/readings", json=payload)
print(res.json())

@app.route('/api/eco_score', methods=['GET'])
def get_eco_score():
    return jsonify(eco_score)

@app.route('/api/predict_co2', methods=['POST'])
def predict_co2():
    data = request.get_json()
    energy = float(data.get("energy", 0))
    temp = float(data.get("temperature", 25))
    predicted = round((energy * 7.0 + temp * 0.5), 2)
    return jsonify({"predicted_co2": predicted})

if __name__ == '__main__':
    app.run(debug=False, use_reloader=False)

