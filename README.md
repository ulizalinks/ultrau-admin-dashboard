# Uliza Tickets

A colorful, modern event ticketing website built with React, TypeScript, Tailwind CSS, and PostgreSQL. Uliza Tickets provides a seamless experience for discovering and purchasing tickets to events across Africa.

## Features

### 🎨 Modern Design
- Vibrant festival-style gradients and colors
- Responsive design that works on all devices
- Smooth animations powered by Framer Motion
- Professional shadcn/ui components

### 🎟️ Event Management
- Featured events grid on homepage
- Individual event detail pages
- Event search and filtering
- Rich event descriptions and imagery

### 💳 Guest Checkout
- No account registration required
- Simple checkout form with contact details
- Optional Paystack payment integration
- Secure payment processing

### 📱 Digital Tickets
- PDF ticket generation with QR codes
- Instant ticket delivery via email
- Unique ticket UIDs for security
- Professional ticket design with branding

### 🔍 CodeREADr Integration
- Ticket validation API endpoint
- QR code scanning support
- Ticket status tracking (unused/used)
- Scan logging and audit trail

## Tech Stack

### Frontend
- **React 18** with TypeScript
- **Tailwind CSS** for styling
- **shadcn/ui** component library
- **Framer Motion** for animations
- **Lucide React** for icons
- **Wouter** for routing
- **TanStack Query** for data fetching

### Backend
- **Express.js** server
- **PostgreSQL** database
- **Drizzle ORM** for database operations
- **PDFKit** for ticket generation
- **QRCode** library for QR code generation

### Development
- **TypeScript** for type safety
- **Vite** for fast development
- **ESBuild** for production builds
- **Tailwind CSS** for styling

## Quick Start

### Prerequisites
- Node.js 18+ 
- PostgreSQL database (provided automatically on Replit)

## 🙋‍♀️ **Frequently Asked Questions**

### **Q: Why is my "Publish" button disabled when creating events?**
**A:** The publish button requires all required fields AND valid data. Common issues:

1. **Image URL Format**: Must end with `.jpg`, `.png`, `.gif`, or `.webp`
   - ❌ `https://images.unsplash.com/photo-123456` 
   - ✅ `https://images.unsplash.com/photo-123456.jpg`

2. **Event Date**: Must be in the future
3. **Venue**: Must be at least 5 characters
4. **Event Name**: Must be 3-100 characters
5. **Ticket Types**: Must have at least one valid ticket with name, price (≥50 KES), and quantity (≥1)

### **Q: How do I use Unsplash images for events?**
**A:** Add `.jpg` to the end of any Unsplash URL:
```
Original: https://images.unsplash.com/photo-1445794544025-63...
Working:  https://images.unsplash.com/photo-1445794544025-63...jpg
```

### **Q: What's the event approval process?**
**A:** 
- **Admins**: Events go live immediately (`is_active: true`)
- **Organizers**: Events require admin approval (`is_active: false` → admin approval → `is_active: true`)

### **Q: How do I manage users and assign roles?**
**A:** **Admin Dashboard** provides complete user management:

1. **View All Users**: Admin Dashboard → Click "Active Users" card
2. **Assign Organizers to Events**: 
   - Click "Total Events" card → Find event → Click purple UserPlus icon
   - Or use the "Edit" button → "Assign Organizer" section
3. **Invite New Users**: Admin Dashboard → "Invite New Users" section
   - **Organizers**: Can create events, manage tickets, assign scanners
   - **Scanners**: Can scan/validate tickets at events
4. **Role Management**: All users show their role (admin/organizer/scanner)

### **Q: How do I change ticket prices and quantities after creating an event?**
**A:** **Organizers** can update their event tickets:

1. **Organizer Dashboard** → "Manage Events" → Find your event
2. **Click Settings icon** next to the event name
3. **Edit each ticket type**:
   - **Price**: Update ticket price in KES
   - **Quantity**: Adjust available tickets
   - **Restriction**: Cannot reduce quantity below already sold tickets
4. **Save changes** - updates apply immediately

**Admin Access**: Admins can edit any event through Admin Dashboard → Edit Event

### Installation

1. **Install dependencies:**
   ```bash
   npm install
   