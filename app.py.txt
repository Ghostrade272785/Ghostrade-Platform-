from fastapi import FastAPI, WebSocket
from fastapi.middleware.cors import CORSMiddleware
import asyncio
import json
from datetime import datetime
import random

app = FastAPI(
    title="Ghostrade Batam Institution",
    version="2.0.0"
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Global state
active_connections = []
current_data = {
    "currencies": {},
    "signals": [],
    "alerts": []
}

# Initialize data
def init_data():
    current_data["currencies"] = {
        'EUR': {'strength': 75, 'trend': 'VERY STRONG ??', 'momentum': 2.5},
        'GBP': {'strength': 60, 'trend': 'STRONG ?', 'momentum': 1.2},
        'USD': {'strength': 25, 'trend': 'VERY WEAK ??', 'momentum': -1.8},
        'JPY': {'strength': 40, 'trend': 'WEAK ?', 'momentum': -0.5},
        'CHF': {'strength': 55, 'trend': 'NEUTRAL ?', 'momentum': 0.3},
        'CAD': {'strength': 35, 'trend': 'WEAK ?', 'momentum': -1.0},
        'AUD': {'strength': 45, 'trend': 'NEUTRAL ?', 'momentum': 0.2},
        'NZD': {'strength': 42, 'trend': 'NEUTRAL ?', 'momentum': 0.1}
    }
    
    pairs_28 = [
        'EURGBP', 'EURUSD', 'EURJPY', 'EURCHF', 'EURCAD', 'EURAUD', 'EURNZD',
        'GBPUSD', 'GBPJPY', 'GBPCHF', 'GBPCAD', 'GBPAUD', 'GBPNZD',
        'USDJPY', 'USDCHF', 'USDCAD', 'USDAUD', 'USDNZD',
        'JPYCHF', 'JPYCAD', 'JPYAUD', 'JPYNZD',
        'CHFCAD', 'CHFAUD', 'CHFNZD', 'CADAUD', 'CADNZD', 'AUDNZD'
    ]
    
    signals = []
    for pair in pairs_28:
        base = pair[:3]
        quote = pair[3:6]
        base_str = current_data["currencies"][base]['strength']
        quote_str = current_data["currencies"][quote]['strength']
        gap = abs(base_str - quote_str)
        
        signal_type = 'BUY' if base_str > quote_str else 'SELL' if quote_str > base_str else 'NEUTRAL'
        quality = 'EXCELLENT' if gap > 40 else 'GOOD' if gap > 30 else 'ACCEPTABLE' if gap > 20 else 'POOR'
        
        signals.append({
            'pair': pair,
            'signal': signal_type,
            'quality': quality,
            'gap': round(gap, 1),
            'base_strength': round(base_str, 1),
            'quote_strength': round(quote_str, 1),
            'price': round(random.uniform(1, 150), 4)
        })
    
    current_data["signals"] = sorted(signals, key=lambda x: x['gap'], reverse=True)
    
    current_data["alerts"] = [
        {
            'id': 1,
            'title': 'EUR turned STRONG',
            'severity': 'HIGH',
            'message': 'EUR strength jumped to 75 on 1h timeframe',
            'timestamp': datetime.utcnow().isoformat()
        }
    ]

init_data()

@app.get("/api/health")
async def health():
    return {"status": "healthy", "version": "2.0.0"}

@app.get("/api/dashboard")
async def get_dashboard():
    return {
        "timestamp": datetime.utcnow().isoformat(),
        "currencies": current_data["currencies"],
        "signals": current_data["signals"][:10],
        "top_buy": [s for s in current_data["signals"] if s['signal'] == 'BUY'][:5],
        "top_sell": [s for s in current_data["signals"] if s['signal'] == 'SELL'][:5],
        "alerts": current_data["alerts"],
        "stats": {
            "strongest": max(current_data["currencies"].items(), key=lambda x: x[1]['strength'])[0],
            "weakest": min(current_data["currencies"].items(), key=lambda x: x[1]['strength'])[0],
            "market_bias": "BULLISH"
        }
    }

@app.websocket("/ws/realtime/{account_id}")
async def websocket_realtime(websocket, account_id: int):
    await websocket.accept()
    active_connections.append(websocket)
    
    try:
        while True:
            for curr in current_data["currencies"]:
                change = random.uniform(-2, 2)
                current_data["currencies"][curr]['strength'] = max(0, min(100, 
                    current_data["currencies"][curr]['strength'] + change))
            
            message = {
                "type": "realtime_update",
                "timestamp": datetime.utcnow().isoformat(),
                "currencies": current_data["currencies"],
                "signals": current_data["signals"][:10],
                "top_buy": [s for s in current_data["signals"] if s['signal'] == 'BUY'][:5],
                "top_sell": [s for s in current_data["signals"] if s['signal'] == 'SELL'][:5],
                "alerts": current_data["alerts"]
            }
            
            await websocket.send_json(message)
            await asyncio.sleep(5)
    
    except Exception as e:
        print(f"Error: {e}")
    finally:
        active_connections.remove(websocket)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("app:app", host="0.0.0.0", port=8000, reload=False)
