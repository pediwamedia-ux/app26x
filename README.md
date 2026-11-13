```react
import React, { useState, useEffect, useRef, useCallback } from 'react';
import { 
  Activity, 
  Settings, 
  TrendingDown, 
  Terminal, 
  Crosshair, 
  AlertTriangle,
  Play,
  Square,
  Wifi,
  WifiOff,
  Target
} from 'lucide-react';
import { 
  LineChart, 
  Line, 
  XAxis, 
  YAxis, 
  CartesianGrid, 
  Tooltip, 
  ResponsiveContainer,
  ReferenceArea
} from 'recharts';

export default function App() {
  // Connection State
  const [apiKey, setApiKey] = useState('');
  const [isConnected, setIsConnected] = useState(false);
  const [connectionStatus, setConnectionStatus] = useState('Disconnected');
  const wsRef = useRef(null);

  // Market Data State
  const [livePrice, setLivePrice] = useState(null);
  const [chartData, setChartData] = useState([]);
  
  // Strategy Parameters
  const [h1Upper, setH1Upper] = useState(51.450);
  const [h1Lower, setH1Lower] = useState(51.350);
  
  // Algorithmic State
  const [currentPhase, setCurrentPhase] = useState(2); // Starts at Phase 2 (Wait State)
  const [alerts, setAlerts] = useState([]);
  const alertsEndRef = useRef(null);

  // --- WebSocket Logic ---
  const connectWebSocket = useCallback(() => {
    if (!apiKey) {
      addAlert('API Key is required to connect.', 'error');
      return;
    }

    setConnectionStatus('Connecting...');
    const wsUrl = `wss://ws.twelvedata.com/v1/quotes/price?apikey=${apiKey}`;
    wsRef.current = new WebSocket(wsUrl);

    wsRef.current.onopen = () => {
      setIsConnected(true);
      setConnectionStatus('Connected');
      addAlert('WebSocket Connected successfully.', 'success');
      
      // Subscribe to Silver Spot
      const subscribeMsg = {
        action: 'subscribe',
        params: {
          symbols: 'XAG/USD'
        }
      };
      wsRef.current.send(JSON.stringify(subscribeMsg));
      addAlert('Subscribed to XAG/USD real-time feed.', 'info');
    };

    wsRef.current.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        
        // Handle subscription success messages
        if (data.event === 'subscribe-status' && data.status === 'ok') {
            addAlert(`Subscription confirmed for ${data.success[0].symbol}`, 'success');
            return;
        }

        // Handle Price Ticks
        if (data.event === 'price' && data.price) {
          const price = parseFloat(data.price);
          const timeLabel = new Date(data.timestamp * 1000).toLocaleTimeString([], { hour12: false, hour: '2-digit', minute: '2-digit', second: '2-digit' });
          
          setLivePrice(price);
          
          setChartData(prev => {
            const newPoint = { time: timeLabel, price: price };
            const newData = [...prev, newPoint];
            // Keep last 100 ticks to avoid memory leaks and chart crowding
            return newData.length > 100 ? newData.slice(newData.length - 100) : newData;
          });
        }
      } catch (e) {
        console.error('Error parsing WS message:', e);
      }
    };

    wsRef.current.onerror = (error) => {
      console.error('WebSocket Error:', error);
      addAlert('WebSocket Error occurred. Check API Key or network.', 'error');
      setConnectionStatus('Error');
    };

    wsRef.current.onclose = () => {
      setIsConnected(false);
      setConnectionStatus('Disconnected');
      addAlert('WebSocket Disconnected.', 'error');
    };
  }, [apiKey]);

  const disconnectWebSocket = useCallback(() => {
    if (wsRef.current) {
      wsRef.current.close();
      wsRef.current = null;
    }
  }, []);

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      if (wsRef.current) wsRef.current.close();
    };
  }, []);

  // --- Algorithmic Strategy Logic ---
  useEffect(() => {
    if (livePrice === null) return;

    // Check Phase 2 (Wait State) -> Phase 3 (CSD Look)
    if (currentPhase === 2) {
      if (livePrice <= h1Upper && livePrice >= h1Lower) {
        addAlert(`ALGO TRIGGER: Price (${livePrice}) entered H1 Premium Zone (${h1Lower} - ${h1Upper}).`, 'critical');
        addAlert('System shifting to Phase 3: Hunting for M15 CSD...', 'info');
        setCurrentPhase(3);
      }
    }
    
    // Auto-scroll logs
    alertsEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [livePrice, h1Upper, h1Lower, currentPhase]);

  const addAlert = (msg, type) => {
    const timeString = new Date().toLocaleTimeString([], { hour12: false, hour: '2-digit', minute: '2-digit', second: '2-digit' });
    setAlerts(prev => {
      const newAlerts = [...prev, { time: timeString, msg, type }];
      return newAlerts.length > 50 ? newAlerts.slice(newAlerts.length - 50) : newAlerts;
    });
  };

  const resetPhases = () => {
    setCurrentPhase(2);
    setChartData([]);
    setLivePrice(null);
    addAlert('Algorithm Matrix Reset. Back to Wait State.', 'info');
  };

  // UI Helpers
  const CustomTooltip = ({ active, payload, label }) => {
    if (active && payload && payload.length) {
      return (
        <div className="bg-slate-900 border border-slate-700 p-2 rounded shadow-lg text-xs font-mono">
          <p className="text-slate-400">{label}</p>
          <p className="text-blue-400 font-bold">XAG/USD: {payload[0].value.toFixed(4)}</p>
        </div>
      );
    }
    return null;
  };

  return (
    <div className="min-h-screen bg-[#0a0e17] text-slate-300 font-sans p-4 sm:p-6 lg:p-8 flex flex-col">
      
      {/* Header & API Connection */}
      <header className="bg-[#111827] border border-slate-800/80 rounded-xl p-4 shadow-xl mb-6 flex flex-col lg:flex-row justify-between items-start lg:items-center gap-4">
        <div className="flex items-center gap-3">
          <div className="p-2 bg-slate-800 rounded-lg">
            <Activity className="w-6 h-6 text-blue-500" />
          </div>
          <div>
            <h1 className="text-lg font-bold text-slate-100 tracking-tight">Real-Time SMC Engine</h1>
            <p className="text-xs text-slate-500 uppercase tracking-widest">Twelve Data Integration</p>
          </div>
        </div>

        <div className="flex flex-col sm:flex-row items-center gap-3 w-full lg:w-auto">
          <input 
            type="password"
            placeholder="Twelve Data API Key"
            value={apiKey}
            onChange={(e) => setApiKey(e.target.value)}
            disabled={isConnected}
            className="w-full sm:w-64 bg-slate-900 border border-slate-700 text-slate-200 text-sm rounded-lg px-3 py-2 focus:outline-none focus:border-blue-500"
          />
          <div className="flex items-center gap-2 w-full sm:w-auto">
            {isConnected ? (
              <button 
                onClick={disconnectWebSocket}
                className="flex-1 sm:flex-none flex items-center justify-center gap-2 bg-red-500/10 hover:bg-red-500/20 text-red-500 border border-red-500/50 px-4 py-2 rounded-lg text-sm font-bold transition-colors"
              >
                <Square className="w-4 h-4" /> Disconnect
              </button>
            ) : (
              <button 
                onClick={connectWebSocket}
                className="flex-1 sm:flex-none flex items-center justify-center gap-2 bg-blue-600 hover:bg-blue-500 text-white px-4 py-2 rounded-lg text-sm font-bold shadow-lg shadow-blue-900/20 transition-colors"
              >
                <Play className="w-4 h-4 fill-current" /> Connect Feed
              </button>
            )}
          </div>
        </div>
      </header>

      {/* Main Content Grid */}
      <div className="grid grid-cols-1 lg:grid-cols-12 gap-6 flex-1">
        
        {/* Left Column: SMC Protocol & Controls */}
        <div className="lg:col-span-4 flex flex-col gap-6">
          
          {/* Status Display */}
          <div className="bg-[#111827] border border-slate-800/80 rounded-xl p-5 shadow-xl flex items-center justify-between">
            <div>
              <span className="text-xs text-slate-500 uppercase font-bold tracking-wider">XAG/USD Live Spot</span>
              <div className={`text-3xl font-mono font-bold tracking-tight mt-1 ${livePrice ? 'text-slate-100' : 'text-slate-600'}`}>
                {livePrice ? livePrice.toFixed(4) : '---.----'}
              </div>
            </div>
            <div className="flex flex-col items-end gap-2">
               {isConnected ? <Wifi className="w-5 h-5 text-green-500" /> : <WifiOff className="w-5 h-5 text-red-500" />}
               <span className={`text-[10px] uppercase font-bold ${isConnected ? 'text-green-500' : 'text-red-500'}`}>
                 {connectionStatus}
               </span>
            </div>
          </div>

          {/* Strategy Matrix Panel */}
          <div className="bg-[#111827] border border-slate-800/80 rounded-xl p-5 shadow-xl flex-1">
            <h2 className="text-xs font-bold text-slate-400 uppercase tracking-wider mb-5 flex items-center gap-2">
              <Crosshair className="w-4 h-4 text-blue-500" /> SMC 4-Phase Protocol
            </h2>

            {/* Target Input Settings */}
            <div className="mb-6 bg-slate-900/50 p-4 rounded-lg border border-slate-800">
              <div className="flex items-center gap-2 mb-3">
                <Target className="w-4 h-4 text-purple-500" />
                <span className="text-xs font-bold text-slate-300">Define H1 Premium Array</span>
              </div>
              <div className="grid grid-cols-2 gap-3">
                <div>
                  <label className="block text-[10px] text-slate-500 uppercase mb-1">Upper Bound</label>
                  <input type="number" step="0.01" value={h1Upper} onChange={(e) => setH1Upper(parseFloat(e.target.value))} className="w-full bg-slate-900 border border-slate-700 text-slate-200 text-sm font-mono rounded px-2 py-1.5 focus:border-purple-500 focus:outline-none" />
                </div>
                <div>
                  <label className="block text-[10px] text-slate-500 uppercase mb-1">Lower Bound</label>
                  <input type="number" step="0.01" value={h1Lower} onChange={(e) => setH1Lower(parseFloat(e.target.value))} className="w-full bg-slate-900 border border-slate-700 text-slate-200 text-sm font-mono rounded px-2 py-1.5 focus:border-purple-500 focus:outline-none" />
                </div>
              </div>
            </div>

            {/* Phases */}
            <div className="space-y-4 relative">
              <div className="absolute left-[11px] top-6 bottom-4 w-px bg-slate-800 z-0"></div>

              {/* Phase 1 */}
              <div className="relative z-10 flex items-start gap-3">
                <div className="w-6 h-6 rounded-full bg-green-900/30 border border-green-500/50 flex items-center justify-center shrink-0 mt-0.5">
                  <div className="w-2 h-2 rounded-full bg-green-500"></div>
                </div>
                <div>
                  <h3 className="text-sm font-bold text-slate-200">Phase 1: Macro Bias (D1)</h3>
                  <p className="text-xs text-slate-500 mt-1">Confirmed Bearish. Looking for shorts only.</p>
                </div>
              </div>

              {/* Phase 2 */}
              <div className={`relative z-10 flex items-start gap-3 ${currentPhase === 2 ? 'opacity-100' : 'opacity-50'}`}>
                <div className={`w-6 h-6 rounded-full flex items-center justify-center shrink-0 mt-0.5 ${currentPhase === 2 ? 'bg-yellow-900/30 border-yellow-500/50' : 'bg-green-900/30 border-green-500/50'}`}>
                  {currentPhase === 2 ? <div className="w-2 h-2 rounded-full bg-yellow-500 animate-pulse"></div> : <div className="w-2 h-2 rounded-full bg-green-500"></div>}
                </div>
                <div>
                  <h3 className={`text-sm font-bold ${currentPhase === 2 ? 'text-yellow-400' : 'text-slate-200'}`}>Phase 2: The Wait State (H1)</h3>
                  <p className="text-xs text-slate-500 mt-1">
                    {currentPhase === 2 ? `Awaiting price to tap ${h1Lower} - ${h1Upper}` : `Tapped H1 Premium Array`}
                  </p>
                </div>
              </div>

              {/* Phase 3 */}
              <div className={`relative z-10 flex items-start gap-3 ${currentPhase === 3 ? 'opacity-100' : 'opacity-40'}`}>
                <div className={`w-6 h-6 rounded-full flex items-center justify-center shrink-0 mt-0.5 ${currentPhase === 3 ? 'bg-blue-900/30 border-blue-500/50' : 'bg-slate-900 border-slate-700'}`}>
                   {currentPhase === 3 && <div className="w-2 h-2 rounded-full bg-blue-500 animate-pulse"></div>}
                </div>
                <div>
                  <h3 className={`text-sm font-bold ${currentPhase === 3 ? 'text-blue-400' : 'text-slate-400'}`}>Phase 3: Orderflow Shift (M15)</h3>
                  <p className="text-xs text-slate-500 mt-1">Look for SMT Divergence & CSD displacement candle.</p>
                </div>
              </div>

              {/* Phase 4 */}
              <div className={`relative z-10 flex items-start gap-3 ${currentPhase >= 4 ? 'opacity-100' : 'opacity-30'}`}>
                <div className="w-6 h-6 rounded-full bg-slate-900 border border-slate-700 flex items-center justify-center shrink-0 mt-0.5"></div>
                <div>
                  <h3 className="text-sm font-bold text-slate-400">Phase 4: Precision Entry (M5)</h3>
                  <p className="text-xs text-slate-500 mt-1">38% Fib Retracement + iFVG alignment.</p>
                </div>
              </div>
            </div>

            <button onClick={resetPhases} className="w-full mt-6 py-2 border border-slate-700 hover:bg-slate-800 text-slate-400 text-xs font-bold rounded uppercase tracking-widest transition-colors">
              Reset Algorithm
            </button>
          </div>
        </div>

        {/* Right Column: Live Chart & Terminal */}
        <div className="lg:col-span-8 flex flex-col gap-6">
          
          {/* Real-time Chart */}
          <div className="bg-[#111827] border border-slate-800/80 rounded-xl p-4 shadow-xl flex-1 min-h-[350px] flex flex-col">
            <div className="flex items-center justify-between mb-4">
               <h2 className="text-xs font-bold text-slate-400 uppercase tracking-wider flex items-center gap-2">
                 <TrendingDown className="w-4 h-4" /> Live Tick Feed
               </h2>
               {currentPhase === 2 && (
                 <span className="text-[10px] uppercase font-bold px-2 py-1 rounded bg-yellow-500/10 text-yellow-500 border border-yellow-500/20">
                   Monitoring Phase 2 Bounds
                 </span>
               )}
            </div>
            
            <div className="flex-1 w-full h-full relative">
              {chartData.length > 0 ? (
                <ResponsiveContainer width="100%" height="100%">
                  <LineChart data={chartData} margin={{ top: 10, right: 30, left: 20, bottom: 0 }}>
                    <CartesianGrid strokeDasharray="3 3" stroke="#1e293b" vertical={false} />
                    <XAxis 
                      dataKey="time" 
                      stroke="#475569" 
                      fontSize={10} 
                      tickMargin={10} 
                      minTickGap={30}
                    />
                    <YAxis 
                      domain={['auto', 'auto']} 
                      stroke="#475569" 
                      fontSize={11}
                      tickFormatter={(val) => val.toFixed(3)}
                      width={60}
                      orientation="right"
                    />
                    <Tooltip content={<CustomTooltip />} />
                    
                    {/* Visual H1 Premium Zone */}
                    <ReferenceArea 
                      y1={h1Lower} 
                      y2={h1Upper} 
                      strokeOpacity={0.3} 
                      fill="url(#premiumPattern)" 
                      fillOpacity={0.1} 
                    />
                    <defs>
                      <pattern id="premiumPattern" patternUnits="userSpaceOnUse" width="4" height="4">
                        <path d="M-1,1 l2,-2 M0,4 l4,-4 M3,5 l2,-2" stroke="#a855f7" strokeWidth="1" opacity="0.3"/>
                      </pattern>
                    </defs>

                    <Line 
                      type="monotone" 
                      dataKey="price" 
                      stroke="#3b82f6" 
                      strokeWidth={2} 
                      dot={false}
                      isAnimationActive={false}
                    />
                  </LineChart>
                </ResponsiveContainer>
              ) : (
                <div className="absolute inset-0 flex items-center justify-center text-slate-600 flex-col gap-2">
                  <Activity className="w-8 h-8 opacity-20" />
                  <p className="text-sm font-mono">{isConnected ? 'Waiting for first tick data from market...' : 'WebSocket Disconnected'}</p>
                </div>
              )}
            </div>
          </div>

          {/* Institutional Alert Terminal */}
          <div className="bg-[#0f141e] border border-slate-800/80 rounded-xl shadow-xl flex flex-col h-64 shrink-0">
             <div className="p-3 border-b border-slate-800/60 flex items-center justify-between bg-[#111827] rounded-t-xl">
              <h2 className="text-xs font-bold text-slate-400 uppercase tracking-wider flex items-center gap-2">
                <Terminal className="w-4 h-4" /> Real-Time Engine Logs
              </h2>
            </div>
            <div className="p-3 overflow-y-auto flex-1 font-mono text-[11px] space-y-2">
              {alerts.length === 0 && (
                 <span className="text-slate-600">Awaiting system initialization...</span>
              )}
              {alerts.map((alert, i) => (
                <div key={i} className="flex items-start gap-3 py-1 px-2 bg-slate-900/30 hover:bg-slate-800/50 rounded transition-colors"
                     style={{ borderLeft: `3px solid ${alert.type === 'critical' ? '#ef4444' : alert.type === 'success' ? '#22c55e' : alert.type === 'error' ? '#f97316' : '#3b82f6'}` }}>
                  <span className="text-slate-500 shrink-0">[{alert.time}]</span>
                  <span className={`${alert.type === 'critical' ? 'text-red-400 font-bold' : alert.type === 'success' ? 'text-green-400' : alert.type === 'error' ? 'text-orange-400' : 'text-slate-300'}`}>
                    {alert.msg}
                  </span>
                </div>
              ))}
              <div ref={alertsEndRef} />
            </div>
          </div>

        </div>
      </div>
    </div>
  );
}
```