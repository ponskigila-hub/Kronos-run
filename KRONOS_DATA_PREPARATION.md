# Kronos: Data Preparation & Troubleshooting Guide

## 📊 Data Preparation Guide

### Step 1: Understand Required Format

Kronos expects CSV files with these exact columns:

```
Column Name | Required | Type | Format | Example
------------|----------|------|--------|----------
timestamps  | ✓ Yes    | str  | Any datetime format | 2024-01-01, 2024-01-01 09:30:00
open        | ✓ Yes    | float| Positive number | 100.5
high        | ✓ Yes    | float| >= open | 101.2
low         | ✓ Yes    | float| <= open | 99.8
close       | ✓ Yes    | float| Positive number | 100.8
volume      | ✓ Yes    | float| Positive number | 1000000
amount      | ✓ Yes    | float| Positive number | 1000000000
```

### Step 2: CSV File Template

Create a file named `SYMBOL_data.csv`:

```csv
timestamps,open,high,low,close,volume,amount
2024-01-01 00:00:00,100.50,101.20,99.80,100.80,1000000,1000000000
2024-01-01 01:00:00,100.80,102.10,100.20,101.50,1200000,1215000000
2024-01-01 02:00:00,101.50,102.80,101.20,102.30,1100000,1123300000
2024-01-01 03:00:00,102.30,103.10,102.00,102.90,950000,970050000
2024-01-01 04:00:00,102.90,103.50,102.50,103.20,1050000,1084560000
```

**Minimum requirements:**
- At least 50-100 rows of data
- Data sorted by date (oldest first)
- No missing values
- Consistent time intervals

### Step 3: Data Validation Script

Before feeding to Kronos, validate your data:

```python
import pandas as pd

def validate_kronos_data(csv_file):
    """Validate CSV for Kronos compatibility"""
    
    print(f"📋 Validating {csv_file}...")
    
    try:
        df = pd.read_csv(csv_file)
    except Exception as e:
        print(f"❌ Error reading CSV: {e}")
        return False
    
    # Check required columns
    required_cols = ['timestamps', 'open', 'high', 'low', 'close', 'volume', 'amount']
    missing_cols = [col for col in required_cols if col not in df.columns]
    
    if missing_cols:
        print(f"❌ Missing columns: {missing_cols}")
        return False
    print("✓ All required columns present")
    
    # Check data types
    numeric_cols = ['open', 'high', 'low', 'close', 'volume', 'amount']
    for col in numeric_cols:
        try:
            df[col] = pd.to_numeric(df[col])
        except:
            print(f"❌ Column '{col}' contains non-numeric values")
            return False
    print("✓ All numeric columns valid")
    
    # Check minimum rows
    if len(df) < 50:
        print(f"⚠️  Warning: Only {len(df)} rows (minimum 50 recommended)")
    else:
        print(f"✓ Data has {len(df)} rows")
    
    # Check for missing values
    missing = df[numeric_cols].isnull().sum().sum()
    if missing > 0:
        print(f"❌ Found {missing} missing values")
        return False
    print("✓ No missing values")
    
    # Validate OHLC relationships
    df['timestamps'] = pd.to_datetime(df['timestamps'])
    df = df.sort_values('timestamps')
    
    invalid_rows = (
        (df['high'] < df['open']) |
        (df['high'] < df['close']) |
        (df['high'] < df['low']) |
        (df['low'] > df['open']) |
        (df['low'] > df['close']) |
        (df['open'] <= 0) |
        (df['close'] <= 0)
    ).sum()
    
    if invalid_rows > 0:
        print(f"❌ Found {invalid_rows} rows with invalid OHLC")
        return False
    print("✓ OHLC relationships valid")
    
    # Check volume
    if (df['volume'] <= 0).any():
        print("⚠️  Warning: Some rows have zero/negative volume")
    print("✓ Volume values valid")
    
    # Date sorting
    if not (df['timestamps'].diff().dt.total_seconds() > 0).all():
        print("⚠️  Warning: Data may not be sorted by date")
        df = df.sort_values('timestamps')
        print("   → Auto-sorted by timestamp")
    print("✓ Data properly sorted")
    
    # Summary
    print("\n✅ Data validation passed!")
    print(f"   Rows: {len(df)}")
    print(f"   Date range: {df['timestamps'].min()} to {df['timestamps'].max()}")
    print(f"   Price range: {df['close'].min():.2f} - {df['close'].max():.2f}")
    print(f"   Avg volume: {df['volume'].mean():.0f}")
    
    return True

# Usage
validate_kronos_data("AAPL_data.csv")
```

### Step 4: Data Cleaning Script

If validation fails, clean your data:

```python
import pandas as pd
import numpy as np

def clean_kronos_data(input_file, output_file):
    """Clean CSV data for Kronos"""
    
    print(f"🧹 Cleaning {input_file}...")
    
    # Load
    df = pd.read_csv(input_file)
    
    # Convert timestamps
    df['timestamps'] = pd.to_datetime(df['timestamps'])
    
    # Sort by date
    df = df.sort_values('timestamps').reset_index(drop=True)
    
    # Remove duplicates (same timestamp)
    df = df.drop_duplicates(subset=['timestamps'])
    
    # Remove missing values
    df = df.dropna()
    
    # Fix negative/zero values
    numeric_cols = ['open', 'high', 'low', 'close', 'volume', 'amount']
    for col in numeric_cols:
        df[col] = df[col].replace(0, np.nan)  # Mark zeros as missing
        df[col] = df[col].abs()  # Make all positive
    
    df = df.dropna()  # Remove rows with zeros
    
    # Fix OHLC relationships
    df['high'] = df[['open', 'high', 'low', 'close']].max(axis=1)
    df['low'] = df[['open', 'high', 'low', 'close']].min(axis=1)
    
    # Forward fill amount if missing
    if df['amount'].isnull().any():
        df['amount'] = df['amount'].fillna(df['volume'])
    
    # Keep only required columns
    df = df[['timestamps', 'open', 'high', 'low', 'close', 'volume', 'amount']]
    
    # Reset index
    df = df.reset_index(drop=True)
    
    # Save
    df.to_csv(output_file, index=False)
    
    print(f"✅ Cleaned data saved to {output_file}")
    print(f"   Rows: {len(df)}")
    print(f"   Date range: {df['timestamps'].min()} to {df['timestamps'].max()}")
    
    return df

# Usage
df_clean = clean_kronos_data("messy_data.csv", "clean_data.csv")
```

### Step 5: Data from Different Sources

#### From CSV File
```python
import pandas as pd

df = pd.read_csv("your_data.csv")
```

#### From Yahoo Finance
```python
import yfinance as yf

# Download AAPL daily data
data = yf.download('AAPL', start='2023-01-01', end='2024-01-01', interval='1d')
data['amount'] = data['Volume']  # Use volume as amount
data.columns = ['open', 'high', 'low', 'close', 'volume', 'amount']
data.to_csv('AAPL.csv')
```

#### From Alpha Vantage
```python
import requests
import pandas as pd

API_KEY = 'your_api_key'
symbol = 'AAPL'

url = f'https://www.alphavantage.co/query?function=TIME_SERIES_DAILY&symbol={symbol}&apikey={API_KEY}'
response = requests.get(url)
data = response.json()

# Parse and convert to DataFrame
# (depends on API response structure)
```

#### From Binance (Crypto)
```python
from binance.client import Client

client = Client(api_key='', api_secret='')
klines = client.get_historical_klines('BTCUSDT', Client.KLINE_INTERVAL_1HOUR, '2023-01-01')

# Convert to DataFrame
import pandas as pd
df = pd.DataFrame(klines, columns=['time', 'open', 'high', 'low', 'close', 'volume', 'close_time', 'quote_asset_volume', 'trades', 'tb_base_volume', 'tb_quote_volume', 'ignore'])
df['amount'] = df['quote_asset_volume']
df = df[['time', 'open', 'high', 'low', 'close', 'volume', 'amount']]
df.to_csv('BTC_data.csv', index=False)
```

#### From AKShare (Chinese Markets)
```python
import akshare as ak

# Get stock data
df = ak.stock_zh_a_hist(symbol='000001', period='daily', start_date='20230101', end_date='20240101', adjust='')
df['timestamps'] = df['日期']
df['open'] = df['开盘']
df['high'] = df['最高']
df['low'] = df['最低']
df['close'] = df['收盘']
df['volume'] = df['成交量']
df['amount'] = df['成交额']

df_clean = df[['timestamps', 'open', 'high', 'low', 'close', 'volume', 'amount']]
df_clean.to_csv('CN_stock.csv', index=False)
```

---

## 🔍 Common Data Issues & Fixes

### Issue 1: Timestamps Not Recognized

**Symptom:**
```python
ValueError: time data '2024-01-01' does not match format '%Y-%m-%d %H:%M:%S'
```

**Fix:**
```python
import pandas as pd

df = pd.read_csv('data.csv')

# Try different formats
possible_formats = [
    '%Y-%m-%d',
    '%Y-%m-%d %H:%M:%S',
    '%d/%m/%Y',
    '%m/%d/%Y',
    '%Y/%m/%d %H:%M'
]

for fmt in possible_formats:
    try:
        df['timestamps'] = pd.to_datetime(df['timestamps'], format=fmt)
        print(f"✓ Format matched: {fmt}")
        break
    except:
        continue
```

### Issue 2: Missing Values

**Symptom:**
```python
ValueError: cannot reindex from a duplicate axis
```

**Fix:**
```python
import pandas as pd

df = pd.read_csv('data.csv')

# Find missing values
print(df.isnull().sum())

# Option 1: Remove rows with any missing value
df_clean = df.dropna()

# Option 2: Forward fill (use previous value)
df_clean = df.fillna(method='ffill')

# Option 3: Interpolate (linear fill)
df_clean = df.interpolate(method='linear')

print(f"Removed {len(df) - len(df_clean)} rows")
```

### Issue 3: Duplicate Timestamps

**Symptom:**
```
Same timestamp appears multiple times
```

**Fix:**
```python
import pandas as pd

df = pd.read_csv('data.csv')
df['timestamps'] = pd.to_datetime(df['timestamps'])

# Find duplicates
duplicates = df[df.duplicated(subset=['timestamps'], keep=False)]
print(f"Found {len(duplicates)} duplicate timestamps")

# Option 1: Keep first occurrence
df_clean = df.drop_duplicates(subset=['timestamps'], keep='first')

# Option 2: Keep last occurrence
df_clean = df.drop_duplicates(subset=['timestamps'], keep='last')

# Option 3: Average duplicate values
df_clean = df.groupby('timestamps').mean()
```

### Issue 4: Wrong Column Names

**Symptom:**
```python
KeyError: 'close'
```

**Fix:**
```python
import pandas as pd

df = pd.read_csv('data.csv')

# Check actual column names
print(df.columns)

# Rename if different
rename_dict = {
    'Price': 'close',
    'Bid': 'open',
    'Ask': 'high',
    'Vol': 'volume',
    # Add more as needed
}

df = df.rename(columns=rename_dict)
```

### Issue 5: Zero or Negative Values

**Symptom:**
```python
Warning: Some rows have zero/negative volume
```

**Fix:**
```python
import pandas as pd

df = pd.read_csv('data.csv')

# Remove zero volumes
before = len(df)
df = df[df['volume'] > 0]
after = len(df)
print(f"Removed {before - after} rows with zero volume")

# Make prices absolute (remove negatives)
df['close'] = df['close'].abs()

# Remove any still invalid
df = df[(df['close'] > 0) & (df['open'] > 0)]
```

### Issue 6: OHLC Logic Error

**Symptom:**
```
High < Low, or Close > High, etc.
```

**Fix:**
```python
import pandas as pd

df = pd.read_csv('data.csv')

# Fix automatically
df['high'] = df[['open', 'high', 'low', 'close']].max(axis=1)
df['low'] = df[['open', 'high', 'low', 'close']].min(axis=1)

# Verify
assert (df['high'] >= df['open']).all()
assert (df['high'] >= df['close']).all()
assert (df['low'] <= df['open']).all()
assert (df['low'] <= df['close']).all()
print("✓ OHLC logic valid")
```

### Issue 7: Inconsistent Decimal Places

**Symptom:**
```
Prices like 100.50 and 100.5 mixed
```

**Fix:**
```python
import pandas as pd

df = pd.read_csv('data.csv')

# Standardize decimal places
numeric_cols = ['open', 'high', 'low', 'close']
for col in numeric_cols:
    df[col] = df[col].round(8)  # 8 decimal places

print(df.head())
```

---

## 🚀 Pre-Kronos Checklist

Before feeding data to Kronos, verify:

```python
import pandas as pd

def pre_kronos_checklist(csv_file):
    df = pd.read_csv(csv_file)
    df['timestamps'] = pd.to_datetime(df['timestamps'])
    
    checks = {
        'Has timestamps': 'timestamps' in df.columns,
        'Has OHLCV': all(col in df.columns for col in ['open', 'high', 'low', 'close', 'volume']),
        'Has amount': 'amount' in df.columns,
        'Rows >= 50': len(df) >= 50,
        'No nulls': df[['timestamps', 'open', 'high', 'low', 'close', 'volume', 'amount']].isnull().sum().sum() == 0,
        'Positive prices': (df[['open', 'high', 'low', 'close']] > 0).all().all(),
        'High >= Low': (df['high'] >= df['low']).all(),
        'Sorted by date': (df['timestamps'].diff().dt.total_seconds()[1:] > 0).all(),
    }
    
    print("📋 Pre-Kronos Checklist:")
    all_pass = True
    for check, result in checks.items():
        status = "✓" if result else "✗"
        print(f"  {status} {check}")
        if not result:
            all_pass = False
    
    if all_pass:
        print("\n✅ Data ready for Kronos!")
        return True
    else:
        print("\n⚠️  Fix issues above before using Kronos")
        return False

# Usage
pre_kronos_checklist('your_data.csv')
```

---

## 📈 Sample Data Files

Find sample data in:

```
Kronos-master/
├── examples/data/           ← Pre-loaded examples
│   ├── XSHG_5min_600977.csv
│   └── ... (other stocks)
└── finetune_csv/data/       ← Training data examples
    └── HK_ali_09988_kline_5min_all.csv
```

### Using Sample Data
```python
import pandas as pd
from model import Kronos, KronosTokenizer, KronosPredictor

# Load from examples
df = pd.read_csv("examples/data/XSHG_5min_600977.csv")

# Continue with Kronos...
```

---

## 🔧 Advanced Data Handling

### Handle Multiple Timeframes
```python
def create_multiframe_dataset(df, base_interval='5min'):
    """Create OHLCV for different timeframes"""
    
    df['timestamps'] = pd.to_datetime(df['timestamps'])
    
    # 1-hour from 5-minute data
    ohlcv_1h = df.set_index('timestamps').resample('1H').agg({
        'open': 'first',
        'high': 'max',
        'low': 'min',
        'close': 'last',
        'volume': 'sum'
    })
    
    # 1-day from 5-minute data
    ohlcv_1d = df.set_index('timestamps').resample('1D').agg({
        'open': 'first',
        'high': 'max',
        'low': 'min',
        'close': 'last',
        'volume': 'sum'
    })
    
    return ohlcv_1h.reset_index(), ohlcv_1d.reset_index()

# Usage
df_1h, df_1d = create_multiframe_dataset(df)
```

### Handle Multiple Symbols
```python
import os
import pandas as pd

def batch_prepare_data(data_dir):
    """Prepare all CSV files in directory"""
    
    for filename in os.listdir(data_dir):
        if not filename.endswith('.csv'):
            continue
        
        input_path = os.path.join(data_dir, filename)
        output_path = os.path.join(data_dir, f'clean_{filename}')
        
        print(f"Processing {filename}...")
        df = clean_kronos_data(input_path, output_path)
        
    print("✓ All files processed")

# Usage
batch_prepare_data('./data')
```

---

## ✅ Data Preparation Checklist

Before running Kronos:

- [ ] CSV file created with correct columns
- [ ] Data validated (run validation script)
- [ ] No missing values
- [ ] Timestamps sorted (oldest first)
- [ ] OHLC logic correct (High >= Low, etc.)
- [ ] All prices positive (> 0)
- [ ] At least 50-100 rows of data
- [ ] Consistent decimal places
- [ ] No duplicate timestamps
- [ ] Amount column populated

Once all checked ✓, your data is ready for Kronos!

---

## 📚 Data Format Examples by Market

### Stock Market (Daily)
```csv
timestamps,open,high,low,close,volume,amount
2024-01-02,100.50,101.20,99.80,100.80,1000000,100000000
2024-01-03,100.80,102.10,100.20,101.50,1200000,121500000
```

### Crypto (Hourly)
```csv
timestamps,open,high,low,close,volume,amount
2024-01-01 00:00:00,43567.89,43789.34,43456.12,43650.23,123.45,5387234.56
2024-01-01 01:00:00,43650.23,43900.56,43623.11,43789.45,145.67,6356423.78
```

### Forex (5-minute)
```csv
timestamps,open,high,low,close,volume,amount
2024-01-01 09:00:00,1.0950,1.0955,1.0945,1.0952,50000,50123.50
2024-01-01 09:05:00,1.0952,1.0960,1.0950,1.0958,48000,48234.60
```

### Chinese Markets
```csv
timestamps,open,high,low,close,volume,amount
2024-01-01 09:30:00,600977.100,600977.150,600977.080,600977.120,1000000,100000000
2024-01-01 09:31:00,600977.120,600977.180,600977.100,600977.160,1100000,110000000
```

---

**Data prep complete! Ready to run Kronos.** ✅
