<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Simulador Educativo de Circuito RL Serie</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
            background-color: #f4f4f4;
            color: #333;
        }
        header {
            text-align: center;
            margin-bottom: 20px;
        }
        section {
            background-color: white;
            padding: 20px;
            margin-bottom: 20px;
            border-radius: 8px;
            box-shadow: 0 0 10px rgba(0,0,0,0.1);
        }
        form {
            display: grid;
            gap: 10px;
        }
        label {
            font-weight: bold;
        }
        input {
            padding: 8px;
            border: 1px solid #ccc;
            border-radius: 4px;
        }
        button {
            padding: 10px;
            background-color: #007bff;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        button:hover {
            background-color: #0056b3;
        }
        #reset-button {
            background-color: #dc3545;
            margin-top: 10px;
        }
        #reset-button:hover {
            background-color: #c82333;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 10px;
        }
        th, td {
            border: 1px solid #ddd;
            padding: 8px;
            text-align: center;
        }
        th {
            background-color: #f2f2f2;
        }
        #graph {
            max-width: 800px;
            margin: 0 auto;
        }
        .error {
            color: red;
            font-weight: bold;
        }
        .resistor-colors {
            display: flex;
            align-items: center;
            gap: 5px;
        }
        .color-band {
            width: 20px;
            height: 40px;
            border: 1px solid #000;
        }
        @media (max-width: 600px) {
            form {
                grid-template-columns: 1fr;
            }
        }
    </style>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body>
    <header>
        <h1>Desarrollo de aplicación web educativa en HTML5, CSS y JavaScript para el modelamiento y análisis del circuito eléctrico RL serie en régimen transitorio, mediante una ecuación diferencial de primer orden homogénea</h1>
    </header>
    <section id="input-section">
        <h2>Ingreso de Datos</h2>
        <form id="rl-form">
            <label for="R">Resistencia R (ohm o kΩ, ej: 1k para 1000 ohm):</label>
            <input type="text" id="R" required>
            <label for="L">Inductancia L (henry, mH o µH, ej: 1mH):</label>
            <input type="text" id="L" required>
            <label for="I0">Corriente inicial I₀ (A):</label>
            <input type="number" id="I0" step="any" required>
            <label for="t_max">Tiempo máximo t_max (s):</label>
            <input type="number" id="t_max" step="any" required>
            <label for="dt">Paso de tiempo Δt (s):</label>
            <input type="number" id="dt" step="any" required>
            <label for="I_objetivo">Corriente objetivo I_objetivo (A, opcional):</label>
            <input type="number" id="I_objetivo" step="any">
            <label for="t_medido">Tiempo de medición t_medido (s, opcional):</label>
            <input type="number" id="t_medido" step="any">
            <label for="I_medido">Corriente medida I_medido (A, opcional):</label>
            <input type="number" id="I_medido" step="any">
            <button type="submit">Calcular</button>
            <button type="button" id="reset-button">Limpiar</button>
        </form>
    </section>
    <section id="results-section" style="display: none;">
        <h2>Resumen de Datos de Entrada</h2>
        <div id="input-summary"></div>
        <h2>Detalles de Componentes</h2>
        <div id="component-details"></div>
        <h2>Constante de Tiempo</h2>
        <div id="tau-result"></div>
        <h2>Función de Corriente I(t)</h2>
        <div id="function-result"></div>
        <h2>Resultados Numéricos Clave</h2>
        <div id="key-results"></div>
        <h2>Tabla de Valores</h2>
        <table id="values-table">
            <thead>
                <tr>
                    <th>t (s)</th>
                    <th>I(t) (A)</th>
                    <th>v_R(t) (V)</th>
                    <th>v_L(t) (V)</th>
                </tr>
            </thead>
            <tbody></tbody>
        </table>
        <h2>Representación Gráfica</h2>
        <canvas id="graph"></canvas>
        <h2>Interpretación del Comportamiento</h2>
        <div id="interpretation"></div>
    </section>
    <script>
        const form = document.getElementById('rl-form');
        const resultsSection = document.getElementById('results-section');
        const resetButton = document.getElementById('reset-button');
        let chart;
        form.addEventListener('submit', (e) => {
            e.preventDefault();
            const errors = validateInputs();
            if (errors.length > 0) {
                alert(errors.join('\n'));
                return;
            }
            performCalculations();
            resultsSection.style.display = 'block';
        });
        resetButton.addEventListener('click', () => {
            form.reset();
            resultsSection.style.display = 'none';
            if (chart) chart.destroy();
            document.getElementById('input-summary').innerHTML = '';
            document.getElementById('component-details').innerHTML = '';
            document.getElementById('tau-result').innerHTML = '';
            document.getElementById('function-result').innerHTML = '';
            document.getElementById('key-results').innerHTML = '';
            document.querySelector('#values-table tbody').innerHTML = '';
            document.getElementById('interpretation').innerHTML = '';
        });
        function parseR(value) {
            value = value.trim().toLowerCase();
            if (value.endsWith('k') || value.endsWith('kω') || value.endsWith('kohm')) {
                return parseFloat(value.slice(0, -1)) * 1000;
            }
            return parseFloat(value);
        }
        function parseL(value) {
            value = value.trim().toLowerCase();
            if (value.endsWith('mh')) {
                return parseFloat(value.slice(0, -2)) * 0.001;
            } else if (value.endsWith('µh') || value.endsWith('uh')) {
                return parseFloat(value.slice(0, -2)) * 1e-6;
            } else if (value.endsWith('h')) {
                return parseFloat(value.slice(0, -1));
            }
            return parseFloat(value);
        }
        function validateInputs() {
            const errors = [];
            const R = parseR(document.getElementById('R').value);
            const L = parseL(document.getElementById('L').value);
            const I0 = parseFloat(document.getElementById('I0').value);
            const t_max = parseFloat(document.getElementById('t_max').value);
            const dt = parseFloat(document.getElementById('dt').value);
            const I_objetivo = parseFloat(document.getElementById('I_objetivo').value) || null;
            const t_medido = parseFloat(document.getElementById('t_medido').value) || null;
            const I_medido = parseFloat(document.getElementById('I_medido').value) || null;
            if (isNaN(R) || R <= 0) errors.push('Resistencia R debe ser mayor que 0.');
            if (isNaN(L) || L <= 0) errors.push('Inductancia L debe ser mayor que 0.');
            if (isNaN(I0)) errors.push('Corriente inicial I₀ es requerida.');
            if (isNaN(t_max) || t_max <= 0) errors.push('Tiempo máximo t_max debe ser mayor que 0.');
            if (isNaN(dt) || dt <= 0) errors.push('Paso de tiempo Δt debe ser mayor que 0.');
            if (I_objetivo !== null && (isNaN(I_objetivo) || I_objetivo > I0 || I_objetivo < 0)) errors.push('Corriente objetivo I_objetivo debe ser entre 0 y I₀.');
            if (t_medido !== null && isNaN(t_medido) || t_medido < 0) errors.push('Tiempo de medición t_medido debe ser positivo.');
            if (I_medido !== null && isNaN(I_medido)) errors.push('Corriente medida I_medido debe ser un número.');
            if (dt > t_max) errors.push('Δt no puede ser mayor que t_max.');
            return errors;
        }
        function getResistorColors(ohms) {
            const colorCodes = {
                0: 'black', 1: 'brown', 2: 'red', 3: 'orange', 4: 'yellow',
                5: 'green', 6: 'blue', 7: 'violet', 8: 'grey', 9: 'white'
            };
            const multipliers = {
                0: 'black', 1: 'brown', 2: 'red', 3: 'orange', 4: 'yellow',
                5: 'green', 6: 'blue', 7: 'violet', 8: 'grey', 9: 'white',
                10: 'gold', 11: 'silver'
            };
            let value = ohms.toString().replace('.', '');
            let digits = value.length;
            let first = parseInt(value[0]);
            let second = digits > 1 ? parseInt(value[1]) : 0;
            let multiplier = digits - 2;
            if (multiplier < 0) multiplier = 10 + Math.abs(multiplier); // gold/silver for <1
            const bands = [colorCodes[first], colorCodes[second], multipliers[multiplier]];
            return bands;
        }
        function estimateInductorParams(L) {
            // Estimaciones simples para un solenoide de aire: L ≈ μ0 * N² * π * (d/2)^2 / h, asumimos h = d para simplificar
            // Asumimos diámetro d=1cm=0.01m, grosor cable AWG 20 (diam 0.81mm), calculamos N aproximado
            const mu0 = 4 * Math.PI * 1e-7;
            const d = 0.01; // diámetro bobina en m
            const h = 0.01; // longitud bobina en m
            const A = Math.PI * (d / 2) ** 2;
            const N = Math.sqrt(L * h / (mu0 * A));
            const wireGauge = 20; // AWG ejemplo
            const wireDiameter = 0.00081; // m
            return { N: Math.round(N), wireGauge, wireDiameter: wireDiameter * 1000, coilDiameter: d * 1000 };
        }
        function performCalculations() {
            const R = parseR(document.getElementById('R').value);
            const L = parseL(document.getElementById('L').value);
            const I0 = parseFloat(document.getElementById('I0').value);
            const t_max = parseFloat(document.getElementById('t_max').value);
            const dt = parseFloat(document.getElementById('dt').value);
            const I_objetivo = parseFloat(document.getElementById('I_objetivo').value) || null;
            const t_medido = parseFloat(document.getElementById('t_medido').value) || null;
            const I_medido = parseFloat(document.getElementById('I_medido').value) || null;
            const tau = L / R;
            // Resumen de entradas
            let summary = `
                R = ${R} Ω<br>
                L = ${L} H<br>
                I₀ = ${I0} A<br>
                t_max = ${t_max} s<br>
                Δt = ${dt} s<br>
            `;
            if (I_objetivo !== null) summary += `I_objetivo = ${I_objetivo} A<br>`;
            if (t_medido !== null) summary += `t_medido = ${t_medido} s<br>`;
            if (I_medido !== null) summary += `I_medido = ${I_medido} A<br>`;
            document.getElementById('input-summary').innerHTML = summary;
            // Detalles de componentes
            const resistorColors = getResistorColors(R);
            let colorsHtml = '<div class="resistor-colors">';
            resistorColors.forEach(color => {
                colorsHtml += `<div class="color-band" style="background-color: ${color};"></div>`;
            });
            colorsHtml += '</div>';
            const inductorParams = estimateInductorParams(L);
            let componentDetails = `
                <b>Resistencia:</b> Código de colores (bandas): ${colorsHtml}<br>
                <b>Inductancia:</b> Estimación aproximada (para bobina de aire simple):<br>
                - Número de vueltas: ${inductorParams.N}<br>
                - Grosor del cable: AWG ${inductorParams.wireGauge} (diámetro ≈ ${inductorParams.wireDiameter} mm)<br>
                - Diámetro de la bobina: ${inductorParams.coilDiameter} mm
            `;
            document.getElementById('component-details').innerHTML = componentDetails;
            // Constante de tiempo
            document.getElementById('tau-result').innerHTML = `τ = ${tau.toFixed(6)} s<br>La constante de tiempo τ representa el tiempo característico en el que la corriente decae a aproximadamente el 37% de su valor inicial.`;
            // Función I(t)
            document.getElementById('function-result').innerHTML = `I(t) = ${I0} × e^(-t / ${tau.toFixed(6)}) A`;
            // Resultados clave
            const I_tmax = I0 * Math.exp(-t_max / tau);
            let keyResults = `
                I(0) = ${I0} A<br>
                I(t_max) = ${I_tmax.toFixed(6)} A<br>
            `;
            let t_objetivo = null;
            if (I_objetivo !== null) {
                t_objetivo = -tau * Math.log(I_objetivo / I0);
                keyResults += `Tiempo para alcanzar I_objetivo: t_objetivo = ${t_objetivo.toFixed(6)} s<br>`;
            }
            let error_rel = null;
            if (t_medido !== null) {
                const I_tmedido = I0 * Math.exp(-t_medido / tau);
                keyResults += `I(t_medido) = ${I_tmedido.toFixed(6)} A<br>`;
                if (I_medido !== null) {
                    error_rel = Math.abs((I_medido - I_tmedido) / I_tmedido) * 100;
                    keyResults += `Error relativo con dato experimental: ${error_rel.toFixed(2)}%<br>`;
                }
            }
            document.getElementById('key-results').innerHTML = keyResults;
            // Tabla de valores
            const tbody = document.querySelector('#values-table tbody');
            tbody.innerHTML = '';
            const times = [];
            const currents = [];
            const vRs = [];
            const vLs = [];
            for (let t = 0; t <= t_max; t += dt) {
                const i = I0 * Math.exp(-t / tau);
                const vR = R * i;
                const vL = -R * i; // vL = -L * di/dt = -R * i
                const row = document.createElement('tr');
                row.innerHTML = `<td>${t.toFixed(4)}</td><td>${i.toFixed(6)}</td><td>${vR.toFixed(6)}</td><td>${vL.toFixed(6)}</td>`;
                tbody.appendChild(row);
                times.push(t);
                currents.push(i);
                vRs.push(vR);
                vLs.push(vL);
            }
            // Añadir t_max si no está exactamente debido a floating point
            if (times[times.length - 1] < t_max) {
                const i = I0 * Math.exp(-t_max / tau);
                const vR = R * i;
                const vL = -R * i;
                const row = document.createElement('tr');
                row.innerHTML = `<td>${t_max.toFixed(4)}</td><td>${i.toFixed(6)}</td><td>${vR.toFixed(6)}</td><td>${vL.toFixed(6)}</td>`;
                tbody.appendChild(row);
                times.push(t_max);
                currents.push(i);
                vRs.push(vR);
                vLs.push(vL);
            }
            // Gráfica
            if (chart) chart.destroy();
            const ctx = document.getElementById('graph').getContext('2d');
            chart = new Chart(ctx, {
                type: 'line',
                data: {
                    labels: times,
                    datasets: [
                        {
                            label: 'I(t) (A)',
                            data: currents,
                            borderColor: 'blue',
                            fill: false
                        },
                        {
                            label: 'v_R(t) (V)',
                            data: vRs,
                            borderColor: 'green',
                            fill: false
                        },
                        {
                            label: 'v_L(t) (V)',
                            data: vLs,
                            borderColor: 'red',
                            fill: false
                        }
                    ]
                },
                options: {
                    scales: {
                        x: { title: { display: true, text: 'Tiempo (s)' } },
                        y: { title: { display: true, text: 'Valor' } }
                    }
                }
            });
            // Interpretación
            let interp = `La corriente decae exponencialmente desde ${I0} A hasta casi 0 A. La constante de tiempo τ = ${tau.toFixed(6)} s indica la rapidez del decaimiento: un τ menor (R mayor o L menor) implica un decaimiento más rápido. En aplicaciones reales, esto afecta el tiempo de conmutación en electrónicos de potencia, el paro de motores, etc.<br>El voltaje en la resistencia v_R(t) sigue la corriente, mientras que v_L(t) es opuesto y decae igualmente.`;
            if (error_rel !== null) interp += `<br>El error relativo con el dato experimental es ${error_rel.toFixed(2)}%, lo que valida el modelo.`;
            document.getElementById('interpretation').innerHTML = interp;
        }
    </script>
</body>
</html>
