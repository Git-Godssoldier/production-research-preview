# Base Theme Definition

This document defines the base theme for the project, including styling definitions, color definitions, packages used, components used, UI interactions, and dependency information.

## Styling Definitions

The project uses Tailwind CSS for styling. The `tailwind.config.ts` file contains the following theme extensions:

```typescript
// tailwind.config.ts
import type { Config } from "tailwindcss";

export default {
  content: [
    "./pages/**/*.{js,ts,jsx,tsx,mdx}",
    "./components/**/*.{js,ts,jsx,tsx,mdx}",
    "./app/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  theme: {
    extend: {
      colors: {
        background: "var(--background)",
        foreground: "var(--foreground)",
      },
      fontFamily: {
        sans: ["var(--font-geist-sans)"],
        mono: ["var(--font-geist-mono)"],
      },
    },
  },
  plugins: [],
} satisfies Config;
```

## Color Definitions

The project uses CSS variables for color definitions. The `AGNTX-AGENTS/Opulentia-Theme.MD` file defines the following color palette:

```typescript
// src/styles/themes/tiffanyDark.ts
export const tiffanyDarkTheme = {
  name: 'tiffany-dark',
  colors: {
    // Base colors
    background: '#121212',
    surface: '#1E1E1E',
    primary: '#0ABAB5', // Tiffany Blue
    primaryHover: '#15DDD8',
    primaryActive: '#2F91AE',

    // Text colors
    textPrimary: '#ECECEC',
    textSecondary: '#A0A0A0',

    // UI elements
    border: '#2A2A2A',
    overlay: 'rgba(0,0,0,0.6)',

    // Status colors
    success: '#3FB980',
    warning: '#F9D53E',
    error: '#FF6B6B',

    // Component-specific
    chatInput: '#1E1E1E',
    messageHover: 'rgba(10,186,181,0.05)', // Tiffany Blue at 5% opacity
    selectedItem: '#0ABAB5',
    focusRing: '0 0 0 2px rgba(10,186,181,0.5)',

    // Financial specific
    positive: '#3FB980', // Green for positive values
    negative: '#FF6B6B', // Red for negative values
    neutral: '#A0A0A0', // Gray for neutral values
    chartGrid: '#2A2A2A',
    chartAxis: '#A0A0A0',
    chartTooltip: '#1E1E1E',
  },

  typography: {
    fontFamily: 'Inter, system-ui, -apple-system, sans-serif',
    fontSize: {
      base: '14px',
      h1: '24px',
      h2: '20px',
      h3: '18px',
      small: '12px',
    },
    fontWeight: {
      normal: 400,
      medium: 500,
      semibold: 600,
    },
  },
};
```

## Packages Used

The following packages are used in the project:

```json
// package.json
{
  "dependencies": {
    "@ai-sdk/anthropic": "1.1.10",
    "@ai-sdk/react": "1.1.18",
    "ai": "4.1.46",
    "classnames": "^2.5.1",
    "framer-motion": "^12.4.4",
    "geist": "^1.3.1",
    "next": "15.1.7",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-markdown": "^9.0.3",
    "sonner": "^2.0.0"
  },
  "devDependencies": {
    "@eslint/eslintrc": "^3",
    "@types/node": "^20",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "eslint": "^9",
    "eslint-config-next": "15.1.7",
    "postcss": "^8",
    "tailwindcss": "^3.4.1",
    "typescript": "^5"
  }
}
```

## Components Used

The project uses the following components:

*   `components/chat.tsx`: Main chat component.
*   `components/messages.tsx`: Displays chat messages.
*   `components/input.tsx`: Input field for typing messages.
*   `components/footnote.tsx`: Displays a footnote.
*   `components/icons.tsx`: Provides icons for the UI.

## UI Interactions

The UI interactions include:

*   Typing messages in the input field.
*   Submitting messages to the chat.
*   Toggling reasoning on/off.
*   Selecting a model.

## Standout Dependency Information

*   **@ai-sdk/react:** Provides hooks and components for building AI-powered React applications.
*   **ai:** Provides primitives for building AI-powered user interfaces.
*   **tailwindcss:** Provides a utility-first CSS framework for styling the application.

## Involved Paths and Theme Codification Patterns

*   **`app/globals.css`**: This file imports Tailwind CSS directives and defines global styles. It uses `@tailwind` directives to inject Tailwind's base, components, and utilities styles.
    *   **Pattern:** Centralized CSS import for global styling.
*   **`tailwind.config.ts`**: This file configures Tailwind CSS, extending the default theme with custom colors and font families. It uses `theme.extend` to add project-specific styling.
    *   **Pattern:** Tailwind configuration for theme extension.
*   **`AGNTX-AGENTS/Opulentia-Theme.MD`**: This file defines the color palette and typography for the Opulentia theme. It uses a JavaScript object to define the theme and exports it for use in other components.
    *   **Pattern:** Centralized theme definition using JavaScript objects.
*   **`components/chat.tsx`**: This file implements the main chat component. It uses React hooks and components from `@ai-sdk/react` to manage the chat state and UI.
    *   **Pattern:** React components for UI implementation.
*   **`components/messages.tsx`**: This file implements the messages component. It uses React components and the `react-markdown` library to display chat messages.
    *   **Pattern:** React components for displaying data.
