# AWS Device Farm MCP Server

A Model Context Protocol (MCP) server that provides tools for AWS Device Farm remote access sessions and mobile device automation via Appium.

## How It Works

This MCP server leverages [AWS Device Farm's managed Appium endpoint feature](https://aws.amazon.com/about-aws/whats-new/2025/11/aws-device-farm-managed-appium-endpoint/) to enable programmatic mobile device automation.

![Appium Endpoint Architecture](images/appium-endpoint.png)

### Demo

![Device Farm MCP Demo](images/demo.gif)

### Architecture Overview

1. **Create Remote Access Session** - The MCP server creates a Device Farm session and receives a managed Appium endpoint URL
2. **WebdriverIO Connection** - Connects to the Appium endpoint using WebdriverIO client over HTTPS
3. **Appium Commands** - Executes mobile automation commands (tap, swipe, type, etc.) via the W3C WebDriver protocol
4. **Real Device Control** - Commands are executed on real physical devices in AWS Device Farm

### Key Benefits

- **No Local Appium Server** - AWS manages the Appium infrastructure
- **Real Devices** - Test on actual Android and iOS devices
- **Secure HTTPS** - All communication over encrypted connections
- **Scalable** - Access to AWS Device Farm's device fleet
- **MCP Integration** - Seamless integration with AI assistants via Model Context Protocol

## Features

### Device Farm Session Management
- **create_session**: Create remote access sessions with optional APK installation
- **list_devices**: List available devices by platform
- **list_active_sessions**: View all active sessions
- **stop_session**: Stop the current session

### Mobile Device Automation
- **mobile_get_screen_size**: Get device screen dimensions
- **mobile_get_orientation**: Get current screen orientation
- **mobile_set_orientation**: Set screen orientation (PORTRAIT/LANDSCAPE)
- **mobile_save_screenshot**: Capture and save screenshots
- **mobile_list_elements_on_screen**: Get page source XML
- **mobile_click_on_screen_at_coordinates**: Click at specific coordinates
- **mobile_double_tap_on_screen**: Double tap at coordinates
- **mobile_long_press_on_screen_at_coordinates**: Long press at coordinates
- **mobile_swipe_on_screen**: Swipe gestures (up/down/left/right)
- **mobile_type_keys**: Type text input
- **mobile_press_button**: Press device buttons (home/back)

### App Management
- **mobile_launch_app**: Launch app by package ID
- **mobile_terminate_app**: Terminate running app
- **mobile_install_app**: Upload and install APK to current session
- **mobile_uninstall_app**: Uninstall app from device
- **mobile_list_apps**: Check if app is installed
- **mobile_get_device_info**: Get device information
- **install_and_launch_app**: Get instructions for manual installation

## Prerequisites

- Node.js v18 or higher
- AWS credentials configured
- AWS Device Farm project ARN

## Installation

### Option 1: NPM Package (Recommended)

```bash
npx devicefarm-mcp-server
```

No installation needed! The package will be downloaded and executed automatically.

### Option 2: From Source

```bash
git clone https://github.com/yoreland/devicefarm-mcp-server.git
cd devicefarm-mcp-server
npm install
node devicefarm-mcp-server.js
```

## Configuration

### Update Project ARN

Edit `devicefarm-mcp-server.js` and set your Device Farm project ARN:

```javascript
const PROJECT_ARN = 'arn:aws:devicefarm:us-west-2:YOUR_ACCOUNT:project:YOUR_PROJECT_ID';
```

### MCP Client Configuration

Configure the MCP server in your MCP client (e.g., Amazon Q Developer CLI `~/.aws/amazonq/mcp.json`):

```json
{
  "mcpServers": {
    "devicefarm": {
      "command": "npx",
      "args": ["devicefarm-mcp-server"],
      "env": {
        "AWS_REGION": "us-west-2",
        "AWS_PROFILE": "default"
      },
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

Or if installed from source:

```json
{
  "mcpServers": {
    "devicefarm": {
      "command": "node",
      "args": ["/path/to/devicefarm-mcp-server/devicefarm-mcp-server.js"],
      "env": {
        "AWS_REGION": "us-west-2",
        "AWS_PROFILE": "default"
      },
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

## Usage

### Create Session with APK

```javascript
// Automatically uploads and installs APK to device
create_session({
  apkPath: "./app.apk",
  platform: "ANDROID",
  sessionName: "My Test Session"
})
```

### Create Session with Pre-uploaded App

```javascript
create_session({
  appArn: "arn:aws:devicefarm:us-west-2:...:upload:...",
  deviceArn: "arn:aws:devicefarm:us-west-2::device:..."
})
```

### Mobile Automation Example

```javascript
// Take screenshot
mobile_save_screenshot({ path: "./screenshot.png" })

// Click at coordinates
mobile_click_on_screen_at_coordinates({ x: 540, y: 1200 })

// Type text
mobile_type_keys({ text: "Hello World" })

// Launch app
mobile_launch_app({ appId: "com.example.app" })
```

## Key Features

### Automatic APK Installation

When creating a session with `apkPath` or `appArn`, the server automatically:
1. Uploads APK to Device Farm (if local path provided)
2. Waits for APK processing
3. Creates remote access session
4. **Automatically installs app to device** using `InstallToRemoteAccessSession` API

No manual installation required!

### Session Management

- Maintains single active session
- Automatic Appium driver initialization
- WebSocket endpoints for interactive access
- Proper cleanup on session stop

## API Reference

### create_session

Creates a Device Farm remote access session with optional app installation.

**Parameters:**
- `deviceArn` (optional): Specific device ARN
- `appArn` (optional): Pre-uploaded app ARN
- `apkPath` (optional): Local APK file path for upload
- `sessionName` (optional): Session name
- `platform` (optional): "ANDROID" or "IOS"

**Returns:**
```json
{
  "sessionArn": "arn:aws:devicefarm:...",
  "appiumEndpoint": "https://...",
  "interactiveEndpoint": "wss://...",
  "status": "RUNNING",
  "device": "Google Pixel 10 Pro XL",
  "uploadedAppArn": "arn:aws:devicefarm:...",
  "appInstalled": true,
  "progress": ["Creating session...", "Session ready!", "App installed!"]
}
```

### mobile_install_app

Uploads and installs APK to the current active session.

**Parameters:**
- `apkPath` (required): Local APK file path

**Returns:**
```
App installed successfully. ARN: arn:aws:devicefarm:...
```

## Architecture

This MCP server bridges AI assistants with AWS Device Farm's real device cloud through the managed Appium endpoint:

```
┌─────────────┐      MCP Protocol      ┌──────────────────┐
│ AI Assistant│ ◄──────────────────────► │  MCP Server      │
│ (Q/Claude)  │      Tool Calls         │  (This Project)  │
└─────────────┘                         └────────┬─────────┘
                                                 │
                                    WebdriverIO  │ HTTPS
                                    W3C Protocol │
                                                 ▼
                                        ┌────────────────┐
                                        │ Device Farm    │
                                        │ Appium Endpoint│
                                        └────────┬───────┘
                                                 │
                                                 ▼
                                        ┌────────────────┐
                                        │  Real Device   │
                                        │  (Android/iOS) │
                                        └────────────────┘
```

### Technical Stack

- **MCP Protocol**: Uses `@modelcontextprotocol/sdk` for tool registration
- **WebdriverIO**: Appium client for mobile automation
- **AWS SDK**: Device Farm API integration (`CreateRemoteAccessSession`, `InstallToRemoteAccessSession`)
- **HTTPS Upload**: Direct S3 upload for APK files
- **Managed Appium**: AWS-hosted Appium server (no local setup required)

### Session Workflow

1. **Session Creation**
   ```javascript
   CreateRemoteAccessSession → Returns Appium endpoint URL
   ```

2. **App Installation** (Automatic)
   ```javascript
   InstallToRemoteAccessSession → Installs APK to device
   ```

3. **Appium Connection**
   ```javascript
   WebdriverIO connects to: https://devicefarm-interactive-global.us-west-2.api.aws/remote-endpoint/...
   ```

4. **Mobile Automation**
   ```javascript
   execute('mobile: clickGesture', {x, y}) → Device performs action
   ```

## Troubleshooting

### Session Creation Fails

- Check AWS credentials and permissions
- Verify Device Farm project ARN
- Ensure device availability in your region

### APK Upload Timeout

- Check network connectivity
- Verify APK file is valid
- Increase timeout in `waitForSessionReady` function

### Appium Commands Fail

- Ensure session is in RUNNING state
- Verify Appium endpoint is accessible
- Check device compatibility with commands

## License

MIT

## Contributing

Contributions welcome! Please open issues or pull requests on GitHub.
