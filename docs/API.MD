# 📡 API Documentation

> **Documentazione completa delle API del sistema Catania Smart Waste Dashboard**

## 📋 Indice

- [🌐 API JavaScript Pubblica](#-api-javascript-pubblica)
- [🔥 Firebase Realtime Database](#-firebase-realtime-database)
- [🗺️ OpenStreetMap & Routing](#️-openstreetmap--routing)
- [📊 Data Models](#-data-models)
- [🔌 Webhook & Events](#-webhook--events)
- [📱 Mobile Integration](#-mobile-integration)
- [🛠️ Development API](#️-development-api)

## 🌐 API JavaScript Pubblica

### `window.CataniaWasteAPI`

API pubblica JavaScript disponibile globalmente per integrazioni esterne.

#### **getStatus()**
Restituisce lo stato attuale del sistema.

```javascript
const status = window.CataniaWasteAPI.getStatus();

// Risposta
{
  "connected": true,
  "totalPoints": 1404,
  "activeTrucks": 6,
  "realDataCoverage": "93.6%",
  "lastUpdate": "2024-12-19T10:30:00Z"
}
```

#### **addReport(lat, lng, type, issue)**
Aggiunge una segnalazione cittadino al sistema.

```javascript
// Parametri
const success = window.CataniaWasteAPI.addReport(
  37.5079,           // Latitudine
  15.0830,           // Longitudine  
  'organico',        // Tipo rifiuto
  'full'            // Tipo problema
);

// Tipi problema supportati
['full', 'broken', 'empty', 'overflow', 'damage']
```

#### **forceRoute(truckId)**
Forza il calcolo di un nuovo percorso per un camion specifico.

```javascript
const success = window.CataniaWasteAPI.forceRoute('CT-001');

// Ritorna: boolean (true se successo)
```

#### **exportData()**
Esporta tutti i dati del sistema in formato JSON.

```javascript
const data = window.CataniaWasteAPI.exportData();

// Struttura risposta
{
  "timestamp": "2024-12-19T10:30:00Z",
  "wastePoints": [...],  // Array punti raccolta
  "trucks": [...],       // Array camion
  "routes": [...],       // Percorsi attivi
  "stats": {...}         // Statistiche sistema
}
```

### Debug & Development API

#### **window.debugDashboard()**
Debug completo del sistema con statistiche dettagliate.

```javascript
const debugInfo = window.debugDashboard();

// Ritorna oggetto con statistiche complete
{
  "summary": "1404 punti, 6 camion attivi",
  "dataQuality": "93.6% dati reali", 
  "performance": "OK"
}
```

#### **window.testSystemIntegrity()**
Esegue test automatici di integrità del sistema.

```javascript
const testResult = window.testSystemIntegrity();

// Ritorna: boolean
// true = tutti i test superati
// false = errori rilevati (check console per dettagli)
```

## 🔥 Firebase Realtime Database

### Struttura Database

```
catania-smart-waste/
├── trucks/
│   ├── CT-001/
│   │   ├── id: "CT-001"
│   │   ├── lat: 37.5079
│   │   ├── lng: 15.0830
│   │   ├── status: "active"
│   │   ├── currentLoad: 45.2
│   │   ├── targetType: "organico"
│   │   └── lastUpdate: "2024-12-19T10:30:00Z"
│   └── ...
├── wastePoints/
│   ├── utenza_157193_1/
│   │   ├── id: "utenza_157193_1"
│   │   ├── lat: 37.5187553
│   │   ├── lng: 15.1005637
│   │   ├── type: "organico"
│   │   ├── fillLevel: 75.2
│   │   ├── status: "active"
│   │   └── lastSync: "2024-12-19T10:30:00Z"
│   └── ...
├── iotSensors/
│   ├── noiseLevel: 65.4
│   ├── trafficLevel: 78.2
│   ├── crowdLevel: 45.1
│   └── timestamp: "2024-12-19T10:30:00Z"
├── manualReports/
│   ├── report_1703068200000/
│   │   ├── id: 1703068200000
│   │   ├── location: {lat: 37.5079, lng: 15.0830}
│   │   ├── issue: "full"
│   │   ├── type: "organico"
│   │   └── timestamp: "2024-12-19T10:30:00Z"
│   └── ...
└── routes/
    ├── CT-001/
    │   ├── truckId: "CT-001"
    │   ├── points: [...]
    │   ├── totalDistance: 12.5
    │   ├── estimatedTime: 45.2
    │   └── createdAt: "2024-12-19T10:30:00Z"
    └── ...
```

### Firebase Rules

```json
{
  "rules": {
    ".read": true,
    ".write": "auth != null",
    "trucks": {
      "$truckId": {
        ".validate": "newData.hasChildren(['id', 'lat', 'lng', 'status'])"
      }
    },
    "wastePoints": {
      "$pointId": {
        ".validate": "newData.hasChildren(['id', 'lat', 'lng', 'type', 'fillLevel'])"
      }
    },
    "manualReports": {
      "$reportId": {
        ".write": true,
        ".validate": "newData.hasChildren(['location', 'issue', 'timestamp'])"
      }
    }
  }
}
```

### Listeners Setup

```javascript
// Listener camion
firebase.database().ref('trucks').on('value', (snapshot) => {
  const trucks = snapshot.val();
  dashboard.updateTrucksFromFirebase(trucks);
});

// Listener punti raccolta
firebase.database().ref('wastePoints').on('value', (snapshot) => {
  const points = snapshot.val();
  dashboard.updateWastePointsFromFirebase(points);
});

// Listener segnalazioni
firebase.database().ref('manualReports').on('child_added', (snapshot) => {
  const report = snapshot.val();
  dashboard.handleManualReports(report);
});
```

## 🗺️ OpenStreetMap & Routing

### OSRM Integration

#### **Route Calculation**
```javascript
// URL OSRM per calcolo percorsi
const osrmUrl = `https://router.project-osrm.org/route/v1/driving/${startLng},${startLat};${endLng},${endLat}`;

// Parametri supportati
{
  overview: 'full',           // Geometria completa
  geometries: 'geojson',      // Formato coordinate
  steps: true,                // Indicazioni dettagliate
  alternatives: false         // Solo percorso migliore
}
```

#### **Response Format**
```json
{
  "routes": [{
    "geometry": {
      "coordinates": [[15.0830, 37.5079], ...],
      "type": "LineString"
    },
    "distance": 12543.2,      // Metri
    "duration": 1854.7,       // Secondi
    "steps": [...]            // Indicazioni turn-by-turn
  }],
  "code": "Ok"
}
```

### Leaflet Map Integration

#### **Map Setup**
```javascript
// Inizializzazione mappa
const map = L.map('map').setView([37.5079, 15.0830], 13);

// Tile layer OpenStreetMap
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
  attribution: '© OpenStreetMap contributors',
  maxZoom: 19
}).addTo(map);
```

#### **Custom Markers**
```javascript
// Marker contenitori
const wasteIcon = L.divIcon({
  html: `<div style="background: ${color}; ...">contenuto</div>`,
  className: 'waste-point-icon',
  iconSize: [20, 20],
  iconAnchor: [10, 10]
});

// Marker camion con stato
const truckIcon = L.divIcon({
  html: `<div class="truck-icon ${status}">🚛</div>`,
  iconSize: [30, 30],
  iconAnchor: [15, 15]
});
```

## 📊 Data Models

### WastePoint Model
```typescript
interface WastePoint {
  id: string;                    // Identificativo unico
  lat: number;                   // Latitudine (37.4-37.6 per Catania)
  lng: number;                   // Longitudine (14.9-15.2 per Catania)
  type: 'organico' | 'carta' | 'plastica' | 'vetro' | 'indifferenziato';
  fillLevel: number;             // Percentuale riempimento (0-100)
  fillRate: number;              // Velocità riempimento (unità/ora)
  lastCollection: Date;          // Ultima raccolta
  zone: string;                  // Zona geografica
  priority: 'high' | 'normal';   // Priorità raccolta
  status: 'active' | 'broken' | 'full' | 'empty';
  gravitationalPull: number;     // Forza attrazione algoritmo
  resetTimer: number;            // Timer reset automatico (secondi)
  density: 'alta' | 'media' | 'bassa';
  source: 'excel_data' | 'firebase' | 'manual_report' | 'fallback';
  
  // Dati aggiuntivi per punti reali
  address?: string;              // Indirizzo completo
  categoria?: string;            // Categoria attività
  kitId?: string;               // ID kit contenitori
  quantita?: number;            // Numero contenitori
  originalTypes?: string[];      // Tipi rifiuto multipli
  isCentroRaccolta?: boolean;   // Flag centro comunale
}
```

### Truck Model
```typescript
interface Truck {
  id: string;                    // ID camion (formato: CT-001)
  lat: number;                   // Posizione corrente lat
  lng: number;                   // Posizione corrente lng
  status: 'active' | 'loading' | 'maintenance';
  capacity: number;              // Capacità massima (litri/kg)
  currentLoad: number;           // Carico attuale (%)
  driver: string;                // Nome operatore
  lastUpdate: Date;              // Ultimo aggiornamento posizione
  route: WastePoint[];          // Percorso assegnato
  targetType: string;           // Tipo rifiuto specializzato
  speed: number;                // Velocità movimento (multiplier)
  efficiency: number;           // Efficienza operativa (0-1)
  isOnRoute: boolean;           // Flag percorso in corso
  pacmanMode: boolean;          // Flag modalità Pac-Man attiva
  firebaseSync: boolean;        // Flag sincronizzazione FB
}
```

### Route Model
```typescript
interface RouteResult {
  route: WastePoint[];          // Sequenza punti ottimizzata
  totalDistance: number;        // Distanza totale (km)
  totalTime: number;           // Tempo stimato (minuti)
  algorithm: string;           // Algoritmo utilizzato
  segments?: RouteSegment[];   // Segmenti percorso dettagliati
  efficiency: number;          // Score efficienza
  createdAt: Date;            // Timestamp creazione
}

interface RouteSegment {
  distance: number;            // Distanza segmento (km)
  duration: number;           // Tempo segmento (minuti)
  coordinates: number[][];    // Coordinate GPS percorso
  geometry?: any;             // Geometria GeoJSON
}
```

### IoT Sensor Data
```typescript
interface IoTSensorData {
  noiseLevel: number;          // Livello rumore (dB)
  crowdLevel: number;          // Livello affluenza (%)
  trafficLevel: number;        // Livello traffico (%)
  timestamp: Date;             // Timestamp lettura
  
  // Dati aggiuntivi opzionali
  temperature?: number;        // Temperatura (°C)
  humidity?: number;          // Umidità (%)
  airQuality?: number;        // Qualità aria (AQI)
}
```

## 🔌 Webhook & Events

### Event System

Il sistema supporta eventi custom per integrazioni esterne:

```javascript
// Event Listeners
document.addEventListener('wastePointUpdated', (event) => {
  console.log('Punto aggiornato:', event.detail);
});

document.addEventListener('routeCalculated', (event) => {
  console.log('Nuovo percorso:', event.detail);
});

document.addEventListener('emergencyReport', (event) => {
  console.log('Segnalazione emergenza:', event.detail);
});

// Event Dispatching
const event = new CustomEvent('wastePointUpdated', {
  detail: {
    pointId: 'utenza_157193_1',
    fillLevel: 85.3,
    timestamp: new Date()
  }
});
document.dispatchEvent(event);
```

### Webhook Configuration

Per integrazioni server-side, il sistema può essere esteso con webhook:

```javascript
// Configurazione webhook (esempio)
const webhookConfig = {
  emergencyReports: 'https://api.comune.catania.it/waste/emergency',
  routeCompleted: 'https://api.comune.catania.it/waste/completed',
  systemAlerts: 'https://api.comune.catania.it/waste/alerts'
};

// Invio webhook
async function sendWebhook(endpoint, data) {
  try {
    const response = await fetch(endpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer YOUR_API_KEY'
      },
      body: JSON.stringify(data)
    });
    
    return response.ok;
  } catch (error) {
    console.error('Webhook error:', error);
    return false;
  }
}
```

## 📱 Mobile Integration

### Responsive API

L'API è ottimizzata per dispositivi mobile:

```javascript
// Rilevamento dispositivo
const isMobile = window.innerWidth <= 768;

// Configurazione mobile-friendly
if (isMobile) {
  dashboard.optimizeForMobile({
    reducedMarkers: true,
    simplifiedUI: true,
    touchOptimized: true
  });
}
```

### PWA Support

Il sistema può essere esteso come Progressive Web App:

```json
// manifest.json
{
  "name": "Catania Smart Waste",
  "short_name": "CataniaWaste",
  "description": "Sistema gestione rifiuti Catania",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#667eea",
  "theme_color": "#764ba2",
  "icons": [
    {
      "src": "icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    }
  ]
}
```

## 🛠️ Development API

### Configuration API

```javascript
// Overriding configurazioni default
window.CataniaWasteConfig = {
  firebase: {
    // Configurazione Firebase custom
  },
  routing: {
    provider: 'osrm',  // 'osrm' | 'google' | 'mapbox'
    timeout: 10000,    // Timeout richieste (ms)
    maxRetries: 3      // Retry automatici
  },
  ui: {
    theme: 'default',  // 'default' | 'dark' | 'light'
    animations: true,  // Abilita animazioni
    notifications: true // Abilita notifiche
  }
};
```

### Plugin System

```javascript
// Registrazione plugin custom
window.CataniaWasteAPI.registerPlugin('customAnalytics', {
  onRouteCalculated: (routeData) => {
    // Logica analytics custom
  },
  onWasteCollected: (pointData) => {
    // Tracking raccolta custom
  }
});
```

### Error Handling

```javascript
// Error events
document.addEventListener('dashboardError', (event) => {
  console.error('Dashboard error:', event.detail);
  
  // Gestione errori custom
  switch(event.detail.type) {
    case 'firebase_connection':
      // Gestisci errore Firebase
      break;
    case 'routing_failed':
      // Gestisci errore routing
      break;
    case 'data_validation':
      // Gestisci errore validazione
      break;
  }
});
```

---

## 📚 Esempi Completi

### Integrazione Base

```html
<!DOCTYPE html>
<html>
<head>
  <title>Integrazione Catania Waste API</title>
</head>
<body>
  <div id="dashboard-container"></div>
  
  <script src="catania-smart-waste.js"></script>
  <script>
    // Inizializzazione
    const dashboard = new CataniaSmartWasteDashboard({
      container: 'dashboard-container',
      firebase: {
        // Configurazione Firebase
      }
    });
    
    // Utilizzo API
    dashboard.on('ready', () => {
      const status = window.CataniaWasteAPI.getStatus();
      console.log('Dashboard ready:', status);
      
      // Aggiungi segnalazione test
      setTimeout(() => {
        window.CataniaWasteAPI.addReport(37.5079, 15.0830, 'organico', 'full');
      }, 5000);
    });
  </script>
</body>
</html>
```

### Integrazione Avanzata con React

```jsx
import React, { useEffect, useState } from 'react';

function CataniaWasteIntegration() {
  const [dashboardStatus, setDashboardStatus] = useState(null);
  
  useEffect(() => {
    // Listener eventi dashboard
    const handleStatusUpdate = (event) => {
      setDashboardStatus(event.detail);
    };
    
    document.addEventListener('dashboardStatusUpdate', handleStatusUpdate);
    
    // Polling status
    const interval = setInterval(() => {
      if (window.CataniaWasteAPI) {
        const status = window.CataniaWasteAPI.getStatus();
        setDashboardStatus(status);
      }
    }, 10000);
    
    return () => {
      document.removeEventListener('dashboardStatusUpdate', handleStatusUpdate);
      clearInterval(interval);
    };
  }, []);
  
  const handleEmergencyReport = () => {
    const success = window.CataniaWasteAPI.addReport(
      37.5079, 15.0830, 'organico', 'full'
    );
    
    if (success) {
      alert('Segnalazione inviata con successo!');
    }
  };
  
  return (
    <div>
      <h2>Status Dashboard</h2>
      {dashboardStatus && (
        <div>
          <p>Punti totali: {dashboardStatus.totalPoints}</p>
          <p>Camion attivi: {dashboardStatus.activeTrucks}</p>
          <p>Copertura dati: {dashboardStatus.realDataCoverage}</p>
        </div>
      )}
      
      <button onClick={handleEmergencyReport}>
        Segnala Emergenza
      </button>
    </div>
  );
}
```

---

**🌱 Sviluppato dal Gruppo Marie Curie per Hackathon Green Mind Catania**
