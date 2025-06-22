<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>WasteLess - Food Expiry Tracker</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    /* Animated gradient background */
    @keyframes gradientBG {
      0% {background-position: 0% 50%;}
      50% {background-position: 100% 50%;}
      100% {background-position: 0% 50%;}
    }
    body {
      font-family: Arial, sans-serif;
      margin: 0;
      padding: 20px;
      background: linear-gradient(-45deg, #ffecd2, #fcb69f, #ff7e5f, #feb47b);
      background-size: 400% 400%;
      animation: gradientBG 15s ease infinite;
      color: #333;
      min-height: 100vh;
    }
    h1 {
      text-align: center;
      margin-bottom: 1rem;
      font-weight: 700;
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      color: #4a2c2a;
      text-shadow: 1px 1px 5px rgba(0,0,0,0.1);
    }
    .container {
      max-width: 720px;
      margin: 0 auto;
      background: rgba(255, 255, 255, 0.9);
      padding: 25px;
      border-radius: 12px;
      box-shadow: 0 8px 20px rgba(0,0,0,0.15);
    }
    input, button {
      padding: 10px;
      margin: 5px 8px 15px 0;
      font-size: 1rem;
      border-radius: 6px;
      border: 1px solid #ccc;
      transition: box-shadow 0.3s ease;
    }
    input:focus, button:focus {
      outline: none;
      box-shadow: 0 0 5px #ff7e5f;
      border-color: #ff7e5f;
    }
    input[type="number"] {
      width: 80px;
    }
    input[type="text"], input[type="date"] {
      width: 220px;
    }
    button {
      background-color: #ff7e5f;
      color: white;
      border: none;
      cursor: pointer;
    }
    button:hover {
      background-color: #f95e3d;
    }
    table {
      width: 100%;
      border-collapse: collapse;
      margin-top: 10px;
      font-size: 0.95rem;
    }
    th, td {
      padding: 12px;
      border: 1px solid #ddd;
      text-align: center;
      vertical-align: middle;
    }
    .near-expiry {
      background-color: #fff3cd;
    }
    .expired {
      background-color: #f8d7da;
      text-decoration: line-through;
      color: #721c24;
    }
    #summary {
      margin-top: 10px;
      font-weight: bold;
    }
    #filterInput {
      margin-bottom: 15px;
      width: 100%;
      padding: 10px;
      font-size: 1rem;
      border-radius: 6px;
      border: 1px solid #ccc;
    }
    .header-graphic {
      display: block;
      margin: 0 auto 15px auto;
      width: 80px;
      filter: drop-shadow(1px 1px 2px rgba(0,0,0,0.15));
    }
    .audio-control {
      margin-top: 10px;
      text-align: center;
    }
    .links {
      margin-top: 20px;
      text-align: center;
    }
    .links a {
      margin: 0 10px;
      color: #ff7e5f;
      text-decoration: none;
      font-weight: 600;
      border-bottom: 1px solid transparent;
      transition: border-color 0.3s ease;
    }
    .links a:hover {
      border-color: #ff7e5f;
    }
  </style>
</head>
<body>
  <div class="container" role="main">
    <img class="header-graphic" src="https://cdn-icons-png.flaticon.com/512/1046/1046784.png" alt="Food icon" />
    <h1>WasteLess - Food Expiry Tracker</h1>

    <input type="text" id="foodName" placeholder="Food Name 🍎" aria-label="Food Name" />
    <input type="number" id="quantity" placeholder="Quantity" min="1" aria-label="Quantity" />
    <input type="date" id="expiryDate" aria-label="Expiry Date" />
    <button onclick="addItem()" aria-label="Add food item">Add Item</button>

    <input type="text" id="filterInput" placeholder="Filter food items..." aria-label="Filter food items" oninput="renderTable()" />

    <table aria-live="polite" aria-label="Food expiry list">
      <thead>
        <tr>
          <th>Food</th><th>Qty</th><th>Expiry</th><th>Days Left</th><th>Action</th>
        </tr>
      </thead>
      <tbody id="foodTable"></tbody>
    </table>

    <div id="summary"></div>

    <canvas id="wasteChart" height="120" aria-label="Food status chart" role="img"></canvas>

    <div class="audio-control">
      <button id="audioBtn" aria-label="Play 'Food added' in Spanish">🔊 Play confirmation sound (Spanish)</button>
    </div>

    <div class="links" aria-label="Helpful links">
      <a href="https://www.foodwise.com.au/" target="_blank" rel="noopener noreferrer">Food Waste Tips</a>
      <a href="https://www.fao.org/food-loss-and-food-waste/en/" target="_blank" rel="noopener noreferrer">FAO Info</a>
      <a href="https://www.epa.gov/recycle/reducing-wasted-food-home" target="_blank" rel="noopener noreferrer">How to Store Food</a>
    </div>
  </div>

  <script>
    let foodList = JSON.parse(localStorage.getItem("foodList") || "[]");
    const table = document.getElementById("foodTable");
    const summary = document.getElementById("summary");
    const audioBtn = document.getElementById("audioBtn");
    let chart;

    // Load audio for alternative language confirmation (Spanish)
    const audio = new Audio('https://actions.google.com/sounds/v1/human_voices/positive_chime.ogg'); // example sound

    audioBtn.onclick = () => {
      audio.play();
    };

    function formatDate(dateStr) {
      const options = { year: 'numeric', month: 'short', day: 'numeric' };
      return new Date(dateStr).toLocaleDateString(undefined, options);
    }

    function daysLeft(expiry) {
      const today = new Date();
      const expiryDate = new Date(expiry);
      const diff = Math.ceil((expiryDate - today) / (1000 * 60 * 60 * 24));
      return diff;
    }

    function renderTable() {
      const filter = document.getElementById('filterInput').value.toLowerCase();
      table.innerHTML = "";

      // Sort by expiry date ascending
      foodList.sort((a, b) => new Date(a.expiry) - new Date(b.expiry));

      let totalQuantity = 0;
      let totalItems = foodList.length;

      foodList.forEach((item, index) => {
        if (!item.name.toLowerCase().includes(filter)) return;

        const diff = daysLeft(item.expiry);
        totalQuantity += Number(item.quantity);

        const row = document.createElement("tr");

        if (diff < 0) {
          row.classList.add("expired");
        } else if (diff <= 3) {
          row.classList.add("near-expiry");
        }

        // Add food emoji dynamically based on name (simple example)
        const emoji = item.name.toLowerCase().includes("apple") ? "🍎" :
                      item.name.toLowerCase().includes("banana") ? "🍌" :
                      item.name.toLowerCase().includes("milk") ? "🥛" :
                      item.name.toLowerCase().includes("bread") ? "🍞" :
                      "🍽️";

        row.innerHTML = `
          <td>${emoji} ${item.name}</td>
          <td>${item.quantity}</td>
          <td>${formatDate(item.expiry)}</td>
          <td>${diff < 0 ? "Expired" : diff}</td>
          <td><button onclick="removeItem(${index})" aria-label="Delete ${item.name}">Delete</button></td>
        `;
        table.appendChild(row);
      });

      summary.textContent = `Total Items: ${totalItems} | Total Quantity: ${totalQuantity}`;

      localStorage.setItem("foodList", JSON.stringify(foodList));
      drawChart();
    }

    function addItem() {
      const nameInput = document.getElementById("foodName");
      const quantityInput = document.getElementById("quantity");
      const expiryInput = document.getElementById("expiryDate");

      const name = nameInput.value.trim();
      const quantity = Number(quantityInput.value);
      const expiry = expiryInput.value;

      if (!name || !quantity || !expiry) {
        alert("Please fill all fields.");
        return;
      }
      if (quantity <= 0 || !Number.isInteger(quantity)) {
        alert("Quantity must be a positive integer.");
        return;
      }
      if (new Date(expiry) < new Date(new Date().toDateString())) {
        alert("Expiry date cannot be in the past.");
        return;
      }

      foodList.push({ name, quantity, expiry });

      nameInput.value = "";
      quantityInput.value = "";
      expiryInput.value = "";

      renderTable();

      // Play audio feedback on add
      audio.play();
    }

    function removeItem(index) {
      if (index >= 0 && index < foodList.length) {
        foodList.splice(index, 1);
        renderTable();
      }
    }

    function drawChart() {
      const ctx = document.getElementById("wasteChart").getContext("2d");

      const expired = foodList.filter(item => new Date(item.expiry) < new Date()).reduce((sum, i) => sum + Number(i.quantity), 0);
      const saved = foodList.reduce((sum, i) => sum + Number(i.quantity), 0) - expired;

      if (chart) chart.destroy();

      chart = new Chart(ctx, {
        type: 'doughnut',
        data: {
          labels: ["Saved", "Expired"],
          datasets: [{
            label: 'Food Status',
            data: [saved, expired],
            backgroundColor: ["#28a745", "#dc3545"]
          }]
        },
        options: {
          responsive: true,
          plugins: {
            legend: {
              position: 'bottom',
            },
          }
        }
      });
    }

    // Initial render
    renderTable();
  </script>
</body>
</html>
