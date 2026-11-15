# MediQuick — Backend Implementation Guide
## Node.js + MongoDB + Swagger REST API

---

## 📁 Project Structure

```
mediquick/
├── server.js                  ← Entry point
├── package.json
├── .env.example               ← Copy to .env and fill in values
│
├── config/
│   ├── db.js                  ← MongoDB connection
│   └── swagger.js             ← Swagger/OpenAPI config
│
├── models/
│   ├── User.js                ← All roles (customer/pharmacy/doctor/admin)
│   ├── Otp.js                 ← OTP with TTL index (auto-expires in 5 min)
│   ├── PharmacyProfile.js     ← Pharmacy registration + geolocation
│   ├── DoctorProfile.js       ← Doctor registration + approval
│   ├── Medicine.js            ← Global medicine catalogue
│   ├── Inventory.js           ← Per-pharmacy stock
│   ├── Order.js               ← Medicine orders with status history
│   ├── Appointment.js         ← Doctor appointments
│   └── Slot.js                ← Doctor availability slots
│
├── routes/
│   ├── auth.js                ← OTP send/verify, register pharmacy/doctor
│   ├── medicines.js           ← Medicine catalogue + search
│   ├── pharmacies.js          ← Pharmacy list, inventory CRUD
│   ├── orders.js              ← Place, track, update, cancel orders
│   ├── doctors.js             ← Doctor list, slot management, appointments
│   └── admin.js               ← Approvals, user management, dashboard
│
├── controllers/
│   └── authController.js      ← Auth business logic
│
├── middleware/
│   ├── auth.js                ← JWT protect + role-based authorize()
│   ├── errorHandler.js        ← Global error handler
│   └── upload.js              ← Multer for prescription/image uploads
│
└── utils/
    └── seed.js                ← Development seed data
```

---

## 🚀 Step-by-Step Setup

### Step 1 — Prerequisites
```bash
node -v    # >= 18.x
npm -v     # >= 9.x
mongod     # MongoDB running locally OR use Atlas URI
```

### Step 2 — Install & configure
```bash
cd mediquick
npm install

# Copy and edit environment variables
cp .env.example .env
```

Edit `.env`:
```env
PORT=5000
NODE_ENV=development
MONGO_URI=mongodb://localhost:27017/mediquick
JWT_SECRET=change_this_to_a_long_random_string
JWT_EXPIRES_IN=7d
```

### Step 3 — Seed test data
```bash
npm run seed
```
This creates:
| Role     | Phone          | Notes                     |
|----------|---------------|---------------------------|
| Admin    | +919000000000 | Full platform access      |
| Pharmacy | +919111111111 | MedPlus — approved        |
| Doctor   | +919444444444 | Dr. Priya Sharma — approved |
| Customer | +919777777777 | Test customer             |

> **In dev mode, OTP is printed to the console — use any 6-digit code returned.**

### Step 4 — Start the server
```bash
npm run dev          # Development (nodemon)
npm start            # Production
```

### Step 5 — Open Swagger UI
```
http://localhost:5000/api-docs
```

---

## 🔌 API Endpoints Reference

### Auth (`/api/v1/auth`)
| Method | Endpoint                    | Access  | Description                       |
|--------|-----------------------------|---------|-----------------------------------|
| POST   | `/send-otp`                 | Public  | Send OTP to phone number          |
| POST   | `/verify-otp`               | Public  | Verify OTP → get JWT token        |
| POST   | `/register/pharmacy`        | Pharmacy| Submit pharmacy registration      |
| POST   | `/register/doctor`          | Doctor  | Submit doctor registration        |
| GET    | `/me`                       | Any     | Get current user profile          |

### Medicines (`/api/v1/medicines`)
| Method | Endpoint      | Access  | Description                        |
|--------|---------------|---------|------------------------------------|
| GET    | `/`           | Public  | List + search (`?q=para&category=`) |
| GET    | `/:id`        | Public  | Single medicine                    |
| POST   | `/`           | Admin   | Add medicine to catalogue          |
| PUT    | `/:id`        | Admin   | Update medicine                    |

### Pharmacies (`/api/v1/pharmacies`)
| Method | Endpoint               | Access   | Description                     |
|--------|------------------------|----------|---------------------------------|
| GET    | `/`                    | Public   | List approved (`?lat=&lng=`)    |
| GET    | `/:id`                 | Public   | Single pharmacy                 |
| GET    | `/:id/inventory`       | Public   | Stock list (`?q=search`)        |
| POST   | `/inventory`           | Pharmacy | Add/update stock                |
| PUT    | `/inventory/:itemId`   | Pharmacy | Edit inventory item             |

### Orders (`/api/v1/orders`)
| Method | Endpoint           | Access   | Description                     |
|--------|--------------------|----------|---------------------------------|
| POST   | `/`                | Customer | Place order                     |
| GET    | `/my`              | Customer | My orders                       |
| GET    | `/pharmacy`        | Pharmacy | Incoming orders (`?status=`)    |
| GET    | `/:id`             | Any      | Order detail                    |
| PATCH  | `/:id/status`      | Pharmacy/Admin | Update status            |
| POST   | `/:id/prescription`| Customer | Upload prescription (multipart) |
| PATCH  | `/:id/cancel`      | Customer | Cancel pending order            |

### Doctors (`/api/v1/doctors`)
| Method | Endpoint                      | Access   | Description                   |
|--------|-------------------------------|----------|-------------------------------|
| GET    | `/`                           | Public   | List doctors (`?specialization=`) |
| GET    | `/:id`                        | Public   | Doctor profile                |
| GET    | `/:id/slots?date=YYYY-MM-DD`  | Public   | Available slots               |
| POST   | `/slots`                      | Doctor   | Add slots                     |
| POST   | `/:id/book`                   | Customer | Book appointment              |
| GET    | `/appointments/my`            | Customer | My appointments               |
| GET    | `/appointments/incoming`      | Doctor   | Doctor's appointments         |
| PATCH  | `/appointments/:id`           | Doctor   | Complete/update appointment   |

### Admin (`/api/v1/admin`)
| Method | Endpoint                     | Access | Description               |
|--------|------------------------------|--------|---------------------------|
| GET    | `/dashboard`                 | Admin  | Platform statistics       |
| GET    | `/pharmacies/pending`        | Admin  | Pending pharmacies        |
| PATCH  | `/pharmacies/:id/approve`    | Admin  | Approve pharmacy          |
| PATCH  | `/pharmacies/:id/reject`     | Admin  | Reject pharmacy           |
| GET    | `/doctors/pending`           | Admin  | Pending doctors           |
| PATCH  | `/doctors/:id/approve`       | Admin  | Approve doctor            |
| PATCH  | `/doctors/:id/reject`        | Admin  | Reject doctor             |
| GET    | `/users`                     | Admin  | All users (paginated)     |
| PATCH  | `/users/:id/suspend`         | Admin  | Suspend user              |
| PATCH  | `/users/:id/activate`        | Admin  | Reactivate user           |

---

## 🔐 Authentication Flow

```
1. Frontend calls POST /auth/send-otp  { phone: "+91..." }
2. OTP sent via Twilio (or printed to console in dev)
3. Frontend calls POST /auth/verify-otp  { phone, otp, name?, role? }
4. Response: { token: "eyJ...", user: { id, name, phone, role } }
5. Store token in localStorage/sessionStorage
6. All subsequent requests: Authorization: Bearer <token>
```

---

## 🔗 Connecting Frontend to Backend

In `mediquickk.html`, replace the mock data calls with real API calls. Example:

```javascript
// Config
const API = 'http://localhost:5000/api/v1';
const getToken = () => localStorage.getItem('mq_token');
const headers = () => ({
  'Content-Type': 'application/json',
  ...(getToken() && { Authorization: `Bearer ${getToken()}` })
});

// Send OTP
async function sendOtp(phone) {
  const res = await fetch(`${API}/auth/send-otp`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ phone })
  });
  return res.json();
}

// Verify OTP
async function verifyOtp(phone, otp, name, role) {
  const res = await fetch(`${API}/auth/verify-otp`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ phone, otp, name, role })
  });
  const data = await res.json();
  if (data.token) localStorage.setItem('mq_token', data.token);
  return data;
}

// Load pharmacies
async function loadPharmacies(lat, lng) {
  const res = await fetch(`${API}/pharmacies?lat=${lat}&lng=${lng}`, { headers: headers() });
  return res.json();
}

// Place order
async function placeOrder(pharmacyId, items, deliveryAddress) {
  const res = await fetch(`${API}/orders`, {
    method: 'POST',
    headers: headers(),
    body: JSON.stringify({ pharmacyId, items, deliveryAddress, paymentMode: 'cod' })
  });
  return res.json();
}
```

---

## 🗄️ MongoDB Collections Summary

| Collection       | Purpose                                 |
|------------------|-----------------------------------------|
| users            | All users across roles                  |
| otps             | Temporary OTPs (TTL auto-deletes)       |
| pharmacyprofiles | Pharmacy business details               |
| doctorprofiles   | Doctor credentials & approval           |
| medicines        | Global medicine catalogue               |
| inventories      | Per-pharmacy stock (unique per medicine)|
| orders           | Medicine orders with status history     |
| appointments     | Doctor consultation bookings            |
| slots            | Doctor availability (unique per day+time)|

---

## 🚢 Production Checklist

- [ ] Set `NODE_ENV=production`
- [ ] Use MongoDB Atlas URI in `MONGO_URI`
- [ ] Set a strong `JWT_SECRET` (32+ chars)
- [ ] Configure Twilio credentials for real OTPs
- [ ] Set `CLIENT_URL` to your frontend domain
- [ ] Use HTTPS (behind nginx/Caddy)
- [ ] Enable MongoDB Atlas IP whitelist
- [ ] Use PM2 for process management: `pm2 start server.js`
- [ ] Set up log rotation

---

## 📦 Key Dependencies

| Package              | Purpose                              |
|----------------------|--------------------------------------|
| express              | Web framework                        |
| mongoose             | MongoDB ODM                          |
| jsonwebtoken         | JWT auth tokens                      |
| bcryptjs             | Password hashing                     |
| swagger-jsdoc        | Generate OpenAPI spec from JSDoc     |
| swagger-ui-express   | Serve Swagger UI                     |
| multer               | File uploads (prescriptions)         |
| twilio               | OTP SMS delivery                     |
| helmet               | Security headers                     |
| cors                 | Cross-origin requests                |
| express-rate-limit   | Rate limiting                        |
| nodemon              | Dev auto-restart                     |
