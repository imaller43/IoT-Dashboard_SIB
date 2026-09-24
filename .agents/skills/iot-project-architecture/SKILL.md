---
name: iot-project-architecture
description: Detailed architectural overview, component hierarchy, and data flow of the IoT Dashboard project.
---

# 🏗️ IoT Dashboard Architecture & Data Flow

This skill provides a high-level overview of the architectural design, component hierarchy, state management, and data flow of the IoT Dashboard project. Use this context to understand how the pieces fit together before modifying the codebase.

## 1. High-Level Architecture

The project is built as a Single Page Application (SPA) using React 18, Vite, and TypeScript. It interfaces with two main data sources:

1. **EMQX MQTT Broker (Real-time Data)**: Provides live sensor readings (temperature, humidity, light density) and handles two-way communication for smart switches (with hardware override logic).
2. **InfluxDB (Historical Data)**: A time-series database providing historical data for charts and tables, queried via an HTTP API (services/api.ts).

### Middleware
A Raspberry Pi 4 running **Node-RED** acts as the middleware. It reads physical sensors, publishes to MQTT, and writes to InfluxDB.

## 2. Component Hierarchy

- **`main.tsx`**: Entry point. Sets up `BrowserRouter`, `ClerkProvider` (Auth), `MqttProvider` (Global MQTT Context), and the Theme State. Defines the `/login` route and the protected `/*` route (wrapped in `<SignedIn>`).
- **`App.tsx`**: The main layout. Contains the Header (MQTT/DB status, Clock, Theme Toggle, Notification Bell, User Profile), Time Range Dropdown, and Room Tabs (Design Room, Room 2, Room 3). Renders `RoomDashboard` based on the active tab.
  - **`RoomDashboard.tsx`**: The core dashboard view for a single room. Fetches InfluxDB data on mount and interval.
    - **`SwitchControl.tsx`**: (Only for Room 1) UI for controlling lights/fans, reacting to hardware override states.
    - **`GaugeWidget.tsx`**: Renders real-time gauge meters (using `react-gauge-component`).
    - **`DataTableWidget.tsx`**: Displays recent historical data in a tabular format.
    - **`ChartWidget.tsx`**: Renders responsive line charts (using `recharts`) for combined Temp/Humidity, Light Density, and Mean Hourly Data.

## 3. State Management & Data Flow

### Real-time Data (MQTT)
Managed globally by **`MqttContext.tsx`**.
- Subscribes to `sapura/bilik1/data`, `bilik2`, `bilik3` and switch topics on mount.
- Parses incoming JSON messages and stores them in `roomsData` state object.
- Manages `switchStates` including `interruptSoftware`, `interruptHardware` (physical switch), and `interruptTempOverride` (high temp).
- Provides a `publishSwitch` function to send commands back to Node-RED.

### Historical Data (InfluxDB)
Managed locally within **`RoomDashboard.tsx`**.
- Calls `fetchHistoricalData` and `fetchMeanHourlyData` (from `services/api.ts`).
- Uses a polling interval that scales dynamically based on the selected `timeRange` to optimize performance and reduce backend load (e.g., 1 min for recent data, 5 mins for 7-day data).
- The InfluxDB queries **must** use `aggregateWindow` (handled in API/backend) to downsample data and prevent browser freezing.

## 4. Authentication & Security
- **Clerk**: Handles all authentication via `@clerk/clerk-react`. The `PUBLISHABLE_KEY` is loaded from `.env`.
- **Auto-Logout**: Implemented via `hooks/useIdleTimeout.ts`, tracking mouse and keyboard activity, logging out after 1 hour of inactivity.

## 5. File Structure
```
src/
├── components/          # Reusable UI widgets (Charts, Gauges, Tables, Switches)
├── context/             # Global Contexts (MqttContext)
├── hooks/               # Custom React Hooks (useIdleTimeout)
├── services/            # API clients and data fetching logic (api.ts for InfluxDB)
├── types/               # TypeScript interfaces
├── App.tsx              # Main Layout & Tab Routing
├── main.tsx             # Entry point, Auth Setup, Providers
├── index.css            # Global Styles and CSS Variables (Dark/Light themes)
└── firebase-messaging-sw.js # Service Worker for Web Push Notifications
```

## 6. Styling approach
- No CSS Modules or Styled Components. Uses standard CSS (`index.css`) with heavily utilized **CSS Variables** (`var(--bg-color)`) to seamlessly switch between Light and Dark themes without React re-renders for styling.
- Responsive design via CSS Grid and media queries (stacking to 1 column on `< 1024px`).
