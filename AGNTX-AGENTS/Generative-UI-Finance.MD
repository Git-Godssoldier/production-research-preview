# UI Components: Generative Financial Agent

This document details the generative UI components used in the AI Financial Agent project for displaying financial data, stocks, portfolios, charts, etc.

## Stock-Related Components

<stockComponents>
  <component>
    <name>StockScreenerTable</name>
    <filePath>components/stock-screener-table.tsx</filePath>
    <imports>
      <import>React</import>
      <import>components/ui/table</import>
      <import>lib/api/stock-filters</import>
    </imports>
    <dependencies>
      <dependency>components/ui/table</dependency>
      <dependency>lib/api/stock-filters</dependency>
    </dependencies>
    <purpose>Displays a table of stocks that match specified financial criteria. Used for screening and filtering stocks based on financial metrics.</purpose>
    <implementation>
      <description>Fetches stock data based on filters defined in lib/api/stock-filters.ts and renders it in a table using components from components/ui/table.tsx.</description>
    </implementation>
  </component>

  <component>
    <name>TickerSuggestions</name>
    <filePath>components/ticker-suggestions.tsx</filePath>
    <imports>
      <import>React</import>
      <import>useSWR</import>
    </imports>
    <dependencies>
      <dependency>useSWR</dependency>
    </dependencies>
    <purpose>Provides suggestions for ticker symbols as the user types. Enhances the user experience by providing quick access to valid ticker symbols.</purpose>
    <implementation>
      <description>Uses useSWR to fetch ticker suggestions from an API endpoint (likely a financial data provider). Displays the suggestions in a dropdown or similar UI element.</description>
    </implementation>
  </component>
</stockComponents>

## Chart Components

<chartComponents>
  <component>
    <name>StockChart</name>
    <filePath>components/ui/stock-chart.tsx</filePath>
    <imports>
      <import>React</import>
      <import>recharts</import>
    </imports>
    <dependencies>
      <dependency>recharts</dependency>
    </dependencies>
    <purpose>Displays a stock chart with historical price data. Allows users to visualize stock price trends over time.</purpose>
    <implementation>
      <description>Uses the recharts library to create a line chart or candlestick chart based on the provided historical price data. Configurable to display different time periods and chart types.</description>
    </implementation>
  </component>
</chartComponents>

## Financial Data Components

<financialDataComponents>
  <component>
    <name>FinancialsTable</name>
    <filePath>components/financials-table.tsx</filePath>
    <imports>
      <import>React</import>
      <import>components/ui/table</import>
    </imports>
    <dependencies>
      <dependency>components/ui/table</dependency>
    </dependencies>
    <purpose>Displays financial data in a tabular format, such as income statements, balance sheets, and cash flow statements. Provides a structured view of financial information for analysis.</purpose>
    <implementation>
      <description>Fetches financial data from an API endpoint (likely using the financialdatasets.ai API) and renders it in a table using components from components/ui/table.tsx. Configurable to display different financial statements and time periods.</description>
    </implementation>
  </component>
</financialDataComponents>

## General UI Components

<generalUIComponents>
  <component>
    <name>UI Table</name>
    <filePath>components/ui/table.tsx</filePath>
    <imports>
      <import>React</import>
      <import>@radix-ui/react-scroll-area</import>
      <import>classnames</import>
    </imports>
    <dependencies>
      <dependency>@radix-ui/react-scroll-area</dependency>
      <dependency>classnames</dependency>
    </dependencies>
    <purpose>A reusable table component for displaying data in a structured format. Provides features like sorting, filtering, and pagination.</purpose>
    <implementation>
      <description>Uses React and Radix UI components to create a flexible and customizable table component. Supports various data types and formatting options.</description>
    </implementation>
  </component>
</generalUIComponents>
