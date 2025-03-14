<agentDetails>
  <name>AI Financial Agent</name>
  <description>This document details the AI Financial Agent's tools, orchestration, integrations, logic, and prompts.</description>

  <tools>
    <tool>
      <name>getStockPrices</name>
      <description>Use this tool to get stock prices and market cap for a company. This tool will return a snapshot of the current price, market cap, and the historical prices over a given time period.</description>
      <parameters>
        <parameter>
          <name>ticker</name>
          <type>string</type>
          <description>The ticker of the company to get historical prices for</description>
        </parameter>
        <parameter>
          <name>start_date</name>
          <type>string</type>
          <description>The start date for historical prices (YYYY-MM-DD)</description>
        </parameter>
        <parameter>
          <name>end_date</name>
          <type>string</type>
          <description>The end date for historical prices (YYYY-MM-DD)</description>
        </parameter>
        <parameter>
          <name>interval</name>
          <type>enum</type>
          <values>second, minute, day, week, month, year</values>
          <default>day</default>
          <description>The interval between price points (e.g. second, minute, day, week, month, year)</description>
        </parameter>
        <parameter>
          <name>interval_multiplier</name>
          <type>number</type>
          <default>1</default>
          <description>The multiplier for the interval (e.g. 1 for second, 60 for minute, 1 for day, 7 for week, 1 for month, 1 for year)</description>
        </parameter>
      </parameters>
      <integration>financialdatasets.ai</integration>
    </tool>
    <tool>
      <name>getIncomeStatements</name>
      <description>Get the income statements of a company</description>
      <parameters>
        <parameter>
          <name>ticker</name>
          <type>string</type>
          <description>The ticker of the company to get income statements for</description>
        </parameter>
        <parameter>
          <name>period</name>
          <type>enum</type>
          <values>quarterly, annual, ttm</values>
          <default>ttm</default>
          <description>The period of the income statements to return</description>
        </parameter>
        <parameter>
          <name>limit</name>
          <type>number</type>
          <optional>true</optional>
          <default>1</default>
          <description>The number of income statements to return</description>
        </parameter>
        <parameter>
          <name>report_period_lte</name>
          <type>string</type>
          <optional>true</optional>
          <description>The less than or equal to date of the income statements to return. This lets us bound the data by date.</description>
        </parameter>
        <parameter>
          <name>report_period_gte</name>
          <type>string</type>
          <optional>true</optional>
          <description>The greater than or equal to date of the income statements to return. This lets us bound the data by date.</description>
        </parameter>
      </parameters>
      <integration>financialdatasets.ai</integration>
    </tool>
    <tool>
      <name>getBalanceSheets</name>
      <description>Get the balance sheets of a company</description>
      <parameters>
        <parameter>
          <name>ticker</name>
          <type>string</type>
          <description>The ticker of the company to get balance sheets for</description>
        </parameter>
        <parameter>
          <name>period</name>
          <type>enum</type>
          <values>quarterly, annual, ttm</values>
          <default>ttm</default>
          <description>The period of the balance sheets to return</description>
        </parameter>
        <parameter>
          <name>limit</name>
          <type>number</type>
          <optional>true</optional>
          <default>1</default>
          <description>The number of balance sheets to return</description>
        </parameter>
        <parameter>
          <name>report_period_lte</name>
          <type>string</type>
          <optional>true</optional>
          <description>The less than or equal to date of the balance sheets to return. This lets us bound the data by date.</description>
        </parameter>
        <parameter>
          <name>report_period_gte</name>
          <type>string</type>
          <optional>true</optional>
          <description>The greater than or equal to date of the balance sheets to return. This lets us bound the data by date.</description>
        </parameter>
      </parameters>
      <integration>financialdatasets.ai</integration>
    </tool>
    <tool>
      <name>getCashFlowStatements</name>
      <description>Get the cash flow statements of a company</description>
      <parameters>
        <parameter>
          <name>ticker</name>
          <type>string</type>
          <description>The ticker of the company to get cash flow statements for</description>
        </parameter>
        <parameter>
          <name>period</name>
          <type>enum</type>
          <values>quarterly, annual, ttm</values>
          <default>ttm</default>
          <description>The period of the cash flow statements to return</description>
        </parameter>
        <parameter>
          <name>limit</name>
          <type>number</type>
          <optional>true</optional>
          <default>1</default>
          <description>The number of cash flow statements to return</description>
        </parameter>
        <parameter>
          <name>report_period_lte</name>
          <type>string</type>
          <optional>true</optional>
          <description>The less than or equal to date of the cash flow statements to return. This lets us bound the data by date.</description>
        </parameter>
        <parameter>
          <name>report_period_gte</name>
          <type>string</type>
          <optional>true</optional>
          <description>The greater than or equal to date of the cash flow statements to return. This lets us bound the data by date.</description>
        </parameter>
      </parameters>
      <integration>financialdatasets.ai</integration>
    </tool>
    <tool>
      <name>getFinancialMetrics</name>
      <description>Get the financial metrics of a company. These financial metrics are derived metrics like P/E ratio, operating income, etc. that cannot be found in the income statement, balance sheet, or cash flow statement.</description>
      <parameters>
        <parameter>
          <name>ticker</name>
          <type>string</type>
          <description>The ticker of the company to get financial metrics for</description>
        </parameter>
        <parameter>
          <name>period</name>
          <type>enum</type>
          <values>quarterly, annual, ttm</values>
          <default>ttm</default>
          <description>The period of the financial metrics to return</description>
        </parameter>
        <parameter>
          <name>limit</name>
          <type>number</type>
          <optional>true</optional>
          <default>1</default>
          <description>The number of financial metrics to return</description>
        </parameter>
        <parameter>
          <name>report_period_lte</name>
          <type>string</type>
          <optional>true</optional>
          <description>The less than or equal to date of the financial metrics to return. This lets us bound the data by date.</description>
        </parameter>
        <parameter>
          <name>report_period_gte</name>
          <type>string</type>
          <optional>true</optional>
          <description>The greater than or equal to date of the financial metrics to return. This lets us bound the data by date.</description>
        </parameter>
      </parameters>
      <integration>financialdatasets.ai</integration>
    </tool>
    <tool>
      <name>searchStocksByFilters</name>
      <description>Search for stocks based on financial criteria. Use this tool when asked to find or screen stocks based on financial metrics like revenue, net income, debt, etc. Examples: "stocks with revenue > 50B", "companies with positive net income", "find stocks with low debt". The tool supports comparing metrics like revenue, net_income, total_debt, total_assets, etc. with values using greater than (gt), less than (lt), equal to (eq), and their inclusive variants (gte, lte).</description>
      <parameters>
        <parameter>
          <name>filters</name>
          <type>array</type>
          <description>The filters to search for (e.g. [{field: "net_income", operator: "gt", value: 1000000000}, {field: "revenue", operator: "gt", value: 50000000000}])</description>
          <item>
            <type>object</type>
            <properties>
              <property>
                <name>field</name>
                <type>enum</type>
                <values>validStockSearchFilters</values>
                <description>The field to filter on (e.g. net_income, revenue, total_debt)</description>
              </property>
              <property>
                <name>operator</name>
                <type>enum</type>
                <values>gt, gte, lt, lte, eq</values>
                <description>The operator to use for the filter (e.g. gt, gte, lt, lte, eq)</description>
              </property>
              <property>
                <name>value</name>
                <type>number</type>
                <description>The value to compare the field to</description>
              </property>
            </properties>
          </item>
        </parameter>
        <parameter>
          <name>period</name>
          <type>enum</type>
          <values>quarterly, annual, ttm</values>
          <optional>true</optional>
          <description>The period of the financial metrics to return</description>
        </parameter>
        <parameter>
          <name>limit</name>
          <type>number</type>
          <optional>true</optional>
          <default>5</default>
          <description>The number of stocks to return</description>
        </parameter>
        <parameter>
          <name>order_by</name>
          <type>enum</type>
          <values>-report_period, report_period</values>
          <optional>true</optional>
          <default>-report_period</default>
          <description>The order of the stocks to return</description>
        </parameter>
      </parameters>
      <integration>financialdatasets.ai</integration>
    </tool>
    <tool>
      <name>createDocument</name>
      <description>Create a document for a writing activity. This tool will call other functions that will generate the contents of the document based on the title and kind.</description>
      <parameters>
        <parameter>
          <name>title</name>
          <type>string</type>
          <description>The title of the document</description>
        </parameter>
        <parameter>
          <name>kind</name>
          <type>enum</type>
          <values>text, code</values>
          <description>The kind of document to create (text or code)</description>
        </parameter>
      </parameters>
      <integration>AI SDK</integration>
    </tool>
    <tool>
      <name>updateDocument</name>
      <description>Update a document with the given description.</description>
      <parameters>
        <parameter>
          <name>id</name>
          <type>string</type>
          <description>The ID of the document to update</description>
        </parameter>
        <parameter>
          <name>description</name>
          <type>string</type>
          <description>The description of changes that need to be made</description>
        </parameter>
      </parameters>
      <integration>AI SDK</integration>
    </tool>
     <tool>
      <name>requestSuggestions</name>
      <description>Request suggestions for a document</description>
      <parameters>
        <parameter>
          <name>documentId</name>
          <type>string</type>
          <description>The ID of the document to request edits</description>
        </parameter>
      </parameters>
      <integration>AI SDK</integration>
    </tool>
  </tools>

  <aiSdkOrchestration>
    <description>The AI SDK is orchestrated within the app/(chat)/api/chat/route.ts file. The streamText function from the 'ai' library is used to manage the interaction with the LLM. The customModel function from '@/lib/ai' is used to initialize the LLM with the OpenAI API key and custom middleware.</description>
    <steps>
      <step>
        <description>The user sends a message to the chat interface.</description>
      </step>
      <step>
        <description>The message is received by the POST handler in app/(chat)/api/chat/route.ts.</description>
      </step>
       <step>
        <description>The user query is decomposed into sub-tasks using generateObject and the gpt-4o-mini model.</description>
      </step>
      <step>
        <description>The streamText function is called with the appropriate model, system prompt, user messages, and available tools.</description>
      </step>
      <step>
        <description>The LLM processes the prompt and determines whether to use a tool.</description>
      </step>
      <step>
        <description>If a tool is called, the corresponding execute function is invoked.</description>
      </step>
      <step>
        <description>The tool's response is fed back to the LLM.</description>
      </step>
      <step>
        <description>The LLM generates a response, which is streamed back to the user interface.</description>
      </step>
    </steps>
  </aiSdkOrchestration>

  <financialDatasetsIntegration>
    <description>The AI Financial Agent integrates with the financialdatasets.ai API to retrieve financial data. The API key is stored in an environment variable and accessed through the getFinancialDatasetsApiKey function in lib/db/api-keys.ts.</description>
    <integrationPoints>
      <integrationPoint>
        <tool>getStockPrices</tool>
        <description>Retrieves historical stock prices and snapshot data.</description>
        <apiEndpoints>
          <apiEndpoint>https://api.financialdatasets.ai/prices/snapshot</apiEndpoint>
          <apiEndpoint>https://api.financialdatasets.ai/prices/</apiEndpoint>
        </apiEndpoints>
      </integrationPoint>
      <integrationPoint>
        <tool>getIncomeStatements</tool>
        <description>Retrieves income statements for a company.</description>
        <apiEndpoint>https://api.financialdatasets.ai/financials/income-statements/</apiEndpoint>
      </integrationPoint>
      <integrationPoint>
        <tool>getBalanceSheets</tool>
        <description>Retrieves balance sheets for a company.</description>
        <apiEndpoint>https://api.financialdatasets.ai/financials/balance-sheets/</apiEndpoint>
      </integrationPoint>
      <integrationPoint>
        <tool>getCashFlowStatements</tool>
        <description>Retrieves cash flow statements for a company.</description>
        <apiEndpoint>https://api.financialdatasets.ai/financials/cash-flow-statements/</apiEndpoint>
      </integrationPoint>
      <integrationPoint>
        <tool>getFinancialMetrics</tool>
        <description>Retrieves financial metrics for a company.</description>
        <apiEndpoint>https://api.financialdatasets.ai/financial-metrics/</apiEndpoint>
      </integrationPoint>
      <integrationPoint>
        <tool>searchStocksByFilters</tool>
        <description>Searches for stocks based on financial criteria.</description>
        <apiEndpoint>https://api.financialdatasets.ai/financials/search/</apiEndpoint>
      </integrationPoint>
    </integrationPoints>
  </financialDatasetsIntegration>

  <agentLogic>
    <description>The agent logic is primarily implemented within the POST handler in app/(chat)/api/chat/route.ts. The agent uses the streamText function from the 'ai' library to interact with the LLM. The agent logic includes task decomposition, tool calling, and response streaming.</description>
    <taskDecomposition>
      <description>The agent uses the generateObject function to decompose the user query into sub-tasks. This allows the agent to break down complex queries into smaller, more manageable steps.</description>
      <model>gpt-4o-mini</model>
      <prompt>You are a reasoning agent. Given the following user query: ${userMessage.content}, break it down to small, tightly-scoped sub-tasks that need to be taken to answer the query. The task name should include the ticker or company name where appropriate. The task name must be in the present progressive tense as if you are telling another agent what to do. The task name should be short (max 5 words), but comprehensive. Create the least number of tasks possible, but make sure they are comprehensive to answer the query. Your output will be given to another LLM, which will use tools to execute the tasks. Make sure your tasks are not too complex and can be completed with the optimal number of tools. Make your task names friendly, concise, easy to understand, and accessible. Example: "Getting current price for AAPL", "Analyzing revenue trends", etc.</prompt>
    </taskDecomposition>
    <toolCalling>
      <description>The agent uses the streamText function from the 'ai' library to call tools. The available tools are passed to the streamText function in the experimental_activeTools parameter. The LLM decides which tool to use based on the prompt and the tool descriptions.</description>
      <availableTools>
        <tool>getStockPrices</tool>
        <tool>getIncomeStatements</tool>
        <tool>getBalanceSheets</tool>
        <tool>getCashFlowStatements</tool>
        <tool>getFinancialMetrics</tool>
        <tool>searchStocksByFilters</tool>
         <tool>createDocument</tool>
        <tool>updateDocument</tool>
         <tool>requestSuggestions</tool>
      </availableTools>
    </toolCalling>
    <responseStreaming>
      <description>The agent uses the createDataStreamResponse function from the 'ai' library to stream the response back to the user interface. The streamText function generates the response in chunks, which are then sent to the client.</description>
    </responseStreaming>
  </agentLogic>

  <prompts>
    <prompt>
      <name>systemPrompt</name>
      <description>The system prompt defines the persona of the AI assistant as a friendly financial assistant. It sets constraints on responses (concise, helpful, no code/markdown/tables/lists in responses). It also includes instructions for financial data retrieval (TTM period, minimize API requests).</description>
      <content>You are a friendly financial assistant. Keep your responses concise and helpful. Do not ever return code, markdown, tables, lists, or any other UI text in your responses. The current date is ${new Date().toLocaleDateString()}. When retrieving recent financial data, use ttm as the default period. Additionally, try to make the least number of API requests as possible, but make sure to get all the information needed to answer the query. Many of our tools let you pass in parameters that can help you get more aggregate data in a single request.</content>
    </prompt>
    <prompt>
      <name>codePrompt</name>
      <description>The code prompt provides instructions for generating Python code snippets. It emphasizes self-contained, runnable code with comments, concise snippets, standard library usage, error handling, and meaningful output.</description>
      <content>You are a Python code generator that creates self-contained, executable code snippets. When writing code: 1. Each snippet should be complete and runnable on its own 2. Prefer using print() statements to display outputs 3. Include helpful comments explaining the code 4. Keep snippets concise (generally under 15 lines) 5. Avoid external dependencies - use Python standard library 6. Handle potential errors gracefully 7. Return meaningful output that demonstrates the code's functionality 8. Don't use input() or other interactive functions 9. Don't access files or network resources 10. Don't use infinite loops Examples of good snippets: \`\`\`python # Calculate factorial iteratively def factorial(n): result = 1 for i in range(1, n + 1): result *= i return result print(f"Factorial of 5 is: {factorial(5)}") \`\`\`</content>
    </prompt>
    <prompt>
      <name>updateDocumentPrompt</name>
      <description>The update document prompt is used to update the contents of a document based on a given description.</description>
      <content>Update the following contents of the document based on the given prompt. ${currentContent}</content>
    </prompt>
  </prompts>
</agentDetails>
