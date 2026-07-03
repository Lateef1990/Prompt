# Build the Novelty International Academy Student Portal

Create a complete Node.js/Express web application called **"Novelty International Academy Student Portal"** — a secondary school result-checking platform for students.

## Tech Stack
- **Runtime:** Node.js with Express
- **Templating:** EJS with express-ejs-layouts
- **Styling:** Tailwind CSS (CDN) + custom CSS
- **Icons:** Font Awesome 6
- **Font:** Google Fonts Inter
- **Database:** MySQL 2 (with read replica support)
- **Session Store:** Redis (primary) / MySQL (fallback)
- **Caching:** Redis (primary) / In-memory Map (fallback)
- **Security:** Helmet, rate-limiting, CSRF, bcrypt, input sanitization

## Project Structure

```
/
├── app.js
├── package.json
├── .env
├── config/
│   └── db.js
├── middleware/
│   ├── auth.js
│   ├── cache.js
│   ├── locals.js
│   └── security.js
├── routes/
│   ├── home.js
│   ├── auth.js
│   ├── results.js
│   └── contact.js
├── views/
│   ├── layout.ejs
│   ├── index.ejs
│   ├── login.ejs
│   ├── dashboard.ejs
│   ├── results.ejs
│   ├── change-password.ejs
│   ├── 404.ejs
│   ├── 429.ejs
│   ├── 500.ejs
│   └── partials/
│       └── footer.ejs
├── public/
│   ├── css/style.css
│   ├── js/
│   │   ├── app.js
│   │   ├── home.js
│   │   ├── dashboard.js
│   │   └── results.js
│   └── images/
│       ├── logo.jpg
│       ├── hero-bg.svg
│       ├── about-school.png
│       ├── vision-icon.svg
│       ├── mission-icon.svg
│       ├── icons/ (icon-*.png & icon-*.svg)
│       └── gallery/ (gallery-1 through gallery-6 .png & .webp)
└── utils/
    ├── bcrypt.js
    └── bcrypt-worker.js
```

## Color Palette
- Primary dark blue: `#1e3a5f`
- Accent gold: `#f0b429` (hover `#d49f1f`)
- Gradient background: `from-[#1e3a5f] to-[#2a5298]`
- Body text: gray-800, gray-600, gray-500, gray-400

## Database Schema (MySQL)
Create database `novelty_academy` with tables:
- **students** — id, full_name, registration_number (unique), password (bcrypt hash), class_id, gender, department, passport_path
- **classes** — id, name
- **sessions** — id, name (e.g. "2024/2025")
- **terms** — id, name (e.g. "First Term")
- **subjects** — id, code, name
- **results** — id, student_id, session_id, term_id, subject_id, ca_score, exam_score, total_score, grade, remark
- **student_ranks** — term_id, class_id, student_id, total_score, position, calculated_at (unique on term_id+class_id+student_id)
- **contact_inquiries** — id, full_name, email, phone, subject, message, created_at
- **newsletter_subscribers** — id, email (unique), created_at

## Pages & Routes

### 1. Home Page (`GET /`)
- Fixed navbar with logo, school name, nav links (Home, About Us, Academics, Gallery, Contact Us), Login button
- Mobile hamburger menu with full-screen overlay
- Hero section with background image, gradient overlay, tagline "Empowering Minds, Shaping Futures", subtitle "Novelty International Academy, Kishi — Excellence in Education", CTA buttons "Check Your Result" and "Learn More"
- About Us section: image left, title "Welcome to Novelty International Academy", description text, animated counters (25 Years, 1500+ Students, 85 Teachers, 12 Awards)
- Vision & Mission section: two cards with icons
- Academics section: 4 cards — Junior Secondary, Science, Arts, Commercial
- Photo Gallery section: 6 images in grid with lightbox on click
- "Why Choose Us" section: 4 cards — Qualified Teachers, Modern Facilities, Character Development, Affordable Fees
- Contact section: form (name, email, phone, subject, message) + contact info (address, phone, email, school hours)
- Footer with logo, description, social links, quick links, contact info, newsletter subscribe form, copyright "© 2026 Novelty International Academy. All rights reserved. Developed by Al-Mansur Software Tech."
- Login modal (triggered by any login/result button)
- Fade-in scroll animations

### 2. Login Page (`GET /login`, `POST /login`)
- Centered card with gradient background, school logo, form with registration number and password
- AJAX submission for modal login, server-side validation, bcrypt password comparison
- Session regeneration on login, student data stored in session
- Rate limiting: 10 attempts per 15 min per registration number, 50 per IP

### 3. Dashboard (`GET /dashboard` — auth required)
- Sidebar with student photo/initial, name, registration number, navigation (Dashboard, View Results, Change Password, Logout)
- Top bar with greeting, class, reg number
- Profile card with student details
- Quick stats cards (Sessions count, Terms count, Class name)
- Result selector with session and term dropdowns
- Change password modal with AJAX submission

### 4. Results Page (`GET /results?session_id=X&term_id=Y` — auth required)
- Header with student info, session/term name
- Results table: #, Subject Code, Subject Name, CA (40), Exam (60), Total (100), Grade, Remark
- Color-coded grade badges (A=green, B=blue, C=yellow, D/E=orange, F=red)
- Summary cards: Total Subjects, Total Score, Average, Position
- Print button, Back to Dashboard, Logout
- No results empty state

### 5. Change Password (`GET /change-password`, `POST /change-password` — auth required)
- Form with current password, new password, confirm new password
- Validation: min 8 chars, uppercase, lowercase, number, special char
- On success: session destroyed, redirect to login

### 6. Contact Form (`POST /contact`)
- Validates name (2-100 chars), email, message (10-5000 chars)
- Inserts into contact_inquiries table
- Displays success/error message on homepage

### 7. Newsletter (`POST /newsletter`)
- Inserts email into newsletter_subscribers (IGNORE duplicate)
- Rate limited to 3 per hour

### 8. Error Pages
- **404**: centered card with warning icon, "Back to Home"
- **429**: Server Busy with hourglass icon
- **500**: Server Error with cog icon

## Middleware Features
- **Security Headers**: X-Content-Type-Options: nosniff, X-Frame-Options: DENY, X-Robots-Tag: noindex/nofollow, Permissions-Policy (no camera/mic/geo)
- **CSRF**: Per-session token, rotated after each POST/PUT/PATCH/DELETE
- **Rate Limiting**: General 200 req/min, login 10/15min per credential, login IP 50/15min, contact 5/hour, newsletter 3/hour
- **Auth**: Redirects to /login if no session, Cache-Control: no-store
- **Compression**: gzip/brotli at level 6, threshold 512 bytes
- **HTTPS Redirect**: In production, redirects HTTP to HTTPS
- **Request Tracing**: UUID X-Request-Id header on every response
- **Caching**: Redis or in-memory cache for sessions, terms, results, ranks (TTLs: sessions/terms 1hr, results 1hr, ranks 30min)

## Security Requirements
- bcrypt password hashing with worker_threads (utils/bcrypt.js)
- Session cookie: httpOnly, secure in production, sameSite strict, 24hr maxAge
- Input validation on all forms (string length checks, regex patterns)
- SQL injection prevention via parameterized queries
- Path traversal protection on passport_path
- Content Security Policy via Helmet (scripts: self + tailwind CDN; styles: self + unsafe-inline + cloudflare + fonts)
- CSP upgrade-insecure-requests enabled
- Referrer policy: strict-origin-when-cross-origin

## Configuration (.env)
```
DB_HOST=localhost
DB_PORT=3306
DB_USER=student_portal_app
DB_PASSWORD="SX0nIB9^Vcx94BmX=X!Yn@8cu2ksyO7&"
DB_NAME=novelty_academy
DB_POOL_SIZE=100
DB_READ_HOST=
DB_READ_POOL_SIZE=100
REDIS_URL=redis://localhost:6379
SESSION_SECRET=<random-32-char-min>
PORT=3003
```

## JavaScript Features (client-side)
- **app.js**: IntersectionObserver for fade-in animations, hide images on error
- **home.js**: Mobile menu toggle, animated counter on scroll, gallery lightbox, login modal with AJAX form submission
- **dashboard.js**: Mobile sidebar toggle, change password modal with client + AJAX validation
- **results.js**: Print button

## CSS (custom)
- Inter font family globally
- Smooth scrolling
- Fade-in animation (translateY + opacity)
- Sticky nav with backdrop blur
- Hero gradient overlay
- Typewriter cursor animation on headline
- Gallery overlay on hover
- Lightbox overlay
- @media print hides `.no-print` elements
