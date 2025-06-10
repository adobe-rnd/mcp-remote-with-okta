# MCP Remote with Okta/Adobe IMS Authentication

A wrapper for `mcp-remote` that handles Adobe IMS/Okta user authentication using implicit grant flow, eliminating the need for manual token management.

## Features

- 🔐 **Automatic User Authentication**: Handles Adobe IMS OAuth implicit grant flow
- 🔄 **Token Management**: Automatically refreshes expired tokens (1-hour lifetime)
- 🖥️ **Cross-Platform**: Works on macOS, Windows, and Linux
- 🚀 **Zero Maintenance**: Set it once, never worry about tokens again
- 🔧 **Configurable**: Support for custom client IDs, scopes, and MCP URLs
- 📱 **Browser Integration**: Opens browser automatically for user authentication
- 🔒 **Secure Storage**: Tokens stored securely in `~/.cursor/`
- 🔑 **Simple Setup**: No client secret required - works with implicit grant

## Installation

### Via npx (Recommended)

```bash
npx mcp-remote-with-okta <mcp-url>
```

### Global Installation

```bash
npm install -g mcp-remote-with-okta
mcp-remote-with-okta <mcp-url>
```

## Configuration

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `ADOBE_CLIENT_ID` | ✅ Required | - | Client ID for Adobe IMS (Implicit Grant) |
| `ADOBE_SCOPE` | Optional | `AdobeID,openid` | OAuth scope for Adobe IMS |

### MCP Configuration

Add to your `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "my-mcp-server": {
      "command": "npx",
      "args": [
        "mcp-remote-with-okta",
        "https://your-mcp-server.com/mcp"
      ],
      "env": {
        "ADOBE_CLIENT_ID": "your_client_id_here"
      }
    }
  }
}
```

## Usage

### As MCP Server (Primary Use Case)

The script automatically detects when called as an MCP server and handles user authentication transparently:

```bash
npx mcp-remote-with-okta https://my.mcp-server.com/mcp
```

### CLI Commands

The package also provides CLI commands for token management:

```bash
# Authenticate user and get token
npx mcp-remote-with-okta <mcp-url> authenticate

# Check token status
npx mcp-remote-with-okta <mcp-url> status

# Display current token
npx mcp-remote-with-okta <mcp-url> token

# Clear stored tokens
npx mcp-remote-with-okta <mcp-url> clear

# Show help
npx mcp-remote-with-okta <mcp-url> help
```

## How It Works

1. **Token Check**: Script checks for valid stored tokens in `~/.cursor/adobe-tokens.json`
2. **User Authentication**: If no valid token exists, launches OAuth implicit flow:
   - Opens browser to Adobe IMS login page
   - User logs in with their Adobe ID
   - Adobe IMS redirects to `localhost:8080` with token in URL fragment
   - JavaScript extracts token and sends to local server
3. **Token Storage**: Securely stores tokens with expiration tracking
4. **MCP Launch**: Launches `mcp-remote` with `Authorization: Bearer <token>` header
5. **Auto-Refresh**: Automatically refreshes tokens when they expire (every hour)

## Authentication Flow

The package uses OAuth 2.0 Implicit Grant flow for simplicity:

```
1. User → Browser → Adobe IMS Login Page
2. User → Authenticates → Adobe ID Credentials
3. Adobe IMS → Redirect → localhost:8080#access_token=...
4. JavaScript → Extract Token → Send to Server
5. Script → Store Token → ~/.cursor/
6. Script → Launch MCP → With Auth Header
```

## Troubleshooting

### Common Issues

**"Client ID not found"**
```bash
# Ensure ADOBE_CLIENT_ID is set in your MCP config
```

**"Port 8080 in use"**
```bash
# The redirect URI is hardcoded to localhost:8080
# Ensure no other service is using this port during authentication
```

**"Browser doesn't open"**
```bash
# Manually visit the authentication URL that appears in the terminal
```

**"Invalid client configuration"**
```bash
# Ensure your Adobe Developer Console project is configured as:
# - Implicit Grant enabled
# - Redirect URI: http://localhost:8080/callback
```

**"unsupported_grant_type"**
```bash
# This means your Adobe IMS client doesn't have Implicit Grant enabled
# Enable "Implicit Grant" in Adobe Developer Console
```

**"Client error for command A system error occurred (spawn npx ENOENT)"**
```bash
# If you encounter this error when using npx in MCP configuration,
# this often happens when the Node.js/npm environment isn't properly set up

# Solution: Create an npx wrapper script
cat > ~/.cursor/npx-wrapper.sh << 'SCRIPT'
#!/bin/bash

# Source nvm to get the correct node version
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# Use your preferred node version (adjust as needed)
nvm use 22.0.0 >/dev/null 2>&1

# Execute npx with all passed arguments
exec npx "$@"
SCRIPT

# Make the script executable
chmod +x ~/.cursor/npx-wrapper.sh

# Update your ~/.cursor/mcp.json to use the wrapper instead of npx:
{
  "mcpServers": {
    "your-server": {
      "command": "/Users/your-username/.cursor/npx-wrapper.sh",
      "args": [
        "mcp-remote-with-okta",
        "https://your-mcp-server.com/mcp"
      ],
      "env": {
        "ADOBE_CLIENT_ID": "your_client_id_here"
      }
    }
  }
}
```

### Debug Mode

The underlying `mcp-remote` runs in debug mode by default. Logs are written to `~/.mcp-auth/`.

### Token Management

```bash
# Check if tokens are valid
npx mcp-remote-with-okta <url> status

# Force re-authentication
npx mcp-remote-with-okta <url> clear
npx mcp-remote-with-okta <url> authenticate
```

## Security Notes

- Tokens are stored locally in `~/.cursor/adobe-tokens.json`
- Uses implicit grant flow (tokens visible in browser URL during auth)
- No client secret required
- Tokens expire after 1 hour and require re-authentication
- User authentication required for each new token

## Adobe Developer Console Setup

To use this package, you need to configure Adobe Developer Console for implicit grant:

### Step-by-Step Setup

1. **Create a Project**:
   - Go to [Adobe Developer Console](https://developer.adobe.com/console/)
   - Create a new project or select existing one

2. **Add OAuth Web App**:
   - Click "Add API" → "Adobe Services" → Choose appropriate service
   - Select "OAuth Web App" credential type

3. **Configure Grant Types**:
   - Enable **"Implicit Grant"** ✅
   - Enable **"Refresh Token Grant"** (optional) ✅
   - Authorization Code Grant is not required

4. **Set Redirect URI**:
   - Add redirect URI: `http://localhost:8080/callback`
   - This must match exactly (including the port)

5. **Configure Scopes**:
   - Add required scopes (minimum: `AdobeID`, `openid`)
   - Add any additional scopes your MCP server requires

6. **Copy Client ID**:
   - Copy the Client ID for your environment configuration
   - No client secret needed for implicit grant

### Required Configuration

- **Grant Type**: Implicit Grant
- **Redirect URI**: `http://localhost:8080/callback`
- **Scopes**: `AdobeID,openid` (minimum)

## Requirements

- Node.js 18.0.0 or higher
- Internet connection for authentication
- Browser for OAuth implicit flow
- Adobe ID account

## Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
