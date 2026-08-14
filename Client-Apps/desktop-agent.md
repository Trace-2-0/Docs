# Desktop Agent Architecture

The Trace desktop client is an Electron application that runs quietly in the background on employee machines. It is responsible for initiating shifts, capturing telemetry (screenshots, idle time, active windows), and handling offline scenarios gracefully.

## Vertical Interconnection Flow

The interaction between the desktop agent and the Express backend is continuous and stateful.

1. **Authentication:** The app reads a persistent API key (`x-agent-token`) from the operating system's native secure storage (e.g., Windows Credential Manager or macOS Keychain) and injects it into every outgoing HTTP request header.
2. **Shift Initiation:** The user clocks in, creating a shift record in the backend with an open `endTime`.
3. **Heartbeat Loop:** The agent pings `POST /api/agent/heartbeat` every 30 seconds to confirm the machine is online and the employee is active.
4. **Media Ingestion:** The agent periodically captures the screen via Electron's `desktopCapturer`, converts the image buffer to Base64, and uploads it to the backend for processing.
5. **Application Tracking:** Using native OS bindings (`active-win`), the agent polls the foreground window title every 5 seconds, aggregates the data, and batch-syncs it to the backend every minute.
6. **Idle Detection:** Using Electron's `powerMonitor`, the app watches for mouse and keyboard inactivity. If inactivity exceeds a configured threshold (e.g., 5 minutes), it informs the backend to mark the period as idle.

## Offline Resilience & Wi-Fi Handling

Employees frequently work on laptops that may temporarily drop Wi-Fi. The Electron app implements an offline buffer to prevent data loss.

If a network error occurs during an upload, the agent writes the screenshot or app-usage payload to a local SQLite/JSON file on disk. The local shift timer continues running without interruption. Once the app detects the OS has regained internet connectivity, it immediately flushes the queued offline buffer to the backend, synchronizing the timestamps seamlessly.

## Graceful Disconnects & OS Shutdowns

A common edge case involves an employee shutting down their computer without explicitly clicking "Stop Shift" in the UI. 

To handle this cleanly, the Electron main process listens to OS-level power events:
```javascript
const { powerMonitor } = require('electron');

powerMonitor.on('shutdown', async () => {
  // Fire a synchronous or rapid disconnect request before the OS cuts the network card
  await api.post('/api/agent/disconnect', { reason: 'shutdown' });
});
```
This tells the backend exactly when the machine powered off, allowing it to close the shift accurately without waiting 90 minutes for the backend Cron Sweeper to declare the shift abandoned.
