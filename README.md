# crypto_signal_tool.html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Crypto Signal Tool</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
            background-color: #f4f4f9;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
            background: #fff;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
        }
        h1 {
            text-align: center;
            color: #333;
        }
        #chart-container {
            width: 100%;
            height: 400px;
            margin-top: 20px;
        }
        #signals {
            margin-top: 20px;
        }
        .signal {
            padding: 10px;
            margin-bottom: 5px;
            border-radius: 5px;
            font-weight: bold;
        }
        .buy {
            background-color: #d4edda;
            color: #155724;
        }
        .sell {
            background-color: #f8d7da;
            color: #721c24;
        }
        .input-group {
            margin-bottom: 20px;
        }
        .input-group input {
            width: 200px;
            padding: 10px;
            font-size: 16px;
            border: 1px solid #ccc;
            border-radius: 5px;
        }
        .input-group button {
            padding: 10px 20px;
            font-size: 16px;
            background-color: #007bff;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }
        .input-group button:hover {
            background-color: #0056b3;
        }
    </style>
</head>
<body>

<div class="container">
    <h1>Crypto Signal Tool</h1>

    <!-- Input for Cryptocurrency Symbol -->
    <div class="input-group">
        <input type="text" id="cryptoSymbol" placeholder="Enter Crypto Symbol (e.g., BTCUSDT)">
        <button onclick="fetchAndDisplayData()">Get Signals</button>
    </div>

    <div id="chart-container">
        <canvas id="priceChart"></canvas>
    </div>

    <div id="signals">
        <h2>Signals</h2>
    </div>
</div>

<script>
    // Fetch cryptocurrency data from Binance API
    async function getCryptoData(symbol = "BTCUSDT", interval = "1h", limit = 100) {
        const url = `https://api.binance.com/api/v3/klines?symbol=${symbol}&interval=${interval}&limit=${limit}`;
        const response = await fetch(url);

        if (!response.ok) {
            alert(`Error fetching data for ${symbol}. Please check the symbol and try again.`);
            return [];
        }

        const data = await response.json();

        // Convert the data into a more usable format
        return data.map(candle => ({
            timestamp: new Date(candle[0]),
            open: parseFloat(candle[1]),
            high: parseFloat(candle[2]),
            low: parseFloat(candle[3]),
            close: parseFloat(candle[4]),
            volume: parseFloat(candle[5])
        }));
    }

    // Generate buy/sell signals using Moving Average Crossover
    function generateSignals(data, shortWindow = 20, longWindow = 50) {
        const closes = data.map(candle => candle.close);

        // Calculate moving averages
        const shortMA = calculateSMA(closes, shortWindow);
        const longMA = calculateSMA(closes, longWindow);

        const signals = [];
        for (let i = longWindow; i < data.length; i++) {
            if (shortMA[i] > longMA[i] && shortMA[i - 1] <= longMA[i - 1]) {
                signals.push({ time: data[i].timestamp, price: data[i].close, signal: 'Buy' });
            } else if (shortMA[i] < longMA[i] && shortMA[i - 1] >= longMA[i - 1]) {
                signals.push({ time: data[i].timestamp, price: data[i].close, signal: 'Sell' });
            }
        }

        return signals;
    }

    // Simple Moving Average (SMA) calculation
    function calculateSMA(prices, window) {
        const sma = [];
        for (let i = 0; i < prices.length; i++) {
            if (i < window - 1) {
                sma.push(null);
            } else {
                const avg = prices.slice(i - window + 1, i + 1).reduce((a, b) => a + b, 0) / window;
                sma.push(avg);
            }
        }
        return sma;
    }

    // Display signals in the UI
    function displaySignals(signals) {
        const signalsDiv = document.getElementById('signals');
        signalsDiv.innerHTML = '<h2>Signals</h2>'; // Clear previous signals

        if (signals.length === 0) {
            signalsDiv.innerHTML += '<p>No signals generated for this period.</p>';
            return;
        }

        signals.forEach(signal => {
            const signalDiv = document.createElement('div');
            signalDiv.className = `signal ${signal.signal.toLowerCase()}`;
            signalDiv.innerHTML = `Time: ${signal.time.toLocaleString()} | Price: ${signal.price.toFixed(2)} | Signal: ${signal.signal}`;
            signalsDiv.appendChild(signalDiv);
        });
    }

    // Plot the price chart with Chart.js
    function plotChart(data, shortMA, longMA) {
        const ctx = document.getElementById('priceChart').getContext('2d');

        const timestamps = data.map(candle => candle.timestamp);
        const closes = data.map(candle => candle.close);

        // Destroy previous chart instance if it exists
        if (window.chartInstance) {
            window.chartInstance.destroy();
        }

        window.chartInstance = new Chart(ctx, {
            type: 'line',
            data: {
                labels: timestamps,
                datasets: [
                    {
                        label: 'Price',
                        data: closes,
                        borderColor: 'blue',
                        borderWidth: 2,
                        fill: false
                    },
                    {
                        label: 'Short MA (20)',
                        data: shortMA,
                        borderColor: 'green',
                        borderWidth: 2,
                        fill: false
                    },
                    {
                        label: 'Long MA (50)',
                        data: longMA,
                        borderColor: 'red',
                        borderWidth: 2,
                        fill: false
                    }
                ]
            },
            options: {
                responsive: true,
                scales: {
                    x: {
                        type: 'time',
                        time: {
                            unit: 'hour'
                        }
                    },
                    y: {
                        beginAtZero: false
                    }
                }
            }
        });
    }

    // Main function to fetch data, generate signals, and display them
    async function fetchAndDisplayData() {
        const symbolInput = document.getElementById('cryptoSymbol').value.toUpperCase();
        if (!symbolInput) {
            alert("Please enter a cryptocurrency symbol.");
            return;
        }

        const symbol = symbolInput;
        const interval = "1h";
        const limit = 100;

        const data = await getCryptoData(symbol, interval, limit);

        if (data.length === 0) return;

        const shortMA = calculateSMA(data.map(candle => candle.close), 20);
        const longMA = calculateSMA(data.map(candle => candle.close), 50);

        const signals = generateSignals(data, 20, 50);

        // Display signals
        displaySignals(signals);

        // Plot the chart
        plotChart(data, shortMA, longMA);
    }
</script>

</body>
</html>
