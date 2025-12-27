# Ammonit AI-ERP

An AI-powered document processing and ERP integration system that automatically extracts structured data from purchase orders and offers received via email, and seamlessly integrates them into existing ERP systems.

## Table of Contents

- [What It Does](#what-it-does)
- [How It Works](#how-it-works)
- [Technology Stack](#technology-stack)
- [Architecture](#architecture)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Development](#development)
- [Deployment](#deployment)

---

## What It Does

**Ammonit AI-ERP** automates the entire workflow of processing business documents from email to ERP integration:

1. **Automatic Email Monitoring**: Continuously monitors configured Outlook/Microsoft 365 email accounts for incoming purchase orders and offers
2. **Intelligent Document Classification**: Uses AI to filter and identify relevant business documents from general email traffic
3. **AI-Powered Data Extraction**: Converts PDF attachments into structured data using advanced GPT-4 Vision and LLM models
4. **Review & Approval Workflow**: Provides a web interface for users to review, edit, and approve extracted data
5. **ERP Integration**: Automatically integrates approved orders into existing ERP systems (SAP Business One, Access DB)
6. **Multi-Language Support**: Handles documents in Spanish, English, and Basque
7. **User-Specific Customization**: Supports per-user business rules and data transformations

---

## How It Works

### High-Level Flow

```
Email (Outlook) → AI Filtering → PDF Extraction → AI Processing → User Approval → ERP Integration
```

### Detailed Processing Pipeline

#### 1. Email Fetching (Background Scheduler)

- A background scheduler runs every 20 seconds per active user
- Connects to Outlook via OAuth 2.0
- Fetches the latest emails from configured accounts
- Prevents duplicate processing by tracking message IDs

#### 2. AI-Powered Email Filtering

- Each email account has customizable AI filters
- Filters are defined in natural language (e.g., "emails with subject containing 'pedido' or 'order'")
- AI determines if an email contains a purchase order or offer
- Irrelevant emails are ignored automatically

#### 3. Document Extraction

- Extracts PDF attachments from qualifying emails
- Links the extracted document to the source email for traceability
- Creates an Order or Offer record in PENDING state

#### 4. AI Document Processing (Multi-Stage)

**Stage 1: PDF to Markdown (Vision AI)**
- Converts PDF pages to JPEG images
- Uses GPT-4 Vision to transcribe the document
- Maintains document structure and extracts text from images
- Output: Markdown representation of the document

**Stage 2: Markdown to Structured Data (LLM)**
- Uses GPT-4 with carefully engineered prompts
- Extracts key fields:
  - Order number
  - Client information
  - Delivery dates
  - Line items (product code, description, quantity, unit, price, currency)
- Applies user-specific business rules and transformations
- Handles multi-language content
- Normalizes dates to ISO 8601 format

**Stage 3: JSON Structuring**
- Validates extracted data against a predefined schema
- Ensures data consistency and completeness

**Stage 4: Preprocessing & CSV Generation**
- Applies user-specific column mappings
- Adds custom columns as needed
- Flattens nested JSON structure into rows
- Generates semicolon-delimited CSV output

#### 5. User Review & Approval

- Web interface displays the original PDF alongside extracted data
- Users can review and edit the extracted information
- Manual approval required (or auto-approval can be enabled per user)
- State transitions: PENDING → APPROVED

#### 6. ERP Integration (Bridge Service)

- A separate bridge service polls for APPROVED orders
- Executes custom integration logic for each ERP system:
  - **SAP Business One**: Uses Service Layer REST API
  - **Access Database**: Uses ODBC connections
- Updates order state to INTEGRATED_OK or INTEGRATED_ERROR
- Provides feedback on integration success/failure

---

## Technology Stack

### Frontend

- **Framework**: React 18.2 with TypeScript
- **Build Tool**: Vite 6.3.0
- **Routing**: TanStack Router 1.19 (file-based routing)
- **State Management**: TanStack Query 5.28 (React Query)
- **UI Libraries**:
  - Chakra UI 3.8.0
  - Material-UI 7.1.0
- **Form Handling**: React Hook Form 7.49
- **HTTP Client**: Axios 1.8 with auto-generated OpenAPI client
- **Additional Features**:
  - React PDF for document viewing
  - React Dropzone for file uploads
  - Theme switching (dark mode)

### Backend

- **Framework**: FastAPI (Python 3.10+)
- **Package Manager**: UV (modern Python package manager)
- **Database ORM**: SQLModel (Pydantic + SQLAlchemy)
- **Database**: PostgreSQL 12
- **Authentication**: JWT tokens
- **Email Integration**: O365 library (Microsoft 365/Outlook OAuth)
- **Background Jobs**: APScheduler
- **Migrations**: Alembic
- **Monitoring**: Sentry

### AI/ML Stack

- **Primary LLM**: OpenAI GPT-4.1 (mini and nano variants)
- **Alternative LLM**: Groq (as fallback)
- **Document Parsing**: LlamaParse, pdfplumber, PyPDF2, pdf2image
- **Data Processing**: pandas
- **Agent Framework**: OpenAI Agents SDK, MCP (Model Context Protocol)

### Infrastructure

- **Containerization**: Docker & Docker Compose
- **Reverse Proxy**: Traefik (production)
- **Web Server**: Nginx (frontend serving)
- **Database UI**: Adminer 5.1.0
- **Deployment**: Multi-environment support (local, staging, production)

---

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────┐
│                         Frontend (React)                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Orders  │  │  Offers  │  │  Emails  │  │  Admin   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │ HTTP/REST
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Backend API (FastAPI)                     │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐  │
│  │   AI Service   │  │ Email Service  │  │ Order Service│  │
│  │  (GPT-4 Vision)│  │   (O365 OAuth) │  │              │  │
│  └────────────────┘  └────────────────┘  └──────────────┘  │
│           │                   │                   │          │
│           └───────────────────┴───────────────────┘          │
│                               │                              │
│                               ▼                              │
│                        ┌─────────────┐                       │
│                        │  PostgreSQL │                       │
│                        └─────────────┘                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Background Scheduler                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Email Polling (every 20s) → AI Filtering → Orders  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     Bridge Service                           │
│  ┌─────────────────┐          ┌─────────────────┐          │
│  │  SAP B1 Client  │          │  Access DB      │          │
│  │  (REST API)     │          │  (ODBC)         │          │
│  └─────────────────┘          └─────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

### Backend Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│                     API Routes Layer                         │
│     /api/v1/login, /orders, /offers, /emails, /users        │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     Services Layer                           │
│  (Business Logic: AI processing, email fetching, etc.)      │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Repositories Layer                         │
│              (Data Access: CRUD operations)                  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     Database (PostgreSQL)                    │
│        Models: User, Order, Offer, Email, EmailData         │
└─────────────────────────────────────────────────────────────┘
```

### Docker Services

- **db**: PostgreSQL 12 database with persistent storage
- **adminer**: Web-based database management interface
- **prestart**: Initialization service (runs migrations, creates superuser)
- **backend**: FastAPI application (4 workers in production)
- **scheduler**: Background job runner for email polling
- **frontend**: React app served via Nginx (multi-stage build)
- **proxy**: Traefik reverse proxy (development and production)

---

## Getting Started

### Prerequisites

- Docker and Docker Compose
- Python 3.10+ (for local development)
- Node.js 20+ (for frontend development)
- PostgreSQL 12 (if running without Docker)

### Quick Start with Docker

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd ammonit-ai-erp
   ```

2. **Configure environment variables**
   ```bash
   cp .env.example .env
   # Edit .env with your configuration
   ```

3. **Start all services**
   ```bash
   docker compose up -d
   ```

4. **Access the application**
   - Frontend: http://dashboard.localhost.tiangolo.com:5173
   - Backend API: http://api.localhost.tiangolo.com:8000
   - API Docs: http://api.localhost.tiangolo.com:8000/docs
   - Adminer: http://adminer.localhost.tiangolo.com:8080

### Local Development (without Docker)

#### Backend

```bash
cd backend

# Install UV package manager
pip install uv

# Install dependencies
uv sync

# Run migrations
alembic upgrade head

# Start development server
fastapi dev app/main.py
```

#### Frontend

```bash
cd frontend

# Install dependencies
npm install

# Generate API client from OpenAPI spec
npm run generate-client

# Start development server
npm run dev
```

---

## Configuration

### Environment Variables

Key environment variables in `.env`:

**General**
- `ENVIRONMENT`: local, staging, or production
- `DOMAIN`: Your domain name
- `CORS_ORIGINS`: Allowed CORS origins (comma-separated)

**Database**
- `POSTGRES_SERVER`: Database host
- `POSTGRES_USER`: Database user
- `POSTGRES_PASSWORD`: Database password
- `POSTGRES_DB`: Database name

**AI Services**
- `OPENAI_API_KEY`: OpenAI API key for GPT models
- `GROQ_API_KEY`: Groq API key (alternative LLM)
- `LLAMA_CLOUD_API_KEY`: LlamaParse API key

**Email Integration**
- `OUTLOOK_CLIENT_ID`: Microsoft Azure app client ID
- `OUTLOOK_CLIENT_SECRET`: Microsoft Azure app client secret

**Monitoring**
- `SENTRY_DSN`: Sentry error tracking DSN

### Database Configuration

Connection pooling is optimized for low-resource environments:
- Pool Size: 2
- Max Overflow: 3
- Pool Timeout: 20s
- Pool Recycle: 1800s

### User-Specific Configuration

Each user can configure:
- **Auto-approval**: Automatically approve extracted orders
- **Additional Rules**: Custom AI extraction rules
- **Particular Rules**: User-specific business logic
- **Column Mappings**: Custom field transformations

---

## Development

### Code Quality Tools

**Backend**
```bash
# Linting
ruff check .

# Type checking
mypy app/

# Format code
ruff format .
```

**Frontend**
```bash
# Format and lint
npm run format
npm run lint
```

### Database Migrations

```bash
# Create a new migration
alembic revision --autogenerate -m "Description"

# Apply migrations
alembic upgrade head

# Rollback one migration
alembic downgrade -1
```

### API Client Generation

The frontend uses an auto-generated TypeScript client from the OpenAPI spec:

```bash
cd frontend
npm run generate-client
```

### Testing

**Backend**
```bash
pytest
```

**Frontend**
```bash
npm run test
```

### Hot Reload

Both frontend and backend support hot reload in development mode:
- Backend: FastAPI's `--reload` flag
- Frontend: Vite's HMR (Hot Module Replacement)

---

## Deployment

### Production Docker Compose

```bash
# Build and start all services
docker compose -f docker-compose.yml up -d

# View logs
docker compose logs -f

# Stop services
docker compose down
```

### Traefik Configuration

Production uses Traefik as a reverse proxy with Let's Encrypt SSL:

```bash
docker compose -f docker-compose.yml -f docker-compose.traefik.yml up -d
```

Services are available at:
- Frontend: https://dashboard.{DOMAIN}
- Backend API: https://api.{DOMAIN}
- Adminer: https://adminer.{DOMAIN}

### Health Checks

All services include health checks:
- Backend: `/api/v1/utils/health-check/`
- Database: `pg_isready` command
- Scheduler: Custom health endpoint

### Monitoring

Production environments use Sentry for:
- Error tracking
- Performance monitoring
- Request tracing

---

## Project Structure

```
ammonit-ai-erp/
├── backend/
│   ├── app/
│   │   ├── main.py                 # FastAPI entry point
│   │   ├── models.py               # Database models
│   │   ├── api/
│   │   │   ├── routes/             # API endpoints
│   │   │   └── deps.py             # Dependencies
│   │   ├── core/
│   │   │   ├── config.py           # Configuration
│   │   │   ├── db.py               # Database connection
│   │   │   └── exceptions.py       # Exception handlers
│   │   ├── services/
│   │   │   ├── ai.py               # AI orchestration
│   │   │   ├── orders.py           # Order processing
│   │   │   ├── emails.py           # Email integration
│   │   │   ├── offers.py           # Offer processing
│   │   │   └── erps/               # ERP integrations
│   │   ├── repositories/           # Data access layer
│   │   └── alembic/                # Database migrations
│   ├── scheduler/                  # Background jobs
│   ├── bridge/                     # ERP bridge service
│   ├── prestart/                   # Initialization scripts
│   ├── pyproject.toml              # Python dependencies
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── routes/                 # Page components
│   │   ├── components/             # Reusable components
│   │   ├── client/                 # Auto-generated API client
│   │   └── main.tsx                # Entry point
│   ├── package.json
│   ├── vite.config.ts
│   └── Dockerfile
├── ai_experiments/                 # AI research and testing
├── docs/                           # Documentation
├── scripts/                        # Utility scripts
├── db_backup/                      # Database backups
├── docker-compose.yml              # Production configuration
├── docker-compose.override.yml     # Development overrides
└── .env                            # Environment variables
```

---

## License

[Add your license information here]

## Contributing

[Add contribution guidelines here]

## Support

For issues and questions, please contact [your contact information].
