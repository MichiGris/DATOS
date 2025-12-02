// ESP32Monitor.js - REEMPLAZA tu archivo actual en GitHub
import React, { useState, useEffect, useRef } from 'react';
import { 
  Activity, Droplets, Thermometer, Zap, 
  CloudRain, AlertTriangle, Wifi, WifiOff, 
  Download, RefreshCw, AlertCircle 
} from 'lucide-react';

const ESP32Monitor = () => {
  const [connected, setConnected] = useState(false);
  const [ipAddress, setIpAddress] = useState('10.80.123.191'); // Cambia esto
  const [sensorData, setSensorData] = useState({
    caudal: { rpm: 0, flujo: 0, estado: 'SIN FLUJO' },
    temperatura: { valor: 0, estado: 'NORMAL' },
    corriente: { valor: 0 },
    turbidez: { estado: 'Normal', alerta: false },
    nivel: { estado: 'Normal', alerta: false }
  });
  const [historial, setHistorial] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');
  const [lastUpdate, setLastUpdate] = useState('');
  const [rssi, setRssi] = useState(0);
  
  const fetchController = useRef(null);

  // Función para obtener datos del ESP32
  const fetchDataFromESP32 = async () => {
    if (!connected || !ipAddress) return;
    
    if (fetchController.current) {
      fetchController.current.abort();
    }
    fetchController.current = new AbortController();
    
    try {
      const response = await fetch(`http://${ipAddress}/api/data`, {
        signal: fetchController.current.signal,
        headers: { 'Accept': 'application/json' }
      });
      
      if (!response.ok) throw new Error(`Error: ${response.status}`);
      
      const data = await response.json();
      
      // Actualizar datos
      setSensorData({
        caudal: { 
          rpm: data.caudal_rpm || 0, 
          flujo: data.caudal || 0, 
          estado: data.caudal_estado || 'SIN FLUJO' 
        },
        temperatura: { 
          valor: data.temperatura || 0, 
          estado: data.temp_estado || 'NORMAL' 
        },
        corriente: { 
          valor: data.corriente || 0 
        },
        turbidez: { 
          estado: data.turbidez || 'Normal', 
          alerta: data.turbidez_alerta || false 
        },
        nivel: { 
          estado: data.nivel || 'Normal', 
          alerta: data.nivel_alerta || false 
        }
      });
      
      // Simular RSSI (en un sistema real vendría del ESP32)
      setRssi(Math.floor(Math.random() * -30) - 50);
      
      // Agregar al historial
      const timestamp = new Date().toLocaleTimeString();
      const fecha = new Date().toLocaleDateString();
      
      setHistorial(prev => {
        const nuevo = [...prev, {
          timestamp,
          fecha,
          caudal: { flujo: data.caudal || 0, rpm: data.caudal_rpm || 0 },
          temperatura: { valor: data.temperatura || 0 },
          corriente: { valor: data.corriente || 0 },
          turbidez: { estado: data.turbidez || 'Normal' },
          nivel: { estado: data.nivel || 'Normal' }
        }];
        return nuevo.slice(-20);
      });
      
      setLastUpdate(timestamp);
      setError('');
      
    } catch (err) {
      if (err.name !== 'AbortError') {
        console.error('Error:', err);
        setError('No se pudo conectar al ESP32. Verifica la IP y conexión.');
        setConnected(false);
      }
    }
  };

  // Polling cuando está conectado
  useEffect(() => {
    if (!connected) return;
    
    const interval = setInterval(fetchDataFromESP32, 1000);
    
    return () => {
      clearInterval(interval);
      if (fetchController.current) {
        fetchController.current.abort();
      }
    };
  }, [connected, ipAddress]);

  const conectar = async () => {
    setLoading(true);
    setError('');
    
    try {
      // Probar conexión
      const response = await fetch(`http://${ipAddress}/`, {
        method: 'GET',
        signal: AbortSignal.timeout(3000)
      });
      
      if (response.ok) {
        setConnected(true);
        setHistorial([]);
      } else {
        setError('El ESP32 no responde correctamente');
      }
    } catch (err) {
      setError(`No se puede alcanzar el ESP32 en ${ipAddress}`);
    } finally {
      setLoading(false);
    }
  };

  const desconectar = () => {
    setConnected(false);
    setError('');
    if (fetchController.current) {
      fetchController.current.abort();
    }
  };

  const descargarHistorial = () => {
    if (historial.length === 0) {
      alert('No hay datos para descargar');
      return;
    }

    const encabezados = ['Fecha', 'Hora', 'Caudal (L/min)', 'RPM', 'Temperatura (°C)', 'Corriente (A)', 'Turbidez', 'Nivel'];
    
    const filasCSV = historial.map(item => [
      item.fecha,
      item.timestamp,
      item.caudal.flujo.toFixed(2),
      item.caudal.rpm.toFixed(2),
      item.temperatura.valor.toFixed(2),
      item.corriente.valor.toFixed(3),
      item.turbidez.estado,
      item.nivel.estado
    ]);

    const contenidoCSV = [
      encabezados.join(','),
      ...filasCSV.map(fila => fila.join(','))
    ].join('\n');

    const blob = new Blob([contenidoCSV], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    
    link.href = url;
    link.download = `datos_esp32_${new Date().toISOString().split('T')[0]}.csv`;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
  };

  // IPs comunes para probar
  const probarIPsComunes = async () => {
    const ips = ['192.168.1.100', '192.168.0.100', '192.168.1.1', '192.168.0.1'];
    
    for (const ip of ips) {
      try {
        const res = await fetch(`http://${ip}/`, { 
          method: 'HEAD',
          signal: AbortSignal.timeout(1000) 
        });
        if (res.ok) {
          setIpAddress(ip);
          setError(`✅ ESP32 encontrado en ${ip}`);
          return;
        }
      } catch {}
    }
    setError('❌ No se encontró el ESP32 en IPs comunes');
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-slate-900 via-blue-900 to-slate-900 text-white p-4 md:p-6">
      {/* Header */}
      <div className="max-w-7xl mx-auto mb-6 md:mb-8">
        <div className="bg-slate-800/50 backdrop-blur rounded-lg p-4 md:p-6 border border-blue-500/30">
          <h1 className="text-2xl md:text-3xl font-bold mb-3 md:mb-4 flex items-center gap-2 md:gap-3">
            <Activity className="w-6 h-6 md:w-8 md:h-8 text-blue-400" />
            Monitor ESP32 - Sistema Integrado de Sensores
          </h1>
          
          {/* Panel de Conexión */}
          <div className="flex flex-col md:flex-row md:items-center gap-3 md:gap-4">
            <div className="flex-1">
              <div className="flex gap-2">
                <input
                  type="text"
                  value={ipAddress}
                  onChange={(e) => setIpAddress(e.target.value)}
                  placeholder="IP del ESP32"
                  disabled={connected || loading}
                  className="px-3 md:px-4 py-2 bg-slate-700 rounded border border-slate-600 focus:border-blue-500 outline-none disabled:opacity-50 flex-1 text-sm md:text-base"
                />
                <button
                  onClick={probarIPsComunes}
                  disabled={connected || loading}
                  className="px-3 md:px-4 py-2 bg-blue-600 hover:bg-blue-700 rounded flex items-center gap-1 md:gap-2 disabled:opacity-50 transition-all"
                  title="Buscar ESP32 automáticamente"
                >
                  <RefreshCw className="w-4 h-4" />
                  <span className="hidden md:inline">Escanear</span>
                </button>
              </div>
            </div>
            
            <button
              onClick={connected ? desconectar : conectar}
              disabled={loading}
              className={`px-4 md:px-6 py-2 rounded font-semibold flex items-center justify-center gap-2 transition-all ${
                connected 
                  ? 'bg-red-600 hover:bg-red-700' 
                  : 'bg-green-600 hover:bg-green-700'
              } ${loading ? 'opacity-50 cursor-not-allowed' : ''}`}
            >
              {loading ? (
                <>
                  <div className="w-4 h-4 md:w-5 md:h-5 border-2 border-white border-t-transparent rounded-full animate-spin" />
                  <span className="hidden md:inline">Conectando...</span>
                </>
              ) : connected ? (
                <>
                  <WifiOff className="w-4 h-4 md:w-5 md:h-5" />
                  <span>Desconectar</span>
                </>
              ) : (
                <>
                  <Wifi className="w-4 h-4 md:w-5 md:h-5" />
                  <span>Conectar</span>
                </>
              )}
            </button>
            
            <div className={`flex items-center gap-2 px-3 md:px-4 py-2 rounded ${
              connected ? 'bg-green-600/20 text-green-400' : 'bg-slate-700 text-slate-400'
            }`}>
              <div className={`w-2 h-2 md:w-3 md:h-3 rounded-full ${connected ? 'bg-green-400 animate-pulse' : 'bg-slate-500'}`} />
              <span>{connected ? 'Conectado' : 'Desconectado'}</span>
            </div>
          </div>
          
          {/* Mensajes de estado */}
          {error && (
            <div className="mt-3 md:mt-4 p-2 md:p-3 bg-red-900/30 border border-red-500/50 rounded text-red-300 text-sm md:text-base flex items-start gap-2">
              <AlertCircle className="w-4 h-4 md:w-5 md:h-5 mt-0.5 flex-shrink-0" />
              <span>{error}</span>
            </div>
          )}
          
          {connected && (
            <div className="mt-2 md:mt-3 text-xs md:text-sm text-slate-400 flex flex-wrap gap-3 md:gap-4">
              {lastUpdate && <span>🕐 Última actualización: {lastUpdate}</span>}
              <span>📶 Señal: {rssi} dBm</span>
              <span>📊 Registros: {historial.length}</span>
            </div>
          )}
        </div>
      </div>

      {/* Paneles de Sensores */}
      <div className="max-w-7xl mx-auto grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4 md:gap-6 mb-6 md:mb-8">
        {/* Caudal */}
        <div className="bg-slate-800/50 backdrop-blur rounded-lg p-4 md:p-6 border border-blue-500/30">
          <div className="flex items-center gap-2 md:gap-3 mb-3 md:mb-4">
            <Droplets className="w-5 h-5 md:w-6 md:h-6 text-blue-400" />
            <h2 className="text-lg md:text-xl font-bold">Caudal</h2>
          </div>
          <div className="space-y-1 md:space-y-2">
            <div className="text-2xl md:text-3xl font-bold text-blue-400">
              {sensorData.caudal.flujo.toFixed(1)} <span className="text-base md:text-lg">L/min</span>
            </div>
            <div className="text-slate-400 text-sm md:text-base">
              RPM: {sensorData.caudal.rpm.toFixed(1)}
            </div>
            <div className={`inline-block px-2 md:px-3 py-1 rounded text-xs md:text-sm font-semibold ${
              sensorData.caudal.estado === 'ALTO' ? 'bg-green-600' :
              sensorData.caudal.estado === 'MODERADO' ? 'bg-yellow-600' :
              'bg-blue-600'
            }`}>
              {sensorData.caudal.estado}
            </div>
          </div>
        </div>

        {/* Temperatura */}
        <div className="bg-slate-800/50 backdrop-blur rounded-lg p-4 md:p-6 border border-orange-500/30">
          <div className="flex items-center gap-2 md:gap-3 mb-3 md:mb-4">
            <Thermometer className="w-5 h-5 md:w-6 md:h-6 text-orange-400" />
            <h2 className="text-lg md:text-xl font-bold">Temperatura</h2>
          </div>
          <div className="space-y-1 md:space-y-2">
            <div className="text-2xl md:text-3xl font-bold text-orange-400">
              {sensorData.temperatura.valor.toFixed(1)} <span className="text-base md:text-lg">°C</span>
            </div>
            <div className={`inline-block px-2 md:px-3 py-1 rounded text-xs md:text-sm font-semibold ${
              sensorData.temperatura.estado === 'CALIENTE' ? 'bg-red-600' :
              sensorData.temperatura.estado === 'FRÍO' ? 'bg-blue-600' :
              'bg-green-600'
            }`}>
              {sensorData.temperatura.estado}
            </div>
          </div>
        </div>

        {/* Corriente */}
        <div className="bg-slate-800/50 backdrop-blur rounded-lg p-4 md:p-6 border border-yellow-500/30">
          <div className="flex items-center gap-2 md:gap-3 mb-3 md:mb-4">
            <Zap className="w-5 h-5 md:w-6 md:h-6 text-yellow-400" />
            <h2 className="text-lg md:text-xl font-bold">Corriente</h2>
          </div>
          <div className="space-y-1 md:space-y-2">
            <div className="text-2xl md:text-3xl font-bold text-yellow-400">
              {sensorData.corriente.valor.toFixed(3)} <span className="text-base md:text-lg">A RMS</span>
            </div>
            <div className="text-slate-400 text-sm md:text-base">Medición continua</div>
          </div>
        </div>

        {/* Turbidez */}
        <div className="bg-slate-800/50 backdrop-blur rounded-lg p-4 md:p-6 border border-cyan-500/30">
          <div className="flex items-center gap-2 md:gap-3 mb-3 md:mb-4">
            <CloudRain className="w-5 h-5 md:w-6 md:h-6 text-cyan-400" />
            <h2 className="text-lg md:text-xl font-bold">Turbidez</h2>
          </div>
          <div className="space-y-1 md:space-y-2">
            <div className={`text-xl md:text-2xl font-bold ${
              sensorData.turbidez.alerta ? 'text-red-400' : 'text-cyan-400'
            }`}>
              {sensorData.turbidez.estado}
            </div>
            {sensorData.turbidez.alerta && (
              <div className="flex items-center gap-1 md:gap-2 text-red-400 text-sm md:text-base">
                <AlertTriangle className="w-4 h-4 md:w-5 md:h-5" />
                <span>Nivel por encima del umbral</span>
              </div>
            )}
          </div>
        </div>

        {/* Nivel */}
        <div className="bg-slate-800/50 backdrop-blur rounded-lg p-4 md:p-6 border border-purple-500/30">
          <div className="flex items-center gap-2 md:gap-3 mb-3 md:mb-4">
            <Activity className="w-5 h-5 md:w-6 md:h-6 text-purple-400" />
            <h2 className="text-lg md:text-xl font-bold">Nivel</h2>
          </div>
          <div className="space-y-1 md:space-y-2">
            <div className={`text-xl md:text-2xl font-bold ${
              sensorData.nivel.alerta ? 'text-red-400' : 'text-purple-400'
            }`}>
              {sensorData.nivel.estado}
            </div>
            {sensorData.nivel.alerta && (
              <div className="flex items-center gap-1 md:gap-2 text-red-400 text-sm md:text-base animate-pulse">
                <AlertTriangle className="w-4 h-4 md:w-5 md:h-5" />
                <span>ALARMA ACTIVADA</span>
              </div>
            )}
          </div>
        </div>
      </div>

      {/* Historial */}
      <div className="max-w-7xl mx-auto">
        <div className="bg-slate-800/50 backdrop-blur rounded-lg p-4 md:p-6 border border-blue-500/

