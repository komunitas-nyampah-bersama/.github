# .github

graph TD
    A[Frontend] -->|React| B[HTML/CSS/JavaScript]
    A -->|Axios| C[API Calls]
    D[Backend] -->|Node.js| E[Express]
    D -->|PostgreSQL| F[Database]
    D -->|JWT| G[Authentication]
    H[Infrastruktur] -->|Docker| I[Containerization]
    H -->|GitHub Actions| J[CI/CD]
    H -->|AWS| K[Cloud Deployment]
    L[Development] -->|ESLint| M[Code Quality]
    L -->|Jest| N[Testing]

    /auth-system/
├── client/              # Frontend
│   ├── public/
│   └── src/
│       ├── assets/      # Gambar, font, dll
│       ├── components/  # Komponen UI reusable
│       ├── pages/       # Halaman aplikasi
│       │   ├── Login/
│       │   │   ├── Login.jsx
│       │   │   ├── Login.css
│       │   │   └── Login.test.js
│       │   └── Register/
│       ├── services/    # API services
│       ├── App.jsx
│       └── index.js
├── server/              # Backend
│   ├── config/          # Konfigurasi database, env
│   ├── controllers/     # Logika bisnis
│   ├── models/          # Model database
│   ├── routes/          # Endpoint API
│   ├── middlewares/     # Auth middleware
│   ├── utils/           # Helper functions
│   ├── .env.example
│   ├── server.js
│   └── package.json
├── .github/workflows/   # CI/CD
│   └── ci-cd.yml
├── docker-compose.yml   # Docker setup
├── .gitignore
└── README.md            # Dokumentasi proyek

name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x]

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'

    # Install dependencies
    - name: Install Frontend Dependencies
      run: |
        cd client
        npm ci
      working-directory: ./client

    - name: Install Backend Dependencies
      run: |
        cd server
        npm ci
      working-directory: ./server

    # Linting
    - name: Run ESLint (Frontend)
      run: npm run lint
      working-directory: ./client

    - name: Run ESLint (Backend)
      run: npm run lint
      working-directory: ./server

    # Testing
    - name: Run Frontend Tests
      run: npm test
      working-directory: ./client

    - name: Run Backend Tests
      run: npm test
      working-directory: ./server
      env:
        DB_TEST_URL: postgresql://test:test@localhost:5432/auth_test

    # Build
    - name: Build Frontend
      run: npm run build
      working-directory: ./client

    # Docker build
    - name: Build Docker Images
      run: docker-compose -f docker-compose.yml build

    # Deploy to Staging (manual trigger)
  deploy-staging:
    needs: build-and-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to AWS Staging
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-southeast-1

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Deploy with Docker Compose
      run: |
        docker-compose -f docker-compose.prod.yml up -d

        sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant CI as CI Server
    participant AWS as AWS

    Dev->>GH: Push code ke branch
    GH->>CI: Trigger workflow
    CI->>CI: Install dependencies
    CI->>CI: Linting & Testing
    alt Semua test pass
        CI->>CI: Build aplikasi
        CI->>CI: Build Docker image
        CI->>AWS: Deploy ke staging
        AWS-->>CI: Konfirmasi deploy
        CI-->>GH: Laporkan status
    else Ada error
        CI-->>GH: Laporkan error
        CI-->>Dev: Kirim notifikasi
    end


    CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_verified BOOLEAN DEFAULT false,
    verification_token VARCHAR(100),
    reset_token VARCHAR(100),
    reset_token_expiry TIMESTAMP
);

CREATE TABLE sessions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    session_token VARCHAR(255) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    device_info TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE social_logins (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL, -- 'google', 'facebook', etc.
    provider_id VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Index untuk pencarian cepat
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_sessions_token ON sessions(session_token);
CREATE INDEX idx_social_logins_provider ON social_logins(provider, provider_id);

erDiagram
    USERS ||--o{ SESSIONS : has
    USERS ||--o{ SOCIAL_LOGINS : has
    USERS {
        int id PK
        varchar(255) email
        varchar(255) password_hash
        varchar(100) full_name
        timestamp created_at
        timestamp updated_at
        boolean is_verified
        varchar(100) verification_token
        varchar(100) reset_token
        timestamp reset_token_expiry
    }
    SESSIONS {
        int id PK
        int user_id FK
        varchar(255) session_token
        timestamp expires_at
        text device_info
        timestamp created_at
    }
    SOCIAL_LOGINS {
        int id PK
        int user_id FK
        varchar(50) provider
        varchar(255) provider_id
        timestamp created_at
    }


# Buat struktur folder
mkdir -p auth-system/{client,server} && cd auth-system
touch .gitignore README.md docker-compose.yml

# Inisialisasi Git
git init

# Setup frontend (React)
npx create-react-app client

# Setup backend (Node.js)
mkdir server
cd server
npm init -y
npm install express dotenv bcrypt jsonwebtoken pg nodemailer
touch server.js

version: '3.8'

services:
  db:
    image: postgres:14
    environment:
      POSTGRES_USER: auth_user
      POSTGRES_PASSWORD: auth_pass
      POSTGRES_DB: auth_db
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

  server:
    build: ./server
    ports:
      - "5000:5000"
    environment:
      DB_HOST: db
      DB_USER: auth_user
      DB_PASSWORD: auth_pass
      DB_NAME: auth_db
      JWT_SECRET: mysecretkey
    depends_on:
      - db

  client:
    build: ./client
    ports:
      - "3000:3000"
    depends_on:
      - server

volumes:
  postgres-data:


  require('dotenv').config();
const express = require('express');
const app = express();
const PORT = process.env.PORT || 5000;

// Middleware dasar
app.use(express.json());

// Test endpoint
app.get('/api/health', (req, res) => {
  res.json({
    status: 'ok',
    message: 'Authentication service is running',
    timestamp: new Date()
  });
});

// Error handling
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Internal Server Error' });
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

gantt
    title Roadmap Pengembangan Sistem Autentikasi
    dateFormat  YYYY-MM-DD
    section Fase 1
    Setup Proyek           :a1, 2023-10-01, 3d
    Implementasi CI/CD     :a2, after a1, 2d
    Desain Database        :a3, after a1, 2d

    section Fase 2
    Endpoint Registrasi    :b1, 2023-10-06, 3d
    Endpoint Login         :b2, after b1, 3d
    Email Verifikasi       :b3, after b2, 2d

    section Fase 3
    Lupa Password         :c1, 2023-10-14, 3d
    Manajemen Sesi        :c2, after c1, 2d
    OAuth Integration     :c3, after c2, 4d

    section Fase 4
    Frontend Login        :d1, 2023-10-20, 3d
    Frontend Registrasi   :d2, after d1, 3d
    Dashboard Pengguna    :d3, after d2, 4d

    section Deployment
    Staging Environment   :e1, 2023-10-30, 2d
    Production Deployment :e2, after e1, 2d
