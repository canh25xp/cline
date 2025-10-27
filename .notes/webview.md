# Cline Webview Guide

## Overview

The Cline webview is the React-based frontend interface for the Cline VSCode extension. It provides the user interface for interacting with the AI assistant, managing settings, viewing conversations, and accessing all Cline features.

## Quick Start

### Prerequisites

- Node.js 18+
- npm or yarn
- VSCode extension development environment

### Building the Webview

The webview is built using Vite and can be developed independently of the main extension.

1. Navigate to the webview-ui directory:
```bash
cd webview-ui
```

2. Install dependencies:
```bash
npm install
```

3. For development with hot reload:
```bash
npm run dev
```

4. For production build:
```bash
npm run build
```

### Running the Webview in VSCode

The webview runs as part of the VSCode extension. To develop with the full extension:

1. Open the project in VSCode
2. Install dependencies for the extension: `npm install`
3. **Start the Cline core service** (if developing with standalone mode):
   ```bash
   npm run compile-standalone
   ./scripts/runclinecore.sh
   ```
   This starts the Cline core API server required for the webview to communicate with backend services.
4. Build the extension: `npm run build`
5. Press F5 to launch extension development host
6. Open Cline in the development host window

### Alternative Development Approaches

1. **Standalone Development**
The webview can run as a standalone React app for faster UI development:

```bash
# From webview-ui directory
npm run preview
```

2. **Storybook for UI Components**
Cline uses Storybook for component development and testing:

```bash
# From webview-ui directory
npm run storybook
```
Opens on http://localhost:6006

3. **Combined Development**
Run both the extension and webview simultaneously for integrated development:
```bash
# Terminal 1: Build and start core service
npm run compile-standalone
./scripts/runclinecore.sh &

# Terminal 2: Start extension development
npm run dev:webview &
npm run watch
```

## Architecture Deep Dive

### Core Technologies

- **React 18** with TypeScript
- **Vite** for build tooling
- **Tailwind CSS** for styling
- **React Router** for routing
- **Context API** for state management

### File Structure

```
webview-ui/
├── src/
│   ├── components/     # Reusable UI components
│   ├── context/        # React context providers
│   ├── hooks/          # Custom React hooks
│   ├── services/       # API integrations
│   ├── utils/          # Utility functions
│   ├── types/          # TypeScript type definitions
│   └── assets/         # Static assets (images, icons)
├── public/            # Public assets
├── .storybook/        # Storybook configuration
└── package.json      # Dependencies and scripts
```

### State Management

#### ExtensionStateContext

The webview communicates with the VSCode extension through a message-passing system. The `ExtensionStateContext` is the central state management solution:

```typescript
interface ExtensionState {
  version: string
  messages: ClineMessage[]
  taskHistory: TaskHistoryItem[]
  theme: 'light' | 'dark' | 'auto'
  apiConfiguration: ApiConfiguration
  mcpServers: Record<string, McpServer>
  // ... additional state properties
}
```

Key responsibilities:
- Synchronizing state from VSCode extension
- Managing local UI state
- Providing reactive state updates
- Handling message passing between webview and extension

#### Message Passing Architecture

Communication between webview and extension uses VSCode's `postMessage` API:

```typescript
// Webview to extension
window.vscode.postMessage({
  type: 'request',
  action: 'someAction',
  payload: { ... }
})

// Extension to webview
webview.postMessage({
  type: 'response',
  action: 'someAction',
  data: { ... }
})
```

### Component Architecture

#### Core Components

1. **App Component**
   - Root component managing routing
   - Theme provider setup
   - Context providers

2. **Router Configuration**
   - React Router for navigation
   - Routes for different views (chat, settings, history)

3. **UI Components**
   - Reusable components following design system
   - Accessibility compliant
   - Responsive design

#### State Flow

```
Extension Host ←→ WebviewProvider ←→ ExtensionStateContext ←→ Components
     ↑                                                           ↓
Controller ←→ Task ←→ API Providers                           UI State
```

### Development Workflow

#### Hot Module Replacement (HMR)

Vite provides fast HMR during development:

```typescript
// vite.config.ts
export default defineConfig({
  plugins: [
    react({
      fastRefresh: true
    })
  ]
})
```

#### Type Generation

Types are generated from Protobuf definitions:

```bash
npm run protos  # From root directory
```

This generates TypeScript interfaces for API communication.

#### Testing

Component testing with Storybook:

```typescript
// Example component story
import { Meta, StoryObj } from '@storybook/react'
import { Button } from './Button'

const meta: Meta<typeof Button> = {
  title: 'UI/Button',
  component: Button,
  parameters: {
    layout: 'centered'
  }
}

export default meta
type Story = StoryObj<typeof meta>

export const Primary: Story = {
  args: {
    children: 'Click me',
    variant: 'primary'
  }
}
```

### Integration Points

#### VSCode Api Bridge

The webview communicates with VSCode APIs through the `vscode` global:

```typescript
declare global {
  interface Window {
    vscode: {
      postMessage(message: any): void
      getState(): any
      setState(state: any): void
      onDidReceiveMessage(handler: (message: any) => void): Disposable
    }
  }
}
```

#### MCP Server Integration

The webview displays and manages MCP (Model Context Protocol) servers:

- Server status monitoring
- Tool auto-approval configuration
- Real-time updates from extension

#### Theme Synchronisation

Automatic theme detection and application:

```typescript
const [theme, setTheme] = useState<'light' | 'dark'>('light')

useEffect(() => {
  const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)')
  setTheme(mediaQuery.matches ? 'dark' : 'light')
  // ... theme change listener
}, [])
```

### Performance Optimizations

1. **Code Splitting**
   Vite automatically splits code for optimal loading

2. **Lazy Loading**
   Components are lazy-loaded for better initial load times

3. **Memoization**
   React.memo and useMemo used for expensive re-renders

4. **Efficient Updates**
   Message streaming prevents full DOM refreshes

### Security Considerations

- Content Security Policy (CSP) headers
- XSS protection through proper message handling
- Secure context isolation from VSCode host

### Extension vs Standalone Mode

The webview can run in different modes:

- **Extension Mode**: Full integration with VSCode
- **Standalone Mode**: Development with mocked services
- **White-label Mode**: Customized for different distributions

### Debugging

#### VSCode Developer Tools
- Use VSCode's Developer: Open Developer Tools command
- Console logs from webview appear in developer tools

#### Webview Inspection
```typescript
// In webview console
window.vscode.postMessage({ type: 'inspect' })

// Check extension state
console.log(window.vscode.getState())
```

### Best Practices

1. **State Updates**: Always use functional updates for state changes
2. **Message Handling**: Validate all incoming messages
3. **Performance**: Minimize re-renders with proper memoization
4. **Accessibility**: Follow WCAG guidelines
5. **Type Safety**: Leverage TypeScript for all API interactions

### Troubleshooting

Common issues and solutions:

- **Messages not updating**: Check ExtensionStateContext is properly mounted
- **Themes not applying**: Verify theme state synchronization
- **Slow performance**: Check for unnecessary re-renders with React DevTools
- **Build failures**: Ensure proto generation ran successfully

## Conclusion

The Cline webview is a sophisticated React application that provides a seamless user experience within the VSCode ecosystem. Its architecture emphasizes modularity, performance, and tight integration with the extension backend, making it both powerful and maintainable.
