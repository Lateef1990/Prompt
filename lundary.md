# Al-Mansury Laundry Service - Full-Stack Web Application Prompt

## Project Overview
Build a **responsive** laundry management web application for **Al-Mansury Laundry Service** using:
- **Backend:** Node.js + Express.js (API-based architecture)
- **Frontend:** Express.js serves EJS/HTML views (server-side rendering)
- **Styling:** Tailwind CSS
- **Database:** MySQL
- **All business logic exposed via REST APIs**

**Contact Info:** 07063970588, 08079839286
**Company Name:** Al-Mansury Laundry Service

---

## Functional Requirements

### 1. Authentication & Authorization
- **Customer Registration:** Full Name, Email, Phone, Password, Address
- **Customer Login:** Email + Password with secure session/JWT
- **Admin Login:** Ask me for admin credentials during development
- **Role-based access:** Customer routes protected; Admin routes protected
- **Password hashing** using bcrypt (cost factor >= 10)
- **Account lockout** after 5 failed login attempts (15min cooldown)
- **Password policy:** Minimum 8 chars, 1 uppercase, 1 number, 1 special char
- **Secure password reset** flow (token expiry, email link)
- **Session management:** HTTP-only, Secure, SameSite=Strict cookies; session timeout after 30min inactivity

### 2. Home Page (Modern UI)
- Hero section with company name, tagline, and CTA buttons
- Services overview (wash, dry-clean, iron, fold, etc.)
- How it works section (steps: Book → Pickup → Process → Deliver)
- Pricing cards
- Customer testimonials / trust badges
- Contact section with phone numbers and address
- Footer with links and social media
- Fully responsive (mobile-first with Tailwind)

### 3. Customer Dashboard
- View booking history
- Track current booking status (Pending, Picked Up, Processing, Ready, Delivered)
- View/download receipts
- Profile management (update name, phone, address, password)
- Loyalty points / discount summary

### 4. Online Booking & Pickup Scheduling
- Select service type: Wash & Fold, Wash & Iron, Dry Clean, Iron Only, etc.
- Select cloth types with quantity (e.g., Shirt, Trouser, Bed Sheet, Blanket, Dress, etc.)
- **Each cloth type has a fixed unit price** stored in DB
- System auto-calculates **subtotal per cloth type** and **grand total**
- Schedule pickup date & time slot
- Choose delivery method: Doorstep Delivery or Self Pickup
- Enter pickup address
- **Auto-generate receipt number** (format: ALM-YYYYMMDD-XXXX)
- Submit booking → store in DB with receipt + total price

### 5. Booking Data Model (per booking)
| Field | Description |
|-------|-------------|
| receipt_no | Auto-generated unique receipt |
| customer_id | FK to customers |
| service_type | e.g., Wash & Fold |
| pickup_date | Scheduled pickup date |
| pickup_time | Time slot |
| delivery_method | Doorstep / Self Pickup |
| pickup_address | Text |
| items | JSON array of {cloth_type, qty, unit_price, subtotal} |
| subtotal | Sum of all items |
| discount_amount | Loyalty discount applied |
| grand_total | Final amount after discount |
| status | pending → picked_up → processing → ready → delivered |
| payment_status | pending / paid |
| created_at | Timestamp |

### 6. Cloth Types & Pricing (Admin-managed — including price updates)
| Cloth Type | Unit Price (₦) |
|------------|----------------|
| Shirt | 300 |
| Trouser | 400 |
| Bed Sheet | 500 |
| Blanket | 800 |
| Dress | 600 |
| Suit | 1000 |
| Towel | 250 |
| Saree | 700 |
| Kurta | 400 |
| Jeans | 350 |
| Agbada with Buba and Sokoto | 1000 |
| Buba with Sokoto | 600 |
| Others | Custom |

All cloth types and prices stored in MySQL `cloth_types` table, CRUD via admin. Admin can update the unit price of any cloth type at any time — price changes take effect on new bookings immediately.

### 7. Receipt & Invoice Printing
- Receipt page with company logo, name, contact info
- Customer details, booking details, itemized list
- Subtotal, discount, grand total
- Payment status
- Print-friendly CSS (no external lib required)

### 8. Payment Management
- Admin marks payment as **Paid** or stays **Pending**
- Admin **cannot edit** revenue reports (read-only)
- Payment history per customer

### 9. Admin Dashboard
- **Statistics Cards:** Total customers, Total bookings (today/week/month), Total revenue, Pending deliveries
- **Charts:** Revenue trend (line/bar), Booking status distribution (pie)
- **Recent bookings table** (latest 10)
- **Quick actions:** New cloth type, View reports

### 10. Reports & Analytics (Read-Only for Admin)
- **Daily Report:** Date filter, total bookings, total revenue, avg order value
- **Weekly Report:** Week selector, aggregated revenue per day, comparison
- **Monthly Report:** Month & year, total revenue, booking count, top service type
- **Yearly Report:** Year selector, revenue per month, YoY comparison
- All revenue reports are **view-only** — no edit/delete allowed
- Export reports as PDF or CSV (optional)

### 11. Customer Management (Admin)
- View all customers
- Search by name, email, phone
- View each customer's booking history
- Loyalty points management

### 12. Order Tracking (Status Management)
- Admin can update booking status: `pending → picked_up → processing → ready → delivered`
- Customer sees real-time status on their dashboard
- Status change timestamps logged

### 13. Customer Loyalty & Discount
- Customers earn **points** per booking (e.g., 1 point per ₦100 spent)
- Points accumulate and can be redeemed for discounts
- Discount tiers:
  - 100 points → 5% off
  - 200 points → 10% off
  - 500 points → 20% off
- Points & discount applied automatically at checkout
- Admin can adjust points manually

### 14. Inventory Management
- Track laundry supplies (detergent, fabric softener, packaging, etc.)
- Admin CRUD: Add/edit/delete inventory items
- Low-stock alerts (show items below threshold)
- Inventory usage log (optional)

### 15. Notifications (Optional / Extendable)
- **Email:** Nodemailer for booking confirmation, status updates, receipts
- **SMS:** Integration placeholder (e.g., Twilio or Africa's Talking)
- Database log for sent notifications

### 16. Doorstep Delivery
- Customers choose **Doorstep Delivery** during booking
- Admin assigns delivery personnel (future scope)
- Delivery address stored per booking

---

## Technical Requirements

### Database Tables (MySQL)
1. `customers` - id, name, email, phone, password, address, loyalty_points, created_at
2. `admins` - id, name, email, password, role
3. `cloth_types` - id, name, unit_price, is_active
4. `service_types` - id, name, description, is_active
5. `bookings` - id, receipt_no, customer_id, service_type_id, pickup_date, pickup_time, delivery_method, pickup_address, subtotal, discount_amount, grand_total, status, payment_status, created_at, updated_at
6. `booking_items` - id, booking_id, cloth_type_id, quantity, unit_price, subtotal
7. `inventory` - id, item_name, quantity, unit, threshold, created_at
8. `revenue_reports` - id, report_type (daily/weekly/monthly/yearly), period, total_bookings, total_revenue, generated_at (auto-populated via triggers/scheduled queries)
9. `notifications` - id, customer_id, type (email/sms), message, sent_at

### API Endpoints Structure

#### Auth
- `POST /api/auth/register` - Customer registration
- `POST /api/auth/login` - Customer login
- `POST /api/admin/login` - Admin login
- `POST /api/auth/logout` - Logout

#### Customers (protected)
- `GET /api/customers/profile` - Get profile
- `PUT /api/customers/profile` - Update profile
- `GET /api/customers/bookings` - My bookings
- `GET /api/customers/bookings/:id` - Booking detail
- `GET /api/customers/loyalty` - Loyalty points & discount info

#### Bookings (customer)
- `GET /api/bookings/new` - Get cloth types & service types for form
- `POST /api/bookings` - Create booking (auto-calc price, generate receipt)
- `GET /api/bookings/:id/receipt` - View receipt
- `GET /api/bookings/:id/print` - Print-friendly receipt

#### Admin: Bookings
- `GET /api/admin/bookings` - All bookings (with filters)
- `GET /api/admin/bookings/:id` - Booking detail
- `PUT /api/admin/bookings/:id/status` - Update status
- `PUT /api/admin/bookings/:id/payment` - Mark paid/pending

#### Admin: Customers
- `GET /api/admin/customers` - List all
- `GET /api/admin/customers/:id` - Customer detail + bookings
- `PUT /api/admin/customers/:id/loyalty` - Adjust loyalty points

#### Admin: Reports (READ-ONLY)
- `GET /api/admin/reports/daily?date=` - Daily report
- `GET /api/admin/reports/weekly?week=&year=` - Weekly report
- `GET /api/admin/reports/monthly?month=&year=` - Monthly report
- `GET /api/admin/reports/yearly?year=` - Yearly report
- `GET /api/admin/reports/summary` - Dashboard summary stats

#### Admin: Cloth Types
- `GET /api/admin/cloth-types` - List all
- `POST /api/admin/cloth-types` - Create
- `PUT /api/admin/cloth-types/:id` - Update
- `DELETE /api/admin/cloth-types/:id` - Delete

#### Admin: Service Types
- `GET /api/admin/service-types` - List all
- `POST /api/admin/service-types` - Create
- `PUT /api/admin/service-types/:id` - Update
- `DELETE /api/admin/service-types/:id` - Delete

#### Admin: Inventory
- `GET /api/admin/inventory` - List all
- `POST /api/admin/inventory` - Add item
- `PUT /api/admin/inventory/:id` - Update
- `DELETE /api/admin/inventory/:id` - Delete

### Frontend Pages (Express Views)
| Route | View | Access |
|-------|------|--------|
| `/` | Home page | Public |
| `/about` | About Us | Public |
| `/services` | Services | Public |
| `/pricing` | Pricing | Public |
| `/contact` | Contact | Public |
| `/login` | Customer login | Public |
| `/register` | Customer registration | Public |
| `/customer/dashboard` | Customer dashboard | Customer |
| `/customer/bookings` | My bookings | Customer |
| `/customer/bookings/new` | New booking | Customer |
| `/customer/bookings/:id` | Booking detail | Customer |
| `/customer/bookings/:id/receipt` | Receipt | Customer |
| `/customer/profile` | Profile | Customer |
| `/customer/loyalty` | Loyalty | Customer |
| `/admin/dashboard` | Admin dashboard | Admin |
| `/admin/bookings` | Manage bookings | Admin |
| `/admin/customers` | Customers list | Admin |
| `/admin/cloth-types` | Cloth types | Admin |
| `/admin/service-types` | Service types | Admin |
| `/admin/inventory` | Inventory | Admin |
| `/admin/reports` | Reports | Admin |
| `/admin/reports/daily` | Daily report | Admin |
| `/admin/reports/weekly` | Weekly report | Admin |
| `/admin/reports/monthly` | Monthly report | Admin |
| `/admin/reports/yearly` | Yearly report | Admin |

### Project File Structure
```
al-mansury-laundry/
├── app.js
├── package.json
├── .env
├── tailwind.config.js
├── public/
│   ├── css/
│   │   └── output.css
│   ├── js/
│   │   └── main.js
│   └── images/
├── src/
│   ├── config/
│   │   └── db.js
│   ├── models/
│   │   ├── Customer.js
│   │   ├── Booking.js
│   │   ├── BookingItem.js
│   │   ├── ClothType.js
│   │   ├── ServiceType.js
│   │   └── Inventory.js
│   ├── controllers/
│   │   ├── authController.js
│   │   ├── customerController.js
│   │   ├── bookingController.js
│   │   ├── adminController.js
│   │   └── reportController.js
│   ├── routes/
│   │   ├── authRoutes.js
│   │   ├── customerRoutes.js
│   │   ├── bookingRoutes.js
│   │   └── adminRoutes.js
│   ├── middleware/
│   │   ├── auth.js
│   │   └── adminAuth.js
│   └── utils/
│       ├── receiptGenerator.js
│       └── loyaltyCalculator.js
├── views/
│   ├── layouts/
│   │   └── main.ejs
│   ├── partials/
│   │   ├── header.ejs
│   │   ├── footer.ejs
│   │   └── admin-sidebar.ejs
│   ├── pages/
│   │   ├── index.ejs
│   │   ├── login.ejs
│   │   ├── register.ejs
│   │   └── ...
│   └── admin/
│       ├── dashboard.ejs
│       ├── bookings.ejs
│       └── ...
├── database/
│   └── schema.sql
└── .gitignore
```

### Key Business Logic Rules
1. **Receipt Number:** `ALM-YYYYMMDD-XXXX` where XXXX is zero-padded sequential daily number
2. **Price Calculation:** `subtotal = Σ(qty × unit_price)`, `discount = based on loyalty tier`, `grand_total = subtotal - discount`
3. **Loyalty Points:** Earned on payment (paid status). Formula: `points = floor(grand_total / 100)`. Redeemed points reduce available balance.
4. **Reports:** Queries aggregate from `bookings` table. NEVER allow UPDATE/DELETE on aggregate data. Reports are generated fresh from transactional data every time.
5. **Payment Guard:** Booking status cannot become `delivered` unless `payment_status = paid`.
6. **Notification Trigger:** When booking status changes, log to `notifications` table (future: send email/SMS).

---

## Deliverable Prompt for AI Coding Agent

> Build a complete, production-ready laundry management system for **Al-Mansury Laundry Service** using **Node.js + Express.js + Tailwind CSS + MySQL**. All functionality must be exposed through REST APIs. Express.js renders EJS views as the frontend.
>
> **Core Features:**
> 1. **Public website:** Modern, responsive home page, about, services, pricing, contact pages. Contact: 07063970588, 08079839286.
> 2. **Customer auth:** Register, login, profile management, loyalty points.
> 3. **Online booking:** Select service type → pick cloth types & quantities → auto-calculate price → schedule pickup → auto-generate receipt (format: ALM-YYYYMMDD-XXXX) → submit booking.
> 4. **Customer dashboard:** View bookings, track status, view/print receipts, loyalty summary.
> 5. **Admin dashboard:** Stats cards, revenue chart, booking status chart, recent bookings.
> 6. **Admin reports (READ-ONLY):** Daily, weekly, monthly, yearly aggregated revenue reports. No edit/delete allowed on reports.
> 7. **Payment management:** Admin marks payment as paid/pending. Payment must be paid before delivery.
> 8. **Order tracking:** Admin updates status (pending→picked_up→processing→ready→delivered). Customer sees live updates.
> 9. **Admin CRUD:** Cloth types (including updating unit prices — changes apply to new bookings immediately), service types, inventory items.
> 10. **Customer management:** Admin views all customers, booking history, adjusts loyalty points.
> 11. **Receipt printing:** Print-friendly receipt page with company details and itemized billing.
> 12. **Loyalty & Discount:** 1 point per ₦100 spent; 100pts=5% off, 200pts=10% off, 500pts=20% off. Auto-applied at checkout.
> 13. **Inventory management:** Track supplies with low-stock alerts.
> 14. **Notifications:** DB logging for email/SMS (extendable with Nodemailer/Twilio).
>
> **Architecture:**
> - MySQL database with customers, admins, cloth_types, service_types, bookings, booking_items, inventory, notifications tables
> - RESTful API endpoints under `/api/`
> - Express.js serves EJS views
> - Tailwind CSS for responsive styling
> - Session or JWT-based auth
> - bcrypt for password hashing
>
> Provide full source code with database schema, API implementation, views, and Tailwind configuration.
