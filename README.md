# Mini ERP and CRM Portal

A full-stack operations portal for a small wholesale or distribution business. It brings customer records, product inventory, stock history, and delivery challans into one web application.

## What the application does

- Stores customer contact and business information.
- Keeps a product catalogue with SKU, price, warehouse location, and stock level.
- Records every deliberate stock addition or removal as a stock movement.
- Creates delivery challans containing one or more product lines.
- Reduces inventory only after a challan is confirmed.
- Supports four roles: Admin, Sales, Warehouse, and Accounts.

## Main workflow

```text
Sign in -> create customers/products -> add stock -> create challan draft
-> confirm challan -> stock is reduced and an OUT movement is recorded
```

The challan confirmation step is protected by a database transaction. If even one requested product does not have enough stock, the confirmation fails and no product quantity is changed.

## Technology used

| Area | Tools |
| --- | --- |
| User interface | React, TypeScript, Vite, React Router |
| API | Node.js, Express, TypeScript |
| Database access | Prisma ORM |
| Database | PostgreSQL |
| Authentication | JSON Web Tokens and bcrypt |
| Input validation | Zod |
| Hosting configuration | Render and Vercel |

## Folder overview

```text
backend/                 Express API and Prisma database layer
  prisma/                Database schema, migrations, seed script
  src/controllers/       Business logic for each module
  src/routes/            API endpoint definitions
  src/middleware/        Authentication, permissions, error handling
  src/validation/        Zod input validation rules

frontend/                React single-page application
  src/api/               Typed requests to the backend
  src/context/           Login state shared by the application
  src/pages/             Customer, product, challan, and login screens
  src/components/        Reusable UI helpers
postman/                 API test collection
```

## Role permissions

| Role | Access |
| --- | --- |
| Admin | Full access to all modules |
| Sales | Create and edit customers; create, edit, confirm, and cancel challans |
| Warehouse | Create and edit products; record stock movements |
| Accounts | Read-only access to customers, products, and challans |

## Run the project locally

### Requirements

- Node.js 20 or later
- PostgreSQL database

### Backend

```bash
cd backend
npm install
```

Create `backend/.env` using `backend/.env.example` as a template. Set a PostgreSQL `DATABASE_URL` and a strong `JWT_SECRET`.

```bash
npx prisma migrate deploy
npm run seed
npm run dev
```

The API starts at `http://localhost:4000`. The health endpoint is available at `http://localhost:4000/health`.

### Frontend

In a second terminal:

```bash
cd frontend
npm install
npm run dev
```

Open the URL displayed by Vite, usually `http://localhost:5173`.

## Demo accounts

After running the seed command, use any of these accounts. The password for each is `Password123!`.

| Role | Email |
| --- | --- |
| Admin | admin@erp.test |
| Sales | sales@erp.test |
| Warehouse | warehouse@erp.test |
| Accounts | accounts@erp.test |

## Manual test cases

Use the following tests after starting both applications.

| ID | Scenario | Steps | Expected result |
| --- | --- | --- | --- |
| TC-01 | Successful login | Sign in as `admin@erp.test` with the seeded password. | Dashboard opens and displays the Admin role. |
| TC-02 | Invalid login | Enter a valid email with an incorrect password. | A login error appears; no token is issued. |
| TC-03 | Customer search | Create a customer, then search using part of the name or mobile number. | The matching customer is returned. |
| TC-04 | Duplicate SKU protection | Create a product, then attempt another product with the same SKU. | The second request is rejected with a SKU validation message. |
| TC-05 | Stock audit trail | Add stock IN for a product, then inspect its stock movement history. | Current stock increases and a dated IN record shows the reason and user. |
| TC-06 | Insufficient inventory | Create a challan requesting more units than currently available, then confirm it. | Confirmation is rejected; current stock remains unchanged. |
| TC-07 | Confirmed challan | Create a draft challan within available stock and confirm it. | Challan status becomes Confirmed, stock decreases, and OUT movements are recorded. |
| TC-08 | Role restriction | Log in as Accounts and attempt to create a product or customer through the API. | The API returns a permission error. |
| TC-09 | Draft-only editing | Confirm a challan, then attempt to edit it. | The system rejects the edit because it is no longer a draft. |

## API testing

Import `postman/ERP-CRM.postman_collection.json` into Postman. Its `baseUrl` collection variable defaults to `http://localhost:4000/api`.

Run an Auth login request first. It saves the JWT in the collection variable used by the protected customer, product, and challan requests.

## Important implementation notes

- Product stock should not be changed directly when editing a product. Use a stock movement so the history stays auditable.
- A challan stores product name, SKU, and price snapshots. Later product edits do not alter past challans.
- Cancelling a confirmed challan does not automatically restore stock. A warehouse user should record a separate IN movement if goods return.
- Login state is held only in browser memory, so refreshing the page signs the user out.

## Deployment

`render.yaml` contains the backend and PostgreSQL setup for Render. The frontend can be deployed from the `frontend` folder to Vercel. Set `VITE_API_BASE_URL` in Vercel to the deployed backend URL plus `/api`, then set `FRONTEND_URL` in Render to the Vercel site URL for CORS.
