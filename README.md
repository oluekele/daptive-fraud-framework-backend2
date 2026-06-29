# Adaptive Fraud Framework - Backend API

A NestJS-based REST API for authentication, session tracking, behavioral telemetry, and fraud prediction.

## Table of Contents

- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Database Setup](#database-setup)
  - [Running the Application](#running-the-application)
- [API Endpoints](#api-endpoints)
- [Database Schema](#database-schema)
- [Configuration](#configuration)
- [Development](#development)
- [Testing](#testing)
- [Deployment](#deployment)

## Overview

This backend API provides the core functionality for the Adaptive Fraud Framework, including:

- **User Authentication**: JWT-based authentication with registration and login
- **Session Management**: Track user sessions and their lifecycle
- **Behavioral Telemetry**: Collect and process user interaction data
- **Fraud Prediction**: Generate risk scores and predictions based on behavioral patterns
- **Feature Extraction**: Compute behavioral features from raw telemetry data

## Tech Stack

- **Runtime**: Node.js
- **Framework**: NestJS 11
- **Database**: PostgreSQL
- **ORM**: Prisma 7
- **Authentication**: JWT (JSON Web Tokens)
- **Validation**: class-validator, class-transformer
- **Documentation**: Swagger/OpenAPI
- **Password Hashing**: bcrypt
- **Language**: TypeScript

## Project Structure

```
src/
├── main.ts                 # Application entry point
├── app.module.ts          # Root module
├── app.controller.ts      # Root controller
├── app.service.ts         # Root service
├── auth/                  # Authentication module
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── auth.module.ts
│   ├── dto/
│   │   ├── login.dto.ts
│   │   └── register.dto.ts
│   ├── guards/
│   │   └── jwt-auth.guard.ts
│   ├── strategies/
│   │   └── jwt.strategy.ts
│   └── interfaces/
│       └── authenticated-request.interface.ts
├── users/                 # Users module
│   ├── users.controller.ts
│   ├── users.service.ts
│   └── users.module.ts
├── sessions/              # Sessions module
│   ├── sessions.controller.ts
│   ├── sessions.service.ts
│   └── sessions.module.ts
├── telemetry/             # Telemetry module
│   ├── telemetry.controller.ts
│   ├── telemetry.service.ts
│   ├── telemetry.module.ts
│   └── dto/
│       └── telemetry-event.dto.ts
├── predictions/           # Predictions module (fraud detection)
├── risk/                  # Risk scoring module
├── features/              # Feature extraction module
├── prisma/                # Prisma service
│   ├── prisma.service.ts
│   └── prisma.module.ts
├── common/                # Common utilities, filters, interceptors
├── config/                # Configuration modules
└── database/              # Database utilities
```

## Getting Started

### Prerequisites

- Node.js (v18 or higher)
- npm or yarn
- PostgreSQL database
- Git

### Installation

1. Navigate to the backend directory:
   ```bash
   cd backend/nestjs-api
   ```

2. Install dependencies:
   ```bash
   npm install
   # or
   yarn install
   ```

3. Create a `.env` file in the root directory:
   ```env
   DATABASE_URL=postgresql://username:password@localhost:5432/fraud_framework?schema=public
   JWT_SECRET=your-secret-key-here
   PORT=4000
   ```

### Database Setup

1. Ensure PostgreSQL is running and the database exists

2. Generate Prisma client:
   ```bash
   npx prisma generate
   ```

3. Run database migrations:
   ```bash
   npx prisma migrate dev
   ```

4. (Optional) Open Prisma Studio to view/edit data:
   ```bash
   npx prisma studio
   ```

### Running the Application

**Development mode** (with hot-reload):
```bash
npm run start:dev
# or
yarn start:dev
```

The application will start on `http://localhost:4000` by default, or the port specified in your `.env` file.

**Production mode**:
```bash
npm run build
npm run start:prod
```

**Debug mode**:
```bash
npm run start:debug
```

## API Endpoints

The API provides the following endpoints:

### Authentication (`/auth`)
- `POST /auth/register` - Register a new user
- `POST /auth/login` - Login and receive JWT token

### Users (`/users`)
- `GET /users` - Get all users (protected)
- `GET /users/:id` - Get user by ID (protected)
- `GET /users/stats` - Get user statistics (total users, recent users)

### Sessions (`/sessions`)
- `GET /sessions/current` - Get the active session from the JWT (protected)
- `GET /sessions/history` - Get session history for the authenticated user (protected)
- `GET /sessions/:sessionId` - Get an owned session by ID (protected)
- `PATCH /sessions/current/end` - End the active session (protected)

### Telemetry (`/telemetry`)
- `POST /telemetry` - Store a keyboard, mouse, touch, or scroll event for the active session
- `GET /telemetry` - Retrieve telemetry for the active session, optionally filtered by `?eventType=`
- `GET /telemetry/session/:sessionId` - Retrieve telemetry for an owned session
- `GET /telemetry/dataset` - Get telemetry dataset for analysis

### Features (`/features`)
- `POST /features/generate` - Generate and store behavioral features for a session
- `GET /features` - Retrieve stored features for the active session
- `GET /features/session/:sessionId` - Retrieve stored features for an owned session
- `GET /features/training-summary` - Return the flattened ML-ready training summary for the active session
- `POST /features/training-summary/all` - Return flattened training summaries for all sessions owned by the authenticated user

### Risk (`/risk`)
- `POST /risk/calculate` - Generate features, calculate risk, save the score, and return the level
- `GET /risk` - Retrieve risk scores for the active session
- `GET /risk/session/:sessionId` - Retrieve risk scores for an owned session

## Session-to-Risk Pipeline

The implemented processing flow is:

```text
Session -> Telemetry -> Feature Extraction -> Risk Engine
```

1. `POST /auth/login` creates a session and returns `accessToken` plus `sessionId`.
2. `POST /telemetry` stores raw behavioral events against the active JWT session.
3. `POST /features/generate` computes and stores average dwell time, average flight time, average mouse speed, average scroll rate, and typing speed.
4. `POST /risk/calculate` generates fresh features, scores the session from `0` to `100`, stores the score, and returns `low`, `medium`, `high`, or `critical`.
5. `GET /features/training-summary` exposes a flattened training record for downstream machine-learning workflows.

### Swagger Documentation
- `GET /docs` - Interactive API documentation

## Database Schema

The application uses the following data models:

### User
- `id` (UUID) - Unique identifier
- `email` (String) - Unique email address
- `passwordHash` (String) - Hashed password
- `createdAt` (DateTime) - Account creation timestamp
- `sessions` (Session[]) - User's sessions

### Session
- `id` (UUID) - Unique identifier
- `userId` (String) - Reference to User
- `startedAt` (DateTime) - Session start time
- `endedAt` (DateTime?) - Session end time (null if active)
- `telemetry` (Telemetry[]) - Telemetry events in session
- `riskScores` (RiskScore[]) - Risk assessments
- `predictions` (Prediction[]) - Fraud predictions

### Telemetry
- `id` (UUID) - Unique identifier
- `sessionId` (String) - Reference to Session
- `eventType` (String) - Type of event (e.g., 'click', 'keystroke', 'scroll')
- `payload` (JSON) - Event data
- `createdAt` (DateTime) - Event timestamp

### Prediction
- `id` (UUID) - Unique identifier
- `sessionId` (String) - Reference to Session
- `modelName` (String) - ML model used
- `fraudProbability` (Float) - Probability score (0-1)
- `createdAt` (DateTime) - Prediction timestamp

### Feature
- `id` (UUID) - Unique identifier
- `sessionId` (String) - Reference to Session
- `avgDwellTime` (Float?) - Average key press duration
- `avgFlightTime` (Float?) - Average time between keys
- `avgMouseSpeed` (Float?) - Average mouse movement speed
- `avgScrollRate` (Float?) - Average scrolling speed
- `typingSpeed` (Float?) - Characters per minute
- `createdAt` (DateTime) - Feature calculation timestamp

### RiskScore
- `id` (UUID) - Unique identifier
- `sessionId` (String) - Reference to Session
- `score` (Float) - Risk score (0-100)
- `level` (String) - Risk level ('low', 'medium', 'high')
- `createdAt` (DateTime) - Score calculation timestamp

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | Required |
| `JWT_SECRET` | Secret key for JWT signing | Required |
| `PORT` | Server port | `4000` |

### Validation

The application uses `class-validator` and `class-transformer` for input validation and transformation. All incoming data is validated against DTOs (Data Transfer Objects) before processing.

## Development

### Available Scripts

| Script | Description |
|--------|-------------|
| `npm run start` | Start the application |
| `npm run start:dev` | Start in development mode with hot-reload |
| `npm run start:debug` | Start in debug mode |
| `npm run start:prod` | Start in production mode |
| `npm run build` | Build the application |
| `npm run lint` | Run ESLint with auto-fix |
| `npm run format` | Format code with Prettier |
| `npm run test` | Run unit tests |
| `npm run test:watch` | Run tests in watch mode |
| `npm run test:cov` | Generate test coverage report |
| `npm run test:e2e` | Run end-to-end tests |

### Code Style

The project uses:
- **ESLint** for code linting
- **Prettier** for code formatting
- **TypeScript** for type safety

Run linting and formatting:
```bash
npm run lint
npm run format
```

## Testing

Run unit tests:
```bash
npm run test
```

Run tests with coverage:
```bash
npm run test:cov
```

Run end-to-end tests:
```bash
npm run test:e2e
```

## Deployment

### Building for Production

```bash
npm run build
npm run start:prod
```

### Docker

The application can be containerized using Docker. See the root `docker/` directory for Docker Compose configuration.

### Environment-specific Configuration

Use environment variables to configure the application for different environments (development, staging, production).

## Additional Resources

- [NestJS Documentation](https://docs.nestjs.com/)
- [Prisma Documentation](https://www.prisma.io/docs/)
- [Swagger/OpenAPI](https://swagger.io/)
- [JWT.io](https://jwt.io/)

## License

UNLICENSED - All rights reserved
# daptive-fraud-framework-backend2
