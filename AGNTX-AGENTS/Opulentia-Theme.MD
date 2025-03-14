
```tsx
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