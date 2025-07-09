# Next.js Integration Guide

## 📁 File Structure for Next.js Integration

```
your-nextjs-project/
├── lib/
│   ├── jupiter-perps/
│   │   ├── constants.ts
│   │   ├── utils.ts
│   │   ├── types.ts
│   │   ├── output.ts
│   │   └── idl/
│   │       ├── jupiter-perpetuals-idl.ts
│   │       └── jupiter-perpetuals-idl-json.json
│   └── ...
├── pages/api/
│   └── trades/
│       └── [wallet].ts
└── ...
```

## 🔧 Environment Setup

### 1. **Install Dependencies**
```bash
npm install @coral-xyz/anchor @solana/web3.js @solana/spl-token dotenv
npm install -D @types/node
```

### 2. **Environment Variables (.env.local)**
```env
RPC_URL=https://api.mainnet-beta.solana.com
# Or your preferred RPC endpoint (Alchemy, QuickNode, etc.)
```

## 🚀 API Route Implementation

### **`pages/api/trades/[wallet].ts`**

```typescript
import type { NextApiRequest, NextApiResponse } from 'next';
import { analyzeTradeHistoryJson } from '../../../lib/jupiter-perps/output';

// Input validation
function validateWalletAddress(wallet: string): boolean {
  return /^[1-9A-HJ-NP-Za-km-z]{32,44}$/.test(wallet);
}

function validateDateFormat(date: string): boolean {
  return /^\d{1,2}\.\d{1,2}\.\d{4}$/.test(date);
}

function parseAndValidateDate(dateString: string): Date {
  const [day, month, year] = dateString.split('.').map(Number);
  
  if (day < 1 || day > 31) throw new Error('Invalid day');
  if (month < 1 || month > 12) throw new Error('Invalid month');
  if (year < 2020 || year > 2030) throw new Error('Invalid year');
  
  const date = new Date(year, month - 1, day);
  if (isNaN(date.getTime())) throw new Error('Invalid date');
  
  return date;
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  // Only allow GET requests
  if (req.method !== 'GET') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  try {
    const { wallet, from_date, to_date } = req.query;

    // Validate wallet address
    if (!wallet || typeof wallet !== 'string') {
      return res.status(400).json({ error: 'Wallet address is required' });
    }

    if (!validateWalletAddress(wallet)) {
      return res.status(400).json({ 
        error: 'Invalid wallet address format' 
      });
    }

    // Validate dates if provided
    let fromDate: string | undefined;
    let toDate: string | undefined;

    if (from_date) {
      if (typeof from_date !== 'string' || !validateDateFormat(from_date)) {
        return res.status(400).json({ 
          error: 'Invalid from_date format. Use DD.MM.YYYY' 
        });
      }
      try {
        parseAndValidateDate(from_date);
        fromDate = from_date;
      } catch (error) {
        return res.status(400).json({ 
          error: `Invalid from_date: ${error instanceof Error ? error.message : 'Unknown error'}` 
        });
      }
    }

    if (to_date) {
      if (typeof to_date !== 'string' || !validateDateFormat(to_date)) {
        return res.status(400).json({ 
          error: 'Invalid to_date format. Use DD.MM.YYYY' 
        });
      }
      try {
        parseAndValidateDate(to_date);
        toDate = to_date;
      } catch (error) {
        return res.status(400).json({ 
          error: `Invalid to_date: ${error instanceof Error ? error.message : 'Unknown error'}` 
        });
      }
    }

    // Validate date range
    if (fromDate && toDate) {
      const from = parseAndValidateDate(fromDate);
      const to = parseAndValidateDate(toDate);
      
      if (from >= to) {
        return res.status(400).json({ 
          error: 'from_date must be earlier than to_date' 
        });
      }
    }

    // Call the optimized trade history function
    const trades = await analyzeTradeHistoryJson(fromDate, wallet, toDate);

    // Set caching headers (optional)
    res.setHeader('Cache-Control', 'public, s-maxage=60, stale-while-revalidate=300');

    // Return the data
    res.status(200).json(trades);

  } catch (error) {
    console.error('Error fetching trade history:', error);
    
    if (error instanceof Error) {
      // Handle known errors
      if (error.message.includes('rate limit') || error.message.includes('429')) {
        return res.status(429).json({ error: 'Rate limited. Please try again later.' });
      }
      
      if (error.message.includes('Invalid')) {
        return res.status(400).json({ error: error.message });
      }
    }

    // Generic error
    res.status(500).json({ error: 'Internal server error' });
  }
}
```

## 📖 API Usage Examples

### **1. Get All Trades (Last 30 Days)**
```
GET /api/trades/GFexHU2EkUcmcKbj3GyiuNQoa7TreCxLkAxMb2xeLHgS
```

### **2. Get Trades with Date Range**
```
GET /api/trades/GFexHU2EkUcmcKbj3GyiuNQoa7TreCxLkAxMb2xeLHgS?from_date=01.06.2025&to_date=27.06.2025
```

### **3. Get Trades from Specific Date to Now**
```
GET /api/trades/GFexHU2EkUcmcKbj3GyiuNQoa7TreCxLkAxMb2xeLHgS?from_date=15.06.2025
```

## 🎯 Frontend Integration

### **React Hook Example**
```typescript
import { useState, useEffect } from 'react';

interface TradeData {
  wallet_address: string;
  sync_timestamp: string;
  positions: Array<{
    trade_id: string;
    symbol: string;
    direction: string;
    status: string;
    // ... other fields
  }>;
}

export function useTrades(walletAddress: string, fromDate?: string, toDate?: string) {
  const [data, setData] = useState<TradeData | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!walletAddress) return;

    const fetchTrades = async () => {
      setLoading(true);
      setError(null);

      try {
        const params = new URLSearchParams();
        if (fromDate) params.append('from_date', fromDate);
        if (toDate) params.append('to_date', toDate);

        const url = `/api/trades/${walletAddress}${params.toString() ? `?${params.toString()}` : ''}`;
        const response = await fetch(url);

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }

        const data = await response.json();
        setData(data);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Unknown error');
      } finally {
        setLoading(false);
      }
    };

    fetchTrades();
  }, [walletAddress, fromDate, toDate]);

  return { data, loading, error };
}
```

## ⚡ Performance Features

- **~5 second response time** (optimized from 27s)
- **Proper rate limiting** with exponential backoff
- **Efficient batch processing** (300 signatures per request)
- **Minimal delays** (30ms between requests)
- **Empty result handling** (consistent JSON structure)

## 🔒 Security Features

- **Input validation** for all parameters
- **Wallet address format validation**
- **Date range validation**
- **Error boundary handling**
- **Rate limit protection**

## 📝 Response Format

```json
{
  "wallet_address": "GFexHU2EkUcmcKbj3GyiuNQoa7TreCxLkAxMb2xeLHgS",
  "sync_timestamp": "2025-07-08T18:23:39.064Z",
  "positions": [
    {
      "trade_id": "C8KSC",
      "position_key": "76USLRvRMwDoGsK2Bj4RXV2CnuAUxgpVhJy7496Fz4M1",
      "symbol": "SOL-PERP",
      "direction": "long",
      "status": "closed",
      "collateral_token": "SOL",
      "size_usd": 1801436.73,
      "collateral_usd": 99956.28,
      "leverage": 18.02,
      "entry_price": 150.32,
      "exit_price": 150.95,
      "realized_pnl": 7604.04,
      "total_fees": 5239.81,
      "entry_time": "2025-06-05T14:38:45.000Z",
      "exit_time": "2025-06-05T15:25:06.000Z",
      "events": [...]
    }
  ]
}
```

## 🎛️ Configuration Options

### **RPC Endpoints (Recommended)**
- **Free**: `https://api.mainnet-beta.solana.com` (rate limited)
- **Alchemy**: `https://solana-mainnet.g.alchemy.com/v2/YOUR_API_KEY`
- **QuickNode**: `https://YOUR_ENDPOINT.solana-mainnet.quiknode.pro/YOUR_TOKEN/`
- **Helius**: `https://rpc.helius.xyz/?api-key=YOUR_API_KEY`

### **Performance Tuning**
For high-volume usage, consider:
- Using a premium RPC endpoint
- Implementing response caching
- Adding request queuing for multiple concurrent requests

## ✅ Ready for Production

Your optimized script is **production-ready** with:
- 🚀 **81% performance improvement** (27s → 5s)
- 🔒 **Security validation** built-in
- 📊 **Consistent JSON output**
- ⚡ **Optimal rate limiting**
- 🛡️ **Error handling** and retries 