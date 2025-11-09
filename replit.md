# AI Adviser

## Overview
AI Adviser is a modular Spring Boot backend designed to analyze product data using the Anthropic Claude API. The system aggregates data from various sources (mock Shopify and Baselinker APIs) to provide AI-driven inventory management recommendations. Its core purpose is to offer intelligent insights for optimizing stock levels and supply chain operations, with ambitions to streamline product management for businesses.

## User Preferences
I prefer that the AI assistant prioritizes clarity and conciseness in its responses, avoiding overly technical jargon where simpler language suffices. I value iterative development, where I can provide feedback and see gradual improvements. Please ask for confirmation before implementing any significant architectural changes or adding new external dependencies. I prefer detailed explanations for complex recommendations, especially regarding the reasoning behind AI suggestions.

## System Architecture
The application follows a layered architecture comprising Controller, Service, Client, Repository, and Model layers. It is built on **Spring Boot 3.2.0** (Web, Data JDBC, WebFlux) with **Java 17** and utilizes **H2 Database** (in-memory) for data storage, **JdbcClient** for database interactions, and **WebClient** for external HTTP calls. **MockServer 5.15.0** is integrated for a demo environment.

**Key Features:**
- **Data Synchronization:** Automatic synchronization of product data from external sources (Shopify, Baselinker) at application startup, including dynamic product discovery and lead time calculation based on stock levels.
- **AI Recommendation Engine:** Provides AI-driven inventory recommendations via the Claude API, including detailed reasoning and dynamic TTL (Time To Live) based on product velocity.
- **Financial Forecasting:** Comprehensive financial analysis system that calculates 5 key metrics for each product recommendation:
  - **Expected Revenue:** Projected revenue based on historical sales velocity and forecast horizon
  - **Expected Profit:** Net profit after deducting COGS from expected revenue
  - **Carrying Cost:** Inventory holding costs calculated at 20% annual rate
  - **Stockout Loss:** Potential revenue loss from out-of-stock situations (15% penalty)
  - **Opportunity Cost:** Lost profit from capital tied up in excess inventory
  - Calculations use 30-day sales history, current stock levels, COGS, and AI-predicted days until stockout. Forecast horizon capped at min(daysUntilStockout, 90 days) to prevent runaway projections.
- **Interactive AI Chat:** A multi-turn chat interface for in-depth consultations with the AI, maintaining context across conversations with persistent session storage.
- **Chat History Persistence:** Chat sessions are stored in the database per SKU with automatic expiration based on AI-determined TTL (2-72 hours). Sessions are automatically cleaned up hourly by a scheduled task.
- **REST API:** Exposes endpoints for product listing, AI recommendations, interactive chat, and chat session management (GET/POST/DELETE `/api/chat/session/{sku}`).
- **Web Interface (SPA):** A vanilla HTML, CSS, JavaScript frontend located at `src/main/resources/static/index.html` for displaying products, triggering AI recommendations, and engaging in interactive chat with the AI. The UI automatically loads saved chat history when reopening product modals, hides the "Detailed Opinion" button after the first interaction, and displays financial forecasts with color-coded metrics and explanatory tooltips.
- **Interactive Help System:** All UI fields include accessible hint icons (ⓘ) that provide contextual explanations. Hints support keyboard navigation (Tab + Enter/Space), touch/click interaction, and screen readers via ARIA attributes, ensuring accessibility for all users.

**System Design Choices:**
- **Database Schema:** `product_data` stores product information (id, sku, product_name, cogs, lead_time_days, timestamps), `sales_history` stores sales records (id, sku, sale_date, quantity, unit_price, total_amount, created_at), and `chat_sessions` stores persistent chat history (id, sku, session_data as JSON, expires_at, created_at, updated_at) with foreign key constraints and indexes for efficient querying.
- **Financial Metrics Architecture:** FinancialForecastService calculates metrics deterministically in the backend (not via AI) for consistency and auditability. Uses safe numeric extraction via Number interface to handle both Integer and Double values from Claude API responses, preventing ClassCastException. Configuration parameters in application.yaml: carrying_cost_rate (20%), stockout_penalty_rate (15%), forecast_horizon_days (30).
- **Scheduled Tasks:** Enabled via `@EnableScheduling` annotation. `ChatSessionCleanupTask` runs hourly (cron: every 3600000ms) to automatically delete expired chat sessions based on the `expires_at` timestamp.
- **TTL Management:** AI recommendations include a `ttlHours` field (3-72 hours) that determines how long the recommendation and associated chat history remain valid. Fast-moving products get shorter TTL, slow-moving products get longer TTL.
- **Configuration:** Supports `demo` and `prod` Spring profiles. The `demo` profile uses MockServer on port 9090 for all external services, generating dynamic, realistic mock data including random TTL values. The `prod` profile uses actual external API endpoints, requiring environment variables for API keys.
- **UI/UX:** The web interface features a modern design with gradients and animations, providing an intuitive user experience for product overview and AI interaction. Chat history persists across sessions and automatically reloads when reopening product recommendations. Financial forecast panel displays all 5 metrics with color-coded indicators (green for revenue/profit, orange for costs, red for losses) and tooltips explaining calculation assumptions.
- **Accessibility:** All help tooltips use accessible button elements with aria-label and aria-describedby attributes, supporting keyboard navigation (focusable), touch/click activation, and screen reader announcements. Tooltips include concise explanations (≤15 words) for non-technical users covering COGS, Lead Time, AI recommendation metrics (Days Until Stockout, TTL Hours, Suggested Order Quantity, Reorder Recommendation, Stockout Risk), and all 5 financial forecast metrics.

## External Dependencies
- **Anthropic Claude API:** For AI recommendations and interactive chat.
- **Shopify API (Mocked in Demo):** Used for fetching order data.
- **Baselinker API (Mocked in Demo):** Used for fetching product inventory data.