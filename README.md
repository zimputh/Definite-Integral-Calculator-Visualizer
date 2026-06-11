<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Simple Integration Calculator</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.8.0/math.js"></script>
    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            margin: 0;
            padding: 0;
            display: flex;
            height: 100vh;
            background-color: #f8f9fa;
        }

        /* Sidebar Control Panel */
        #sidebar {
            width: 350px;
            background-color: #ffffff;
            box-shadow: 2px 0 10px rgba(0,0,0,0.1);
            padding: 24px;
            box-sizing: border-box;
            display: flex;
            flex-direction: column;
            gap: 20px;
            z-index: 10;
            overflow-y: auto;
        }

        h2 {
            margin-top: 0;
            color: #333;
            font-size: 1.5rem;
            border-bottom: 2px solid #eaeaea;
            padding-bottom: 10px;
        }

        .input-group {
            display: flex;
            flex-direction: column;
            gap: 6px;
        }

        label {
            font-weight: 600;
            font-size: 0.85rem;
            color: #555;
            text-transform: uppercase;
            letter-spacing: 0.5px;
        }

        input {
            padding: 10px 12px;
            border: 1px solid #ccc;
            border-radius: 6px;
            font-size: 1rem;
            outline: none;
            transition: border-color 0.2s;
        }

        input:focus {
            border-color: #0076ff;
        }

        .bounds-group {
            display: flex;
            gap: 12px;
        }

        .bounds-group .input-group {
            flex: 1;
        }

        /* Results Panel */
        #results {
            margin-top: 10px;
            padding: 16px;
            background-color: #f0f7ff;
            border-left: 4px solid #0076ff;
            border-radius: 4px;
        }

        .result-title {
            font-size: 0.85rem;
            color: #0056b3;
            font-weight: bold;
            margin-bottom: 4px;
        }

        .result-value {
            font-size: 1.6rem;
            font-weight: 700;
            color: #111;
            font-family: monospace;
        }

        .error-message {
            color: #dc3545;
            background-color: #fdf2f2;
            border-left: 4px solid #dc3545;
            padding: 12px;
            border-radius: 4px;
            font-size: 0.9rem;
            display: none;
        }

        /* Main Graph Area */
        #graph-container {
            flex: 1;
            position: relative;
            height: 100%;
            background-color: #fafafa;
        }

        canvas {
            display: block;
            width: 100%;
            height: 100%;
        }

        .help-text {
            font-size: 0.8rem;
            color: #777;
            line-height: 1.4;
        }
    </style>
</head>
<body>

    <div id="sidebar">
        <h2>Integration Visualizer</h2>
        
        <div class="input-group">
            <label for="fx">Function f(x)</label>
            <input type="text" id="fx" value="x^2 - 2x" placeholder="e.g., x^2 - 2*x">
        </div>

        <div class="bounds-group">
            <div class="input-group">
                <label for="bound-a">Lower Bound (a)</label>
                <input type="number" id="bound-a" value="-1" step="any">
            </div>
            <div class="input-group">
                <label for="bound-b">Upper Bound (b)</label>
                <input type="number" id="bound-b" value="3" step="any">
            </div>
        </div>

        <div id="error" class="error-message"></div>

        <div id="results">
            <div class="result-title">Definite Integral Value</div>
            <div id="integral-value" class="result-value">0.0000</div>
        </div>

        <div class="help-text">
            <strong>Supported expressions:</strong><br>
            Multiplication requires standard symbols (e.g., use <code>2*x</code> instead of <code>2x</code>). You can use functions like <code>sin(x)</code>, <code>cos(x)</code>, <code>tan(x)</code>, <code>ln(x)</code>, <code>exp(x)</code>, <code>sqrt(x)</code>, and constants like <code>pi</code> or <code>e</code>.
        </div>
    </div>

    <div id="graph-container">
        <canvas id="graphCanvas"></canvas>
    </div>

    <script>
        const canvas = document.getElementById('graphCanvas');
        const ctx = canvas.getContext('2d');
        
        // Input elements
        const fxInput = document.getElementById('fx');
        const boundAInput = document.getElementById('bound-a');
        const boundBInput = document.getElementById('bound-b');
        const integralValueDisplay = document.getElementById('integral-value');
        const errorDiv = document.getElementById('error');

        // Coordinate system state (Dynamic zoom/pan centered on bounds)
        let scaleX = 40; // pixels per unit
        let scaleY = 40; // pixels per unit
        let offsetX = 0; // pixel offset of origin from center
        let offsetY = 0; // pixel offset of origin from center

        // Handle canvas resizing
        function resizeCanvas() {
            canvas.width = canvas.parentElement.clientWidth;
            canvas.height = canvas.parentElement.clientHeight;
            updateGraph();
        }
        window.addEventListener('resize', resizeCanvas);

        // Core compilation & evaluation
        function getFunction(exprString) {
            try {
                const compiled = math.compile(exprString);
                return function(x) {
                    return compiled.evaluate({ x: x });
                };
            } catch (e) {
                throw new Error("Invalid expression syntax.");
            }
        }

        // Numerical Integration using Simpson's Rule
        function calculateIntegral(f, a, b, n = 1000) {
            if (a === b) return 0;
            // Ensure n is even for Simpson's rule
            if (n % 2 !== 0) n++;
            
            const h = (b - a) / n;
            let sum = f(a) + f(b);

            for (let i = 1; i < n; i++) {
                const x = a + i * h;
                if (i % 2 === 0) {
                    sum += 2 * f(x);
                } else {
                    sum += 4 * f(x);
                }
            }
            return (h / 3) * sum;
        }

        // Setup the graph viewport to frame the integration bounds nicely
        function autoFocusGrid(a, b) {
            const centerX = (a + b) / 2;
            scaleX = (canvas.width * 0.6) / Math.max(Math.abs(b - a), 1);
            // Lock scaleY to match scaleX for an isotropic aspect ratio
            scaleY = scaleX; 
            
            offsetX = canvas.width / 2 - centerX * scaleX;
            offsetY = canvas.height / 2;
        }

        // Convert Math coordinates to Pixel coordinates
        function toPixelX(x) { return offsetX + x * scaleX; }
        function toPixelY(y) { return offsetY - y * scaleY; } // Canvas Y goes downwards

        // Convert Pixel coordinates to Math coordinates
        function toMathX(px) { return (px - offsetX) / scaleX; }

        function drawGrid() {
            ctx.strokeStyle = '#e2e8f0';
            ctx.lineWidth = 1;
            
            // Grid spacing dynamically calculated based on scale
            const idealSpacing = 50; 
            const approxUnitSpacing = idealSpacing / scaleX;
            // Find next power of 10 or friendly fraction (0.1, 0.2, 0.5, 1, 2, 5, 10...)
            let step = Math.pow(10, Math.floor(Math.log10(approxUnitSpacing)));
            if (approxUnitSpacing / step > 5) step *= 5;
            else if (approxUnitSpacing / step > 2) step *= 2;

            // Horizontal grid lines (constant Y)
            const startY = toPixelY(Math.ceil(canvas.height / scaleY));
            const endY = toPixelY(Math.floor(-canvas.height / scaleY));
            const mathStartY = Math.floor(toMathX(0)); // generic span helper
            
            // X Grid Lines
            const minX = toMathX(0);
            const maxX = toMathX(canvas.width);
            const startX = Math.floor(minX / step) * step;
            
            for (let x = startX; x <= maxX; x += step) {
                let px = toPixelX(x);
                ctx.beginPath();
                ctx.moveTo(px, 0);
                ctx.lineTo(px, canvas.height);
                ctx.stroke();

                // Draw X labels
                if (Math.abs(x) > 0.0001) {
                    ctx.fillStyle = '#888';
                    ctx.font = '10px sans-serif';
                    ctx.fillText(Number(x.toFixed(4)), px + 4, offsetY - 6);
                }
            }

            // Y Grid Lines
            const minY = (offsetY - canvas.height) / scaleY;
            const maxY = offsetY / scaleY;
            const startYMath = Math.floor(minY / step) * step;

            for (let y = startYMath; y <= maxY; y += step) {
                let py = toPixelY(y);
                ctx.beginPath();
                ctx.moveTo(0, py);
                ctx.lineTo(canvas.width, py);
                ctx.stroke();

                // Draw Y labels
                if (Math.abs(y) > 0.0001) {
                    ctx.fillStyle = '#888';
                    ctx.font = '10px sans-serif';
                    ctx.fillText(Number(y.toFixed(4)), offsetX + 6, py - 4);
                }
            }

            // Main Axes
            ctx.strokeStyle = '#333333';
            ctx.lineWidth = 2;

            // X Axis
            ctx.beginPath();
            ctx.moveTo(0, offsetY);
            ctx.lineTo(canvas.width, offsetY);
            ctx.stroke();

            // Y Axis
            ctx.beginPath();
            ctx.moveTo(offsetX, 0);
            ctx.lineTo(offsetX, canvas.height);
            ctx.stroke();
            
            // Origin Label
            ctx.fillStyle = '#333';
            ctx.fillText('0', offsetX - 12, offsetY + 14);
        }

        function updateGraph() {
            // Clear canvas
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            errorDiv.style.display = 'none';

            const expr = fxInput.value;
            const a = parseFloat(boundAInput.value);
            const b = parseFloat(boundBInput.value);

            if (isNaN(a) || !isFinite(a) || isNaN(b) || !isFinite(b)) {
                showError("Please enter valid real numbers for your bounds.");
                return;
            }

            try {
                const f = getFunction(expr);
                
                // 1. Calculate and update numerical integration text
                const integralVal = calculateIntegral(f, a, b);
                if(isNaN(integralVal) || !isFinite(integralVal)) {
                    integralValueDisplay.textContent = "Undefined";
                } else {
                    integralValueDisplay.textContent = Number(integralVal.toFixed(5));
                }

                // 2. Adjust grid view smoothly relative to bounds
                // Only change focus automatically if canvas matches default initialized metrics
                if (window.autoFocusNeeded) {
                    autoFocusGrid(a, b);
                    window.autoFocusNeeded = false;
                }

                // Draw grid mapping elements
                drawGrid();

                // 3. Draw shaded area of integration
                const startX = Math.min(a, b);
                const endX = Math.max(a, b);
                
                ctx.fillStyle = 'rgba(0, 118, 255, 0.25)';
                ctx.beginPath();
                
                let firstPoint = true;
                const fillResolution = 2; // pixel steps
                
                for (let px = toPixelX(startX); px <= toPixelX(endX); px += fillResolution) {
                    const x = toMathX(px);
                    try {
                        const y = f(x);
                        if (typeof y === 'number' && isFinite(y)) {
                            if (firstPoint) {
                                ctx.moveTo(px, toPixelY(0));
                                ctx.lineTo(px, toPixelY(y));
                                firstPoint = false;
                            } else {
                                ctx.lineTo(px, toPixelY(y));
                            }
                        }
                    } catch(e) {}
                }
                ctx.lineTo(toPixelX(endX), toPixelY(0));
                ctx.closePath();
                ctx.fill();

                // 4. Draw continuous function curve
                ctx.strokeStyle = '#0076ff';
                ctx.lineWidth = 3;
                ctx.beginPath();
                
                let isDrawing = false;
                for (let px = 0; px <= canvas.width; px++) {
                    const x = toMathX(px);
                    try {
                        const y = f(x);
                        if (typeof y === 'number' && isFinite(y)) {
                            const py = toPixelY(y);
                            if (!isDrawing) {
                                ctx.moveTo(px, py);
                                isDrawing = true;
                            } else {
                                // Prevent chaotic vertical lines when passing through asymptotes
                                if (Math.abs(py) < canvas.height * 5) {
                                    ctx.lineTo(px, py);
                                } else {
                                    ctx.moveTo(px, py);
                                }
                            }
                        } else {
                            isDrawing = false;
                        }
                    } catch (e) {
                        isDrawing = false;
                    }
                }
                ctx.stroke();

                // 5. Draw individual bounded limits lines
                ctx.strokeStyle = '#dc3545';
                ctx.lineWidth = 1.5;
                ctx.setLineDash([4, 4]);
                
                // Boundary a
                ctx.beginPath();
                ctx.moveTo(toPixelX(a), 0);
                ctx.lineTo(toPixelX(a), canvas.height);
                ctx.stroke();

                // Boundary b
                ctx.beginPath();
                ctx.moveTo(toPixelX(b), 0);
                ctx.lineTo(toPixelX(b), canvas.height);
                ctx.stroke();
                ctx.setLineDash([]); // Reset dash line

            } catch (error) {
                showError(error.message);
                drawGrid();
            }
        }

        function showError(msg) {
            errorDiv.style.display = 'block';
            errorDiv.textContent = msg;
            integralValueDisplay.textContent = "Error";
        }

        // Live updating interactions
        [fxInput, boundAInput, boundBInput].forEach(element => {
            element.addEventListener('input', () => {
                if(element !== fxInput) {
                    // Refocus view grid frame if user alters limits bounds specifically
                    const a = parseFloat(boundAInput.value);
                    const b = parseFloat(boundBInput.value);
                    if(!isNaN(a) && !isNaN(b)) autoFocusGrid(a, b);
                }
                updateGraph();
            });
        });

        // Initialize App State
        window.autoFocusNeeded = true;
        setTimeout(() => {
            resizeCanvas();
        }, 50);
    </script>
</body>
</html>
