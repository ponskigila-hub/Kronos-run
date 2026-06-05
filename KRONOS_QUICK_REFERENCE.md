# Kronos Quick Reference Guide

## ⚡ TL;DR - 5 Minute Setup

```bash
# 1. Extract and navigate
unzip Kronos-master.zip && cd Kronos-master

# 2. Create environment
python3 -m venv kronos_env
source kronos_env/bin/activate  # Mac/Linux
# or
kronos_env\Scripts\activate.bat  # Windows

# 3. Install dependencies
pip install -r requirements.txt

# 4. Run example
python3 examples/prediction_example.py

# Done! ✅
```

---

## 🎯 Three Ways to Use Kronos

### Quick Script (5 minutes)
```bash
python3 examples/prediction_example.py
```

### Custom Script (15 minutes)
```bash
python3 my_prediction.py  # Your custom script
```

### Web UI (Point & Click)
```bash
cd webui
pip install -r requirements.txt
python3 run.py
# Open: http://localhost:7070
```

---

## 📝 Minimal Working Example

```python
import pandas as pd
from model import Kronos, KronosTokenizer, KronosPredictor

# Load model (first time: 2-5 min)
tokenizer = KronosTokenizer.from_pretrained("NeoQuasar/Kronos-Tokenizer-base")
model = Kronos.from_pretrained("NeoQuasar/Kronos-small")
predictor = KronosPredictor(model, tokenizer, max_context=512)

# Load data
df = pd.read_csv("your_data.csv")
df['timestamps'] = pd.to_datetime(df['timestamps'])

# Predict
pred = predictor.predict(
    df=df.iloc[:400][['open','high','low','close','volume','amount']],
    x_timestamp=df.iloc[:400]['timestamps'],
    y_timestamp=df.iloc[400:520]['timestamps'],
    pred_len=120,
    verbose=True
)

print(pred.head())
```

---

## 📊 CSV Format Required

```
timestamps,open,high,low,close,volume,amount
2024-01-01 00:00,100.5,101.2,99.8,100.8,1000000,1000000000
2024-01-01 01:00,100.8,102.1,100.2,101.5,1200000,1215000000
...
```

**Minimum requirements:**
- ✓ At least 50 rows
- ✓ Sorted by date (oldest first)
- ✓ Valid numeric values
- ✓ Timestamps convertible to datetime

---

## 🔧 Model Selection Quick Guide

```
Use Kronos-mini (4.1M)    → Fast predictions, laptops, real-time
Use Kronos-small (24.7M)  → Balance of speed & accuracy (recommended)
Use Kronos-base (102.3M)  → Most accurate, needs good GPU
```

## Change Model

```python
# Default
model = Kronos.from_pretrained("NeoQuasar/Kronos-small")

# Faster
model = Kronos.from_pretrained("NeoQuasar/Kronos-mini")

# More accurate
model = Kronos.from_pretrained("NeoQuasar/Kronos-base")
```

---

## 🎚️ Parameter Cheat Sheet

```python
predictor.predict(
    df=data,           # Historical OHLCV
    pred_len=120,      # Predict 120 candles (↑ = longer forecast)
    T=1.0,             # Temperature (0.5-2.0, 1.0 = normal)
    top_p=0.9,         # Sampling (0.7-0.99, 0.9 = normal)
    sample_count=1,    # Multiple predictions (1-5, 1 = single)
    verbose=True       # Show progress
)
```

---

## 🐛 Common Issues & Fixes

| Problem | Quick Fix |
|---------|-----------|
| `ModuleNotFoundError: torch` | `pip install torch` |
| `CUDA out of memory` | Use `Kronos-mini` instead |
| `Port 7070 already in use` | `python3 run.py` → edit port in app.py |
| `No such file or directory` | Make sure in `Kronos-master/` folder |
| `Very slow first run` | Models downloading (normal, be patient) |
| CSV parse error | Check CSV format, ensure columns exist |
| GPU not detected | `pip install torch --index-url https://download.pytorch.org/whl/cu118` |

---

## 📂 File Structure

```
Kronos-master/
├── examples/               ← Try these first
│   ├── prediction_example.py
│   ├── prediction_batch_example.py
│   └── run_backtest_kronos.py
├── model/                  ← Core model code
├── webui/                  ← Web interface
├── finetune_csv/          ← Fine-tune your own model
├── requirements.txt        ← Dependencies
└── README.md              ← Full documentation
```

---

## ✅ Verification Steps

```bash
# 1. Check Python
python3 --version  # Should be 3.10+

# 2. Check packages installed
python3 -c "import torch, pandas; print('✅ OK')"

# 3. Check GPU (optional)
python3 -c "import torch; print('GPU:', torch.cuda.is_available())"

# 4. Run test
cd Kronos-master
python3 -m pytest tests/  # Run tests
```

---

## 🎯 Common Use Cases

### Use Case 1: Predict Next Hour
```python
lookback = 100      # Last 100 candles
pred_len = 1        # Predict 1 candle
```

### Use Case 2: Day-Ahead Forecast
```python
lookback = 400      # Last 400 5-min candles = ~33 hours
pred_len = 288      # Next 288 candles = ~24 hours
```

### Use Case 3: Quick Screening
```python
lookback = 50       # Use last 50 candles
pred_len = 10       # Quick 10-candle look
model = Kronos.from_pretrained("NeoQuasar/Kronos-mini")
```

---

## 🚀 Running Examples

```bash
cd Kronos-master

# Basic example
python3 examples/prediction_example.py

# With visualization
python3 examples/prediction_new.py

# Batch predictions
python3 examples/prediction_batch_example.py

# Backtest
python3 examples/run_backtest_kronos.py

# Web UI
cd webui && python3 run.py
```

---

## 🌐 Web UI Endpoints

After starting Web UI (`python3 webui/run.py`):

- **Main page:** http://localhost:7070
- **API upload:** POST to `/api/upload`
- **API predict:** POST to `/api/predict`
- **Results:** http://localhost:7070/results

---

## 💾 Output Files

After prediction, files saved to:

```
webui/prediction_results/
└── prediction_YYYYMMDD_HHMMSS.json
```

Contains:
- Predictions (OHLCV)
- Timestamps
- Metadata

---

## 🔄 Typical Workflow

```
1. Get Data
   ↓
2. Format as CSV (timestamps, OHLCV)
   ↓
3. Load Model & Tokenizer
   ↓
4. Create Predictor
   ↓
5. Run predict()
   ↓
6. Visualize results
   ↓
7. Validate accuracy
   ↓
8. (Optional) Fine-tune for better results
```

---

## 📊 Expected Results

**Typical Accuracy:** 55-70% directional accuracy (better than coin flip)

**Variables affecting accuracy:**
- Model size (larger = better)
- Data quality (clean = better)
- Prediction length (shorter = better)
- Market conditions (trending = better)

---

## 🎓 Learning Path

1. **Beginner** (15 min)
   - Run: `examples/prediction_example.py`
   - Change: `lookback` and `pred_len`

2. **Intermediate** (30 min)
   - Load own CSV data
   - Try different models
   - Create custom script

3. **Advanced** (1-2 hours)
   - Fine-tune on your data
   - Build trading signals
   - Backtest predictions

4. **Expert** (varies)
   - Deploy in production
   - Real-time predictions
   - Model evaluation

---

## 🔗 Important Links

| Resource | URL |
|----------|-----|
| GitHub | https://github.com/shiyu-coder/Kronos |
| Models | https://huggingface.co/NeoQuasar |
| Paper | https://arxiv.org/abs/2508.02739 |
| Live Demo | https://shiyu-coder.github.io/Kronos-demo/ |

---

## 💡 Pro Tips

1. **First run slow?** ✓ Models downloading (1-3GB) - only happens once
2. **Want faster?** Use `Kronos-mini` instead of `Kronos-base`
3. **Out of memory?** Reduce `lookback` from 400 to 200
4. **Multiple predictions?** Set `sample_count=5` to get variety
5. **Better results?** Use more data (lookback=512) and fine-tune
6. **Production ready?** Always validate with backtesting first

---

## 🆘 Emergency Troubleshooting

```bash
# Start fresh (if issues persist)
rm -rf kronos_env
python3 -m venv kronos_env
source kronos_env/bin/activate
pip install -r requirements.txt
python3 examples/prediction_example.py

# Check logs
# Linux/Mac:
tail -f ~/.cache/huggingface/hub/*

# Windows:
type %USERPROFILE%\.cache\huggingface\hub\*

# Clear cache if needed
rm -rf ~/.cache/huggingface/  # Linux/Mac
rmdir %USERPROFILE%\.cache\huggingface\  # Windows (then reinstall)
```

---

## 📝 Template: Your First Prediction

```python
#!/usr/bin/env python3
"""My First Kronos Prediction"""

import pandas as pd
import matplotlib.pyplot as plt
from model import Kronos, KronosTokenizer, KronosPredictor

# 1. SETUP
print("🚀 Loading models...")
tokenizer = KronosTokenizer.from_pretrained("NeoQuasar/Kronos-Tokenizer-base")
model = Kronos.from_pretrained("NeoQuasar/Kronos-small")
predictor = KronosPredictor(model, tokenizer, max_context=512)

# 2. DATA
print("📊 Loading data...")
df = pd.read_csv("your_data.csv")
df['timestamps'] = pd.to_datetime(df['timestamps'])

# 3. PARAMETERS
lookback = 400
pred_len = 120

# 4. PREDICT
print(f"🔮 Predicting {pred_len} candles...")
x_df = df.loc[:lookback-1, ['open','high','low','close','volume','amount']]
pred = predictor.predict(
    df=x_df,
    x_timestamp=df.loc[:lookback-1, 'timestamps'],
    y_timestamp=df.loc[lookback:lookback+pred_len-1, 'timestamps'],
    pred_len=pred_len,
    verbose=True
)

# 5. RESULTS
print("✅ Done!")
print(pred.head())

# 6. PLOT
plt.plot(df['close'].iloc[:lookback], label='Historical')
plt.plot(pred['close'].values, label='Predicted')
plt.legend()
plt.show()
```

Save as `my_kronos.py`, then run:
```bash
python3 my_kronos.py
```

---

**You're ready to use Kronos!** 🎉
