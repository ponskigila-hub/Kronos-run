# Kronos: Complete Setup & Running Guide

## 📋 What is Kronos?

**Kronos** is an open-source AI foundation model for predicting financial market candlesticks (K-lines). It:
- Predicts OHLCV (Open, High, Low, Close, Volume) data
- Works with 45+ global exchanges
- Uses advanced transformer architecture with custom tokenizer
- Comes in multiple sizes: mini, small, base
- Supports both CLI and Web UI interfaces

---

## 🎯 Prerequisites Check

Before starting, make sure you have:

```
✓ Python 3.10 or higher
✓ pip (Python package manager)
✓ 4GB+ RAM (8GB recommended)
✓ 2GB+ disk space (for models)
✓ Internet connection (to download pre-trained models)
✓ Optional: GPU (CUDA 11.8+) for faster predictions
```

### Check Your System
```bash
# Check Python version
python3 --version

# Check pip
pip --version

# Check GPU availability (optional)
python3 -c "import torch; print('GPU Available:', torch.cuda.is_available())"
```

---

## ⚙️ Installation Steps

### Step 1: Extract the Kronos Archive
```bash
# If on Windows
# Right-click Kronos-master.zip → Extract All → Choose folder

# If on Mac/Linux
unzip Kronos-master.zip
cd Kronos-master
```

### Step 2: Create Python Virtual Environment (Recommended)
This isolates dependencies and prevents conflicts:

**On Mac/Linux:**
```bash
# Create virtual environment
python3 -m venv kronos_env

# Activate it
source kronos_env/bin/activate

# You should see (kronos_env) prefix in terminal
```

**On Windows (PowerShell):**
```powershell
# Create virtual environment
python -m venv kronos_env

# Activate it
.\kronos_env\Scripts\Activate.ps1

# If you get execution policy error:
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
# Then retry activate command
```

**On Windows (Command Prompt):**
```bash
# Create virtual environment
python -m venv kronos_env

# Activate it
source kronos_env/Scripts/activate
```

### Step 3: Install Dependencies
```bash
# Install all required packages
pip install -r requirements.txt

# This installs:
# - numpy, pandas (data handling)
# - torch>=2.0.0 (AI framework)
# - einops (tensor operations)
# - huggingface_hub (model downloads)
# - matplotlib, plotly (visualization)
# - safetensors (model format)
# - tqdm (progress bars)
```

**⚠️ Note on PyTorch Installation:**
The default pip install might install CPU-only version. If you have GPU:

```bash
# For NVIDIA GPU (CUDA 11.8):
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118

# For AMD GPU (ROCm):
pip install torch --index-url https://download.pytorch.org/whl/rocm5.7
```

### Step 4: Verify Installation
```bash
# Check if all imports work
python3 -c "
import torch
import pandas
import matplotlib
from huggingface_hub import hf_hub_download
print('✅ All dependencies installed successfully!')
print(f'PyTorch version: {torch.__version__}')
print(f'GPU Available: {torch.cuda.is_available()}')
"
```

---

## 🚀 Running Kronos - Three Methods

### METHOD 1: Simple Python Script (Recommended for Beginners)

#### Step 1: Create Your First Prediction Script
Create a file named `my_first_prediction.py`:

```python
import pandas as pd
import matplotlib.pyplot as plt
import sys
from model import Kronos, KronosTokenizer, KronosPredictor

print("🚀 Starting Kronos prediction...")

# Step 1: Download and load model (first time takes 2-5 minutes)
print("📥 Loading model (this may take a moment on first run)...")
tokenizer = KronosTokenizer.from_pretrained("NeoQuasar/Kronos-Tokenizer-base")
model = Kronos.from_pretrained("NeoQuasar/Kronos-small")
print("✅ Model loaded!")

# Step 2: Initialize predictor
predictor = KronosPredictor(model, tokenizer, max_context=512)

# Step 3: Prepare sample data
# Option A: Load from included example
print("📊 Loading sample data...")
df = pd.read_csv("./examples/data/XSHG_5min_600977.csv")

# Option B: Use your own CSV
# df = pd.read_csv("your_data.csv")
# CSV should have columns: open, high, low, close, volume, amount, timestamps

# Step 4: Convert timestamps to datetime
df['timestamps'] = pd.to_datetime(df['timestamps'])

# Step 5: Set prediction parameters
lookback = 400        # Use last 400 candles to predict
pred_len = 120        # Predict next 120 candles

# Step 6: Prepare input data
x_df = df.loc[:lookback-1, ['open', 'high', 'low', 'close', 'volume', 'amount']]
x_timestamp = df.loc[:lookback-1, 'timestamps']
y_timestamp = df.loc[lookback:lookback+pred_len-1, 'timestamps']

# Step 7: Run prediction
print(f"🔮 Predicting {pred_len} candles...")
pred_df = predictor.predict(
    df=x_df,
    x_timestamp=x_timestamp,
    y_timestamp=y_timestamp,
    pred_len=pred_len,
    T=1.0,           # Temperature (1.0 = normal, higher = more creative)
    top_p=0.9,       # Top-p sampling (0.9 = use top 90% probable tokens)
    sample_count=1,  # Number of predictions to generate
    verbose=True
)

# Step 8: Display results
print("\n✅ Prediction complete!")
print("\n📈 Predicted values (first 10 rows):")
print(pred_df.head(10))

# Step 9: Visualize
kline_df = df.loc[:lookback+pred_len-1]

plt.figure(figsize=(14, 6))

# Plot close prices
plt.subplot(2, 1, 1)
plt.plot(kline_df.index[:len(kline_df)-pred_len], kline_df['close'][:len(kline_df)-pred_len], 
         label='Historical', color='blue', linewidth=1.5)
plt.plot(kline_df.index[lookback:], pred_df['close'].values, 
         label='Predicted', color='red', linewidth=1.5, linestyle='--')
plt.ylabel('Close Price')
plt.legend()
plt.grid(True)
plt.title('Kronos Price Prediction')

# Plot volumes
plt.subplot(2, 1, 2)
plt.plot(kline_df.index[:len(kline_df)-pred_len], kline_df['volume'][:len(kline_df)-pred_len], 
         label='Historical', color='blue', linewidth=1.5)
plt.plot(kline_df.index[lookback:], pred_df['volume'].values, 
         label='Predicted', color='red', linewidth=1.5, linestyle='--')
plt.ylabel('Volume')
plt.xlabel('Time')
plt.legend()
plt.grid(True)

plt.tight_layout()
plt.savefig('kronos_prediction.png', dpi=150)
print("\n📊 Chart saved as 'kronos_prediction.png'")
plt.show()
```

#### Step 2: Run the Script
```bash
# Make sure you're in the Kronos-master directory
cd Kronos-master

# Run prediction
python3 my_first_prediction.py
```

**Expected Output:**
```
🚀 Starting Kronos prediction...
📥 Loading model (this may take a moment on first run)...
✅ Model loaded!
📊 Loading sample data...
🔮 Predicting 120 candles...
Token generated:  10%|██        | 12/120 [00:05<00:45, 2.38 tokens/s]
✅ Prediction complete!
📈 Predicted values (first 10 rows):
    open    high     low   close  volume  amount
0  ...
📊 Chart saved as 'kronos_prediction.png'
```

---

### METHOD 2: Using Built-in Examples

Kronos includes several pre-built example scripts:

#### Simple Example
```bash
cd Kronos-master
python3 examples/prediction_example.py
```

#### Example with Volume Prediction
```bash
python3 examples/prediction_wo_vol_example.py
```

#### Batch Predictions (Multiple Symbols)
```bash
python3 examples/prediction_batch_example.py
```

#### Backtest Strategy with Predictions
```bash
python3 examples/run_backtest_kronos.py
```

#### New GUI Example (With Interface)
```bash
python3 examples/prediction_new_GUI.py
```

---

### METHOD 3: Web Interface (User-Friendly)

The web UI provides a graphical interface without coding.

#### Step 1: Install Web UI Dependencies
```bash
cd Kronos-master/webui
pip install -r requirements.txt
```

#### Step 2: Start Web Server
```bash
# Option A: Using provided script
./start.sh              # Mac/Linux
# or
start.sh               # Windows PowerShell

# Option B: Using Python directly
python3 run.py
```

**Expected Output:**
```
🚀 Starting Kronos Web UI...
==================================================
✅ All dependencies installed
✅ Kronos model library available

🌐 Starting Web server...
✅ Web server started successfully!
🌐 Access URL: http://localhost:7070
💡 Tip: Press Ctrl+C to stop server
```

#### Step 3: Access Web Interface
- Open browser → Go to `http://localhost:7070`
- Upload CSV file with OHLCV data
- Select prediction parameters
- Click "Predict"
- View results with interactive charts

---

## 📊 Data Format Requirements

Your CSV file should have these columns:

```
timestamps    | open   | high   | low    | close  | volume  | amount
2024-01-01   | 100.5  | 101.2  | 99.8   | 100.8  | 1000000 | 1000000000
2024-01-02   | 100.8  | 102.1  | 100.2  | 101.5  | 1200000 | 1215000000
...
```

### Important Notes:
- **timestamps**: Can be any date/time format (script will convert)
- **open, high, low, close**: Stock/crypto price
- **volume**: Trading volume
- **amount**: Trading amount (can duplicate volume if not available)
- **At least 50+ rows** of historical data recommended
- Data should be sorted by date (oldest first)

### Sample Data Locations:
```
Kronos-master/
├── examples/data/  ← Pre-loaded example files
└── finetune_csv/data/  ← Additional sample datasets
```

---

## 🎚️ Key Parameters Explained

### Prediction Parameters:

```python
predictor.predict(
    df=x_df,                    # Input DataFrame with OHLCV
    x_timestamp=x_timestamp,    # Timestamps of historical data
    y_timestamp=y_timestamp,    # Timestamps of future predictions
    pred_len=120,               # How many candles to predict
    T=1.0,                      # Temperature (0.5-2.0)
                                # 0.5 = more conservative
                                # 1.0 = normal
                                # 2.0 = more creative
    top_p=0.9,                  # Probability threshold (0.7-0.99)
                                # 0.7 = more selective
                                # 0.99 = less selective
    sample_count=1,             # Predictions to generate
                                # 1 = one prediction
                                # 5 = five different predictions
    verbose=True                # Show progress
)
```

### Model Selection:

| Model | Size | Speed | Accuracy | Use Case |
|-------|------|-------|----------|----------|
| Kronos-mini | 4.1M | ⚡⚡⚡ Very Fast | Good | Mobile, real-time |
| Kronos-small | 24.7M | ⚡⚡ Fast | Better | Standard use |
| Kronos-base | 102.3M | ⚡ Slower | Best | Serious analysis |

---

## 🔧 Troubleshooting

### Issue 1: "ModuleNotFoundError: No module named 'torch'"
**Solution:**
```bash
# Reinstall PyTorch
pip install torch --upgrade

# Or with GPU support
pip install torch --index-url https://download.pytorch.org/whl/cu118
```

### Issue 2: "No such file or directory: ./data/..."
**Solution:**
```bash
# Make sure you're in correct directory
pwd  # Check current location
cd Kronos-master  # Navigate to main folder
ls examples/  # List available example files
```

### Issue 3: "CUDA out of memory"
**Solution:**
```python
# Use smaller model
model = Kronos.from_pretrained("NeoQuasar/Kronos-mini")

# Or reduce lookback
lookback = 200  # Instead of 400
```

### Issue 4: Very Slow First Run
**Reason:** Models are downloaded from Hugging Face (1-3GB)
**Solution:** Be patient! Download happens only once. Check:
```bash
# See download progress
~/.cache/huggingface/hub/  # Linux/Mac
%USERPROFILE%\.cache\huggingface\hub\  # Windows
```

### Issue 5: Web UI Port Already in Use
**Solution:**
```python
# Edit webui/app.py
# Change: app.run(port=7070)
# To:     app.run(port=8080)

# Or kill existing process
lsof -ti:7070 | xargs kill -9  # Mac/Linux
netstat -ano | findstr :7070   # Windows
```

---

## 📈 Using Your Own Data

### Step 1: Prepare CSV File
```bash
# Create my_stock_data.csv
# Format: timestamps,open,high,low,close,volume,amount
```

### Step 2: Load and Predict
```python
import pandas as pd
from model import Kronos, KronosTokenizer, KronosPredictor

# Load your data
df = pd.read_csv("my_stock_data.csv")
df['timestamps'] = pd.to_datetime(df['timestamps'])

# Rest of prediction code (same as before)
```

### Step 3: Example with Different Symbols

```python
# Predict multiple symbols
symbols = ['AAPL', 'GOOGL', 'MSFT']

for symbol in symbols:
    print(f"\n📊 Predicting {symbol}...")
    df = pd.read_csv(f"{symbol}_data.csv")
    df['timestamps'] = pd.to_datetime(df['timestamps'])
    
    # Your prediction code here...
```

---

## 🎯 Advanced Usage

### Multiple Predictions (Ensemble)

Get multiple predictions to understand uncertainty:

```python
predictions = []
for i in range(5):
    pred = predictor.predict(
        df=x_df,
        x_timestamp=x_timestamp,
        y_timestamp=y_timestamp,
        pred_len=pred_len,
        sample_count=1,
        verbose=False
    )
    predictions.append(pred)

# Calculate average prediction
avg_pred = pd.concat(predictions).groupby(level=0).mean()

# Calculate std (uncertainty)
std_pred = pd.concat(predictions).groupby(level=0).std()

print("Average predicted close:", avg_pred['close'].mean())
print("Uncertainty (std):", std_pred['close'].mean())
```

### Fine-tuning on Custom Data

For advanced users wanting to adapt the model:

```bash
cd Kronos-master/finetune_csv

# Edit config file
nano configs/config_yourstock.yaml

# Run fine-tuning
python3 finetune_base_model.py --config configs/config_yourstock.yaml
```

### Real-time Prediction Loop

```python
import time

while True:
    # Fetch latest data
    latest_df = get_latest_data()  # Your data fetching function
    
    # Predict next period
    pred = predictor.predict(
        df=latest_df,
        pred_len=1,
        verbose=False
    )
    
    print(f"Predicted next close: {pred['close'].iloc[0]:.2f}")
    
    # Wait for next candle
    time.sleep(300)  # 5 minutes
```

---

## 📊 Interpreting Results

### Output Format
```
    open    high     low   close  volume  amount
0   100.2   101.5   99.8   100.8  1000000 1000000000
1   100.9   102.1  100.1   101.2  1100000 1111000000
...
```

### Key Metrics to Check

1. **Directional Accuracy**
   - Did it predict up/down correctly?
   - Check pred_close vs historical_close

2. **Magnitude Accuracy**
   - How much was price predicted to move?
   - Compare predicted range vs actual

3. **Volume Predictions**
   - Useful for identifying significant moves
   - Can indicate market strength

### Visualization Tips

```python
# Plot confidence band
plt.fill_between(
    range(len(pred_df)),
    pred_df['low'],
    pred_df['high'],
    alpha=0.3,
    label='Predicted Range'
)
```

---

## ⚡ Performance Tips

### Make Predictions Faster

1. **Use smaller model:**
   ```python
   model = Kronos.from_pretrained("NeoQuasar/Kronos-mini")
   ```

2. **Reduce context:**
   ```python
   lookback = 100  # Instead of 400
   ```

3. **Use GPU:**
   ```python
   # Ensure CUDA is available
   import torch
   print(torch.cuda.is_available())  # Should be True
   ```

4. **Batch predictions:**
   ```python
   # Predict multiple symbols at once
   predictions = {sym: predictor.predict(...) for sym in symbols}
   ```

---

## 🚀 Next Steps

1. **Run examples:**
   - Start with `examples/prediction_example.py`
   - Try different models (mini, small, base)

2. **Test with your data:**
   - Prepare your CSV
   - Run prediction on it

3. **Integrate into trading:**
   - Combine with your strategy
   - Backtest predictions
   - Paper trade first

4. **Fine-tune model:**
   - Adapt to specific markets
   - Use `finetune_csv/` scripts

5. **Deploy to production:**
   - Use Web UI for manual predictions
   - Or build API wrapper for automated trading

---

## 📚 Additional Resources

- **GitHub:** https://github.com/shiyu-coder/Kronos
- **HuggingFace:** https://huggingface.co/NeoQuasar
- **Paper:** https://arxiv.org/abs/2508.02739
- **Live Demo:** https://shiyu-coder.github.io/Kronos-demo/

---

## ✅ Success Checklist

You've successfully set up Kronos when:

- [ ] Python 3.10+ installed
- [ ] Virtual environment created and activated
- [ ] Dependencies installed (`pip install -r requirements.txt`)
- [ ] Can import: `from model import Kronos, KronosTokenizer, KronosPredictor`
- [ ] At least one example script runs without errors
- [ ] Can see predictions and visualizations
- [ ] Can load and predict on your own data

---

**Happy Predicting! 🚀📈**

Remember: This is a prediction model, not financial advice. Always validate predictions and use proper risk management!
