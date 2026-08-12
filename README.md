# Mini ERP + CRM Portal

A full-stack ERP and CRM web application for managing customers, products, challans, and stock operations.

## Live Demo

Frontend: https://mini-erp-crm-portal-rho.vercel.app/

Backend: https://mini-erp-crm-portal-g6ej.onrender.com/

## Features

- User login and authentication
- Role-based access
- Customer management
- Product management
- Challan management
- Stock management
- Dashboard
- PostgreSQL database

## User Roles

- Admin
- Sales
- Warehouse
- Accounts

## Tech Stack

**Frontend**
- React
- Vite
- JavaScript
- CSS

**Backend**
- Node.js
- Express.js
- TypeScript
- JWT

**Database**
- PostgreSQL
- Prisma ORM

**Deployment**
- Vercel
- Render

## Demo Credentials

### Admin
Email: `admin@erp.test`  
Password: `Password123!`

### Sales
Email: `sales@erp.test`  
Password: `Password123!`

### Warehouse
Email: `warehouse@erp.test`  
Password: `Password123!`

### Accounts
Email: `accounts@erp.test`  
Password: `Password123!`

## Project Structure

```text
mini-erp-crm-portal/
├── frontend/
├── backend/
│   ├── src/
│   └── prisma/
├── README.md
└── .gitignore
```

## Running Locally

### Clone the repository

```bash
git clone https://github.com/JCharanjithRao/mini-erp-crm-portal.git
cd mini-erp-crm-portal
```

### Backend

```bash
cd backend
npm install
npm run dev
```

### Frontend

Open another terminal:

```bash
cd frontend
npm install
npm run dev
```

Create the required `.env` files with your database, JWT, and API configuration before running the application.

## API

Backend API:

https://mini-erp-crm-portal-g6ej.onrender.com/

Health Check:

https://mini-erp-crm-portal-g6ej.onrender.com/health

## Deployment

The frontend is deployed using Vercel and the backend is deployed using Render.

The backend uses a PostgreSQL database hosted through Render and Prisma for database management.

## Security

- JWT authentication
- Password hashing
- Role-based access control
- Environment variables for sensitive information

Do not commit `.env` files or secrets to GitHub.

## Future Improvements

- Advanced analytics
- Inventory reports
- Sales reports
- PDF generation
- Notifications
- Search and filtering
- Improved mobile responsiveness

## Author

J Charanjith Rao

GitHub: https://github.com/JCharanjithRao


## Architecture

```text
                  USER
                    |
                    v
          +------------------+
          |  React + Vite    |
          |    Frontend      |
          |    Vercel        |
          +--------+---------+
                   |
                REST API
                   |
                   v
          +------------------+
          | Node.js +        |
          | Express +        |
          | TypeScript       |
          |    Render        |
          +--------+---------+
                   |
                   v
          +------------------+
          |     Prisma       |
          |       ORM        |
          +--------+---------+
                   |
                   v
          +------------------+
          |    PostgreSQL    |
          |     Render       |
          +------------------+
```

### Architecture Flow

The user interacts with the React and Vite frontend, which is deployed on Vercel.

The frontend communicates with the Node.js, Express, and TypeScript backend through REST APIs.

The backend handles authentication, authorization, business logic, and API requests. It uses Prisma ORM to communicate with the PostgreSQL database.

The PostgreSQL database stores the application's persistent data and is hosted on Render.

Overall data flow:

```text
User → React Frontend → REST API → Node.js Backend → Prisma ORM → PostgreSQL
```

