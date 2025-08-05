# Infinity Server

A TypeScript monorepo project for building scalable web applications.

## 📦 Packages

- **API Server** (`packages/api`) - Backend API service built with TypeScript
- **Web App** (`packages/web`) - React frontend application

## 🚀 Getting Started

### Prerequisites

- Node.js (v16 or higher)
- npm or yarn

### Installation

```bash
# Clone the repository
git clone https://github.com/alwayswron9/infinity-server-monorepo.git
cd infinity-server

# Install dependencies
npm install

# Build all packages
npm run build
```

## 🛠️ Development

```bash
# Start development mode (watch mode)
npm run dev

# Clean build artifacts
npm run clean
```

## 📁 Project Structure

```
infinity-server/
├── packages/
│   ├── api/          # Backend API server
│   └── web/          # React frontend app
├── src/              # Shared source code
├── package.json      # Root package configuration
├── tsconfig.json     # TypeScript configuration
└── README.md         # This file
```

## 🤝 Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📄 License

This project is licensed under the ISC License.