# Build Uliza Tickets: Complete African Event Ticketing Platform

## 🚀 Project Overview
Create a production-ready event ticketing platform from scratch called "Uliza Tickets" for African events. This system will handle everything from event discovery to ticket validation with professional-grade features.

### Core Features to Build:
- **Festival-style landing page** with event discovery
- **Guest checkout system** (no registration required)  
- **Multi-payment integration** (Paystack for African markets)
- **Digital tickets with QR codes** sent via email
- **Professional scanning system** (CodeREADr integration)
- **Multi-role dashboard** (admin, organizer, scanner interfaces)
- **Box office system** for physical ticket sales
- **Waiting room management** for high-traffic events
- **Real-time analytics** and revenue tracking
- **Production-ready deployment** with robust error handling

## 📋 Step 1: Project Initialization

### Create New Replit Project
1. **Create Node.js project** on Replit
2. **Set up folder structure**:
```
/
├── server/           # Backend API
├── client/src/       # Frontend React app  
├── shared/           # Shared types and schemas
├── types/            # TypeScript declarations
└── attached_assets/  # Static images and assets
```

### Install Dependencies
```bash
# Backend dependencies
npm install express cors helmet compression
npm install @types/express @types/cors @types/node
npm install drizzle-orm @neondatabase/serverless
npm install drizzle-kit drizzle-zod zod
npm install bcryptjs jsonwebtoken @types/bcryptjs @types/jsonwebtoken
npm install nodemailer @sendgrid/mail @getbrevo/brevo
npm install pdfkit qrcode @types/pdfkit @types/qrcode
npm install express-session connect-pg-simple memorystore
npm install tsx typescript

# Frontend dependencies  
npm install react react-dom @types/react @types/react-dom
npm install @vitejs/plugin-react vite
npm install tailwindcss @tailwindcss/typography postcss autoprefixer
npm install @radix-ui/react-* (all UI components)
npm install lucide-react react-icons
npm install framer-motion
npm install wouter @tanstack/react-query
npm install react-hook-form @hookform/resolvers
npm install class-variance-authority clsx tailwind-merge
npm install cmdk vaul input-otp

# Payment & External APIs
npm install paystack-api flutterwave-node-v3
npm install ws @types/ws
```

## 📋 Step 2: Database Setup

### Create Neon PostgreSQL Database
1. **Provision Neon database** in Replit
2. **Get DATABASE_URL** from Replit Database tab
3. **Create schema file** `shared/schema.ts`:

```typescript
import { pgTable, uuid, varchar, text, timestamp, boolean, integer, index } from 'drizzle-orm/pg-core';
import { createInsertSchema } from 'drizzle-zod';
import { sql } from 'drizzle-orm';
import { z } from 'zod';

// Core Events Table
export const events = pgTable("events", {
  id: uuid("id").primaryKey().default(sql`gen_random_uuid()`),
  slug: varchar("slug", { length: 255 }).notNull().unique(),
  name: text("name").notNull(),
  description: text("description").notNull(),
  venue: text("venue").notNull(),
  date_from: timestamp("date_from").notNull(),
  date_to: timestamp("date_to"),
  image_url: text("image_url").notNull(),
  price_kes: integer("price_kes").notNull(),
  is_active: boolean("is_active").default(true),
  codereadr_database_id: varchar("codereadr_database_id", { length: 50 }),
  created_at: timestamp("created_at").default(sql`now()`),
}, (table) => ({
  slugIdx: index("events_slug_idx").on(table.slug),
  activeIdx: index("events_active_idx").on(table.is_active),
  dateIdx: index("events_date_idx").on(table.date_from),
}));

// Tickets Table with Guest Checkout Linking
export const tickets = pgTable("tickets", {
  id: uuid("id").primaryKey().default(sql`gen_random_uuid()`),
  uid: varchar("uid", { length: 255 }).notNull().unique(),
  event_id: uuid("event_id").references(() => events.id).notNull(),
  buyer_name: text("buyer_name").notNull(),
  buyer_email: varchar("buyer_email", { length: 255 }).notNull(),
  buyer_phone: varchar("buyer_phone", { length: 20 }),
  quantity: integer("quantity").notNull().default(1),
  price_kes: integer("price_kes").notNull(),
  total_paid_kes: integer("total_paid_kes").notNull(),
  payment_method: varchar("payment_method", { length: 20 }).notNull(),
  paid: boolean("paid").default(false),
  status: varchar("status", { length: 20 }).notNull().default("unused"),
  created_at: timestamp("created_at").default(sql`now()`),
  checked_in_at: timestamp("checked_in_at"),
  user_id: uuid("user_id").references(() => users.id), // Link to user account
  is_guest_purchase: boolean("is_guest_purchase").default(false), // Track guest checkouts
  linked_at: timestamp("linked_at"), // When guest ticket was linked to account
}, (table) => ({
  eventIdIdx: index("tickets_event_id_idx").on(table.event_id),
  buyerEmailIdx: index("tickets_buyer_email_idx").on(table.buyer_email),
  paidStatusIdx: index("tickets_paid_status_idx").on(table.paid, table.status),
  uidIdx: index("tickets_uid_idx").on(table.uid),
  userIdIdx: index("tickets_user_id_idx").on(table.user_id),
}));

// Scans Table (Audit Trail)
export const scans = pgTable("scans", {
  id: uuid("id").primaryKey().default(sql`gen_random_uuid()`),
  ticket_uid: varchar("ticket_uid", { length: 255 }).notNull(),
  event_id: uuid("event_id").references(() => events.id),
  device_label: text("device_label"),
  result: varchar("result", { length: 20 }).notNull(),
  created_at: timestamp("created_at").default(sql`now()`),
}, (table) => ({
  ticketUidIdx: index("scans_ticket_uid_idx").on(table.ticket_uid),
  eventIdResultIdx: index("scans_event_id_result_idx").on(table.event_id, table.result),
}));

// Payment Requests for Organizers
export const payment_requests = pgTable("payment_requests", {
  id: uuid("id").primaryKey().default(sql`gen_random_uuid()`),
  organizer_id: uuid("organizer_id").references(() => users.id).notNull(),
  event_id: uuid("event_id").references(() => events.id).notNull(),
  amount_kes: integer("amount_kes").notNull(),
  tickets_sold: integer("tickets_sold").notNull(),
  platform_fee_kes: integer("platform_fee_kes").notNull(),
  net_amount_kes: integer("net_amount_kes").notNull(),
  status: varchar("status", { length: 20 }).default("pending"), // pending, approved, paid, rejected
  requested_at: timestamp("requested_at").default(sql`now()`),
  processed_at: timestamp("processed_at"),
  notes: text("notes"),
  bank_details: text("bank_details"), // JSON string with bank info
  processed_by: uuid("processed_by").references(() => users.id),
}, (table) => ({
  organizerIdx: index("payment_requests_organizer_idx").on(table.organizer_id),
  statusIdx: index("payment_requests_status_idx").on(table.status),
  eventIdx: index("payment_requests_event_idx").on(table.event_id),
}));

// Guest Checkout Email Tracking for Account Linking
export const guest_checkouts = pgTable("guest_checkouts", {
  id: uuid("id").primaryKey().default(sql`gen_random_uuid()`),
  email: varchar("email", { length: 255 }).notNull(),
  ticket_ids: varchar("ticket_ids", { length: 1000 }).notNull(), // JSON array of ticket IDs
  created_at: timestamp("created_at").default(sql`now()`),
  linked_user_id: uuid("linked_user_id").references(() => users.id),
  linked_at: timestamp("linked_at"),
}, (table) => ({
  emailIdx: index("guest_checkouts_email_idx").on(table.email),
  linkedUserIdx: index("guest_checkouts_linked_user_idx").on(table.linked_user_id),
}));

// Multi-Role Users Table
export const users = pgTable("users", {
  id: uuid("id").primaryKey().default(sql`gen_random_uuid()`),
  email: varchar("email", { length: 255 }).notNull().unique(),
  password_hash: text("password_hash").notNull(),
  first_name: text("first_name").notNull(),
  last_name: text("last_name").notNull(),
  phone: text("phone"),
  role: varchar("role", { length: 20 }).notNull().default("user"), // user, organizer, scanner, admin
  is_active: boolean("is_active").default(true),
  email_verified: boolean("email_verified").default(false),
  created_at: timestamp("created_at").default(sql`now()`),
  updated_at: timestamp("updated_at").default(sql`now()`),
});

// Enhanced Events with Organizer Management
export const enhanced_events = pgTable("enhanced_events", {
  id: uuid("id").primaryKey().default(sql`gen_random_uuid()`),
  slug: varchar("slug", { length: 255 }).notNull().unique(),
  name: text("name").notNull(),
  description: text("description").notNull(),
  venue: text("venue").notNull(),
  date_from: timestamp("date_from").notNull(),
  date_to: timestamp("date_to"),
  start_time: varchar("start_time", { length: 10 }),
  image_url: text("image_url").notNull(),
  organizer_id: uuid("organizer_id").references(() => users.id),
  is_active: boolean("is_active").default(false),
  codereadr_database_id: varchar("codereadr_database_id", { length: 50 }),
  created_at: timestamp("created_at").default(sql`now()`),
  updated_at: timestamp("updated_at").default(sql`now()`),
});

// Ticket Types for Multiple Pricing
export const ticket_types = pgTable("ticket_types", {
  id: uuid("id").primaryKey().default(sql`gen_random_uuid()`),
  event_id: uuid("event_id").references(() => enhanced_events.id).notNull(),
  name: varchar("name", { length: 100 }).notNull(),
  price_kes: integer("price_kes").notNull(),
  available_quantity: integer("available_quantity").notNull(),
  sold_quantity: integer("sold_quantity").notNull().default(0),
  is_active: boolean("is_active").default(true),
  created_at: timestamp("created_at").default(sql`now()`),
});

// Create insert schemas and types
export const insertEventSchema = createInsertSchema(events).omit({
  id: true,
  created_at: true,
});

export const insertTicketSchema = createInsertSchema(tickets).omit({
  id: true,
  uid: true, 
  created_at: true,
  checked_in_at: true,
});

export const insertUserSchema = createInsertSchema(users).omit({
  id: true,
  created_at: true,
  updated_at: true,
});

// Export types
export type Event = typeof events.$inferSelect;
export type InsertEvent = z.infer<typeof insertEventSchema>;
export type Ticket = typeof tickets.$inferSelect;
export type InsertTicket = z.infer<typeof insertTicketSchema>;
export type User = typeof users.$inferSelect;
export type InsertUser = z.infer<typeof insertUserSchema>;
```

### Configure Database Connection
Create `server/db.ts`:
```typescript
import { Pool, neonConfig } from '@neondatabase/serverless';
import { drizzle } from 'drizzle-orm/neon-serverless';
import ws from "ws";
import * as schema from "@shared/schema";

neonConfig.webSocketConstructor = ws;

if (!process.env.DATABASE_URL) {
  throw new Error("DATABASE_URL must be set. Did you forget to provision a database?");
}

export const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle({ client: pool, schema });
```

## 📋 Step 3: Production-Ready Server Setup

### Environment Validation System
Create `server/env-validation.ts`:
```typescript
import { log } from "./vite";

interface RequiredEnvVar {
  name: string;
  description: string;
  required: boolean;
}

const REQUIRED_ENV_VARS: RequiredEnvVar[] = [
  { name: 'DATABASE_URL', description: 'PostgreSQL database connection string', required: true },
  { name: 'JWT_SECRET', description: 'Secret key for JWT token signing', required: true },
  { name: 'BREVO_SMTP_PASSWORD', description: 'SMTP password for Brevo email service', required: true },
  { name: 'CODEREADR_API_KEY', description: 'API key for CodeREADr ticket validation service', required: true },
  { name: 'PAYSTACK_SECRET_KEY', description: 'Secret key for Paystack payment processing', required: true },
  { name: 'PORT', description: 'Server port (defaults to 5000)', required: false },
  { name: 'NODE_ENV', description: 'Application environment', required: false }
];

export class EnvironmentValidationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'EnvironmentValidationError';
  }
}

export async function validateEnvironmentVariables(): Promise<void> {
  const missingRequired: string[] = [];
  const warnings: string[] = [];

  log('🔍 Validating environment variables...');

  for (const envVar of REQUIRED_ENV_VARS) {
    const value = process.env[envVar.name];
    
    if (!value) {
      if (envVar.required) {
        missingRequired.push(`${envVar.name}: ${envVar.description}`);
      } else {
        warnings.push(`${envVar.name} not set: ${envVar.description}`);
      }
    } else {
      const displayValue = envVar.name.toLowerCase().includes('secret') || 
                          envVar.name.toLowerCase().includes('password') || 
                          envVar.name.toLowerCase().includes('key')
        ? '***' 
        : value;
      log(`✅ ${envVar.name}: ${displayValue}`);
    }
  }

  if (warnings.length > 0) {
    log('⚠️ Environment warnings:');
    warnings.forEach(warning => log(`   ${warning}`));
  }

  if (missingRequired.length > 0) {
    const errorMessage = `Missing required environment variables:\n${missingRequired.map(v => `  - ${v}`).join('\n')}`;
    throw new EnvironmentValidationError(errorMessage);
  }

  if (!process.env.NODE_ENV) {
    process.env.NODE_ENV = 'production';
    log('🔧 NODE_ENV not set, defaulting to production');
  }

  log('✅ Environment validation completed successfully');
}

export async function validateDatabaseConnection(db: any): Promise<void> {
  const maxRetries = 3;
  const retryDelay = 2000;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      log(`🔌 Attempting database connection (attempt ${attempt}/${maxRetries})...`);
      await db.execute('SELECT 1 as test');
      log('✅ Database connection validated successfully');
      return;
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error';
      log(`❌ Database connection attempt ${attempt} failed: ${errorMessage}`);
      
      if (attempt === maxRetries) {
        throw new EnvironmentValidationError(
          `Failed to connect to database after ${maxRetries} attempts. ` +
          'Please check your DATABASE_URL and ensure the database is accessible.'
        );
      }
      
      await new Promise(resolve => setTimeout(resolve, retryDelay));
    }
  }
}
```

### Robust Server Initialization
Create `server/index.ts`:
```typescript
import express, { type Request, Response, NextFunction } from "express";
import { registerRoutes } from "./routes";
import { setupVite, serveStatic, log } from "./vite";
import { validateEnvironmentVariables, validateDatabaseConnection, EnvironmentValidationError } from "./env-validation";
import { db } from "./db";

const app = express();
app.use(express.json());
app.use(express.urlencoded({ extended: false }));

// Request logging middleware
app.use((req, res, next) => {
  const start = Date.now();
  const path = req.path;
  let capturedJsonResponse: Record<string, any> | undefined = undefined;

  const originalResJson = res.json;
  res.json = function (bodyJson, ...args) {
    capturedJsonResponse = bodyJson;
    return originalResJson.apply(res, [bodyJson, ...args]);
  };

  res.on("finish", () => {
    const duration = Date.now() - start;
    if (path.startsWith("/api")) {
      let logLine = `${req.method} ${path} ${res.statusCode} in ${duration}ms`;
      if (capturedJsonResponse) {
        logLine += ` :: ${JSON.stringify(capturedJsonResponse)}`;
      }
      if (logLine.length > 80) {
        logLine = logLine.slice(0, 79) + "…";
      }
      log(logLine);
    }
  });

  next();
});

async function startServer() {
  try {
    log('🚀 Starting Uliza Tickets server...');
    
    // Step 1: Validate environment variables
    await validateEnvironmentVariables();
    
    // Step 2: Validate database connection  
    await validateDatabaseConnection(db);
    
    // Step 3: Register API routes
    log('📁 Registering API routes...');
    const server = await registerRoutes(app);
    log('✅ API routes registered successfully');

    // Step 4: Setup error handling
    app.use((err: any, _req: Request, res: Response, _next: NextFunction) => {
      const status = err.status || err.statusCode || 500;
      const message = err.message || "Internal Server Error";
      
      log(`❌ Server error: ${status} - ${message}`);
      if (err.stack && process.env.NODE_ENV === 'development') {
        console.error(err.stack);
      }

      res.status(status).json({ message });
    });

    // Step 5: Setup development/production environment
    const isProduction = process.env.NODE_ENV === 'production';
    
    if (!isProduction) {
      log('🔧 Setting up development environment with Vite...');
      await setupVite(app, server);
      log('✅ Vite development server configured');
    } else {
      log('📦 Setting up production static file serving...');
      serveStatic(app);
      log('✅ Static file serving configured');
    }

    // Step 6: Start HTTP server
    const port = parseInt(process.env.PORT || '5000', 10);
    
    server.listen({
      port,
      host: "0.0.0.0",
      reusePort: true,
    }, () => {
      log('🎉 Server startup completed successfully!');
      log(`🌐 Server running on port ${port}`);
      log(`🏷️  Environment: ${process.env.NODE_ENV}`);
    });

  } catch (error) {
    log('💥 Server startup failed!');
    
    if (error instanceof EnvironmentValidationError) {
      log('❌ Environment validation error:');
      log(error.message);
    } else {
      log('❌ Unexpected startup error:');
      log(error instanceof Error ? error.message : String(error));
      if (error instanceof Error && error.stack) {
        console.error(error.stack);
      }
    }
    
    log('🔧 Suggested fixes:');
    log('   1. Check that all required environment variables are set');
    log('   2. Ensure DATABASE_URL points to an accessible database');
    log('   3. Verify network connectivity to external services');
    
    process.exit(1);
  }
}

// Handle uncaught exceptions and unhandled rejections
process.on('uncaughtException', (error) => {
  log('💥 Uncaught Exception:');
  console.error(error);
  process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  log('💥 Unhandled Rejection at Promise');
  log('Reason: ' + String(reason));
  process.exit(1);
});

// Graceful shutdown
process.on('SIGTERM', () => {
  log('🛑 SIGTERM received, shutting down gracefully...');
  process.exit(0);
});

process.on('SIGINT', () => {
  log('🛑 SIGINT received, shutting down gracefully...');
  process.exit(0);
});

startServer();
```

## 📋 Step 4: External API Integrations

### CodeREADr Integration for Ticket Scanning
Add to `server/routes.ts`:
```typescript
// CodeREADr API for professional ticket scanning
const CODEREADR_API_BASE = 'https://api.codereadr.com/api/';

async function makeCodeREADrRequest(section: string, action: string, params: Record<string, any> = {}) {
  const formData = new FormData();
  formData.append('api_key', process.env.CODEREADR_API_KEY!);
  formData.append('section', section);
  formData.append('action', action);
  
  Object.entries(params).forEach(([key, value]) => {
    formData.append(key, String(value));
  });
  
  try {
    const response = await fetch(CODEREADR_API_BASE, {
      method: 'POST',
      body: formData
    });
    
    const responseText = await response.text();
    
    try {
      return JSON.parse(responseText);
    } catch {
      if (responseText.includes('<status>1</status>')) {
        return { status: 1, data: responseText };
      } else {
        throw new Error(`CodeREADr API error: ${responseText}`);
      }
    }
  } catch (error) {
    console.error('CodeREADr API request failed:', error);
    throw error;
  }
}

async function createCodeREADrDatabase(eventName: string, eventId: string) {
  const databaseName = `${eventName.replace(/[^a-zA-Z0-9]/g, '_')}_${eventId.slice(-8)}`;
  
  const response = await makeCodeREADrRequest('databases', 'create', {
    database_name: databaseName,
  });
  
  console.log(`✅ Created CodeREADr database: ${databaseName}`, response);
  return response;
}

async function uploadTicketToCodeREADr(databaseId: string, ticketUID: string, eventName: string, buyerName: string) {
  const response = await makeCodeREADrRequest('databases', 'upsertvalue', {
    database_id: databaseId,
    value: ticketUID,
    response: `Valid ticket for ${eventName} - ${buyerName}`,
    validity: 1,
    trim_value: 1
  });
  
  console.log(`✅ Uploaded ticket ${ticketUID} to CodeREADr`);
  return response;
}
```

### Paystack Payment Integration
```typescript
// TypeScript declaration for Paystack
// Create types/paystack-api.d.ts:
declare module "paystack-api" {
  interface PaystackTransaction {
    initialize(data: any): Promise<any>;
    verify(reference: string): Promise<any>;
  }

  interface PaystackAPI {
    transaction: PaystackTransaction;
  }

  function PaystackLib(secretKey: string): PaystackAPI;
  export = PaystackLib;
}

// In routes.ts:
import PaystackLib from "paystack-api";

// Initialize Paystack
const paystack = PaystackLib(process.env.PAYSTACK_SECRET_KEY!);

// Payment initialization endpoint
app.post("/api/purchase", async (req, res) => {
  try {
    const { event_id, buyer_name, buyer_email, buyer_phone, quantity } = req.body;
    
    const event = await storage.getEventById(event_id);
    if (!event) {
      return res.status(404).json({ error: "Event not found" });
    }

    const totalAmount = event.price_kes * quantity;
    
    const paystackData = {
      email: buyer_email,
      amount: totalAmount * 100, // Paystack expects kobo
      currency: 'KES',
      reference: `TK-${randomUUID().substring(0, 8).toUpperCase()}`,
      callback_url: `${req.protocol}://${req.get('host')}/api/payment/callback`,
      metadata: {
        event_id,
        buyer_name,
        buyer_phone,
        quantity: quantity.toString(),
        event_name: event.name,
        venue: event.venue,
        date: event.date_from.toDateString()
      }
    };

    const response = await paystack.transaction.initialize(paystackData);
    
    if (response.status) {
      res.json({
        payment_url: response.data.authorization_url,
        reference: response.data.reference
      });
    } else {
      res.status(400).json({ error: 'Payment initialization failed' });
    }
  } catch (error) {
    console.error('Payment initialization error:', error);
    res.status(500).json({ error: 'Payment initialization failed' });
  }
});

// Payment verification webhook
app.get("/api/payment/callback", async (req, res) => {
  try {
    const { reference } = req.query;
    
    const response = await fetch(`https://api.paystack.co/transaction/verify/${reference}`, {
      headers: {
        Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
      },
    });
    
    const verification = await response.json();
    
    if (verification.status && verification.data.status === 'success') {
      const metadata = verification.data.metadata;
      
      // Create ticket after successful payment
      const ticket = await storage.createTicket({
        event_id: metadata.event_id,
        buyer_name: metadata.buyer_name,
        buyer_email: verification.data.customer.email,
        buyer_phone: metadata.buyer_phone,
        quantity: parseInt(metadata.quantity),
        paid: true
      }, reference);

      // Sync to CodeREADr
      try {
        const event = await storage.getEventById(metadata.event_id);
        if (event) {
          let databaseResponse = await createCodeREADrDatabase(event.name, metadata.event_id);
          await uploadTicketToCodeREADr(
            databaseResponse.id,
            ticket.uid,
            event.name,
            metadata.buyer_name
          );
        }
      } catch (error) {
        console.error('CodeREADr sync failed (non-critical):', error);
      }

      // Send ticket email
      await sendTicketEmail(
        verification.data.customer.email,
        metadata.buyer_name,
        reference,
        metadata.event_name,
        metadata.venue,
        metadata.date
      );

      res.redirect(`/success?uid=${ticket.uid}`);
    } else {
      res.redirect('/error?message=Payment failed');
    }
  } catch (error) {
    console.error('Payment verification error:', error);
    res.redirect('/error?message=Payment verification failed');
  }
});
```

### Email System with Brevo
```typescript
import nodemailer from 'nodemailer';

// Initialize SMTP transporter
const transporter = nodemailer.createTransporter({
  host: 'smtp-relay.brevo.com',
  port: 587,
  secure: false,
  auth: {
    user: '9678c7001@smtp-brevo.com',
    pass: process.env.BREVO_SMTP_PASSWORD
  }
});

async function sendTicketEmail(email: string, buyerName: string, ticketUID: string, eventName: string, venue: string, date: string) {
  // Generate PDF ticket with QR code
  const pdfBuffer = await generateTicketPDF(ticketUID, eventName, buyerName, venue, date);
  
  const mailOptions = {
    from: '"Uliza Tickets" <tickets@uliza.co.ke>',
    to: email,
    subject: `🎫 Your ${eventName} Tickets - ${ticketUID}`,
    html: `
      <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
        <h1 style="color: #e11d48;">🎉 Your tickets are ready!</h1>
        <p>Hi ${buyerName},</p>
        <p>Thank you for purchasing tickets to <strong>${eventName}</strong>!</p>
        
        <div style="background: linear-gradient(135deg, #e11d48, #f59e0b); padding: 20px; border-radius: 10px; color: white; margin: 20px 0;">
          <h2 style="margin: 0 0 10px 0;">📅 Event Details</h2>
          <p style="margin: 5px 0;"><strong>Event:</strong> ${eventName}</p>
          <p style="margin: 5px 0;"><strong>Venue:</strong> ${venue}</p>
          <p style="margin: 5px 0;"><strong>Date:</strong> ${date}</p>
          <p style="margin: 5px 0;"><strong>Ticket ID:</strong> ${ticketUID}</p>
        </div>
        
        <p><strong>🎯 Entry Instructions:</strong></p>
        <ul>
          <li>Show the PDF on your phone or print it out</li>
          <li>Arrive early - gates open 1 hour before</li>
          <li>Bring valid photo ID</li>
          <li>Check event guidelines for prohibited items</li>
        </ul>
        
        <p style="color: #e11d48; font-weight: bold;">🌟 GET READY TO EXPERIENCE THE MAGIC! 🌟</p>
        <p><em>Powered by Uliza Tickets - Africa's Premier Event Platform</em></p>
      </div>
    `,
    attachments: [
      {
        filename: `uliza-ticket-${ticketUID}.pdf`,
        content: pdfBuffer,
        contentType: 'application/pdf'
      }
    ]
  };

  const result = await transporter.sendMail(mailOptions);
  console.log('✅ Ticket email sent successfully!');
  return result.messageId;
}
```

## 📋 Step 5: Advanced Features

### High-Traffic Waiting Room System
```typescript
// Waiting room for high-demand events
const waitingRooms = new Map<string, {
  users: Array<{
    sessionId: string;
    joinedAt: Date;
    position: number;
    status: 'waiting' | 'active' | 'expired';
  }>;
  maxActiveUsers: number;
  currentActiveUsers: number;
}>();

app.post('/api/events/:eventId/waiting-room/join', async (req, res) => {
  try {
    const { eventId } = req.params;
    const sessionId = `session_${randomUUID()}`;
    
    let room = waitingRooms.get(eventId);
    if (!room) {
      room = {
        users: [],
        maxActiveUsers: 50, // Allow 50 concurrent purchasers
        currentActiveUsers: 0
      };
      waitingRooms.set(eventId, room);
    }

    // Check if user can proceed immediately
    if (room.currentActiveUsers < room.maxActiveUsers) {
      room.currentActiveUsers++;
      return res.json({
        status: 'proceed',
        message: 'You can proceed to purchase tickets',
        sessionId
      });
    }

    // Add to waiting queue
    const position = room.users.length + 1;
    room.users.push({
      sessionId,
      joinedAt: new Date(),
      position,
      status: 'waiting'
    });

    res.json({
      status: 'waiting',
      message: `You're in the waiting room. Position: ${position}`,
      sessionId,
      position,
      estimatedWaitTime: Math.ceil(position * 0.5), // ~30 seconds per person
      queueLength: room.users.length
    });

  } catch (error) {
    const err = error as Error;
    console.error('Waiting room error:', err.message);
    res.status(500).json({ error: 'Failed to join waiting room' });
  }
});
```

### Box Office System for Physical Sales
```typescript
// Box office ticket sales for events
app.post("/api/box-office/sell", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const { eventId, ticketTypeId, quantity, customerName, customerEmail, customerPhone, paymentMethod } = req.body;
    const userRole = req.user?.role;

    // Only admin, organizer, and scanner can sell tickets
    if (!['admin', 'organizer', 'scanner'].includes(userRole)) {
      return res.status(403).json({ error: "Access denied" });
    }

    // Get ticket type and check availability
    const [ticketType] = await db
      .select()
      .from(ticket_types)
      .where(eq(ticket_types.id, ticketTypeId));

    if (!ticketType || !ticketType.is_active) {
      return res.status(404).json({ error: "Ticket type not found" });
    }

    const availableQuantity = ticketType.available_quantity - ticketType.sold_quantity;
    if (quantity > availableQuantity) {
      return res.status(400).json({ error: `Only ${availableQuantity} tickets available` });
    }

    // Create tickets
    const ticketPromises = [];
    for (let i = 0; i < quantity; i++) {
      const ticketUID = `TK-${randomUUID().substring(0, 8).toUpperCase()}`;
      
      const ticketPromise = db
        .insert(tickets)
        .values({
          uid: ticketUID,
          event_id: eventId,
          buyer_name: customerName,
          buyer_email: customerEmail,
          buyer_phone: customerPhone,
          quantity: 1,
          payment_method: paymentMethod,
          price_kes: ticketType.price_kes,
          total_paid_kes: ticketType.price_kes,
          paid: true,
          created_at: new Date(),
        })
        .returning();
      
      ticketPromises.push(ticketPromise);
    }

    const ticketResults = await Promise.all(ticketPromises);
    const createdTickets = ticketResults.map(result => result[0]);

    // Update sold quantity
    await db
      .update(ticket_types)
      .set({ 
        sold_quantity: ticketType.sold_quantity + quantity 
      })
      .where(eq(ticket_types.id, ticketTypeId));

    // Sync to CodeREADr for instant validation
    try {
      const event = await db.select().from(events).where(eq(events.id, eventId));
      if (event[0]) {
        for (const ticket of createdTickets) {
          await uploadTicketToCodeREADr(
            event[0].codereadr_database_id!,
            ticket.uid,
            event[0].name,
            ticket.buyer_name
          );
        }
      }
    } catch (error) {
      console.error('CodeREADr sync failed (non-critical):', error);
    }

    res.json({
      message: `Successfully sold ${quantity} tickets`,
      tickets: createdTickets.map(t => ({ uid: t.uid, buyer_name: t.buyer_name })),
      totalAmount: ticketType.price_kes * quantity
    });

  } catch (error) {
    console.error("Box office sale error:", error);
    res.status(500).json({ error: "Failed to process sale" });
  }
});
```

### Analytics and Reporting System
```typescript
// Real-time analytics dashboard
app.get("/api/admin/analytics", authenticateToken, async (req: AuthRequest, res) => {
  try {
    if (req.user?.role !== 'admin') {
      return res.status(403).json({ error: "Admin access required" });
    }

    // Total revenue
    const revenueResult = await db
      .select({ total: sql<number>`SUM(total_paid_kes)` })
      .from(tickets)
      .where(eq(tickets.paid, true));

    // Commission calculation (5%)
    const totalRevenue = revenueResult[0]?.total || 0;
    const totalCommission = Math.floor(totalRevenue * 0.05);

    // Ticket sales by status
    const salesByStatus = await db
      .select({
        status: tickets.status,
        count: sql<number>`COUNT(*)`
      })
      .from(tickets)
      .where(eq(tickets.paid, true))
      .groupBy(tickets.status);

    // Daily sales trend (last 7 days)
    const dailySales = await db
      .select({
        date: sql<string>`DATE(created_at)`,
        revenue: sql<number>`SUM(total_paid_kes)`,
        tickets: sql<number>`COUNT(*)`
      })
      .from(tickets)
      .where(and(
        eq(tickets.paid, true),
        gte(tickets.created_at, new Date(Date.now() - 7 * 24 * 60 * 60 * 1000))
      ))
      .groupBy(sql`DATE(created_at)`)
      .orderBy(sql`DATE(created_at)`);

    // Top performing events
    const topEvents = await db
      .select({
        eventId: tickets.event_id,
        eventName: events.name,
        revenue: sql<number>`SUM(tickets.total_paid_kes)`,
        ticketsSold: sql<number>`COUNT(tickets.id)`
      })
      .from(tickets)
      .innerJoin(events, eq(tickets.event_id, events.id))
      .where(eq(tickets.paid, true))
      .groupBy(tickets.event_id, events.name)
      .orderBy(sql`SUM(tickets.total_paid_kes) DESC`)
      .limit(10);

    res.json({
      totalRevenue,
      totalCommission,
      salesByStatus,
      dailySales,
      topEvents
    });

  } catch (error) {
    console.error("Analytics error:", error);
    res.status(500).json({ error: "Failed to fetch analytics" });
  }
});
```

## 📋 Step 6: Frontend Development

### Create React App with Vite
Set up `client/src/App.tsx`:
```typescript
import React from 'react';
import { Route, Switch } from 'wouter';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Toaster } from '@/components/ui/toaster';

// Pages
import { LandingPage } from '@/pages/LandingPage';
import { EventDetailsPage } from '@/pages/EventDetailsPage';
import { CheckoutPage } from '@/pages/CheckoutPage';
import { SuccessPage } from '@/pages/SuccessPage';
import { AdminDashboard } from '@/pages/AdminDashboard';
import { OrganizerDashboard } from '@/pages/OrganizerDashboard';
import { ScannerDashboard } from '@/pages/ScannerDashboard';

// Create query client
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      cacheTime: 10 * 60 * 1000, // 10 minutes
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <div className="min-h-screen bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500">
        <Switch>
          <Route path="/" component={LandingPage} />
          <Route path="/events/:slug" component={EventDetailsPage} />
          <Route path="/checkout/:eventId" component={CheckoutPage} />
          <Route path="/success" component={SuccessPage} />
          <Route path="/admin" component={AdminDashboard} />
          <Route path="/organizer" component={OrganizerDashboard} />
          <Route path="/scanner" component={ScannerDashboard} />
          <Route>
            <div className="flex items-center justify-center min-h-screen text-white">
              <h1 className="text-4xl font-bold">404 - Page Not Found</h1>
            </div>
          </Route>
        </Switch>
        <Toaster />
      </div>
    </QueryClientProvider>
  );
}

export default App;
```

### PWA Configuration for High-Performance Scanning
Create `public/manifest.json`:
```json
{
  "name": "Uliza Tickets - PWA Scanner",
  "short_name": "Uliza Scanner",
  "description": "High-performance ticket scanning for events across Africa",
  "start_url": "/scanner",
  "display": "standalone",
  "orientation": "portrait",
  "theme_color": "#ec4899",
  "background_color": "#ec4899",
  "icons": [
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/scanner-mobile.png",
      "sizes": "390x844",
      "type": "image/png",
      "form_factor": "narrow"
    },
    {
      "src": "/screenshots/scanner-desktop.png",
      "sizes": "1920x1080",
      "type": "image/png",
      "form_factor": "wide"
    }
  ],
  "categories": ["business", "productivity", "utilities"],
  "lang": "en",
  "scope": "/",
  "prefer_related_applications": false
}
```

Create `public/sw.js` (Service Worker for offline scanning):
```javascript
const CACHE_NAME = 'uliza-scanner-v1';
const urlsToCache = [
  '/scanner',
  '/offline.html',
  '/icons/icon-192x192.png',
  '/icons/icon-512x512.png'
];

// Install event - cache essential files
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => cache.addAll(urlsToCache))
  );
});

// Fetch event - serve from cache when offline
self.addEventListener('fetch', (event) => {
  // Only handle GET requests
  if (event.request.method !== 'GET') return;
  
  // Handle scanner page requests
  if (event.request.url.includes('/scanner')) {
    event.respondWith(
      caches.match(event.request)
        .then((response) => {
          // Return cached version or fetch from network
          return response || fetch(event.request);
        })
        .catch(() => {
          // Return offline page if network fails
          return caches.match('/offline.html');
        })
    );
  }
  
  // Handle API requests with background sync
  if (event.request.url.includes('/api/')) {
    event.respondWith(
      fetch(event.request)
        .catch(() => {
          // Store failed requests for background sync
          return new Response(
            JSON.stringify({ error: 'Offline mode', offline: true }),
            { 
              status: 200,
              headers: { 'Content-Type': 'application/json' }
            }
          );
        })
    );
  }
});

// Background sync for offline scans
self.addEventListener('sync', (event) => {
  if (event.tag === 'background-sync-scans') {
    event.waitUntil(syncOfflineScans());
  }
});

// Sync offline scans when online
async function syncOfflineScans() {
  try {
    const offlineScans = await getOfflineScans();
    if (offlineScans.length === 0) return;

    const response = await fetch('/api/scanner/sync-offline', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${localStorage.getItem('auth_token')}`,
      },
      body: JSON.stringify({ scans: offlineScans }),
    });

    if (response.ok) {
      await clearOfflineScans();
      // Notify UI of successful sync
      self.clients.matchAll().then(clients => {
        clients.forEach(client => {
          client.postMessage({
            type: 'SYNC_COMPLETE',
            count: offlineScans.length
          });
        });
      });
    }
  } catch (error) {
    console.error('Background sync failed:', error);
  }
}

// Helper functions for offline storage
async function getOfflineScans() {
  return JSON.parse(localStorage.getItem('offline_scans') || '[]');
}

async function clearOfflineScans() {
  localStorage.removeItem('offline_scans');
}
```

Create `public/offline.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Uliza Scanner - Offline</title>
  <style>
    body {
      font-family: system-ui, -apple-system, sans-serif;
      background: linear-gradient(135deg, #ec4899, #ef4444, #eab308);
      color: white;
      text-align: center;
      padding: 2rem;
      min-height: 100vh;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
    }
    .icon { font-size: 4rem; margin-bottom: 1rem; }
    h1 { font-size: 2rem; margin-bottom: 1rem; }
    p { font-size: 1.2rem; opacity: 0.9; margin-bottom: 2rem; }
    button {
      background: rgba(255,255,255,0.2);
      border: 1px solid rgba(255,255,255,0.3);
      color: white;
      padding: 1rem 2rem;
      border-radius: 0.5rem;
      font-size: 1rem;
      cursor: pointer;
      backdrop-filter: blur(10px);
    }
    button:hover { background: rgba(255,255,255,0.3); }
  </style>
</head>
<body>
  <div class="icon">📱</div>
  <h1>Scanner Offline Mode</h1>
  <p>You're offline, but you can still scan tickets!<br>Scans will sync automatically when you're back online.</p>
  <button onclick="window.location.reload()">Try Again</button>
  
  <script>
    // Auto-retry when back online
    window.addEventListener('online', () => {
      window.location.reload();
    });
  </script>
</body>
</html>
```

### CodeREADr Professional Integration Setup

#### Step 1: Create CodeREADr Account and Database
1. **Sign up at CodeREADr.com**:
   - Visit https://www.codereadr.com
   - Create professional account ($10-50/month based on scanning volume)
   - Note your API key from Account Settings

2. **Create Event-Specific Databases**:
   ```bash
   # For each event, create a dedicated database via CodeREADr API
   curl -X POST "https://api.codereadr.com/api/databases/create" \
     -H "Content-Type: application/json" \
     -d '{
       "api_key": "YOUR_CODEREADR_API_KEY",
       "name": "Event-{EVENT_ID}-Database",
       "description": "Tickets for {EVENT_NAME}",
       "type": "Scan Validation"
     }'
   ```

3. **Database Structure for Each Event**:
   ```
   Database Name: Event-{EVENT_ID}-Database
   Fields:
   - ticket_uid (Primary Key)
   - buyer_name
   - buyer_email
   - ticket_type (Regular/VIP/VVIP)
   - price_paid
   - event_name
   - purchase_date
   - scan_count (tracks multiple scans)
   - first_scan_time
   - last_scan_time
   - scanner_device
   ```

#### Step 2: Backend CodeREADr Integration
Add to `server/routes.ts`:
```typescript
// CodeREADr API Configuration
const CODEREADR_BASE_URL = 'https://api.codereadr.com/api';
const CODEREADR_API_KEY = process.env.CODEREADR_API_KEY;

// Create CodeREADr database for new event
app.post("/api/codereadr/create-database", authenticateToken, async (req: AuthRequest, res) => {
  try {
    if (!CODEREADR_API_KEY) {
      return res.status(500).json({ error: "CodeREADr API key not configured" });
    }

    const { eventId, eventName } = req.body;
    
    // Create database in CodeREADr
    const response = await fetch(`${CODEREADR_BASE_URL}/databases/create`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        api_key: CODEREADR_API_KEY,
        name: `Event-${eventId}-Database`,
        description: `Tickets for ${eventName}`,
        type: 'Scan Validation'
      })
    });

    const result = await response.json();
    
    if (result.status === 'success') {
      // Update event with CodeREADr database ID
      await db
        .update(events)
        .set({ codereadr_database_id: result.database_id })
        .where(eq(events.id, eventId));

      res.json({ 
        message: "CodeREADr database created successfully",
        database_id: result.database_id 
      });
    } else {
      res.status(400).json({ error: result.message });
    }
  } catch (error) {
    console.error("CodeREADr database creation error:", error);
    res.status(500).json({ error: "Failed to create CodeREADr database" });
  }
});

// Sync tickets to CodeREADr database
app.post("/api/codereadr/sync-tickets", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const { eventId } = req.body;

    // Get event with CodeREADr database ID
    const [event] = await db
      .select({
        id: events.id,
        name: events.name,
        codereadr_database_id: events.codereadr_database_id
      })
      .from(events)
      .where(eq(events.id, eventId));

    if (!event?.codereadr_database_id) {
      return res.status(400).json({ error: "Event not configured for CodeREADr" });
    }

    // Get all tickets for this event
    const tickets = await db
      .select({
        uid: tickets.uid,
        buyer_name: tickets.buyer_name,
        buyer_email: tickets.buyer_email,
        price_kes: tickets.price_kes,
        created_at: tickets.created_at,
      })
      .from(tickets)
      .where(eq(tickets.event_id, eventId));

    // Sync tickets to CodeREADr in batches
    const batchSize = 100;
    let syncedCount = 0;

    for (let i = 0; i < tickets.length; i += batchSize) {
      const batch = tickets.slice(i, i + batchSize);
      
      const batchData = batch.map(ticket => ({
        value: ticket.uid,
        response: JSON.stringify({
          buyer_name: ticket.buyer_name,
          buyer_email: ticket.buyer_email,
          price_paid: ticket.price_kes,
          event_name: event.name,
          purchase_date: ticket.created_at
        })
      }));

      const response = await fetch(`${CODEREADR_BASE_URL}/databases/${event.codereadr_database_id}/upload`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          api_key: CODEREADR_API_KEY,
          values: batchData
        })
      });

      const result = await response.json();
      if (result.status === 'success') {
        syncedCount += batch.length;
      }
    }

    res.json({ 
      message: `Successfully synced ${syncedCount} tickets to CodeREADr`,
      synced_count: syncedCount,
      total_tickets: tickets.length
    });
  } catch (error) {
    console.error("CodeREADr sync error:", error);
    res.status(500).json({ error: "Failed to sync tickets to CodeREADr" });
  }
});

// Record scan in CodeREADr
app.post("/api/codereadr/record-scan", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const { ticket_uid, event_id, scanner_device, scan_timestamp } = req.body;

    const [event] = await db
      .select({ codereadr_database_id: events.codereadr_database_id })
      .from(events)
      .where(eq(events.id, event_id));

    if (!event?.codereadr_database_id) {
      return res.json({ message: "Event not configured for CodeREADr" });
    }

    // Record scan in CodeREADr
    const response = await fetch(`${CODEREADR_BASE_URL}/scans/create`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        api_key: CODEREADR_API_KEY,
        database_id: event.codereadr_database_id,
        value: ticket_uid,
        timestamp: scan_timestamp,
        device: scanner_device,
        location: 'Event Entrance'
      })
    });

    const result = await response.json();
    res.json(result);
  } catch (error) {
    console.error("CodeREADr scan recording error:", error);
    res.status(500).json({ error: "Failed to record scan in CodeREADr" });
  }
});

// Test CodeREADr API connection
app.post("/api/codereadr/test", authenticateToken, async (req: AuthRequest, res) => {
  try {
    if (!CODEREADR_API_KEY) {
      return res.status(500).json({ error: "CodeREADr API key not configured" });
    }

    const response = await fetch(`${CODEREADR_BASE_URL}/databases`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        api_key: CODEREADR_API_KEY
      })
    });

    const result = await response.json();
    
    if (result.status === 'success') {
      res.json({ 
        message: "CodeREADr API connection successful",
        databases_count: result.databases?.length || 0
      });
    } else {
      res.status(400).json({ error: result.message });
    }
  } catch (error) {
    console.error("CodeREADr test error:", error);
    res.status(500).json({ error: "Failed to test CodeREADr connection" });
  }
});
```

### Back Button Implementation for All Pages

Add back button components to all pages. Update each page component:

```typescript
// Add to top of every page component
import { ArrowLeft } from 'lucide-react';
import { Link } from 'wouter';

// Add this header to every page (modify the href as appropriate):
<div className="sticky top-0 z-50 bg-white/10 backdrop-blur-md border-b border-white/20 px-4 py-3">
  <div className="flex items-center justify-between max-w-7xl mx-auto">
    <Link href="/">
      <Button size="sm" variant="ghost" className="text-white hover:bg-white/20">
        <ArrowLeft className="w-4 h-4 mr-1" />
        Back
      </Button>
    </Link>
    <h1 className="text-lg font-bold text-white">[Page Title]</h1>
    <div></div> {/* Spacer for center alignment */}
  </div>
</div>
```

### Mobile-First Landing Page with Clickable Event Cards
Create `client/src/pages/LandingPage.tsx`:
```typescript
import React, { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { motion } from 'framer-motion';
import { Calendar, MapPin, Ticket, Search, Menu, User, ChevronRight } from 'lucide-react';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Badge } from '@/components/ui/badge';
import { Link, useLocation } from 'wouter';

export function LandingPage() {
  const [searchQuery, setSearchQuery] = useState('');
  const [, navigate] = useLocation();

  // Fetch featured events with error handling
  const { data: events, isLoading, error } = useQuery({
    queryKey: ['/api/events/featured'],
    queryFn: async () => {
      const response = await fetch('/api/events/featured');
      if (!response.ok) {
        throw new Error('Failed to fetch events');
      }
      return response.json();
    },
    retry: 2,
    staleTime: 5 * 60 * 1000, // 5 minutes
  });

  // Filter events based on search
  const filteredEvents = events?.filter((event: any) =>
    event.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
    event.venue.toLowerCase().includes(searchQuery.toLowerCase()) ||
    event.description.toLowerCase().includes(searchQuery.toLowerCase())
  ) || [];

  // Handle event card click - navigate to event details
  const handleEventClick = (eventSlug: string) => {
    navigate(`/events/${eventSlug}`);
  };

  return (
    <div className="min-h-screen">
      {/* Mobile-friendly header */}
      <div className="sticky top-0 z-50 bg-white/10 backdrop-blur-md border-b border-white/20 px-4 py-3">
        <div className="flex items-center justify-between max-w-7xl mx-auto">
          <div className="flex items-center space-x-2">
            <h1 className="text-xl font-bold text-white">Uliza</h1>
          </div>
          <div className="flex items-center space-x-2">
            <Link href="/login">
              <Button size="sm" variant="ghost" className="text-white hover:bg-white/20">
                <User className="w-4 h-4 mr-1" />
                Login
              </Button>
            </Link>
          </div>
        </div>
      </div>

      {/* Hero Section - Mobile Optimized */}
      <motion.section 
        initial={{ opacity: 0, y: 20 }}
        animate={{ opacity: 1, y: 0 }}
        transition={{ duration: 0.8 }}
        className="relative py-12 md:py-20 px-4 text-center text-white"
      >
        <div className="max-w-4xl mx-auto">
          <motion.h1 
            initial={{ opacity: 0, y: 30 }}
            animate={{ opacity: 1, y: 0 }}
            transition={{ delay: 0.2, duration: 0.8 }}
            className="text-4xl md:text-6xl lg:text-8xl font-bold mb-4 md:mb-6 bg-gradient-to-r from-white to-yellow-200 bg-clip-text text-transparent leading-tight"
          >
            Uliza Tickets
          </motion.h1>
          <motion.p 
            initial={{ opacity: 0, y: 30 }}
            animate={{ opacity: 1, y: 0 }}
            transition={{ delay: 0.4, duration: 0.8 }}
            className="text-lg md:text-xl lg:text-2xl mb-8 opacity-90 px-4"
          >
            Discover Amazing Events Across Africa 🌍
          </motion.p>
          
          {/* Mobile-friendly Search Bar */}
          <motion.div 
            initial={{ opacity: 0, y: 30 }}
            animate={{ opacity: 1, y: 0 }}
            transition={{ delay: 0.6, duration: 0.8 }}
            className="max-w-md mx-auto mb-8 md:mb-12"
          >
            <div className="relative">
              <Search className="absolute left-3 top-1/2 transform -translate-y-1/2 text-gray-400 w-5 h-5" />
              <Input 
                placeholder="Search events, venues..."
                value={searchQuery}
                onChange={(e) => setSearchQuery(e.target.value)}
                className="pl-10 py-3 text-base md:text-lg bg-white/20 border-white/30 text-white placeholder-white/70 backdrop-blur-sm rounded-full"
                data-testid="search-events"
              />
            </div>
          </motion.div>

          {/* Quick Stats */}
          <motion.div 
            initial={{ opacity: 0, y: 30 }}
            animate={{ opacity: 1, y: 0 }}
            transition={{ delay: 0.8, duration: 0.8 }}
            className="flex flex-wrap justify-center gap-4 md:gap-8 text-white/80"
          >
            <div className="text-center">
              <p className="text-2xl font-bold text-white">{events?.length || 0}+</p>
              <p className="text-sm">Live Events</p>
            </div>
            <div className="text-center">
              <p className="text-2xl font-bold text-white">50K+</p>
              <p className="text-sm">Happy Attendees</p>
            </div>
            <div className="text-center">
              <p className="text-2xl font-bold text-white">500+</p>
              <p className="text-sm">Organizers</p>
            </div>
          </motion.div>
        </div>
      </motion.section>

      {/* Featured Events - Mobile-First Grid */}
      <section className="py-8 md:py-16 px-4">
        <div className="max-w-7xl mx-auto">
          <motion.div
            initial={{ opacity: 0, y: 20 }}
            whileInView={{ opacity: 1, y: 0 }}
            transition={{ duration: 0.6 }}
            className="text-center mb-8 md:mb-12"
          >
            <h2 className="text-3xl md:text-4xl font-bold text-white mb-2">
              🔥 Featured Events
            </h2>
            <p className="text-white/80 text-base md:text-lg">
              Don't miss out on these amazing experiences
            </p>
          </motion.div>
          
          {error ? (
            <div className="text-center py-12">
              <p className="text-white/70 mb-4">Unable to load events. Please try again later.</p>
              <Button 
                onClick={() => window.location.reload()} 
                className="bg-gradient-to-r from-pink-500 to-yellow-500"
              >
                Retry
              </Button>
            </div>
          ) : isLoading ? (
            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 md:gap-8">
              {[...Array(6)].map((_, i) => (
                <Card key={i} className="bg-white/10 border-white/20 backdrop-blur-sm animate-pulse">
                  <CardContent className="p-4 md:p-6">
                    <div className="h-40 md:h-48 bg-white/20 rounded-lg mb-4"></div>
                    <div className="h-6 bg-white/20 rounded mb-2"></div>
                    <div className="h-4 bg-white/20 rounded mb-4"></div>
                    <div className="h-10 bg-white/20 rounded"></div>
                  </CardContent>
                </Card>
              ))}
            </div>
          ) : filteredEvents.length > 0 ? (
            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 md:gap-6 lg:gap-8">
              {filteredEvents.map((event: any, index: number) => (
                <motion.div
                  key={event.id}
                  initial={{ opacity: 0, y: 30 }}
                  whileInView={{ opacity: 1, y: 0 }}
                  transition={{ delay: index * 0.1, duration: 0.6 }}
                  whileHover={{ scale: 1.02, transition: { duration: 0.2 } }}
                  whileTap={{ scale: 0.98 }}
                  className="cursor-pointer"
                  onClick={() => handleEventClick(event.slug)}
                  data-testid={`event-card-${event.slug}`}
                >
                  <Card className="bg-white/10 border-white/20 backdrop-blur-sm hover:bg-white/20 transition-all duration-300 overflow-hidden h-full">
                    {/* Event Image */}
                    <div className="relative">
                      <img 
                        src={event.image_url}
                        alt={event.name}
                        className="w-full h-40 md:h-48 object-cover"
                        loading="lazy"
                      />
                      
                      {/* Price Badge */}
                      <div className="absolute top-3 right-3 bg-gradient-to-r from-pink-500 to-yellow-500 text-white px-2 md:px-3 py-1 rounded-full text-xs md:text-sm font-bold">
                        From KES {event.price_kes ? event.price_kes.toLocaleString() : '100'}
                      </div>

                      {/* Date Badge */}
                      <div className="absolute top-3 left-3 bg-black/70 text-white px-2 py-1 rounded text-xs font-medium backdrop-blur-sm">
                        {new Date(event.date_from).toLocaleDateString('en-US', { 
                          month: 'short', 
                          day: 'numeric' 
                        })}
                      </div>
                    </div>

                    {/* Event Details */}
                    <CardHeader className="text-white p-3 md:p-4">
                      <CardTitle className="text-base md:text-lg font-bold line-clamp-2 leading-tight">
                        {event.name}
                      </CardTitle>
                      
                      <div className="space-y-1">
                        <CardDescription className="text-white/80 flex items-center gap-1 text-xs md:text-sm">
                          <Calendar className="w-3 h-3 md:w-4 md:h-4 flex-shrink-0" />
                          <span className="truncate">
                            {new Date(event.date_from).toLocaleDateString('en-US', {
                              weekday: 'short',
                              month: 'short',
                              day: 'numeric'
                            })}
                          </span>
                        </CardDescription>
                        
                        <CardDescription className="text-white/80 flex items-center gap-1 text-xs md:text-sm">
                          <MapPin className="w-3 h-3 md:w-4 md:h-4 flex-shrink-0" />
                          <span className="truncate">{event.venue}</span>
                        </CardDescription>
                      </div>

                      {/* Event Description Preview */}
                      <p className="text-white/70 text-xs md:text-sm line-clamp-2 mt-2">
                        {event.description}
                      </p>
                    </CardHeader>

                    {/* Action Button */}
                    <CardContent className="pt-0 p-3 md:p-4">
                      <Button 
                        className="w-full bg-gradient-to-r from-pink-500 to-yellow-500 hover:from-pink-600 hover:to-yellow-600 border-0 font-bold text-sm md:text-base py-2 md:py-3"
                        onClick={(e) => {
                          e.stopPropagation(); // Prevent card click
                          handleEventClick(event.slug);
                        }}
                        data-testid={`get-tickets-${event.slug}`}
                      >
                        <Ticket className="w-3 h-3 md:w-4 md:h-4 mr-1 md:mr-2" />
                        Get Tickets
                        <ChevronRight className="w-3 h-3 md:w-4 md:h-4 ml-1" />
                      </Button>
                    </CardContent>
                  </Card>
                </motion.div>
              ))}
            </div>
          ) : (
            <div className="text-center py-12">
              <p className="text-white/70 text-lg mb-4">
                {searchQuery ? `No events found for "${searchQuery}"` : 'No events available at the moment'}
              </p>
              {searchQuery && (
                <Button 
                  onClick={() => setSearchQuery('')}
                  variant="outline"
                  className="bg-white/10 border-white/30 text-white hover:bg-white/20"
                >
                  Clear Search
                </Button>
              )}
            </div>
          )}

          {/* Load More Button - only show if there are many events */}
          {filteredEvents.length > 12 && (
            <div className="text-center mt-8 md:mt-12">
              <Button 
                variant="outline"
                className="bg-white/10 border-white/30 text-white hover:bg-white/20 px-8 py-3"
              >
                Load More Events
              </Button>
            </div>
          )}
        </div>
      </section>

      {/* Call to Action Section */}
      <section className="py-12 md:py-16 px-4 bg-black/20">
        <div className="max-w-4xl mx-auto text-center">
          <motion.div
            initial={{ opacity: 0, y: 20 }}
            whileInView={{ opacity: 1, y: 0 }}
            transition={{ duration: 0.6 }}
          >
            <h3 className="text-2xl md:text-3xl font-bold text-white mb-4">
              Ready to Create Your Own Event? 🎪
            </h3>
            <p className="text-white/80 text-base md:text-lg mb-6 md:mb-8 px-4">
              Join hundreds of organizers who trust Uliza Tickets to manage their events
            </p>
            <div className="flex flex-col sm:flex-row gap-4 justify-center">
              <Link href="/signup">
                <Button 
                  size="lg"
                  className="bg-gradient-to-r from-pink-500 to-yellow-500 hover:from-pink-600 hover:to-yellow-600 text-white font-bold px-8 py-3"
                  data-testid="create-account"
                >
                  Create Account
                </Button>
              </Link>
              <Link href="/login">
                <Button 
                  size="lg"
                  variant="outline"
                  className="bg-white/10 border-white/30 text-white hover:bg-white/20 px-8 py-3"
                  data-testid="sign-in"
                >
                  Sign In
                </Button>
              </Link>
            </div>
          </motion.div>
        </div>
      </section>
    </div>
  );
}
```

### Authentication Pages
Create `client/src/pages/LoginPage.tsx`:
```typescript
import React, { useState } from 'react';
import { motion } from 'framer-motion';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from '@/components/ui/form';
import { useToast } from '@/hooks/use-toast';
import { Link, useLocation } from 'wouter';
import { Eye, EyeOff, LogIn, User } from 'lucide-react';

const loginSchema = z.object({
  email: z.string().email('Please enter a valid email address'),
  password: z.string().min(6, 'Password must be at least 6 characters'),
});

type LoginForm = z.infer<typeof loginSchema>;

export function LoginPage() {
  const [, setLocation] = useLocation();
  const [showPassword, setShowPassword] = useState(false);
  const [isLoading, setIsLoading] = useState(false);
  const { toast } = useToast();

  const form = useForm<LoginForm>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      email: '',
      password: '',
    },
  });

  const onSubmit = async (data: LoginForm) => {
    setIsLoading(true);
    try {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });

      const result = await response.json();

      if (response.ok) {
        localStorage.setItem('auth_token', result.token);
        toast({
          title: 'Welcome back!',
          description: 'Successfully logged in',
        });

        // Redirect based on role
        switch (result.user.role) {
          case 'admin':
            setLocation('/admin');
            break;
          case 'organizer':
            setLocation('/organizer');
            break;
          case 'scanner':
            setLocation('/scanner');
            break;
          default:
            setLocation('/');
        }
      } else {
        toast({
          title: 'Login failed',
          description: result.error || 'Invalid credentials',
          variant: 'destructive',
        });
      }
    } catch (error) {
      toast({
        title: 'Login failed',
        description: 'Network error. Please try again.',
        variant: 'destructive',
      });
    } finally {
      setIsLoading(false);
    }
  };

  // Helper function for form validation
  const validateAndSubmit = async () => {
    const email = form.getValues('email');
    const password = form.getValues('password');
    
    if (!email || !password) {
      toast({
        title: 'Missing Information',
        description: 'Please enter both email and password',
        variant: 'destructive',
      });
      return;
    }
    
    await onSubmit({ email, password });
  };

  return (
    <div className="min-h-screen flex items-center justify-center p-4">
      <motion.div
        initial={{ opacity: 0, y: 20 }}
        animate={{ opacity: 1, y: 0 }}
        transition={{ duration: 0.6 }}
        className="w-full max-w-md"
      >
        <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
          <CardHeader className="text-center">
            <motion.div
              initial={{ scale: 0.9 }}
              animate={{ scale: 1 }}
              transition={{ delay: 0.2, duration: 0.4 }}
            >
              <CardTitle className="text-2xl font-bold text-white mb-2">
                Welcome Back! 🎭
              </CardTitle>
              <CardDescription className="text-white/80">
                Sign in to your Uliza Tickets account
              </CardDescription>
            </motion.div>
          </CardHeader>
          <CardContent className="space-y-6">
            <Form {...form}>
              <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
                <FormField
                  control={form.control}
                  name="email"
                  render={({ field }) => (
                    <FormItem>
                      <FormLabel className="text-white">Email</FormLabel>
                      <FormControl>
                        <Input
                          {...field}
                          type="email"
                          placeholder="Enter your email"
                          className="bg-white/20 border-white/30 text-white placeholder-white/70"
                          data-testid="login-email"
                        />
                      </FormControl>
                      <FormMessage className="text-red-300" />
                    </FormItem>
                  )}
                />

                <FormField
                  control={form.control}
                  name="password"
                  render={({ field }) => (
                    <FormItem>
                      <FormLabel className="text-white">Password</FormLabel>
                      <FormControl>
                        <div className="relative">
                          <Input
                            {...field}
                            type={showPassword ? 'text' : 'password'}
                            placeholder="Enter your password"
                            className="bg-white/20 border-white/30 text-white placeholder-white/70 pr-10"
                            data-testid="login-password"
                          />
                          <Button
                            type="button"
                            variant="ghost"
                            size="sm"
                            className="absolute right-0 top-0 h-full px-3 text-white/70 hover:text-white"
                            onClick={() => setShowPassword(!showPassword)}
                          >
                            {showPassword ? <EyeOff className="w-4 h-4" /> : <Eye className="w-4 h-4" />}
                          </Button>
                        </div>
                      </FormControl>
                      <FormMessage className="text-red-300" />
                    </FormItem>
                  )}
                />

                <Button
                  type="submit"
                  className="w-full bg-gradient-to-r from-pink-500 to-yellow-500 hover:from-pink-600 hover:to-yellow-600 text-white font-bold"
                  disabled={isLoading}
                  data-testid="login-submit"
                >
                  {isLoading ? (
                    <>
                      <div className="w-4 h-4 border-2 border-white/30 border-t-white rounded-full animate-spin mr-2" />
                      Signing in...
                    </>
                  ) : (
                    <>
                      <LogIn className="w-4 h-4 mr-2" />
                      Sign In
                    </>
                  )}
                </Button>
              </form>
            </Form>

            {/* First Time User Help */}
            <div className="border-t border-white/20 pt-4">
              <p className="text-white/80 text-sm text-center mb-3">First time user?</p>
              <div className="text-center">
                <Link href="/signup">
                  <Button
                    variant="outline"
                    size="sm"
                    className="bg-green-500/20 border-green-300/30 text-green-100 hover:bg-green-500/30"
                    data-testid="create-account-link"
                  >
                    ✨ Create Your Account
                  </Button>
                </Link>
              </div>
            </div>

            <div className="text-center">
              <p className="text-white/80 text-sm">
                Don't have an account?{' '}
                <Link href="/signup" className="text-yellow-300 hover:text-yellow-200 font-medium">
                  Sign up here
                </Link>
              </p>
            </div>
          </CardContent>
        </Card>
      </motion.div>
    </div>
  );
}
```

Create `client/src/pages/SignupPage.tsx`:
```typescript
import React, { useState } from 'react';
import { motion } from 'framer-motion';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from '@/components/ui/form';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { useToast } from '@/hooks/use-toast';
import { Link, useLocation } from 'wouter';
import { Eye, EyeOff, UserPlus } from 'lucide-react';

const signupSchema = z.object({
  firstName: z.string().min(2, 'First name must be at least 2 characters'),
  lastName: z.string().min(2, 'Last name must be at least 2 characters'),
  email: z.string().email('Please enter a valid email address'),
  phone: z.string().min(10, 'Phone number must be at least 10 digits'),
  password: z.string().min(6, 'Password must be at least 6 characters'),
  confirmPassword: z.string(),
  role: z.enum(['user', 'organizer'], { required_error: 'Please select a role' }),
}).refine((data) => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ["confirmPassword"],
});

type SignupForm = z.infer<typeof signupSchema>;

export function SignupPage() {
  const [, setLocation] = useLocation();
  const [showPassword, setShowPassword] = useState(false);
  const [showConfirmPassword, setShowConfirmPassword] = useState(false);
  const [isLoading, setIsLoading] = useState(false);
  const { toast } = useToast();

  const form = useForm<SignupForm>({
    resolver: zodResolver(signupSchema),
    defaultValues: {
      firstName: '',
      lastName: '',
      email: '',
      phone: '',
      password: '',
      confirmPassword: '',
      role: 'user',
    },
  });

  const onSubmit = async (data: SignupForm) => {
    setIsLoading(true);
    try {
      const response = await fetch('/api/auth/register', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          first_name: data.firstName,
          last_name: data.lastName,
          email: data.email,
          phone: data.phone,
          password: data.password,
          role: data.role,
        }),
      });

      const result = await response.json();

      if (response.ok) {
        if (data.role === 'organizer') {
          toast({
            title: 'Registration Successful! 🎉',
            description: 'Your organizer application is pending admin approval. You will be notified via email once approved.',
          });
        } else {
          localStorage.setItem('auth_token', result.token);
          toast({
            title: 'Welcome to Uliza Tickets! 🎉',
            description: 'Your account has been created successfully',
          });
        }
        setLocation('/login');
      } else {
        toast({
          title: 'Registration failed',
          description: result.error || 'Something went wrong',
          variant: 'destructive',
        });
      }
    } catch (error) {
      toast({
        title: 'Registration failed',
        description: 'Network error. Please try again.',
        variant: 'destructive',
      });
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center p-4">
      <motion.div
        initial={{ opacity: 0, y: 20 }}
        animate={{ opacity: 1, y: 0 }}
        transition={{ duration: 0.6 }}
        className="w-full max-w-md"
      >
        <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
          <CardHeader className="text-center">
            <motion.div
              initial={{ scale: 0.9 }}
              animate={{ scale: 1 }}
              transition={{ delay: 0.2, duration: 0.4 }}
            >
              <CardTitle className="text-2xl font-bold text-white mb-2">
                Join Uliza Tickets! 🌟
              </CardTitle>
              <CardDescription className="text-white/80">
                Create your account to get started
              </CardDescription>
            </motion.div>
          </CardHeader>
          <CardContent>
            <Form {...form}>
              <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
                <div className="grid grid-cols-2 gap-4">
                  <FormField
                    control={form.control}
                    name="firstName"
                    render={({ field }) => (
                      <FormItem>
                        <FormLabel className="text-white">First Name</FormLabel>
                        <FormControl>
                          <Input
                            {...field}
                            placeholder="John"
                            className="bg-white/20 border-white/30 text-white placeholder-white/70"
                            data-testid="signup-firstname"
                          />
                        </FormControl>
                        <FormMessage className="text-red-300" />
                      </FormItem>
                    )}
                  />

                  <FormField
                    control={form.control}
                    name="lastName"
                    render={({ field }) => (
                      <FormItem>
                        <FormLabel className="text-white">Last Name</FormLabel>
                        <FormControl>
                          <Input
                            {...field}
                            placeholder="Doe"
                            className="bg-white/20 border-white/30 text-white placeholder-white/70"
                            data-testid="signup-lastname"
                          />
                        </FormControl>
                        <FormMessage className="text-red-300" />
                      </FormItem>
                    )}
                  />
                </div>

                <FormField
                  control={form.control}
                  name="email"
                  render={({ field }) => (
                    <FormItem>
                      <FormLabel className="text-white">Email</FormLabel>
                      <FormControl>
                        <Input
                          {...field}
                          type="email"
                          placeholder="john@example.com"
                          className="bg-white/20 border-white/30 text-white placeholder-white/70"
                          data-testid="signup-email"
                        />
                      </FormControl>
                      <FormMessage className="text-red-300" />
                    </FormItem>
                  )}
                />

                <FormField
                  control={form.control}
                  name="phone"
                  render={({ field }) => (
                    <FormItem>
                      <FormLabel className="text-white">Phone Number</FormLabel>
                      <FormControl>
                        <Input
                          {...field}
                          placeholder="+254712345678"
                          className="bg-white/20 border-white/30 text-white placeholder-white/70"
                          data-testid="signup-phone"
                        />
                      </FormControl>
                      <FormMessage className="text-red-300" />
                    </FormItem>
                  )}
                />

                <FormField
                  control={form.control}
                  name="role"
                  render={({ field }) => (
                    <FormItem>
                      <FormLabel className="text-white">Account Type</FormLabel>
                      <Select onValueChange={field.onChange} defaultValue={field.value}>
                        <FormControl>
                          <SelectTrigger className="bg-white/20 border-white/30 text-white" data-testid="signup-role">
                            <SelectValue placeholder="Select your role" />
                          </SelectTrigger>
                        </FormControl>
                        <SelectContent>
                          <SelectItem value="user">User - Buy tickets and attend events</SelectItem>
                          <SelectItem value="organizer">Organizer - Create and manage events</SelectItem>
                        </SelectContent>
                      </Select>
                      <FormMessage className="text-red-300" />
                    </FormItem>
                  )}
                />

                <FormField
                  control={form.control}
                  name="password"
                  render={({ field }) => (
                    <FormItem>
                      <FormLabel className="text-white">Password</FormLabel>
                      <FormControl>
                        <div className="relative">
                          <Input
                            {...field}
                            type={showPassword ? 'text' : 'password'}
                            placeholder="Create a password"
                            className="bg-white/20 border-white/30 text-white placeholder-white/70 pr-10"
                            data-testid="signup-password"
                          />
                          <Button
                            type="button"
                            variant="ghost"
                            size="sm"
                            className="absolute right-0 top-0 h-full px-3 text-white/70 hover:text-white"
                            onClick={() => setShowPassword(!showPassword)}
                          >
                            {showPassword ? <EyeOff className="w-4 h-4" /> : <Eye className="w-4 h-4" />}
                          </Button>
                        </div>
                      </FormControl>
                      <FormMessage className="text-red-300" />
                    </FormItem>
                  )}
                />

                <FormField
                  control={form.control}
                  name="confirmPassword"
                  render={({ field }) => (
                    <FormItem>
                      <FormLabel className="text-white">Confirm Password</FormLabel>
                      <FormControl>
                        <div className="relative">
                          <Input
                            {...field}
                            type={showConfirmPassword ? 'text' : 'password'}
                            placeholder="Confirm your password"
                            className="bg-white/20 border-white/30 text-white placeholder-white/70 pr-10"
                            data-testid="signup-confirm-password"
                          />
                          <Button
                            type="button"
                            variant="ghost"
                            size="sm"
                            className="absolute right-0 top-0 h-full px-3 text-white/70 hover:text-white"
                            onClick={() => setShowConfirmPassword(!showConfirmPassword)}
                          >
                            {showConfirmPassword ? <EyeOff className="w-4 h-4" /> : <Eye className="w-4 h-4" />}
                          </Button>
                        </div>
                      </FormControl>
                      <FormMessage className="text-red-300" />
                    </FormItem>
                  )}
                />

                <Button
                  type="submit"
                  className="w-full bg-gradient-to-r from-pink-500 to-yellow-500 hover:from-pink-600 hover:to-yellow-600 text-white font-bold"
                  disabled={isLoading}
                  data-testid="signup-submit"
                >
                  {isLoading ? (
                    <>
                      <div className="w-4 h-4 border-2 border-white/30 border-t-white rounded-full animate-spin mr-2" />
                      Creating Account...
                    </>
                  ) : (
                    <>
                      <UserPlus className="w-4 h-4 mr-2" />
                      Create Account
                    </>
                  )}
                </Button>
              </form>
            </Form>

            <div className="text-center mt-6">
              <p className="text-white/80 text-sm">
                Already have an account?{' '}
                <Link href="/login" className="text-yellow-300 hover:text-yellow-200 font-medium">
                  Sign in here
                </Link>
              </p>
            </div>
          </CardContent>
        </Card>
      </motion.div>
    </div>
  );
}
```

### Complete Admin Dashboard
Create `client/src/pages/AdminDashboard.tsx`:
```typescript
import React, { useState } from 'react';
import { motion } from 'framer-motion';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { Input } from '@/components/ui/input';
import { useToast } from '@/hooks/use-toast';
import { 
  Users, 
  DollarSign, 
  Calendar, 
  TrendingUp, 
  UserCheck, 
  UserX, 
  Search,
  Eye,
  CheckCircle,
  XCircle,
  Crown,
  Building2
} from 'lucide-react';

export function AdminDashboard() {
  const [userSearchQuery, setUserSearchQuery] = useState('');
  const [eventSearchQuery, setEventSearchQuery] = useState('');
  const [selectedUserRole, setSelectedUserRole] = useState<string>('');
  const [selectedCategory, setSelectedCategory] = useState<string>('');
  const [createEventForm, setCreateEventForm] = useState({
    name: '',
    description: '',
    venue: '',
    date_from: '',
    date_to: '',
    price_kes: '',
    category: 'general',
    organizer_id: '',
    image_url: ''
  });
  const [showCreateEvent, setShowCreateEvent] = useState(false);
  const { toast } = useToast();
  const queryClient = useQueryClient();

  // Fetch system statistics
  const { data: stats, isLoading: statsLoading } = useQuery({
    queryKey: ['/api/admin/stats'],
  });

  // Fetch all users with search and filters
  const { data: users, isLoading: usersLoading } = useQuery({
    queryKey: ['/api/admin/users', userSearchQuery, selectedUserRole],
    queryFn: async () => {
      const params = new URLSearchParams();
      if (userSearchQuery) params.append('search', userSearchQuery);
      if (selectedUserRole) params.append('role', selectedUserRole);
      params.append('limit', '100');
      
      const token = localStorage.getItem('auth_token');
      const response = await fetch(`/api/admin/users?${params}`, {
        headers: { 'Authorization': `Bearer ${token}` },
      });
      if (!response.ok) throw new Error('Failed to fetch users');
      return response.json();
    },
  });

  // Fetch all events with search and filters
  const { data: events, isLoading: eventsLoading } = useQuery({
    queryKey: ['/api/admin/events', eventSearchQuery, selectedCategory],
    queryFn: async () => {
      const params = new URLSearchParams();
      if (eventSearchQuery) params.append('search', eventSearchQuery);
      if (selectedCategory) params.append('category', selectedCategory);
      params.append('limit', '100');
      
      const token = localStorage.getItem('auth_token');
      const response = await fetch(`/api/admin/events?${params}`, {
        headers: { 'Authorization': `Bearer ${token}` },
      });
      if (!response.ok) throw new Error('Failed to fetch events');
      return response.json();
    },
  });

  // Approve/reject user mutation
  const updateUserMutation = useMutation({
    mutationFn: async ({ userId, approved, role }: { userId: string; approved?: boolean; role?: string }) => {
      const token = localStorage.getItem('auth_token');
      const response = await fetch(`/api/admin/users/${userId}`, {
        method: 'PATCH',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`,
        },
        body: JSON.stringify({ approved, role }),
      });
      if (!response.ok) throw new Error('Failed to update user');
      return response.json();
    },
    onSuccess: (data) => {
      queryClient.invalidateQueries({ queryKey: ['/api/admin/users'] });
      queryClient.invalidateQueries({ queryKey: ['/api/admin/stats'] });
      toast({
        title: '✅ Success',
        description: data.message || 'User updated successfully',
      });
    },
  });

  // Create event mutation
  const createEventMutation = useMutation({
    mutationFn: async (eventData: any) => {
      const token = localStorage.getItem('auth_token');
      const response = await fetch('/api/admin/events', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`,
        },
        body: JSON.stringify(eventData),
      });
      if (!response.ok) throw new Error('Failed to create event');
      return response.json();
    },
    onSuccess: (data) => {
      queryClient.invalidateQueries({ queryKey: ['/api/admin/events'] });
      queryClient.invalidateQueries({ queryKey: ['/api/admin/stats'] });
      toast({
        title: '🎉 Event Created',
        description: data.message || 'Event created successfully',
      });
      setShowCreateEvent(false);
      setCreateEventForm({
        name: '',
        description: '',
        venue: '',
        date_from: '',
        date_to: '',
        price_kes: '',
        category: 'general',
        organizer_id: '',
        image_url: ''
      });
    },
  });

  // Delete event mutation
  const deleteEventMutation = useMutation({
    mutationFn: async (eventId: string) => {
      const token = localStorage.getItem('auth_token');
      const response = await fetch(`/api/admin/events/${eventId}`, {
        method: 'DELETE',
        headers: { 'Authorization': `Bearer ${token}` },
      });
      if (!response.ok) throw new Error('Failed to delete event');
      return response.json();
    },
    onSuccess: (data) => {
      queryClient.invalidateQueries({ queryKey: ['/api/admin/events'] });
      queryClient.invalidateQueries({ queryKey: ['/api/admin/stats'] });
      toast({
        title: '🗑️ Event Deleted',
        description: data.message || 'Event deleted successfully',
      });
    },
  });

  // Get approved organizers for event creation
  const approvedOrganizers = users?.filter((user: any) => 
    user.role === 'organizer' && user.approved
  ) || [];

  // Filter pending users
  const pendingUsers = users?.filter((user: any) => !user.approved) || [];

  const handleCreateEvent = () => {
    if (!createEventForm.name || !createEventForm.venue || !createEventForm.date_from || !createEventForm.price_kes || !createEventForm.organizer_id) {
      toast({
        title: 'Missing Information',
        description: 'Please fill in all required fields',
        variant: 'destructive',
      });
      return;
    }

    createEventMutation.mutate(createEventForm);
  };

  return (
    <div className="min-h-screen p-6 bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500">
      <div className="max-w-7xl mx-auto">
        <motion.div
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          transition={{ duration: 0.6 }}
        >
          {/* Header */}
          <div className="mb-8">
            <h1 className="text-4xl font-bold text-white mb-2">
              Admin Dashboard 👑
            </h1>
            <p className="text-xl text-white/80">
              Platform Overview and Management
            </p>
          </div>

          {/* System Statistics */}
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-8">
            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.1 }}
            >
              <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-white">Total Revenue</CardTitle>
                  <DollarSign className="h-4 w-4 text-yellow-400" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-white" data-testid="admin-total-revenue">
                    KES {statsLoading ? '...' : (stats?.total_revenue || 0).toLocaleString()}
                  </div>
                  <p className="text-xs text-white/70 mt-1">
                    Tickets Sold: {statsLoading ? '...' : (stats?.total_tickets_sold || 0).toLocaleString()}
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.2 }}
            >
              <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-white">Total Events</CardTitle>
                  <Calendar className="h-4 w-4 text-blue-400" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-white" data-testid="admin-total-events">
                    {statsLoading ? '...' : stats?.total_events || 0}
                  </div>
                  <p className="text-xs text-white/70 mt-1">
                    Upcoming: {statsLoading ? '...' : stats?.upcoming_events || 0}
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.3 }}
            >
              <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-white">Total Users</CardTitle>
                  <Users className="h-4 w-4 text-green-400" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-white" data-testid="admin-total-users">
                    {statsLoading ? '...' : stats?.total_users || 0}
                  </div>
                  <p className="text-xs text-white/70 mt-1">
                    Pending: {statsLoading ? '...' : stats?.pending_users || 0}
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.4 }}
            >
              <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-white">Pending Payments</CardTitle>
                  <TrendingUp className="h-4 w-4 text-orange-400" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-white" data-testid="admin-pending-payments">
                    {statsLoading ? '...' : stats?.pending_payments || 0}
                  </div>
                  <p className="text-xs text-white/70 mt-1">
                    Scanned: {statsLoading ? '...' : stats?.tickets_scanned || 0}
                  </p>
                </CardContent>
              </Card>
            </motion.div>
          </div>

          {/* Create Event Section */}
          <motion.div
            initial={{ opacity: 0, y: 20 }}
            animate={{ opacity: 1, y: 0 }}
            transition={{ duration: 0.6, delay: 0.5 }}
            className="mb-8"
          >
            <Card className="bg-white/10 backdrop-blur-sm border-white/20">
              <CardHeader>
                <div className="flex items-center justify-between">
                  <div>
                    <CardTitle className="text-white flex items-center">
                      <Calendar className="w-6 h-6 mr-2" />
                      Event Management
                    </CardTitle>
                    <CardDescription className="text-white/70">
                      Create and manage events on the platform
                    </CardDescription>
                  </div>
                  <Button
                    onClick={() => setShowCreateEvent(!showCreateEvent)}
                    className="bg-blue-500 hover:bg-blue-600"
                    data-testid="toggle-create-event"
                  >
                    {showCreateEvent ? 'Cancel' : '+ Create Event'}
                  </Button>
                </div>
              </CardHeader>
              
              {showCreateEvent && (
                <CardContent className="space-y-4">
                  <div className="bg-white/5 rounded-lg p-4">
                    <h3 className="text-white font-semibold mb-4">Create New Event</h3>
                    
                    <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                      <div>
                        <label className="text-white/80 text-sm">Event Name *</label>
                        <Input
                          value={createEventForm.name}
                          onChange={(e) => setCreateEventForm({...createEventForm, name: e.target.value})}
                          placeholder="Enter event name"
                          className="bg-white/10 border-white/20 text-white"
                          data-testid="event-name-input"
                        />
                      </div>
                      
                      <div>
                        <label className="text-white/80 text-sm">Venue *</label>
                        <Input
                          value={createEventForm.venue}
                          onChange={(e) => setCreateEventForm({...createEventForm, venue: e.target.value})}
                          placeholder="Event venue"
                          className="bg-white/10 border-white/20 text-white"
                          data-testid="event-venue-input"
                        />
                      </div>
                      
                      <div>
                        <label className="text-white/80 text-sm">Start Date *</label>
                        <Input
                          type="datetime-local"
                          value={createEventForm.date_from}
                          onChange={(e) => setCreateEventForm({...createEventForm, date_from: e.target.value})}
                          className="bg-white/10 border-white/20 text-white"
                          data-testid="event-date-input"
                        />
                      </div>
                      
                      <div>
                        <label className="text-white/80 text-sm">End Date</label>
                        <Input
                          type="datetime-local"
                          value={createEventForm.date_to}
                          onChange={(e) => setCreateEventForm({...createEventForm, date_to: e.target.value})}
                          className="bg-white/10 border-white/20 text-white"
                        />
                      </div>
                      
                      <div>
                        <label className="text-white/80 text-sm">Price (KES) *</label>
                        <Input
                          type="number"
                          value={createEventForm.price_kes}
                          onChange={(e) => setCreateEventForm({...createEventForm, price_kes: e.target.value})}
                          placeholder="0"
                          className="bg-white/10 border-white/20 text-white"
                          data-testid="event-price-input"
                        />
                      </div>
                      
                      <div>
                        <label className="text-white/80 text-sm">Category</label>
                        <select
                          value={createEventForm.category}
                          onChange={(e) => setCreateEventForm({...createEventForm, category: e.target.value})}
                          className="w-full p-2 rounded bg-white/10 border border-white/20 text-white"
                          data-testid="event-category-select"
                        >
                          <option value="general" className="bg-gray-800">General</option>
                          <option value="technology" className="bg-gray-800">Technology</option>
                          <option value="business" className="bg-gray-800">Business</option>
                          <option value="music" className="bg-gray-800">Music</option>
                          <option value="culture" className="bg-gray-800">Culture</option>
                          <option value="health" className="bg-gray-800">Health</option>
                          <option value="education" className="bg-gray-800">Education</option>
                          <option value="environment" className="bg-gray-800">Environment</option>
                        </select>
                      </div>
                      
                      <div className="md:col-span-2">
                        <label className="text-white/80 text-sm">Organizer *</label>
                        <select
                          value={createEventForm.organizer_id}
                          onChange={(e) => setCreateEventForm({...createEventForm, organizer_id: e.target.value})}
                          className="w-full p-2 rounded bg-white/10 border border-white/20 text-white"
                          data-testid="event-organizer-select"
                        >
                          <option value="" className="bg-gray-800">Select organizer</option>
                          {approvedOrganizers.map((organizer) => (
                            <option key={organizer.id} value={organizer.id} className="bg-gray-800">
                              {organizer.name} ({organizer.email})
                            </option>
                          ))}
                        </select>
                      </div>
                      
                      <div className="md:col-span-2">
                        <label className="text-white/80 text-sm">Description</label>
                        <textarea
                          value={createEventForm.description}
                          onChange={(e) => setCreateEventForm({...createEventForm, description: e.target.value})}
                          placeholder="Event description"
                          className="w-full p-2 rounded bg-white/10 border border-white/20 text-white"
                          rows={3}
                          data-testid="event-description-input"
                        />
                      </div>
                      
                      <div className="md:col-span-2">
                        <label className="text-white/80 text-sm">Image URL</label>
                        <Input
                          value={createEventForm.image_url}
                          onChange={(e) => setCreateEventForm({...createEventForm, image_url: e.target.value})}
                          placeholder="https://example.com/image.jpg (optional)"
                          className="bg-white/10 border-white/20 text-white"
                        />
                      </div>
                    </div>
                    
                    <div className="mt-6">
                      <Button
                        onClick={handleCreateEvent}
                        disabled={createEventMutation.isPending}
                        className="bg-green-500 hover:bg-green-600"
                        data-testid="create-event-submit"
                      >
                        {createEventMutation.isPending ? (
                          <>
                            <div className="animate-spin w-4 h-4 border-2 border-white border-t-transparent rounded-full mr-2"></div>
                            Creating...
                          </>
                        ) : (
                          <>
                            <Calendar className="w-4 h-4 mr-2" />
                            Create Event
                          </>
                        )}
                      </Button>
                    </div>
                  </div>
                </CardContent>
              )}
            </Card>
          </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.2 }}
            >
              <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-white">Total Users</CardTitle>
                  <Users className="h-4 w-4 text-blue-400" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-white" data-testid="admin-total-users">
                    {usersLoading ? '...' : users?.length || 0}
                  </div>
                  <p className="text-xs text-white/70 mt-1">
                    Active: {users?.filter((u: any) => u.is_active).length || 0}
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.3 }}
            >
              <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-white">Total Events</CardTitle>
                  <Calendar className="h-4 w-4 text-green-400" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-white" data-testid="admin-total-events">
                    {eventsLoading ? '...' : events?.length || 0}
                  </div>
                  <p className="text-xs text-white/70 mt-1">
                    Active: {events?.filter((e: any) => e.is_active).length || 0}
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.4 }}
            >
              <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-white">Daily Performance</CardTitle>
                  <TrendingUp className="h-4 w-4 text-purple-400" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-white" data-testid="admin-daily-sales">
                    {analyticsLoading ? '...' : analytics?.dailySales?.length || 0}
                  </div>
                  <p className="text-xs text-white/70 mt-1">
                    Sales this week
                  </p>
                </CardContent>
              </Card>
            </motion.div>
          </div>

          {/* Pending Organizer Approvals */}
          <Card className="mb-8 bg-white/10 border-white/20 backdrop-blur-sm">
            <CardHeader>
              <CardTitle className="text-white flex items-center">
                <UserCheck className="h-5 w-5 mr-2" />
                Pending Organizer Approvals
                {pendingOrganizers.length > 0 && (
                  <Badge variant="destructive" className="ml-2">
                    {pendingOrganizers.length}
                  </Badge>
                )}
              </CardTitle>
              <CardDescription className="text-white/80">
                Review and approve organizer applications
              </CardDescription>
            </CardHeader>
            <CardContent>
              {pendingOrganizers.length === 0 ? (
                <p className="text-white/70 text-center py-8">No pending organizer applications</p>
              ) : (
                <div className="space-y-4">
                  {pendingOrganizers.map((user: any) => (
                    <div key={user.id} className="flex items-center justify-between p-4 bg-white/5 rounded-lg">
                      <div className="flex items-center space-x-4">
                        <div className="w-10 h-10 bg-gradient-to-r from-purple-400 to-pink-400 rounded-full flex items-center justify-center">
                          <Building2 className="w-5 h-5 text-white" />
                        </div>
                        <div>
                          <p className="text-white font-medium">
                            {user.first_name} {user.last_name}
                          </p>
                          <p className="text-white/70 text-sm">{user.email}</p>
                          <p className="text-white/50 text-xs">
                            Applied: {new Date(user.created_at).toLocaleDateString()}
                          </p>
                        </div>
                      </div>
                      <div className="flex space-x-2">
                        <Button
                          size="sm"
                          className="bg-green-500 hover:bg-green-600"
                          onClick={() => approveOrganizerMutation.mutate({ userId: user.id, approve: true })}
                          disabled={approveOrganizerMutation.isPending}
                          data-testid={`approve-organizer-${user.id}`}
                        >
                          <CheckCircle className="w-4 h-4 mr-1" />
                          Approve
                        </Button>
                        <Button
                          size="sm"
                          variant="destructive"
                          onClick={() => approveOrganizerMutation.mutate({ userId: user.id, approve: false })}
                          disabled={approveOrganizerMutation.isPending}
                          data-testid={`reject-organizer-${user.id}`}
                        >
                          <XCircle className="w-4 h-4 mr-1" />
                          Reject
                        </Button>
                      </div>
                    </div>
                  ))}
                </div>
              )}
            </CardContent>
          </Card>

          {/* Pending Event Approvals */}
          <Card className="mb-8 bg-white/10 border-white/20 backdrop-blur-sm">
            <CardHeader>
              <CardTitle className="text-white flex items-center">
                <Calendar className="h-5 w-5 mr-2" />
                Pending Event Approvals
                {pendingEvents.length > 0 && (
                  <Badge variant="destructive" className="ml-2">
                    {pendingEvents.length}
                  </Badge>
                )}
              </CardTitle>
              <CardDescription className="text-white/80">
                Review and approve event submissions
              </CardDescription>
            </CardHeader>
            <CardContent>
              {pendingEvents.length === 0 ? (
                <p className="text-white/70 text-center py-8">No pending event approvals</p>
              ) : (
                <div className="space-y-4">
                  {pendingEvents.map((event: any) => (
                    <div key={event.id} className="flex items-center justify-between p-4 bg-white/5 rounded-lg">
                      <div className="flex items-center space-x-4">
                        <img
                          src={event.image_url}
                          alt={event.name}
                          className="w-16 h-16 object-cover rounded-lg"
                        />
                        <div>
                          <p className="text-white font-medium">{event.name}</p>
                          <p className="text-white/70 text-sm">{event.venue}</p>
                          <p className="text-white/50 text-xs">
                            {new Date(event.date_from).toLocaleDateString()}
                          </p>
                        </div>
                      </div>
                      <div className="flex space-x-2">
                        <Button
                          size="sm"
                          className="bg-green-500 hover:bg-green-600"
                          onClick={() => eventApprovalMutation.mutate({ eventId: event.id, action: 'approve' })}
                          disabled={eventApprovalMutation.isPending}
                          data-testid={`approve-event-${event.id}`}
                        >
                          <CheckCircle className="w-4 h-4 mr-1" />
                          Approve
                        </Button>
                        <Button
                          size="sm"
                          variant="destructive"
                          onClick={() => eventApprovalMutation.mutate({ eventId: event.id, action: 'reject' })}
                          disabled={eventApprovalMutation.isPending}
                          data-testid={`reject-event-${event.id}`}
                        >
                          <XCircle className="w-4 h-4 mr-1" />
                          Reject
                        </Button>
                      </div>
                    </div>
                  ))}
                </div>
              )}
            </CardContent>
          </Card>

          {/* All Users Management */}
          <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
            <CardHeader>
              <CardTitle className="text-white flex items-center">
                <Users className="h-5 w-5 mr-2" />
                User Management
              </CardTitle>
              <CardDescription className="text-white/80">
                View and manage all platform users
              </CardDescription>
            </CardHeader>
            <CardContent>
              <div className="mb-4">
                <div className="relative">
                  <Search className="absolute left-3 top-3 h-4 w-4 text-white/40" />
                  <Input
                    placeholder="Search users by name or email..."
                    value={userSearchQuery}
                    onChange={(e) => setUserSearchQuery(e.target.value)}
                    className="pl-10 bg-white/20 border-white/30 text-white placeholder-white/70"
                    data-testid="users-search"
                  />
                </div>
              </div>

              {usersLoading ? (
                <div className="space-y-4">
                  {[1, 2, 3].map((i) => (
                    <div key={i} className="animate-pulse flex items-center space-x-4">
                      <div className="h-10 w-10 bg-white/20 rounded-full" />
                      <div className="flex-1">
                        <div className="h-4 bg-white/20 rounded w-1/2 mb-2" />
                        <div className="h-3 bg-white/20 rounded w-1/4" />
                      </div>
                    </div>
                  ))}
                </div>
              ) : (
                <div className="space-y-2 max-h-96 overflow-y-auto">
                  {filteredUsers.map((user: any) => (
                    <div key={user.id} className="flex items-center justify-between p-3 bg-white/5 rounded-lg">
                      <div className="flex items-center space-x-3">
                        <div className="w-8 h-8 bg-gradient-to-r from-blue-400 to-purple-400 rounded-full flex items-center justify-center">
                          {user.role === 'admin' ? (
                            <Crown className="w-4 h-4 text-white" />
                          ) : (
                            <span className="text-white text-sm font-bold">
                              {user.first_name[0]}{user.last_name[0]}
                            </span>
                          )}
                        </div>
                        <div>
                          <p className="text-white text-sm font-medium">
                            {user.first_name} {user.last_name}
                          </p>
                          <p className="text-white/60 text-xs">{user.email}</p>
                        </div>
                      </div>
                      <div className="flex items-center space-x-2">
                        <Badge 
                          variant={user.role === 'admin' ? 'default' : user.role === 'organizer' ? 'secondary' : 'outline'}
                          className="text-xs"
                        >
                          {user.role}
                        </Badge>
                        <Badge 
                          variant={user.is_active ? 'default' : 'destructive'}
                          className="text-xs"
                        >
                          {user.is_active ? 'Active' : 'Pending'}
                        </Badge>
                      </div>
                    </div>
                  ))}
                </div>
              )}
            </CardContent>
          </Card>
        </motion.div>
      </div>
    </div>
  );
}
```

### Complete Organizer Dashboard
Create `client/src/pages/OrganizerDashboard.tsx`:
```typescript
import React, { useState } from 'react';
import { motion } from 'framer-motion';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Badge } from '@/components/ui/badge';
import { useToast } from '@/hooks/use-toast';
import { Link } from 'wouter';
import { 
  DollarSign, 
  Ticket, 
  Calendar, 
  TrendingUp,
  Search,
  Eye,
  Plus,
  Banknote,
  Scan,
  Settings,
  Building2,
  CheckCircle,
  XCircle
} from 'lucide-react';

export function OrganizerDashboard() {
  const [eventSearchQuery, setEventSearchQuery] = useState('');
  const [showEarningsModal, setShowEarningsModal] = useState(false);
  const { toast } = useToast();
  const queryClient = useQueryClient();

  // Fetch organizer analytics
  const { data: analytics, isLoading: analyticsLoading } = useQuery({
    queryKey: ['/api/organizer/analytics'],
    queryFn: async () => {
      const token = localStorage.getItem('auth_token');
      const response = await fetch('/api/organizer/analytics', {
        headers: { 'Authorization': `Bearer ${token}` },
      });
      if (!response.ok) throw new Error('Failed to fetch analytics');
      return response.json();
    },
  });

  // Fetch organizer events
  const { data: events, isLoading: eventsLoading } = useQuery({
    queryKey: ['/api/organizer/events'],
    queryFn: async () => {
      const token = localStorage.getItem('auth_token');
      const response = await fetch('/api/organizer/events', {
        headers: { 'Authorization': `Bearer ${token}` },
      });
      if (!response.ok) throw new Error('Failed to fetch events');
      return response.json();
    },
  });

  // Fetch earnings breakdown
  const { data: earnings, isLoading: earningsLoading } = useQuery({
    queryKey: ['/api/organizer/earnings'],
    queryFn: async () => {
      const token = localStorage.getItem('auth_token');
      const response = await fetch('/api/organizer/earnings', {
        headers: { 'Authorization': `Bearer ${token}` },
      });
      if (!response.ok) throw new Error('Failed to fetch earnings');
      return response.json();
    },
  });

  // Fetch scan analytics
  const { data: scanAnalytics, isLoading: scanAnalyticsLoading } = useQuery({
    queryKey: ['/api/scanner/analytics'],
    queryFn: async () => {
      const token = localStorage.getItem('auth_token');
      const response = await fetch('/api/scanner/analytics', {
        headers: { 'Authorization': `Bearer ${token}` },
      });
      if (!response.ok) throw new Error('Failed to fetch scan analytics');
      return response.json();
    },
  });

  // Sync tickets to CodeREADr
  const syncTicketsMutation = useMutation({
    mutationFn: async (eventId: string) => {
      const token = localStorage.getItem('auth_token');
      const response = await fetch('/api/codereadr/sync-tickets', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`,
        },
        body: JSON.stringify({ eventId }),
      });
      if (!response.ok) throw new Error('Failed to sync tickets');
      return response.json();
    },
    onSuccess: () => {
      toast({
        title: 'Success!',
        description: 'Tickets synced to CodeREADr successfully',
      });
    },
  });

  // Test CodeREADr API
  const testCodeREADrAPI = async () => {
    try {
      const token = localStorage.getItem('auth_token');
      const response = await fetch('/api/codereadr/test', {
        method: 'POST',
        headers: { 'Authorization': `Bearer ${token}` },
      });
      const result = await response.json();

      if (response.ok) {
        toast({
          title: 'API Test Successful!',
          description: 'CodeREADr connection is working properly',
        });
      } else {
        toast({
          title: 'API Test Failed',
          description: result.error || 'Connection failed',
          variant: 'destructive',
        });
      }
    } catch (error) {
      toast({
        title: 'API Test Failed',
        description: 'Network error occurred',
        variant: 'destructive',
      });
    }
  };

  // Filter events by search query
  const filteredEvents = events?.filter((event: any) =>
    event.name.toLowerCase().includes(eventSearchQuery.toLowerCase()) ||
    event.venue.toLowerCase().includes(eventSearchQuery.toLowerCase()) ||
    event.description?.toLowerCase().includes(eventSearchQuery.toLowerCase())
  ) || [];

  return (
    <div className="min-h-screen p-6 bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500">
      <div className="max-w-7xl mx-auto">
        <motion.div
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          transition={{ duration: 0.6 }}
        >
          {/* Header */}
          <div className="mb-8">
            <h1 className="text-3xl font-bold text-white mb-2">
              Organizer Dashboard 🎭
            </h1>
            <p className="text-lg text-white/80">
              Manage your events and track sales performance
            </p>
          </div>

          {/* Key Stats */}
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-6 gap-4 mb-8">
            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.1 }}
            >
              <Card 
                className="bg-gradient-to-br from-blue-50 to-blue-100 border-blue-200 cursor-pointer hover:shadow-lg transition-shadow"
                onClick={() => setShowEarningsModal(true)}
                data-testid="organizer-earnings-card"
              >
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-blue-800">Your Earnings</CardTitle>
                  <Banknote className="h-4 w-4 text-blue-600" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-blue-900" data-testid="organizer-total-earnings">
                    KES {earningsLoading ? '...' : (earnings?.totalEarnings || 0).toLocaleString()}
                  </div>
                  <p className="text-xs text-blue-600 mt-1">
                    Click to view breakdown
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.2 }}
            >
              <Card className="bg-gradient-to-br from-green-50 to-green-100 border-green-200">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-green-800">Tickets Sold</CardTitle>
                  <Ticket className="h-4 w-4 text-green-600" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-green-900" data-testid="organizer-tickets-sold">
                    {analyticsLoading ? '...' : analytics?.ticketsSold || 0}
                  </div>
                  <p className="text-xs text-green-600 mt-1">
                    Across all events
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.3 }}
            >
              <Card className="bg-gradient-to-br from-purple-50 to-purple-100 border-purple-200">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-purple-800">Active Events</CardTitle>
                  <Calendar className="h-4 w-4 text-purple-600" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-purple-900" data-testid="organizer-active-events">
                    {analyticsLoading ? '...' : analytics?.activeEvents || 0}
                  </div>
                  <p className="text-xs text-purple-600 mt-1">
                    Currently managing
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.4 }}
            >
              <Card className="bg-gradient-to-br from-orange-50 to-orange-100 border-orange-200">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-orange-800">Total Sales</CardTitle>
                  <DollarSign className="h-4 w-4 text-orange-600" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-orange-900" data-testid="organizer-total-sales">
                    KES {analyticsLoading ? '...' : (analytics?.totalRevenue || 0).toLocaleString()}
                  </div>
                  <p className="text-xs text-orange-600 mt-1">
                    Total revenue generated
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.5 }}
            >
              <Card className="bg-gradient-to-br from-green-50 to-green-100 border-green-200 cursor-pointer hover:shadow-lg transition-all duration-200 hover:scale-105">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-green-800">Valid Scans Today</CardTitle>
                  <CheckCircle className="h-4 w-4 text-green-600" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-green-900" data-testid="organizer-valid-scans">
                    {scanAnalyticsLoading ? '...' : scanAnalytics?.validScansToday || 0}
                  </div>
                  <p className="text-xs text-green-600 mt-1">
                    Click to view history
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.6 }}
            >
              <Card className="bg-gradient-to-br from-red-50 to-red-100 border-red-200 cursor-pointer hover:shadow-lg transition-all duration-200 hover:scale-105">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-red-800">Invalid Scans Today</CardTitle>
                  <XCircle className="h-4 w-4 text-red-600" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-red-900" data-testid="organizer-invalid-scans">
                    {scanAnalyticsLoading ? '...' : scanAnalytics?.invalidScansToday || 0}
                  </div>
                  <p className="text-xs text-red-600 mt-1">
                    Click to view history
                  </p>
                </CardContent>
              </Card>
            </motion.div>
          </div>

          {/* Management Cards */}
          <div className="grid grid-cols-1 lg:grid-cols-3 gap-6 mb-8">
            {/* Financial Management */}
            <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
              <CardHeader>
                <CardTitle className="flex items-center text-white">
                  <Banknote className="h-5 w-5 mr-2" />
                  Financial Management
                </CardTitle>
                <CardDescription className="text-white/80">
                  Earnings and payment requests
                </CardDescription>
              </CardHeader>
              <CardContent className="space-y-3">
                <div className="text-center p-4 bg-white/10 rounded-lg">
                  <p className="text-white/80 text-sm">Available to withdraw</p>
                  <p className="text-2xl font-bold text-white" data-testid="organizer-available-earnings">
                    KES {earningsLoading ? '...' : (earnings?.availableEarnings || 0).toLocaleString()}
                  </p>
                </div>
                <Link href="/organizer/payment-request">
                  <Button className="w-full bg-gradient-to-r from-green-600 to-blue-600 hover:from-green-700 hover:to-blue-700" data-testid="organizer-request-payment">
                    <Banknote className="h-4 w-4 mr-2" />
                    Request Payment
                  </Button>
                </Link>
                <p className="text-xs text-white/60 text-center">
                  Withdraw via bank transfer or mobile money
                </p>
              </CardContent>
            </Card>

            {/* Box Office */}
            <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
              <CardHeader>
                <CardTitle className="flex items-center text-white">
                  <Building2 className="h-5 w-5 mr-2" />
                  Box Office POS
                </CardTitle>
                <CardDescription className="text-white/80">
                  Sell tickets directly at events
                </CardDescription>
              </CardHeader>
              <CardContent className="space-y-3">
                <Link href="/box-office">
                  <Button className="w-full bg-gradient-to-r from-orange-600 to-red-600 hover:from-orange-700 hover:to-red-700" data-testid="organizer-box-office">
                    <Ticket className="h-4 w-4 mr-2" />
                    Open Box Office
                  </Button>
                </Link>
                <p className="text-xs text-white/60 text-center">
                  Instant ticket sales with QR code generation
                </p>
              </CardContent>
            </Card>

            {/* Professional Scanning */}
            <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
              <CardHeader>
                <CardTitle className="flex items-center text-white">
                  <Scan className="h-5 w-5 mr-2" />
                  Professional Scanning
                </CardTitle>
                <CardDescription className="text-white/80">
                  CodeREADr integration for events
                </CardDescription>
              </CardHeader>
              <CardContent className="space-y-3">
                <Button 
                  variant="default" 
                  className="w-full justify-start bg-gradient-to-r from-purple-600 to-indigo-600 hover:from-purple-700 hover:to-indigo-700" 
                  data-testid="organizer-setup-scanning"
                  onClick={() => window.open('https://www.codereadr.com', '_blank')}
                >
                  <Building2 className="h-4 w-4 mr-2" />
                  Setup CodeREADr
                </Button>
                <Button 
                  variant="outline" 
                  className="w-full justify-start bg-white/10 border-white/30 text-white hover:bg-white/20"
                  data-testid="organizer-test-api"
                  onClick={testCodeREADrAPI}
                >
                  <Settings className="h-4 w-4 mr-2" />
                  Test API
                </Button>
                <Link href="/scanner">
                  <Button 
                    variant="outline" 
                    className="w-full justify-start bg-white/10 border-white/30 text-white hover:bg-white/20"
                    data-testid="organizer-view-scanner"
                  >
                    <Scan className="h-4 w-4 mr-2" />
                    PWA Scanner
                  </Button>
                </Link>
                <p className="text-xs text-white/60 text-center">
                  Professional ticket validation at events
                </p>
              </CardContent>
            </Card>
          </div>

          {/* Your Events */}
          <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
            <CardHeader>
              <CardTitle className="flex items-center text-white">
                <Eye className="h-5 w-5 mr-2" />
                Your Events
              </CardTitle>
              <CardDescription className="text-white/80">
                Events you're managing
              </CardDescription>
            </CardHeader>
            <CardContent>
              {/* Events Search Bar */}
              <div className="mb-4">
                <div className="relative">
                  <Search className="absolute left-3 top-3 h-4 w-4 text-gray-400" />
                  <Input
                    placeholder="Search events by name, venue, or description..."
                    value={eventSearchQuery}
                    onChange={(e) => setEventSearchQuery(e.target.value)}
                    className="pl-10 bg-white/20 border-white/30 text-white placeholder-white/70"
                    data-testid="events-search"
                  />
                </div>
              </div>

              {eventsLoading ? (
                <div className="space-y-4">
                  {[1, 2, 3].map((i) => (
                    <div key={i} className="animate-pulse flex items-center space-x-4">
                      <div className="h-16 w-16 bg-white/20 rounded-lg" />
                      <div className="flex-1">
                        <div className="h-4 bg-white/20 rounded w-1/2 mb-2" />
                        <div className="h-3 bg-white/20 rounded w-1/4 mb-1" />
                        <div className="h-3 bg-white/20 rounded w-1/3" />
                      </div>
                    </div>
                  ))}
                </div>
              ) : events && events.length > 0 ? (
                <div className="space-y-4">
                  {filteredEvents.slice(0, 5).map((event: any) => (
                    <div 
                      key={event.id} 
                      className="flex items-center space-x-4 p-4 border border-white/20 rounded-lg hover:bg-white/5 transition-colors cursor-pointer"
                      data-testid={`event-card-${event.id}`}
                    >
                      <img 
                        src={event.image_url} 
                        alt={event.name}
                        className="h-16 w-16 object-cover rounded-lg"
                      />
                      <div className="flex-1">
                        <h3 className="text-white font-medium">{event.name}</h3>
                        <p className="text-white/70 text-sm">{event.venue}</p>
                        <div className="flex items-center space-x-4 mt-1">
                          <p className="text-white/60 text-xs">
                            {new Date(event.date_from).toLocaleDateString()}
                          </p>
                          <Badge variant={event.is_active ? "default" : "secondary"} className="text-xs">
                            {event.is_active ? "Active" : "Pending"}
                          </Badge>
                          <p className="text-white/60 text-xs">
                            {event.ticket_count || 0} tickets sold
                          </p>
                        </div>
                      </div>
                      <div className="flex items-center space-x-2">
                        <Button
                          size="sm"
                          variant="outline"
                          className="bg-white/10 border-white/30 text-white hover:bg-white/20"
                          onClick={() => syncTicketsMutation.mutate(event.id)}
                          disabled={syncTicketsMutation.isPending}
                          data-testid={`sync-tickets-${event.id}`}
                        >
                          <Scan className="h-4 w-4 mr-1" />
                          Sync
                        </Button>
                        <Link href={`/events/${event.slug}`}>
                          <Button size="sm" className="bg-gradient-to-r from-pink-500 to-yellow-500">
                            <Eye className="h-4 w-4 mr-1" />
                            View
                          </Button>
                        </Link>
                      </div>
                    </div>
                  ))}
                  
                  {filteredEvents.length > 5 && (
                    <div className="text-center pt-4">
                      <Link href="/organizer/all-events">
                        <Button variant="outline" className="bg-white/10 border-white/30 text-white hover:bg-white/20">
                          View All Events ({filteredEvents.length})
                        </Button>
                      </Link>
                    </div>
                  )}
                </div>
              ) : (
                <div className="text-center py-12">
                  <Calendar className="h-12 w-12 text-white/40 mx-auto mb-4" />
                  <p className="text-white/70 mb-4">No events found</p>
                  <Link href="/create-event">
                    <Button className="bg-gradient-to-r from-pink-500 to-yellow-500 hover:from-pink-600 hover:to-yellow-600">
                      <Plus className="h-4 w-4 mr-2" />
                      Create Your First Event
                    </Button>
                  </Link>
                </div>
              )}
            </CardContent>
          </Card>
        </motion.div>
      </div>
    </div>
  );
}
```

### High-Performance PWA Scanner Dashboard for 100,000+ Tickets
Create `client/src/pages/ScannerDashboard.tsx`:
```typescript
import React, { useState, useEffect, useRef, useCallback } from 'react';
import { motion } from 'framer-motion';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Badge } from '@/components/ui/badge';
import { useToast } from '@/hooks/use-toast';
import { Link } from 'wouter';
import { 
  Camera, 
  Scan, 
  CheckCircle, 
  XCircle, 
  Search,
  Clock,
  User,
  Calendar,
  Smartphone,
  AlertTriangle,
  ArrowLeft,
  Wifi,
  WifiOff,
  Database,
  Zap
} from 'lucide-react';

// High-performance scanning optimizations
const SCAN_DEBOUNCE_MS = 100;
const MAX_SCAN_HISTORY = 50;
const BATCH_SYNC_SIZE = 100;

export function ScannerDashboard() {
  const [isScanning, setIsScanning] = useState(false);
  const [manualInput, setManualInput] = useState('');
  const [lastScanResult, setLastScanResult] = useState<any>(null);
  const [scanHistory, setScanHistory] = useState<any[]>([]);
  const [selectedEvent, setSelectedEvent] = useState<string>('');
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  const [offlineScans, setOfflineScans] = useState<any[]>([]);
  const [scanCount, setScanCount] = useState(0);
  const videoRef = useRef<HTMLVideoElement>(null);
  const scanTimeoutRef = useRef<NodeJS.Timeout>();
  const { toast } = useToast();
  const queryClient = useQueryClient();

  // Fetch scanner analytics
  const { data: analytics, isLoading: analyticsLoading } = useQuery({
    queryKey: ['/api/scanner/analytics'],
    queryFn: async () => {
      const token = localStorage.getItem('auth_token');
      const response = await fetch('/api/scanner/analytics', {
        headers: { 'Authorization': `Bearer ${token}` },
      });
      if (!response.ok) throw new Error('Failed to fetch analytics');
      return response.json();
    },
  });

  // Fetch scan history
  const { data: scanHistoryData, isLoading: historyLoading } = useQuery({
    queryKey: ['/api/scanner/scan-history'],
    queryFn: async () => {
      const token = localStorage.getItem('auth_token');
      const response = await fetch('/api/scanner/scan-history', {
        headers: { 'Authorization': `Bearer ${token}` },
      });
      if (!response.ok) throw new Error('Failed to fetch scan history');
      return response.json();
    },
  });

  // Fetch available events for scanning
  const { data: events } = useQuery({
    queryKey: ['/api/scanner/events'],
    queryFn: async () => {
      const token = localStorage.getItem('auth_token');
      const response = await fetch('/api/scanner/events', {
        headers: { 'Authorization': `Bearer ${token}` },
      });
      if (!response.ok) throw new Error('Failed to fetch events');
      return response.json();
    },
  });

  // Network status monitoring for PWA
  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);
    
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  // Offline scan synchronization
  useEffect(() => {
    if (isOnline && offlineScans.length > 0) {
      syncOfflineScans();
    }
  }, [isOnline, offlineScans]);

  // High-performance ticket validation with offline support
  const validateTicket = useCallback(async (ticketUID: string) => {
    if (scanTimeoutRef.current) {
      clearTimeout(scanTimeoutRef.current);
    }

    try {
      setScanCount(prev => prev + 1);
      const timestamp = new Date();
      
      if (!isOnline) {
        // Store for offline sync
        const offlineScan = {
          uid: ticketUID,
          event_id: selectedEvent,
          scanned_at: timestamp,
          device: 'PWA Scanner (Offline)',
          status: 'pending_sync'
        };
        
        setOfflineScans(prev => [...prev, offlineScan]);
        
        toast({
          title: '📡 Offline Mode',
          description: 'Scan saved. Will sync when online.',
          duration: 2000,
        });
        
        return { valid: true, reason: 'offline_mode', offline: true };
      }

      const token = localStorage.getItem('auth_token');
      const response = await fetch('/api/validate', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`,
        },
        body: JSON.stringify({
          uid: ticketUID,
          event_id: selectedEvent,
          device: 'PWA Scanner',
          scan_timestamp: timestamp.toISOString()
        }),
      });

      const result = await response.json();
      
      // Optimized result handling
      const scanResult = {
        uid: ticketUID,
        valid: result.valid,
        reason: result.reason,
        buyer_name: result.buyer_name,
        event_name: result.event_name,
        scanned_at: timestamp,
        scan_count: scanCount,
      };
      
      setLastScanResult(scanResult);

      // High-performance history management
      setScanHistory(prev => {
        const newHistory = [scanResult, ...prev];
        return newHistory.slice(0, MAX_SCAN_HISTORY); // Limit history for performance
      });

      // Optimized toast notifications
      if (result.valid) {
        toast({
          title: '✅ Valid Ticket',
          description: `Welcome ${result.buyer_name || 'Guest'}!`,
          duration: 2000,
        });
      } else {
        toast({
          title: '❌ Invalid Ticket',
          description: result.reason || 'Ticket validation failed',
          variant: 'destructive',
          duration: 2000,
        });
      }

      // Auto-sync to CodeREADr if configured
      if (result.valid && selectedEvent) {
        syncToCodeREADr(ticketUID, selectedEvent);
      }

      return result;
    } catch (error) {
      console.error('Validation error:', error);
      
      if (!isOnline) {
        // Store failed scan for retry
        setOfflineScans(prev => [...prev, {
          uid: ticketUID,
          event_id: selectedEvent,
          scanned_at: new Date(),
          status: 'failed_validation',
          error: error.message
        }]);
      }
      
      toast({
        title: 'Validation Error',
        description: 'Failed to validate ticket',
        variant: 'destructive',
        duration: 2000,
      });
      return { valid: false, reason: 'network_error' };
    }
  }, [isOnline, selectedEvent, scanCount, toast]);

  // Sync offline scans when back online
  const syncOfflineScans = async () => {
    if (offlineScans.length === 0) return;

    try {
      const token = localStorage.getItem('auth_token');
      
      // Process in batches for performance
      for (let i = 0; i < offlineScans.length; i += BATCH_SYNC_SIZE) {
        const batch = offlineScans.slice(i, i + BATCH_SYNC_SIZE);
        
        await fetch('/api/scanner/sync-offline', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`,
          },
          body: JSON.stringify({ scans: batch }),
        });
      }

      setOfflineScans([]);
      toast({
        title: '🔄 Sync Complete',
        description: `${offlineScans.length} offline scans synchronized`,
      });
    } catch (error) {
      console.error('Offline sync error:', error);
    }
  };

  // CodeREADr integration for professional scanning
  const syncToCodeREADr = async (ticketUID: string, eventId: string) => {
    try {
      const token = localStorage.getItem('auth_token');
      await fetch('/api/codereadr/record-scan', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`,
        },
        body: JSON.stringify({
          ticket_uid: ticketUID,
          event_id: eventId,
          scanner_device: 'PWA Scanner',
          scan_timestamp: new Date().toISOString()
        }),
      });
    } catch (error) {
      console.error('CodeREADr sync error:', error);
    }
  };

  // Start camera scanning
  const startScanning = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ 
        video: { facingMode: 'environment' } 
      });
      
      if (videoRef.current) {
        videoRef.current.srcObject = stream;
        setIsScanning(true);
      }
    } catch (error) {
      console.error('Camera access error:', error);
      toast({
        title: 'Camera Access Denied',
        description: 'Please allow camera access to scan QR codes',
        variant: 'destructive',
      });
    }
  };

  // Stop camera scanning
  const stopScanning = () => {
    if (videoRef.current?.srcObject) {
      const stream = videoRef.current.srcObject as MediaStream;
      stream.getTracks().forEach(track => track.stop());
      videoRef.current.srcObject = null;
    }
    setIsScanning(false);
  };

  // Manual ticket validation
  const handleManualValidation = async () => {
    if (!manualInput.trim()) {
      toast({
        title: 'Invalid Input',
        description: 'Please enter a ticket UID',
        variant: 'destructive',
      });
      return;
    }

    await validateTicket(manualInput.trim());
    setManualInput('');
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500">
      {/* Mobile-friendly header with back button */}
      <div className="sticky top-0 z-50 bg-white/10 backdrop-blur-md border-b border-white/20 px-4 py-3">
        <div className="flex items-center justify-between max-w-7xl mx-auto">
          <div className="flex items-center space-x-3">
            <Link href="/organizer">
              <Button size="sm" variant="ghost" className="text-white hover:bg-white/20">
                <ArrowLeft className="w-4 h-4 mr-1" />
                Back
              </Button>
            </Link>
            <div>
              <h1 className="text-xl font-bold text-white">PWA Scanner</h1>
              <div className="flex items-center text-xs text-white/70">
                {isOnline ? (
                  <><Wifi className="w-3 h-3 mr-1" />Online</>
                ) : (
                  <><WifiOff className="w-3 h-3 mr-1" />Offline</>
                )}
                {offlineScans.length > 0 && (
                  <span className="ml-2 bg-orange-500 text-white px-2 py-0.5 rounded-full text-xs">
                    {offlineScans.length} pending sync
                  </span>
                )}
              </div>
            </div>
          </div>
          <div className="flex items-center space-x-2">
            <Badge variant="outline" className="bg-white/10 border-white/30 text-white">
              <Zap className="w-3 h-3 mr-1" />
              {scanCount} scans
            </Badge>
            <Badge variant="outline" className="bg-white/10 border-white/30 text-white">
              <Database className="w-3 h-3 mr-1" />
              {MAX_SCAN_HISTORY} max
            </Badge>
          </div>
        </div>
      </div>

      <div className="p-4 md:p-6 max-w-7xl mx-auto">
        <motion.div
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          transition={{ duration: 0.6 }}
        >
          {/* Performance Stats */}
          <div className="mb-6">
            <div className="grid grid-cols-2 md:grid-cols-4 gap-3">
              <div className="bg-white/10 backdrop-blur-sm rounded-lg p-3 text-center">
                <p className="text-white/70 text-xs">Scan Speed</p>
                <p className="text-white font-bold">&lt;{SCAN_DEBOUNCE_MS}ms</p>
              </div>
              <div className="bg-white/10 backdrop-blur-sm rounded-lg p-3 text-center">
                <p className="text-white/70 text-xs">History Limit</p>
                <p className="text-white font-bold">{MAX_SCAN_HISTORY}</p>
              </div>
              <div className="bg-white/10 backdrop-blur-sm rounded-lg p-3 text-center">
                <p className="text-white/70 text-xs">Batch Size</p>
                <p className="text-white font-bold">{BATCH_SYNC_SIZE}</p>
              </div>
              <div className="bg-white/10 backdrop-blur-sm rounded-lg p-3 text-center">
                <p className="text-white/70 text-xs">Status</p>
                <p className="text-white font-bold">{isOnline ? 'Online' : 'Offline'}</p>
              </div>
            </div>
          </div>

          {/* Analytics Cards */}
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-8">
            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.1 }}
            >
              <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-white">Valid Scans Today</CardTitle>
                  <CheckCircle className="h-4 w-4 text-green-400" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-white" data-testid="scanner-valid-scans">
                    {analyticsLoading ? '...' : analytics?.validScansToday || 0}
                  </div>
                  <p className="text-xs text-white/70 mt-1">
                    Successful validations
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.2 }}
            >
              <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-white">Invalid Scans</CardTitle>
                  <XCircle className="h-4 w-4 text-red-400" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-white" data-testid="scanner-invalid-scans">
                    {analyticsLoading ? '...' : analytics?.invalidScansToday || 0}
                  </div>
                  <p className="text-xs text-white/70 mt-1">
                    Failed validations
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.3 }}
            >
              <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-white">Success Rate</CardTitle>
                  <Clock className="h-4 w-4 text-blue-400" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-white" data-testid="scanner-success-rate">
                    {analyticsLoading ? '...' : analytics?.successRate || 100}%
                  </div>
                  <p className="text-xs text-white/70 mt-1">
                    Validation accuracy
                  </p>
                </CardContent>
              </Card>
            </motion.div>

            <motion.div
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: 0.4 }}
            >
              <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
                  <CardTitle className="text-sm font-medium text-white">Total Scans</CardTitle>
                  <Scan className="h-4 w-4 text-purple-400" />
                </CardHeader>
                <CardContent>
                  <div className="text-2xl font-bold text-white" data-testid="scanner-total-scans">
                    {analyticsLoading ? '...' : analytics?.totalScans || 0}
                  </div>
                  <p className="text-xs text-white/70 mt-1">
                    All time scans
                  </p>
                </CardContent>
              </Card>
            </motion.div>
          </div>

          {/* Scanner Interface */}
          <div className="grid grid-cols-1 lg:grid-cols-2 gap-8 mb-8">
            {/* Camera Scanner */}
            <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
              <CardHeader>
                <CardTitle className="text-white flex items-center">
                  <Camera className="h-5 w-5 mr-2" />
                  QR Code Scanner
                </CardTitle>
                <CardDescription className="text-white/80">
                  Use camera to scan ticket QR codes
                </CardDescription>
              </CardHeader>
              <CardContent className="space-y-4">
                {/* Event Selection */}
                <div>
                  <label className="text-white text-sm font-medium mb-2 block">
                    Select Event to Scan For:
                  </label>
                  <select 
                    value={selectedEvent}
                    onChange={(e) => setSelectedEvent(e.target.value)}
                    className="w-full p-2 rounded bg-white/20 border border-white/30 text-white"
                    data-testid="scanner-event-select"
                  >
                    <option value="">Choose an event...</option>
                    {events?.map((event: any) => (
                      <option key={event.id} value={event.id} className="text-black">
                        {event.name} - {event.venue}
                      </option>
                    ))}
                  </select>
                </div>

                {/* Camera View */}
                <div className="relative">
                  <video
                    ref={videoRef}
                    autoPlay
                    playsInline
                    className="w-full h-64 bg-black rounded-lg object-cover"
                    data-testid="scanner-video"
                  />
                  {!isScanning && (
                    <div className="absolute inset-0 flex items-center justify-center bg-black/50 rounded-lg">
                      <div className="text-center">
                        <Camera className="h-12 w-12 text-white/50 mx-auto mb-2" />
                        <p className="text-white/70">Camera not active</p>
                      </div>
                    </div>
                  )}
                </div>

                {/* Scanner Controls */}
                <div className="flex space-x-2">
                  {!isScanning ? (
                    <Button 
                      onClick={startScanning}
                      className="flex-1 bg-green-500 hover:bg-green-600"
                      disabled={!selectedEvent}
                      data-testid="scanner-start"
                    >
                      <Camera className="h-4 w-4 mr-2" />
                      Start Scanning
                    </Button>
                  ) : (
                    <Button 
                      onClick={stopScanning}
                      variant="destructive"
                      className="flex-1"
                      data-testid="scanner-stop"
                    >
                      <XCircle className="h-4 w-4 mr-2" />
                      Stop Scanning
                    </Button>
                  )}
                </div>

                {!selectedEvent && (
                  <div className="flex items-center space-x-2 p-3 bg-yellow-500/20 border border-yellow-500/30 rounded-lg">
                    <AlertTriangle className="h-4 w-4 text-yellow-400" />
                    <p className="text-yellow-100 text-sm">
                      Please select an event before scanning
                    </p>
                  </div>
                )}
              </CardContent>
            </Card>

            {/* Manual Input & Last Result */}
            <div className="space-y-6">
              {/* Manual Input */}
              <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                <CardHeader>
                  <CardTitle className="text-white flex items-center">
                    <Smartphone className="h-5 w-5 mr-2" />
                    Manual Input
                  </CardTitle>
                  <CardDescription className="text-white/80">
                    Type ticket UID for validation
                  </CardDescription>
                </CardHeader>
                <CardContent className="space-y-4">
                  <div className="flex space-x-2">
                    <Input
                      placeholder="Enter ticket UID (e.g., TK-ABC123)"
                      value={manualInput}
                      onChange={(e) => setManualInput(e.target.value)}
                      className="bg-white/20 border-white/30 text-white placeholder-white/70"
                      data-testid="scanner-manual-input"
                      onKeyPress={(e) => {
                        if (e.key === 'Enter') {
                          handleManualValidation();
                        }
                      }}
                    />
                    <Button 
                      onClick={handleManualValidation}
                      disabled={!selectedEvent || !manualInput.trim()}
                      className="bg-blue-500 hover:bg-blue-600"
                      data-testid="scanner-manual-validate"
                    >
                      <Scan className="h-4 w-4" />
                    </Button>
                  </div>
                </CardContent>
              </Card>

              {/* Last Scan Result */}
              {lastScanResult && (
                <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
                  <CardHeader>
                    <CardTitle className="text-white flex items-center">
                      <Clock className="h-5 w-5 mr-2" />
                      Last Scan Result
                    </CardTitle>
                  </CardHeader>
                  <CardContent>
                    <div className={`p-4 rounded-lg ${
                      lastScanResult.valid 
                        ? 'bg-green-500/20 border border-green-500/30' 
                        : 'bg-red-500/20 border border-red-500/30'
                    }`}>
                      <div className="flex items-center space-x-2 mb-2">
                        {lastScanResult.valid ? (
                          <CheckCircle className="h-5 w-5 text-green-400" />
                        ) : (
                          <XCircle className="h-5 w-5 text-red-400" />
                        )}
                        <Badge 
                          variant={lastScanResult.valid ? "default" : "destructive"}
                          className="text-sm"
                        >
                          {lastScanResult.valid ? 'VALID' : 'INVALID'}
                        </Badge>
                      </div>
                      
                      <div className="space-y-1">
                        <p className="text-white font-medium">
                          Ticket: {lastScanResult.uid}
                        </p>
                        {lastScanResult.buyer_name && (
                          <p className="text-white/80">
                            Holder: {lastScanResult.buyer_name}
                          </p>
                        )}
                        {lastScanResult.reason && !lastScanResult.valid && (
                          <p className="text-red-300 text-sm">
                            Reason: {lastScanResult.reason.replace(/_/g, ' ').toLowerCase()}
                          </p>
                        )}
                        <p className="text-white/60 text-xs">
                          Scanned: {lastScanResult.scanned_at.toLocaleTimeString()}
                        </p>
                      </div>
                    </div>
                  </CardContent>
                </Card>
              )}
            </div>
          </div>

          {/* Recent Scan History */}
          <Card className="bg-white/10 border-white/20 backdrop-blur-sm">
            <CardHeader>
              <CardTitle className="text-white flex items-center">
                <Search className="h-5 w-5 mr-2" />
                Recent Scan History
              </CardTitle>
              <CardDescription className="text-white/80">
                Latest ticket validations
              </CardDescription>
            </CardHeader>
            <CardContent>
              {historyLoading ? (
                <div className="space-y-3">
                  {[1, 2, 3].map((i) => (
                    <div key={i} className="animate-pulse flex items-center space-x-4">
                      <div className="h-8 w-8 bg-white/20 rounded-full" />
                      <div className="flex-1">
                        <div className="h-4 bg-white/20 rounded w-1/3 mb-1" />
                        <div className="h-3 bg-white/20 rounded w-1/4" />
                      </div>
                    </div>
                  ))}
                </div>
              ) : (scanHistoryData && scanHistoryData.length > 0) || scanHistory.length > 0 ? (
                <div className="space-y-2 max-h-96 overflow-y-auto">
                  {(scanHistoryData || scanHistory).slice(0, 20).map((scan: any, index: number) => (
                    <div 
                      key={index} 
                      className="flex items-center justify-between p-3 bg-white/5 rounded-lg"
                    >
                      <div className="flex items-center space-x-3">
                        {scan.result === 'valid' || scan.valid ? (
                          <CheckCircle className="h-5 w-5 text-green-400" />
                        ) : (
                          <XCircle className="h-5 w-5 text-red-400" />
                        )}
                        <div>
                          <p className="text-white text-sm font-medium">
                            {scan.ticket_uid || scan.uid}
                          </p>
                          {scan.buyer_name && (
                            <p className="text-white/60 text-xs">
                              {scan.buyer_name}
                            </p>
                          )}
                        </div>
                      </div>
                      <div className="text-right">
                        <Badge 
                          variant={(scan.result === 'valid' || scan.valid) ? "default" : "destructive"}
                          className="text-xs mb-1"
                        >
                          {(scan.result === 'valid' || scan.valid) ? 'Valid' : 'Invalid'}
                        </Badge>
                        <p className="text-white/50 text-xs">
                          {new Date(scan.created_at || scan.scanned_at).toLocaleTimeString()}
                        </p>
                      </div>
                    </div>
                  ))}
                </div>
              ) : (
                <div className="text-center py-12">
                  <Scan className="h-12 w-12 text-white/40 mx-auto mb-4" />
                  <p className="text-white/70">No scan history yet</p>
                  <p className="text-white/50 text-sm">
                    Start scanning tickets to see history
                  </p>
                </div>
              )}
            </CardContent>
          </Card>
        </motion.div>
      </div>
    </div>
  );
}
```

### Event Details Page with Mobile-First Design
Create `client/src/pages/EventDetailsPage.tsx`:
```typescript
import React, { useState } from 'react';
import { motion } from 'framer-motion';
import { useQuery, useMutation } from '@tanstack/react-query';
import { useParams, Link } from 'wouter';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { useToast } from '@/hooks/use-toast';
import { 
  Calendar, 
  MapPin, 
  Clock, 
  Ticket, 
  Users, 
  Share2, 
  ArrowLeft,
  Star,
  Heart,
  ShoppingCart,
  CheckCircle
} from 'lucide-react';

export function EventDetailsPage() {
  const { slug } = useParams();
  const [selectedTicketType, setSelectedTicketType] = useState<any>(null);
  const [quantity, setQuantity] = useState(1);
  const { toast } = useToast();

  // Fetch event details with error handling
  const { data: event, isLoading, error } = useQuery({
    queryKey: ['/api/events', slug],
    queryFn: async () => {
      const response = await fetch(`/api/events/${slug}`);
      if (!response.ok) {
        if (response.status === 404) {
          throw new Error('Event not found');
        }
        throw new Error('Failed to fetch event');
      }
      return response.json();
    },
    retry: false, // Don't retry on 404
  });

  // Fetch ticket types for this event
  const { data: ticketTypes, isLoading: ticketTypesLoading } = useQuery({
    queryKey: ['/api/events', slug, 'ticket-types'],
    queryFn: async () => {
      const response = await fetch(`/api/events/${slug}/ticket-types`);
      if (!response.ok) return [];
      return response.json();
    },
    enabled: !!event,
  });

  // Handle ticket purchase
  const purchaseTicketMutation = useMutation({
    mutationFn: async (purchaseData: any) => {
      const response = await fetch('/api/purchase', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(purchaseData),
      });
      if (!response.ok) throw new Error('Purchase failed');
      return response.json();
    },
    onSuccess: (data) => {
      if (data.paymentUrl) {
        window.location.href = data.paymentUrl; // Redirect to Paystack
      } else {
        toast({
          title: 'Purchase Successful!',
          description: 'Your tickets have been confirmed',
        });
      }
    },
  });

  // Handle event not found
  if (error?.message === 'Event not found') {
    return (
      <div className="min-h-screen flex items-center justify-center p-4 bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500">
        <motion.div
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          className="text-center text-white"
        >
          <h1 className="text-4xl font-bold mb-4">Event Not Found 😕</h1>
          <p className="text-xl mb-6 opacity-80">
            The event you're looking for doesn't exist or has been removed.
          </p>
          <Link href="/">
            <Button className="bg-gradient-to-r from-pink-500 to-yellow-500 hover:from-pink-600 hover:to-yellow-600">
              <ArrowLeft className="w-4 h-4 mr-2" />
              Back to Events
            </Button>
          </Link>
        </motion.div>
      </div>
    );
  }

  if (isLoading) {
    return (
      <div className="min-h-screen bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500">
        <div className="animate-pulse p-4">
          <div className="h-64 bg-white/20 rounded-lg mb-4" />
          <div className="h-8 bg-white/20 rounded mb-2" />
          <div className="h-4 bg-white/20 rounded mb-4" />
          <div className="h-32 bg-white/20 rounded" />
        </div>
      </div>
    );
  }

  const handlePurchase = () => {
    if (!selectedTicketType) {
      toast({
        title: 'Please select a ticket type',
        variant: 'destructive',
      });
      return;
    }

    // Scroll to checkout form or handle purchase
    const checkoutSection = document.getElementById('checkout-section');
    if (checkoutSection) {
      checkoutSection.scrollIntoView({ behavior: 'smooth' });
    }
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500">
      {/* Mobile-first hero section */}
      <div className="relative">
        <img 
          src={event.image_url}
          alt={event.name}
          className="w-full h-64 md:h-80 object-cover"
        />
        <div className="absolute inset-0 bg-black/50" />
        
        {/* Back button - mobile friendly */}
        <div className="absolute top-4 left-4 z-10">
          <Link href="/">
            <Button size="sm" className="bg-white/20 backdrop-blur-sm border-0 text-white">
              <ArrowLeft className="w-4 h-4 mr-1" />
              Back
            </Button>
          </Link>
        </div>

        {/* Share button - mobile friendly */}
        <div className="absolute top-4 right-4 z-10">
          <Button 
            size="sm" 
            className="bg-white/20 backdrop-blur-sm border-0 text-white"
            onClick={() => {
              if (navigator.share) {
                navigator.share({
                  title: event.name,
                  text: event.description,
                  url: window.location.href,
                });
              } else {
                navigator.clipboard.writeText(window.location.href);
                toast({ title: 'Link copied to clipboard!' });
              }
            }}
          >
            <Share2 className="w-4 h-4" />
          </Button>
        </div>

        {/* Event title overlay */}
        <div className="absolute bottom-0 left-0 right-0 p-4 bg-gradient-to-t from-black/80 to-transparent">
          <h1 className="text-2xl md:text-4xl font-bold text-white mb-2">
            {event.name}
          </h1>
          <div className="flex flex-wrap items-center gap-2 text-white/90">
            <div className="flex items-center">
              <Calendar className="w-4 h-4 mr-1" />
              <span className="text-sm">
                {new Date(event.date_from).toLocaleDateString()}
              </span>
            </div>
            <div className="flex items-center">
              <MapPin className="w-4 h-4 mr-1" />
              <span className="text-sm">{event.venue}</span>
            </div>
            {event.start_time && (
              <div className="flex items-center">
                <Clock className="w-4 h-4 mr-1" />
                <span className="text-sm">{event.start_time}</span>
              </div>
            )}
          </div>
        </div>
      </div>

      {/* Mobile-optimized buy tickets section - NEAR TOP */}
      <div className="sticky top-0 z-20 bg-white/95 backdrop-blur-sm border-b border-white/20 p-4">
        <div className="flex items-center justify-between">
          <div>
            <p className="text-sm text-gray-600">Starting from</p>
            <p className="text-lg font-bold text-gray-900">
              KES {ticketTypes && ticketTypes.length > 0 ? Math.min(...ticketTypes.map((t: any) => t.price_kes)).toLocaleString() : '100'}
            </p>
          </div>
          <Button 
            className="bg-gradient-to-r from-pink-500 to-yellow-500 hover:from-pink-600 hover:to-yellow-600 text-white font-bold px-6"
            onClick={handlePurchase}
            data-testid="buy-tickets-mobile"
          >
            <Ticket className="w-4 h-4 mr-2" />
            Buy Tickets
          </Button>
        </div>
      </div>

      <div className="p-4 max-w-4xl mx-auto space-y-6">
        {/* Event description */}
        <Card className="bg-white/95 backdrop-blur-sm border-white/30">
          <CardHeader>
            <CardTitle className="flex items-center">
              <Star className="w-5 h-5 mr-2 text-yellow-500" />
              About This Event
            </CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-gray-700 leading-relaxed">{event.description}</p>
          </CardContent>
        </Card>

        {/* Ticket types with mobile-friendly layout */}
        <Card className="bg-white/95 backdrop-blur-sm border-white/30">
          <CardHeader>
            <CardTitle className="flex items-center">
              <Ticket className="w-5 h-5 mr-2 text-pink-500" />
              Choose Your Tickets
            </CardTitle>
          </CardHeader>
          <CardContent className="space-y-3">
            {ticketTypesLoading ? (
              <div className="space-y-3">
                {[1, 2, 3].map((i) => (
                  <div key={i} className="animate-pulse bg-gray-200 h-16 rounded-lg" />
                ))}
              </div>
            ) : ticketTypes && ticketTypes.length > 0 ? (
              ticketTypes.map((ticketType: any) => (
                <motion.div
                  key={ticketType.id}
                  whileHover={{ scale: 1.02 }}
                  whileTap={{ scale: 0.98 }}
                  className={`p-4 border-2 rounded-lg cursor-pointer transition-all ${
                    selectedTicketType?.id === ticketType.id
                      ? 'border-pink-500 bg-pink-50'
                      : 'border-gray-200 hover:border-gray-300'
                  }`}
                  onClick={() => setSelectedTicketType(ticketType)}
                  data-testid={`ticket-type-${ticketType.name.toLowerCase()}`}
                >
                  <div className="flex items-center justify-between">
                    <div className="flex-1">
                      <div className="flex items-center space-x-2">
                        <h3 className="font-semibold text-gray-900">{ticketType.name}</h3>
                        {ticketType.name.toLowerCase().includes('vip') && (
                          <Badge className="bg-gradient-to-r from-yellow-400 to-orange-500 text-white">
                            ⭐ Premium
                          </Badge>
                        )}
                      </div>
                      <p className="text-2xl font-bold text-pink-600">
                        KES {ticketType.price_kes.toLocaleString()}
                      </p>
                      <div className="flex items-center text-sm text-gray-600 mt-1">
                        <Users className="w-4 h-4 mr-1" />
                        <span>
                          {ticketType.available_quantity - ticketType.sold_quantity} available
                        </span>
                      </div>
                    </div>
                    <div className="flex items-center">
                      {selectedTicketType?.id === ticketType.id && (
                        <CheckCircle className="w-6 h-6 text-pink-500" />
                      )}
                    </div>
                  </div>
                </motion.div>
              ))
            ) : (
              <div className="text-center py-8 text-gray-500">
                <Ticket className="w-12 h-12 mx-auto mb-4 text-gray-300" />
                <p>Ticket information not available</p>
              </div>
            )}
          </CardContent>
        </Card>

        {/* Quantity selector */}
        {selectedTicketType && (
          <Card className="bg-white/95 backdrop-blur-sm border-white/30">
            <CardContent className="pt-6">
              <div className="flex items-center justify-between">
                <div>
                  <h3 className="font-semibold text-gray-900">Quantity</h3>
                  <p className="text-sm text-gray-600">How many tickets?</p>
                </div>
                <div className="flex items-center space-x-3">
                  <Button
                    variant="outline"
                    size="sm"
                    onClick={() => setQuantity(Math.max(1, quantity - 1))}
                    disabled={quantity <= 1}
                  >
                    -
                  </Button>
                  <span className="text-xl font-semibold w-8 text-center">{quantity}</span>
                  <Button
                    variant="outline"
                    size="sm"
                    onClick={() => setQuantity(quantity + 1)}
                    disabled={quantity >= (selectedTicketType.available_quantity - selectedTicketType.sold_quantity)}
                  >
                    +
                  </Button>
                </div>
              </div>
            </CardContent>
          </Card>
        )}

        {/* Purchase summary */}
        {selectedTicketType && (
          <Card className="bg-gradient-to-r from-pink-50 to-yellow-50 border-2 border-pink-200">
            <CardContent className="pt-6">
              <div className="space-y-3">
                <div className="flex justify-between items-center">
                  <span className="text-gray-700">Ticket Type:</span>
                  <span className="font-semibold">{selectedTicketType.name}</span>
                </div>
                <div className="flex justify-between items-center">
                  <span className="text-gray-700">Quantity:</span>
                  <span className="font-semibold">{quantity}</span>
                </div>
                <div className="flex justify-between items-center">
                  <span className="text-gray-700">Price per ticket:</span>
                  <span className="font-semibold">KES {selectedTicketType.price_kes.toLocaleString()}</span>
                </div>
                <hr className="border-gray-300" />
                <div className="flex justify-between items-center text-lg">
                  <span className="font-bold text-gray-900">Total:</span>
                  <span className="font-bold text-pink-600">
                    KES {(selectedTicketType.price_kes * quantity).toLocaleString()}
                  </span>
                </div>
                
                <Link href={`/checkout/${event.id}?ticketType=${selectedTicketType.id}&quantity=${quantity}`}>
                  <Button className="w-full bg-gradient-to-r from-pink-500 to-yellow-500 hover:from-pink-600 hover:to-yellow-600 text-white font-bold py-3 text-lg">
                    <ShoppingCart className="w-5 h-5 mr-2" />
                    Proceed to Checkout
                  </Button>
                </Link>
              </div>
            </CardContent>
          </Card>
        )}

        {/* Event details */}
        <Card className="bg-white/95 backdrop-blur-sm border-white/30">
          <CardHeader>
            <CardTitle>Event Details</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div className="space-y-3">
                <div className="flex items-start space-x-3">
                  <Calendar className="w-5 h-5 text-pink-500 mt-0.5" />
                  <div>
                    <p className="font-semibold text-gray-900">Date & Time</p>
                    <p className="text-gray-600">
                      {new Date(event.date_from).toLocaleDateString('en-US', {
                        weekday: 'long',
                        year: 'numeric',
                        month: 'long',
                        day: 'numeric'
                      })}
                    </p>
                    {event.start_time && (
                      <p className="text-gray-600">Starts at {event.start_time}</p>
                    )}
                    {event.date_to && (
                      <p className="text-gray-600">
                        Ends: {new Date(event.date_to).toLocaleDateString()}
                      </p>
                    )}
                  </div>
                </div>
                
                <div className="flex items-start space-x-3">
                  <MapPin className="w-5 h-5 text-pink-500 mt-0.5" />
                  <div>
                    <p className="font-semibold text-gray-900">Venue</p>
                    <p className="text-gray-600">{event.venue}</p>
                  </div>
                </div>
              </div>
            </div>
          </CardContent>
        </Card>
      </div>
    </div>
  );
}
```

### Payment Confirmation Page
Create `client/src/pages/SuccessPage.tsx`:
```typescript
import React, { useEffect, useState } from 'react';
import { motion } from 'framer-motion';
import { useQuery } from '@tanstack/react-query';
import { useSearchParams, Link } from 'wouter';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { 
  CheckCircle, 
  Download, 
  Mail, 
  Calendar, 
  MapPin, 
  Ticket,
  Home,
  Clock,
  User,
  CreditCard
} from 'lucide-react';

export function SuccessPage() {
  const [searchParams] = useSearchParams();
  const [showConfetti, setShowConfetti] = useState(true);
  const reference = searchParams.get('reference');
  const ticketId = searchParams.get('ticket');

  // Fetch payment confirmation details
  const { data: confirmation, isLoading } = useQuery({
    queryKey: ['/api/payment/confirm', reference],
    queryFn: async () => {
      if (!reference) return null;
      const response = await fetch(`/api/payment/confirm?reference=${reference}`);
      if (!response.ok) throw new Error('Failed to confirm payment');
      return response.json();
    },
    enabled: !!reference,
  });

  // Hide confetti after 3 seconds
  useEffect(() => {
    const timer = setTimeout(() => setShowConfetti(false), 3000);
    return () => clearTimeout(timer);
  }, []);

  // Download ticket function
  const downloadTicket = async () => {
    try {
      const response = await fetch(`/api/tickets/${confirmation.ticket_uid}/download`);
      const blob = await response.blob();
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `ticket-${confirmation.ticket_uid}.pdf`;
      a.click();
      window.URL.revokeObjectURL(url);
    } catch (error) {
      console.error('Download failed:', error);
    }
  };

  if (isLoading) {
    return (
      <div className="min-h-screen bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500 flex items-center justify-center p-4">
        <Card className="bg-white/95 backdrop-blur-sm border-white/30 max-w-md w-full">
          <CardContent className="pt-6">
            <div className="text-center">
              <div className="w-16 h-16 border-4 border-pink-500 border-t-transparent rounded-full animate-spin mx-auto mb-4" />
              <p className="text-gray-600">Confirming your payment...</p>
            </div>
          </CardContent>
        </Card>
      </div>
    );
  }

  if (!confirmation) {
    return (
      <div className="min-h-screen bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500 flex items-center justify-center p-4">
        <Card className="bg-white/95 backdrop-blur-sm border-white/30 max-w-md w-full">
          <CardContent className="pt-6 text-center">
            <h1 className="text-2xl font-bold text-gray-900 mb-4">Payment Not Found</h1>
            <p className="text-gray-600 mb-6">We couldn't find details for this payment reference.</p>
            <Link href="/">
              <Button className="w-full bg-gradient-to-r from-pink-500 to-yellow-500">
                <Home className="w-4 h-4 mr-2" />
                Back to Events
              </Button>
            </Link>
          </CardContent>
        </Card>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500 relative overflow-hidden">
      {/* Confetti Animation */}
      {showConfetti && (
        <div className="absolute inset-0 pointer-events-none">
          {[...Array(50)].map((_, i) => (
            <motion.div
              key={i}
              initial={{ 
                y: -100, 
                x: Math.random() * window.innerWidth,
                rotate: 0,
                scale: Math.random() * 0.5 + 0.5
              }}
              animate={{ 
                y: window.innerHeight + 100,
                rotate: Math.random() * 360
              }}
              transition={{ 
                duration: Math.random() * 3 + 2,
                ease: "easeInOut"
              }}
              className={`absolute w-3 h-3 rounded-full ${
                ['bg-pink-400', 'bg-yellow-400', 'bg-blue-400', 'bg-green-400', 'bg-purple-400'][i % 5]
              }`}
              style={{
                left: Math.random() * 100 + '%',
                animationDelay: Math.random() * 2 + 's'
              }}
            />
          ))}
        </div>
      )}

      <div className="p-4 max-w-2xl mx-auto">
        <motion.div
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          transition={{ duration: 0.6 }}
          className="space-y-6"
        >
          {/* Success Header */}
          <Card className="bg-white/95 backdrop-blur-sm border-white/30 text-center">
            <CardContent className="pt-8 pb-6">
              <motion.div
                initial={{ scale: 0 }}
                animate={{ scale: 1 }}
                transition={{ delay: 0.2, type: "spring", stiffness: 200 }}
              >
                <CheckCircle className="w-24 h-24 text-green-500 mx-auto mb-4" />
              </motion.div>
              <h1 className="text-3xl font-bold text-gray-900 mb-2">
                Payment Successful! 🎉
              </h1>
              <p className="text-gray-600 text-lg">
                Your tickets have been confirmed and sent to your email
              </p>
              <Badge className="mt-4 bg-green-100 text-green-800 hover:bg-green-100">
                <CreditCard className="w-4 h-4 mr-1" />
                Payment Ref: {confirmation.reference}
              </Badge>
            </CardContent>
          </Card>

          {/* Ticket Information */}
          <Card className="bg-white/95 backdrop-blur-sm border-white/30">
            <CardHeader>
              <CardTitle className="flex items-center">
                <Ticket className="w-5 h-5 mr-2 text-pink-500" />
                Your Ticket Details
              </CardTitle>
            </CardHeader>
            <CardContent className="space-y-4">
              <div className="bg-gradient-to-r from-pink-50 to-yellow-50 p-4 rounded-lg">
                <h3 className="font-bold text-lg text-gray-900 mb-2">
                  {confirmation.event_name}
                </h3>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4 text-sm">
                  <div className="flex items-center space-x-2">
                    <User className="w-4 h-4 text-gray-500" />
                    <div>
                      <span className="text-gray-600">Ticket Holder:</span>
                      <p className="font-medium">{confirmation.buyer_name}</p>
                    </div>
                  </div>
                  
                  <div className="flex items-center space-x-2">
                    <Mail className="w-4 h-4 text-gray-500" />
                    <div>
                      <span className="text-gray-600">Email:</span>
                      <p className="font-medium">{confirmation.buyer_email}</p>
                    </div>
                  </div>
                  
                  <div className="flex items-center space-x-2">
                    <Ticket className="w-4 h-4 text-gray-500" />
                    <div>
                      <span className="text-gray-600">Ticket ID:</span>
                      <p className="font-medium font-mono">{confirmation.ticket_uid}</p>
                    </div>
                  </div>
                  
                  <div className="flex items-center space-x-2">
                    <CreditCard className="w-4 h-4 text-gray-500" />
                    <div>
                      <span className="text-gray-600">Amount Paid:</span>
                      <p className="font-bold text-green-600">
                        KES {confirmation.total_paid_kes.toLocaleString()}
                      </p>
                    </div>
                  </div>
                </div>
              </div>

              {/* Event Details */}
              <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                <div className="flex items-start space-x-3">
                  <Calendar className="w-5 h-5 text-pink-500 mt-0.5" />
                  <div>
                    <p className="font-semibold text-gray-900">Event Date</p>
                    <p className="text-gray-600">
                      {new Date(confirmation.event_date).toLocaleDateString('en-US', {
                        weekday: 'long',
                        year: 'numeric',
                        month: 'long',
                        day: 'numeric'
                      })}
                    </p>
                  </div>
                </div>
                
                <div className="flex items-start space-x-3">
                  <MapPin className="w-5 h-5 text-pink-500 mt-0.5" />
                  <div>
                    <p className="font-semibold text-gray-900">Venue</p>
                    <p className="text-gray-600">{confirmation.event_venue}</p>
                  </div>
                </div>
              </div>
            </CardContent>
          </Card>

          {/* Action Buttons */}
          <Card className="bg-white/95 backdrop-blur-sm border-white/30">
            <CardContent className="pt-6">
              <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                <Button 
                  onClick={downloadTicket}
                  className="bg-gradient-to-r from-blue-500 to-purple-500 hover:from-blue-600 hover:to-purple-600 text-white"
                  data-testid="download-ticket"
                >
                  <Download className="w-4 h-4 mr-2" />
                  Download Ticket PDF
                </Button>
                
                <Button 
                  variant="outline"
                  onClick={() => {
                    const subject = encodeURIComponent(`My ticket for ${confirmation.event_name}`);
                    const body = encodeURIComponent(`I just bought a ticket for ${confirmation.event_name}! Ticket ID: ${confirmation.ticket_uid}`);
                    window.open(`mailto:?subject=${subject}&body=${body}`);
                  }}
                  data-testid="share-ticket"
                >
                  <Mail className="w-4 h-4 mr-2" />
                  Share via Email
                </Button>
              </div>
            </CardContent>
          </Card>

          {/* Important Information */}
          <Card className="bg-blue-50 border-blue-200">
            <CardContent className="pt-6">
              <h3 className="font-bold text-blue-900 mb-3 flex items-center">
                <Clock className="w-5 h-5 mr-2" />
                Important Information
              </h3>
              <ul className="space-y-2 text-sm text-blue-800">
                <li>• Your ticket has been sent to your email address</li>
                <li>• Please bring a valid ID and your ticket (digital or printed) to the event</li>
                <li>• Your ticket contains a unique QR code for fast entry</li>
                <li>• Screenshots of tickets are accepted for entry</li>
                <li>• Contact support if you have any issues with your ticket</li>
              </ul>
            </CardContent>
          </Card>

          {/* Navigation */}
          <div className="flex flex-col sm:flex-row gap-4">
            <Link href="/" className="flex-1">
              <Button variant="outline" className="w-full bg-white/50 hover:bg-white/70">
                <Home className="w-4 h-4 mr-2" />
                Browse More Events
              </Button>
            </Link>
          </div>
        </motion.div>
      </div>
    </div>
  );
}
```

### Backend Admin Event Management System

Add comprehensive event management endpoints to `server/routes.ts`:

```typescript
// Admin: Create new event
app.post("/api/admin/events", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userRole = req.user?.role;
    const userId = req.user?.id;

    if (userRole !== 'admin') {
      return res.status(403).json({ error: "Admin access required" });
    }

    const {
      name,
      description,
      venue,
      date_from,
      date_to,
      price_kes,
      category,
      image_url,
      organizer_id
    } = req.body;

    // Validate required fields
    if (!name || !venue || !date_from || !price_kes || !organizer_id) {
      return res.status(400).json({ error: "Missing required fields" });
    }

    // Generate slug from name
    const slug = name.toLowerCase()
      .replace(/[^a-z0-9]+/g, '-')
      .replace(/(^-|-$)/g, '');

    // Verify organizer exists and is approved
    const [organizer] = await db
      .select()
      .from(users)
      .where(and(
        eq(users.id, organizer_id),
        eq(users.role, 'organizer'),
        eq(users.approved, true)
      ));

    if (!organizer) {
      return res.status(400).json({ error: "Invalid or unapproved organizer" });
    }

    const [newEvent] = await db
      .insert(events)
      .values({
        name,
        slug,
        description: description || '',
        venue,
        date_from: new Date(date_from),
        date_to: date_to ? new Date(date_to) : new Date(date_from),
        price_kes: parseInt(price_kes),
        category: category || 'general',
        image_url: image_url || 'https://images.unsplash.com/photo-1540575467063-178a50c2df87?ixlib=rb-4.0.3&auto=format&fit=crop&w=800&q=80',
        organizer_id,
        created_by: userId
      })
      .returning();

    res.json({
      success: true,
      event: newEvent,
      message: "Event created successfully"
    });

  } catch (error) {
    console.error("Admin event creation error:", error);
    res.status(500).json({ error: "Failed to create event" });
  }
});

// Admin: Get all events with search and filters
app.get("/api/admin/events", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userRole = req.user?.role;

    if (userRole !== 'admin') {
      return res.status(403).json({ error: "Admin access required" });
    }

    const { search, category, status, organizer, limit = 50, offset = 0 } = req.query;

    let query = db
      .select({
        id: events.id,
        name: events.name,
        slug: events.slug,
        description: events.description,
        venue: events.venue,
        date_from: events.date_from,
        date_to: events.date_to,
        price_kes: events.price_kes,
        category: events.category,
        image_url: events.image_url,
        created_at: events.created_at,
        organizer_name: users.name,
        organizer_email: users.email,
        organizer_id: events.organizer_id,
        tickets_sold: sql<number>`(
          SELECT COUNT(*) FROM ${tickets} 
          WHERE ${tickets.event_id} = ${events.id} 
          AND ${tickets.paid} = true
        )`,
        total_revenue: sql<number>`(
          SELECT COALESCE(SUM(${tickets.total_paid_kes}), 0) FROM ${tickets} 
          WHERE ${tickets.event_id} = ${events.id} 
          AND ${tickets.paid} = true
        )`
      })
      .from(events)
      .leftJoin(users, eq(events.organizer_id, users.id));

    // Apply search filter
    if (search && typeof search === 'string') {
      query = query.where(
        or(
          ilike(events.name, `%${search}%`),
          ilike(events.venue, `%${search}%`),
          ilike(events.description, `%${search}%`),
          ilike(users.name, `%${search}%`)
        )
      );
    }

    // Apply category filter
    if (category && typeof category === 'string') {
      query = query.where(eq(events.category, category));
    }

    // Apply organizer filter
    if (organizer && typeof organizer === 'string') {
      query = query.where(eq(events.organizer_id, organizer));
    }

    const adminEvents = await query
      .orderBy(desc(events.created_at))
      .limit(parseInt(limit as string))
      .offset(parseInt(offset as string));

    res.json(adminEvents);

  } catch (error) {
    console.error("Admin events fetch error:", error);
    res.status(500).json({ error: "Failed to fetch events" });
  }
});

// Admin: Update event
app.patch("/api/admin/events/:id", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userRole = req.user?.role;

    if (userRole !== 'admin') {
      return res.status(403).json({ error: "Admin access required" });
    }

    const { id } = req.params;
    const updates = req.body;

    // Remove undefined values
    const cleanUpdates = Object.fromEntries(
      Object.entries(updates).filter(([_, value]) => value !== undefined)
    );

    if (Object.keys(cleanUpdates).length === 0) {
      return res.status(400).json({ error: "No updates provided" });
    }

    // Update slug if name is updated
    if (cleanUpdates.name) {
      cleanUpdates.slug = cleanUpdates.name.toLowerCase()
        .replace(/[^a-z0-9]+/g, '-')
        .replace(/(^-|-$)/g, '');
    }

    // Convert dates
    if (cleanUpdates.date_from) {
      cleanUpdates.date_from = new Date(cleanUpdates.date_from);
    }
    if (cleanUpdates.date_to) {
      cleanUpdates.date_to = new Date(cleanUpdates.date_to);
    }

    const [updatedEvent] = await db
      .update(events)
      .set(cleanUpdates)
      .where(eq(events.id, id))
      .returning();

    if (!updatedEvent) {
      return res.status(404).json({ error: "Event not found" });
    }

    res.json({
      success: true,
      event: updatedEvent,
      message: "Event updated successfully"
    });

  } catch (error) {
    console.error("Admin event update error:", error);
    res.status(500).json({ error: "Failed to update event" });
  }
});

// Admin: Delete event
app.delete("/api/admin/events/:id", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userRole = req.user?.role;

    if (userRole !== 'admin') {
      return res.status(403).json({ error: "Admin access required" });
    }

    const { id } = req.params;

    // Check if event has tickets sold
    const ticketCount = await db
      .select({ count: count() })
      .from(tickets)
      .where(and(
        eq(tickets.event_id, id),
        eq(tickets.paid, true)
      ));

    if (ticketCount[0]?.count > 0) {
      return res.status(400).json({ 
        error: "Cannot delete event with sold tickets",
        tickets_sold: ticketCount[0].count
      });
    }

    // Delete unsold tickets first
    await db
      .delete(tickets)
      .where(eq(tickets.event_id, id));

    // Delete the event
    const [deletedEvent] = await db
      .delete(events)
      .where(eq(events.id, id))
      .returning();

    if (!deletedEvent) {
      return res.status(404).json({ error: "Event not found" });
    }

    res.json({
      success: true,
      message: "Event deleted successfully"
    });

  } catch (error) {
    console.error("Admin event deletion error:", error);
    res.status(500).json({ error: "Failed to delete event" });
  }
});

// Admin: Get system statistics
app.get("/api/admin/stats", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userRole = req.user?.role;

    if (userRole !== 'admin') {
      return res.status(403).json({ error: "Admin access required" });
    }

    // Get comprehensive system statistics
    const [stats] = await db.execute(sql`
      SELECT 
        (
          SELECT COUNT(*) FROM ${events}
        ) as total_events,
        (
          SELECT COUNT(*) FROM ${events} 
          WHERE ${events.date_from} >= NOW()
        ) as upcoming_events,
        (
          SELECT COUNT(*) FROM ${users}
        ) as total_users,
        (
          SELECT COUNT(*) FROM ${users} 
          WHERE ${users.approved} = false
        ) as pending_users,
        (
          SELECT COUNT(*) FROM ${tickets}
          WHERE ${tickets.paid} = true
        ) as total_tickets_sold,
        (
          SELECT COALESCE(SUM(${tickets.total_paid_kes}), 0) FROM ${tickets}
          WHERE ${tickets.paid} = true
        ) as total_revenue,
        (
          SELECT COUNT(*) FROM ${tickets}
          WHERE ${tickets.status} = 'scanned'
        ) as tickets_scanned,
        (
          SELECT COUNT(*) FROM ${payment_requests}
          WHERE ${payment_requests.status} = 'pending'
        ) as pending_payments
    `);

    res.json(stats);

  } catch (error) {
    console.error("Admin stats error:", error);
    res.status(500).json({ error: "Failed to fetch statistics" });
  }
});

// Admin: Get all users with search and filters
app.get("/api/admin/users", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userRole = req.user?.role;

    if (userRole !== 'admin') {
      return res.status(403).json({ error: "Admin access required" });
    }

    const { search, role, approved, limit = 50, offset = 0 } = req.query;

    let query = db
      .select({
        id: users.id,
        name: users.name,
        email: users.email,
        phone: users.phone,
        role: users.role,
        approved: users.approved,
        created_at: users.created_at,
        last_login: users.last_login,
        events_created: sql<number>`(
          SELECT COUNT(*) FROM ${events} 
          WHERE ${events.organizer_id} = ${users.id}
        )`,
        tickets_purchased: sql<number>`(
          SELECT COUNT(*) FROM ${tickets} 
          WHERE ${tickets.user_id} = ${users.id}
          AND ${tickets.paid} = true
        )`
      })
      .from(users);

    // Apply search filter
    if (search && typeof search === 'string') {
      query = query.where(
        or(
          ilike(users.name, `%${search}%`),
          ilike(users.email, `%${search}%`),
          ilike(users.phone, `%${search}%`)
        )
      );
    }

    // Apply role filter
    if (role && typeof role === 'string') {
      query = query.where(eq(users.role, role));
    }

    // Apply approval filter
    if (approved !== undefined) {
      query = query.where(eq(users.approved, approved === 'true'));
    }

    const allUsers = await query
      .orderBy(desc(users.created_at))
      .limit(parseInt(limit as string))
      .offset(parseInt(offset as string));

    res.json(allUsers);

  } catch (error) {
    console.error("Admin users fetch error:", error);
    res.status(500).json({ error: "Failed to fetch users" });
  }
});

// Admin: Update user (approve/reject, change role)
app.patch("/api/admin/users/:id", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userRole = req.user?.role;
    const adminId = req.user?.id;

    if (userRole !== 'admin') {
      return res.status(403).json({ error: "Admin access required" });
    }

    const { id } = req.params;
    const { approved, role, notes } = req.body;

    // Prevent admin from modifying themselves
    if (id === adminId) {
      return res.status(400).json({ error: "Cannot modify your own account" });
    }

    const updates: any = {};
    if (approved !== undefined) updates.approved = approved;
    if (role && ['user', 'organizer', 'scanner', 'admin'].includes(role)) {
      updates.role = role;
    }

    if (Object.keys(updates).length === 0) {
      return res.status(400).json({ error: "No valid updates provided" });
    }

    const [updatedUser] = await db
      .update(users)
      .set(updates)
      .where(eq(users.id, id))
      .returning({
        id: users.id,
        name: users.name,
        email: users.email,
        role: users.role,
        approved: users.approved
      });

    if (!updatedUser) {
      return res.status(404).json({ error: "User not found" });
    }

    res.json({
      success: true,
      user: updatedUser,
      message: `User ${approved !== undefined ? (approved ? 'approved' : 'rejected') : 'updated'} successfully`
    });

  } catch (error) {
    console.error("Admin user update error:", error);
    res.status(500).json({ error: "Failed to update user" });
  }
});
```

### Essential Backend API Enhancements
Add these critical API endpoints to `server/routes.ts`:

```typescript
// Enhanced featured events endpoint with ticket types
app.get("/api/events/featured", async (req, res) => {
  try {
    const featuredEvents = await db
      .select({
        id: events.id,
        slug: events.slug,
        name: events.name,
        description: events.description,
        venue: events.venue,
        date_from: events.date_from,
        date_to: events.date_to,
        image_url: events.image_url,
        price_kes: events.price_kes,
        is_active: events.is_active,
      })
      .from(events)
      .where(eq(events.is_active, true))
      .orderBy(events.date_from)
      .limit(20);

    // Enhance with minimum ticket prices
    const eventsWithPricing = await Promise.all(
      featuredEvents.map(async (event) => {
        const minPrice = await db
          .select({ min_price: sql<number>`MIN(price_kes)` })
          .from(ticket_types)
          .where(eq(ticket_types.event_id, event.id));

        return {
          ...event,
          price_kes: minPrice[0]?.min_price || event.price_kes || 100,
        };
      })
    );

    res.json(eventsWithPricing);
  } catch (error) {
    console.error("Featured events error:", error);
    res.status(500).json({ error: "Failed to fetch featured events" });
  }
});

// Event details with comprehensive error handling
app.get("/api/events/:slug", async (req, res) => {
  try {
    const { slug } = req.params;

    if (!slug) {
      return res.status(400).json({ error: "Event slug is required" });
    }

    const [event] = await db
      .select()
      .from(events)
      .where(eq(events.slug, slug));

    if (!event) {
      return res.status(404).json({ error: "Event not found" });
    }

    if (!event.is_active) {
      return res.status(404).json({ error: "Event is not available" });
    }

    res.json(event);
  } catch (error) {
    console.error(`Event details error for slug ${req.params.slug}:`, error);
    res.status(500).json({ error: "Failed to fetch event details" });
  }
});

// Ticket types for specific event
app.get("/api/events/:slug/ticket-types", async (req, res) => {
  try {
    const { slug } = req.params;

    // First get the event
    const [event] = await db
      .select({ id: events.id })
      .from(events)
      .where(and(eq(events.slug, slug), eq(events.is_active, true)));

    if (!event) {
      return res.status(404).json({ error: "Event not found" });
    }

    // Get ticket types for this event
    const ticketTypes = await db
      .select()
      .from(ticket_types)
      .where(and(
        eq(ticket_types.event_id, event.id),
        eq(ticket_types.is_active, true)
      ))
      .orderBy(ticket_types.price_kes); // Order by price: Regular, VIP, VVIP

    res.json(ticketTypes);
  } catch (error) {
    console.error(`Ticket types error for slug ${req.params.slug}:`, error);
    res.status(500).json({ error: "Failed to fetch ticket types" });
  }
});

// Payment confirmation endpoint
app.get("/api/payment/confirm", async (req, res) => {
  try {
    const { reference } = req.query;

    if (!reference) {
      return res.status(400).json({ error: "Payment reference is required" });
    }

    // Find ticket by payment reference
    const [ticket] = await db
      .select({
        ticket_uid: tickets.uid,
        buyer_name: tickets.buyer_name,
        buyer_email: tickets.buyer_email,
        total_paid_kes: tickets.total_paid_kes,
        event_name: events.name,
        event_date: events.date_from,
        event_venue: events.venue,
        reference: sql<string>`${reference}`,
      })
      .from(tickets)
      .innerJoin(events, eq(tickets.event_id, events.id))
      .where(eq(tickets.uid, reference.toString()))
      .limit(1);

    if (!ticket) {
      return res.status(404).json({ error: "Payment confirmation not found" });
    }

    res.json(ticket);
  } catch (error) {
    console.error("Payment confirmation error:", error);
    res.status(500).json({ error: "Failed to confirm payment" });
  }
});

// Admin event approval endpoints
app.post("/api/admin/events/:eventId/approve", authenticateToken, async (req: AuthRequest, res) => {
  try {
    if (req.user?.role !== 'admin') {
      return res.status(403).json({ error: "Admin access required" });
    }

    const { eventId } = req.params;
    
    await db
      .update(events)
      .set({ 
        is_active: true,
        updated_at: new Date()
      })
      .where(eq(events.id, eventId));

    res.json({ message: "Event approved successfully" });
  } catch (error) {
    console.error("Event approval error:", error);
    res.status(500).json({ error: "Failed to approve event" });
  }
});

app.post("/api/admin/events/:eventId/reject", authenticateToken, async (req: AuthRequest, res) => {
  try {
    if (req.user?.role !== 'admin') {
      return res.status(403).json({ error: "Admin access required" });
    }

    const { eventId } = req.params;
    
    await db
      .update(events)
      .set({ 
        is_active: false,
        updated_at: new Date()
      })
      .where(eq(events.id, eventId));

    res.json({ message: "Event rejected successfully" });
  } catch (error) {
    console.error("Event rejection error:", error);
    res.status(500).json({ error: "Failed to reject event" });
  }
});

// Ticket download endpoint
app.get("/api/tickets/:uid/download", async (req, res) => {
  try {
    const { uid } = req.params;

    const [ticket] = await db
      .select({
        uid: tickets.uid,
        buyer_name: tickets.buyer_name,
        buyer_email: tickets.buyer_email,
        event_name: events.name,
        event_date: events.date_from,
        event_venue: events.venue,
        price_kes: tickets.price_kes,
      })
      .from(tickets)
      .innerJoin(events, eq(tickets.event_id, events.id))
      .where(eq(tickets.uid, uid));

    if (!ticket) {
      return res.status(404).json({ error: "Ticket not found" });
    }

    // Generate PDF using PDFKit (implementation depends on your PDF generation setup)
    const PDFDocument = require('pdfkit');
    const QRCode = require('qrcode');

    const doc = new PDFDocument();
    
    // Set headers for PDF download
    res.setHeader('Content-Type', 'application/pdf');
    res.setHeader('Content-Disposition', `attachment; filename="ticket-${uid}.pdf"`);

    // Pipe PDF to response
    doc.pipe(res);

    // Add ticket content
    doc.fontSize(20).text('Uliza Tickets', 50, 50);
    doc.fontSize(16).text(ticket.event_name, 50, 100);
    doc.fontSize(12).text(`Venue: ${ticket.event_venue}`, 50, 130);
    doc.text(`Date: ${new Date(ticket.event_date).toLocaleDateString()}`, 50, 150);
    doc.text(`Ticket Holder: ${ticket.buyer_name}`, 50, 170);
    doc.text(`Ticket ID: ${ticket.uid}`, 50, 190);

    // Generate and add QR code
    const qrCodeDataURL = await QRCode.toDataURL(ticket.uid);
    const qrCodeBuffer = Buffer.from(qrCodeDataURL.split(',')[1], 'base64');
    doc.image(qrCodeBuffer, 400, 100, { width: 100 });

    doc.end();
  } catch (error) {
    console.error("Ticket download error:", error);
    res.status(500).json({ error: "Failed to generate ticket PDF" });
  }
});
```

### Mobile-Optimized CSS Enhancements
Add to `client/src/index.css`:

```css
/* Mobile-first responsive design */
@media (max-width: 640px) {
  .min-h-screen {
    min-height: 100vh;
    min-height: 100dvh; /* Dynamic viewport height for mobile */
  }
  
  /* Prevent zoom on input focus */
  input, select, textarea {
    font-size: 16px !important;
  }
  
  /* Smooth scrolling */
  html {
    scroll-behavior: smooth;
  }
  
  /* Touch-friendly tap targets */
  button, .cursor-pointer {
    min-height: 44px;
    min-width: 44px;
  }
  
  /* Prevent horizontal scroll */
  body {
    overflow-x: hidden;
  }
  
  /* Card hover effects for touch */
  .card-hover-mobile {
    transition: transform 0.2s ease;
    -webkit-tap-highlight-color: transparent;
  }
  
  .card-hover-mobile:active {
    transform: scale(0.98);
  }
}

/* Text truncation utility */
.line-clamp-2 {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.line-clamp-3 {
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

/* Loading states */
@keyframes shimmer {
  0% {
    background-position: -200px 0;
  }
  100% {
    background-position: calc(200px + 100%) 0;
  }
}

.shimmer {
  background: linear-gradient(90deg, transparent, rgba(255,255,255,0.2), transparent);
  background-size: 200px 100%;
  animation: shimmer 1.5s infinite;
}

/* Backdrop blur support */
.backdrop-blur-sm {
  backdrop-filter: blur(4px);
  -webkit-backdrop-filter: blur(4px);
}

.backdrop-blur-md {
  backdrop-filter: blur(8px);
  -webkit-backdrop-filter: blur(8px);
}

/* Festival gradient backgrounds */
.gradient-festival {
  background: linear-gradient(135deg, #ff6b6b 0%, #ffd93d 25%, #6bcf7f  50%, #4d7fff 75%, #ff6b6b 100%);
  background-size: 400% 400%;
  animation: gradientShift 8s ease infinite;
}

@keyframes gradientShift {
  0% { background-position: 0% 50%; }
  50% { background-position: 100% 50%; }
  100% { background-position: 0% 50%; }
}

/* Safe area insets for mobile devices */
.safe-area-inset {
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
  padding-left: env(safe-area-inset-left);
  padding-right: env(safe-area-inset-right);
}
```

### PWA Registration and Enhanced App Configuration
Update your `client/src/App.tsx` with PWA support and mobile optimization:
```typescript
import React, { useEffect } from 'react';
import { Route, Switch } from 'wouter';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Toaster } from '@/components/ui/toaster';
import { Button } from '@/components/ui/button';
import { Link } from 'wouter';

// Pages
import { LandingPage } from '@/pages/LandingPage';
import { EventDetailsPage } from '@/pages/EventDetailsPage';
import { CheckoutPage } from '@/pages/CheckoutPage';
import { SuccessPage } from '@/pages/SuccessPage';
import { LoginPage } from '@/pages/LoginPage';
import { SignupPage } from '@/pages/SignupPage';
import { UserDashboard } from '@/pages/UserDashboard';
import { AdminDashboard } from '@/pages/AdminDashboard';
import { OrganizerDashboard } from '@/pages/OrganizerDashboard';
import { ScannerDashboard } from '@/pages/ScannerDashboard';

// Create query client with mobile-optimized settings
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      gcTime: 10 * 60 * 1000, // 10 minutes (renamed from cacheTime)
      retry: (failureCount, error: any) => {
        // Don't retry on 404s
        if (error?.status === 404) return false;
        return failureCount < 3;
      },
      refetchOnWindowFocus: false, // Prevent unnecessary mobile refetches
      networkMode: 'offlineFirst', // PWA offline-first strategy
    },
  },
});

// PWA Service Worker Registration
function registerServiceWorker() {
  if ('serviceWorker' in navigator) {
    window.addEventListener('load', () => {
      navigator.serviceWorker.register('/sw.js')
        .then((registration) => {
          console.log('SW registered: ', registration);
          
          // Handle updates
          registration.addEventListener('updatefound', () => {
            const newWorker = registration.installing;
            if (newWorker) {
              newWorker.addEventListener('statechange', () => {
                if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
                  // New content available, prompt user to refresh
                  if (confirm('New version available! Refresh to update?')) {
                    window.location.reload();
                  }
                }
              });
            }
          });
        })
        .catch((registrationError) => {
          console.log('SW registration failed: ', registrationError);
        });
    });
  }
}

function App() {
  useEffect(() => {
    // Register PWA service worker
    registerServiceWorker();
    
    // Set up PWA install prompt
    let deferredPrompt: any;
    
    window.addEventListener('beforeinstallprompt', (e) => {
      deferredPrompt = e;
      // Show install button if needed
    });

    // Listen for service worker messages (offline sync notifications)
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.addEventListener('message', (event) => {
        if (event.data.type === 'SYNC_COMPLETE') {
          // Show sync completion notification
          console.log(`${event.data.count} offline scans synchronized`);
        }
      });
    }

    // Performance optimization for mobile
    if (typeof window !== 'undefined') {
      // Disable context menu on long press (mobile)
      document.addEventListener('contextmenu', (e) => {
        if (window.innerWidth <= 768) { // Mobile viewport
          e.preventDefault();
        }
      });

      // Disable zoom on double tap
      let lastTouchEnd = 0;
      document.addEventListener('touchend', (event) => {
        const now = (new Date()).getTime();
        if (now - lastTouchEnd <= 300) {
          event.preventDefault();
        }
        lastTouchEnd = now;
      }, false);
    }
  }, []);

  return (
    <QueryClientProvider client={queryClient}>
      <div className="min-h-screen bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500">
        <Switch>
          <Route path="/" component={LandingPage} />
          <Route path="/login" component={LoginPage} />
          <Route path="/signup" component={SignupPage} />
          <Route path="/events/:slug" component={EventDetailsPage} />
          <Route path="/checkout/:eventId" component={CheckoutPage} />
          <Route path="/success" component={SuccessPage} />
          <Route path="/admin" component={AdminDashboard} />
          <Route path="/organizer" component={OrganizerDashboard} />
          <Route path="/scanner" component={ScannerDashboard} />
          <Route path="/dashboard" component={UserDashboard} />
          <Route>
            {/* 404 Page with back button */}
            <div className="min-h-screen flex flex-col">
              {/* Back button header */}
              <div className="sticky top-0 z-50 bg-white/10 backdrop-blur-md border-b border-white/20 px-4 py-3">
                <div className="flex items-center justify-between max-w-7xl mx-auto">
                  <Link href="/">
                    <Button size="sm" variant="ghost" className="text-white hover:bg-white/20">
                      <ArrowLeft className="w-4 h-4 mr-1" />
                      Back to Home
                    </Button>
                  </Link>
                  <h1 className="text-lg font-bold text-white">Page Not Found</h1>
                  <div></div>
                </div>
              </div>
              
              {/* 404 Content */}
              <div className="flex-1 flex items-center justify-center text-white p-4">
                <div className="text-center">
                  <h1 className="text-4xl md:text-6xl font-bold mb-4">404</h1>
                  <p className="text-xl md:text-2xl mb-8 opacity-80">Page Not Found</p>
                  <Link href="/">
                    <Button className="bg-gradient-to-r from-pink-500 to-yellow-500 hover:from-pink-600 hover:to-yellow-600">
                      Back to Home
                    </Button>
                  </Link>
                </div>
              </div>
            </div>
          </Route>
        </Switch>
        
        {/* Mobile-optimized toaster */}
        <Toaster 
          position="top-center"
          className="safe-area-inset"
        />
      </div>
    </QueryClientProvider>
  );
}

export default App;
```

### Back Button Implementation for All Dashboard Pages

Update all dashboard pages to include consistent back button headers:

#### Admin Dashboard Back Button
```typescript
// Add to AdminDashboard.tsx at the top of the return statement:
<div className="sticky top-0 z-50 bg-white/10 backdrop-blur-md border-b border-white/20 px-4 py-3 mb-6">
  <div className="flex items-center justify-between max-w-7xl mx-auto">
    <Link href="/login">
      <Button size="sm" variant="ghost" className="text-white hover:bg-white/20">
        <ArrowLeft className="w-4 h-4 mr-1" />
        Logout
      </Button>
    </Link>
    <h1 className="text-lg font-bold text-white">Admin Dashboard</h1>
    <div></div>
  </div>
</div>
```

#### Organizer Dashboard Back Button
```typescript
// Add to OrganizerDashboard.tsx at the top of the return statement:
<div className="sticky top-0 z-50 bg-white/10 backdrop-blur-md border-b border-white/20 px-4 py-3 mb-6">
  <div className="flex items-center justify-between max-w-7xl mx-auto">
    <Link href="/login">
      <Button size="sm" variant="ghost" className="text-white hover:bg-white/20">
        <ArrowLeft className="w-4 h-4 mr-1" />
        Logout
      </Button>
    </Link>
    <h1 className="text-lg font-bold text-white">Organizer Dashboard</h1>
    <div className="flex items-center space-x-2">
      <Link href="/scanner">
        <Button size="sm" className="bg-purple-500 hover:bg-purple-600">
          <Scan className="w-4 h-4 mr-1" />
          Scanner
        </Button>
      </Link>
    </div>
  </div>
</div>
```

#### Login/Signup Page Back Buttons
```typescript
// Add to LoginPage.tsx and SignupPage.tsx after the main content:
<div className="absolute top-4 left-4">
  <Link href="/">
    <Button size="sm" variant="ghost" className="text-white hover:bg-white/20">
      <ArrowLeft className="w-4 h-4 mr-1" />
      Back to Events
    </Button>
  </Link>
</div>
```

#### Checkout Page Back Button
```typescript
// Add to CheckoutPage.tsx at the top:
<div className="sticky top-0 z-50 bg-white/10 backdrop-blur-md border-b border-white/20 px-4 py-3">
  <div className="flex items-center justify-between max-w-7xl mx-auto">
    <Button 
      size="sm" 
      variant="ghost" 
      className="text-white hover:bg-white/20"
      onClick={() => window.history.back()}
    >
      <ArrowLeft className="w-4 h-4 mr-1" />
      Back to Event
    </Button>
    <h1 className="text-lg font-bold text-white">Checkout</h1>
    <div></div>
  </div>
</div>
```

### Performance Optimizations for 100,000+ Tickets

Add to `vite.config.ts`:
```typescript
export default defineConfig({
  plugins: [
    react(),
    // PWA plugin for enhanced performance
    {
      name: 'pwa-optimization',
      configureServer(server) {
        // Enable compression
        server.middlewares.use(compression());
        
        // Cache static assets
        server.middlewares.use('/assets', express.static('public', {
          maxAge: '1y',
          etag: false
        }));
      }
    }
  ],
  build: {
    // Optimize for production
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true
      }
    },
    // Enable code splitting for large datasets
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': ['react', 'react-dom'],
          'scanner': ['@zxing/library'],
          'ui': ['framer-motion', 'lucide-react']
        }
      }
    },
    // Increase chunk size warning limit for large ticket datasets
    chunkSizeWarningLimit: 1000
  },
  // Optimize development for large datasets
  optimizeDeps: {
    include: ['react', 'react-dom', '@tanstack/react-query']
  }
});
```

### Mobile-Optimized Ticket PDF Generation System

Add comprehensive PDF ticket generation to `server/routes.ts`:

```typescript
import PDFDocument from 'pdfkit';
import QRCode from 'qrcode';
import crypto from 'crypto';
import { format } from 'date-fns';
import { zonedTimeToUtc, utcToZonedTime } from 'date-fns-tz';

// HMAC signature generation for ticket security
const generateTicketSignature = (uid: string, eventId: string): string => {
  const hmacSecret = process.env.HMAC_SECRET || 'your-secure-hmac-secret';
  const payload = `${uid}|${eventId}`;
  return crypto.createHmac('sha256', hmacSecret).update(payload).digest('hex');
};

// Format date for Kenya timezone (Africa/Nairobi)
const formatEventDate = (date: Date): string => {
  const timezone = 'Africa/Nairobi';
  const zonedDate = utcToZonedTime(date, timezone);
  return format(zonedDate, 'MMMM d, yyyy · h:mm a');
};

// Format phone number with spaces
const formatPhoneNumber = (phone: string): string => {
  return phone.replace(/(\+254)(\d{3})(\d{3})(\d{3})/, '$1 $2 $3 $4');
};

// Generate short ticket ID
const generateShortTicketId = (uid: string): string => {
  return `#${uid.slice(0, 8).toUpperCase()}`;
};

// Mobile-optimized PDF ticket generation
const generateTicketPDF = async (ticketData: any): Promise<Buffer> => {
  return new Promise((resolve, reject) => {
    try {
      // Create PDF document - mobile optimized (390 × 844 pt)
      const doc = new PDFDocument({
        size: [390, 844],
        margins: { top: 16, bottom: 16, left: 16, right: 16 }
      });

      const buffers: Buffer[] = [];
      doc.on('data', buffers.push.bind(buffers));
      doc.on('end', () => resolve(Buffer.concat(buffers)));

      const pageWidth = 390;
      const pageHeight = 844;
      const margin = 16;
      const contentWidth = pageWidth - (margin * 2);

      // Background card with rounded corners
      doc.roundedRect(margin, margin, contentWidth, pageHeight - (margin * 2), 16)
         .fillAndStroke('#FAFAFA', '#E5E5E5');

      // Header section with event image placeholder
      const headerHeight = 120;
      doc.roundedRect(margin + 8, margin + 8, contentWidth - 16, headerHeight - 16, 12)
         .fill('#F0F0F0');

      // Event image or placeholder
      if (ticketData.event.image_url) {
        // In production, you'd fetch and embed the actual image
        doc.fontSize(12)
           .fillColor('#666666')
           .text('Event Poster', margin + contentWidth/2 - 30, margin + headerHeight/2 - 6);
      } else {
        // Event initials as fallback
        const initials = ticketData.event.name
          .split(' ')
          .map(word => word[0])
          .join('')
          .substring(0, 3)
          .toUpperCase();
        
        doc.fontSize(24)
           .fillColor('#666666')
           .text(initials, margin + contentWidth/2 - 20, margin + headerHeight/2 - 12);
      }

      let yPosition = margin + headerHeight + 24;

      // Event Title (bold, large)
      doc.fontSize(22)
         .fillColor('#1A1A1A')
         .font('Helvetica-Bold')
         .text(ticketData.event.name, margin + 8, yPosition, {
           width: contentWidth - 16,
           lineGap: 2
         });

      yPosition += 32;

      // Optional subtitle
      if (ticketData.event.description) {
        doc.fontSize(14)
           .fillColor('#666666')
           .font('Helvetica')
           .text(ticketData.event.description.substring(0, 80) + '...', margin + 8, yPosition, {
             width: contentWidth - 16
           });
        yPosition += 24;
      }

      // Venue & Schedule Section
      yPosition += 16;
      const leftColumnX = margin + 8;
      const rightColumnX = margin + contentWidth/2 + 8;
      const labelColor = '#8B8B8B';
      const valueColor = '#1A1A1A';

      // Left column - Venue & Schedule
      doc.fontSize(9)
         .fillColor(labelColor)
         .font('Helvetica-Bold')
         .text('VENUE', leftColumnX, yPosition);
      
      doc.fontSize(13)
         .fillColor(valueColor)
         .font('Helvetica-Bold')
         .text(ticketData.event.venue, leftColumnX, yPosition + 12, {
           width: contentWidth/2 - 16
         });

      yPosition += 40;

      doc.fontSize(9)
         .fillColor(labelColor)
         .font('Helvetica-Bold')
         .text('DATE', leftColumnX, yPosition);
      
      doc.fontSize(13)
         .fillColor(valueColor)
         .font('Helvetica-Bold')
         .text(formatEventDate(new Date(ticketData.event.date_from)), leftColumnX, yPosition + 12, {
           width: contentWidth/2 - 16
         });

      // Right column - Ticket Info
      yPosition -= 40; // Reset to top of left column

      doc.fontSize(9)
         .fillColor(labelColor)
         .font('Helvetica-Bold')
         .text('TICKET TYPE', rightColumnX, yPosition);
      
      doc.fontSize(13)
         .fillColor(valueColor)
         .font('Helvetica-Bold')
         .text('General Admission', rightColumnX, yPosition + 12, {
           width: contentWidth/2 - 16
         });

      yPosition += 40;

      doc.fontSize(9)
         .fillColor(labelColor)
         .font('Helvetica-Bold')
         .text('TICKET No.', rightColumnX, yPosition);
      
      doc.fontSize(13)
         .fillColor(valueColor)
         .font('Helvetica-Bold')
         .text(generateShortTicketId(ticketData.uid), rightColumnX, yPosition + 12);

      yPosition += 40;

      // Full UID (monospace, smaller)
      doc.fontSize(8)
         .fillColor('#999999')
         .font('Courier')
         .text(`UID: ${ticketData.uid}`, rightColumnX, yPosition, {
           width: contentWidth/2 - 16
         });

      yPosition += 32;

      // Attendee Information
      doc.fontSize(9)
         .fillColor(labelColor)
         .font('Helvetica-Bold')
         .text('ATTENDEE', leftColumnX, yPosition);
      
      doc.fontSize(13)
         .fillColor(valueColor)
         .font('Helvetica-Bold')
         .text(ticketData.buyer_name, leftColumnX, yPosition + 12, {
           width: contentWidth - 16
         });

      yPosition += 32;

      doc.fontSize(9)
         .fillColor(labelColor)
         .font('Helvetica-Bold')
         .text('PHONE', leftColumnX, yPosition);
      
      doc.fontSize(13)
         .fillColor(valueColor)
         .font('Helvetica-Bold')
         .text(formatPhoneNumber(ticketData.buyer_phone), leftColumnX, yPosition + 12);

      doc.fontSize(9)
         .fillColor(labelColor)
         .font('Helvetica-Bold')
         .text('EMAIL', rightColumnX, yPosition);
      
      const email = ticketData.buyer_email;
      const truncatedEmail = email.length > 25 ? 
        email.substring(0, 12) + '...' + email.substring(email.length - 10) : 
        email;
      
      doc.fontSize(13)
         .fillColor(valueColor)
         .font('Helvetica-Bold')
         .text(truncatedEmail, rightColumnX, yPosition + 12, {
           width: contentWidth/2 - 16
         });

      yPosition += 60;

      // QR Code section
      const qrSize = 180;
      const qrX = margin + (contentWidth - qrSize) / 2;
      
      // Generate QR code data with HMAC signature
      const signature = generateTicketSignature(ticketData.uid, ticketData.event_id);
      const qrPayload = JSON.stringify({
        uid: ticketData.uid,
        eventId: ticketData.event_id,
        sig: signature
      });

      // Generate QR code (in production, use a server-safe QR library)
      QRCode.toBuffer(qrPayload, { 
        width: qrSize,
        margin: 0,
        color: { dark: '#000000', light: '#FFFFFF' }
      }).then(qrBuffer => {
        doc.image(qrBuffer, qrX, yPosition, { width: qrSize, height: qrSize });

        yPosition += qrSize + 16;

        // Short link under QR
        const shortLink = `${process.env.BASE_URL || 'https://uliza.app'}/t/${ticketData.uid}`;
        doc.fontSize(12)
           .fillColor('#666666')
           .font('Helvetica')
           .text(shortLink, margin + 8, yPosition, {
             width: contentWidth - 16,
             align: 'center'
           });

        yPosition += 24;

        // Fine print section
        const finePrint = `
This ticket is valid for one-time entry only. Present this QR code at the venue entrance. 
Screenshots accepted. Non-transferable except through official platform. Entry subject to venue terms and conditions. 
No refunds unless event is cancelled. Support: +254 700 000 000 | support@uliza.co.ke
        `.trim();

        doc.fontSize(8)
           .fillColor('#999999')
           .font('Helvetica')
           .text(finePrint, margin + 8, yPosition, {
             width: contentWidth - 16,
             lineGap: 1
           });

        doc.end();
      }).catch(reject);

    } catch (error) {
      reject(error);
    }
  });
};

// Get individual ticket PDF
app.get("/api/tickets/:uid.pdf", async (req, res) => {
  try {
    const { uid } = req.params;
    
    // Fetch ticket with related data
    const [ticket] = await db
      .select({
        id: tickets.id,
        uid: tickets.uid,
        buyer_name: tickets.buyer_name,
        buyer_email: tickets.buyer_email,
        buyer_phone: tickets.buyer_phone,
        event_id: tickets.event_id,
        paid: tickets.paid,
        user_id: tickets.user_id,
        event: {
          id: events.id,
          name: events.name,
          description: events.description,
          venue: events.venue,
          date_from: events.date_from,
          image_url: events.image_url,
          organizer_id: events.organizer_id
        }
      })
      .from(tickets)
      .leftJoin(events, eq(tickets.event_id, events.id))
      .where(eq(tickets.uid, uid));

    if (!ticket) {
      return res.status(404).json({ error: "Ticket not found" });
    }

    if (!ticket.paid) {
      return res.status(403).json({ error: "Ticket not paid" });
    }

    // Access control - allow ticket owner, organizer, or admin
    const token = req.headers.authorization?.replace('Bearer ', '');
    let hasAccess = false;

    if (token) {
      try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET!) as any;
        const userId = decoded.userId;
        
        // Check if user owns ticket, is organizer, or is admin
        const [user] = await db
          .select()
          .from(users)
          .where(eq(users.id, userId));

        if (user) {
          hasAccess = 
            ticket.user_id === userId || // Ticket owner
            ticket.event.organizer_id === userId || // Event organizer
            user.role === 'admin'; // Admin
        }
      } catch (error) {
        // Token invalid, check other access methods
      }
    }

    // Allow access by email/phone match (for guest purchases)
    if (!hasAccess) {
      const { email, phone } = req.query;
      if (email && phone) {
        hasAccess = 
          ticket.buyer_email.toLowerCase() === email.toString().toLowerCase() &&
          ticket.buyer_phone === phone.toString();
      }
    }

    if (!hasAccess) {
      return res.status(403).json({ error: "Access denied" });
    }

    // Generate PDF
    const pdfBuffer = await generateTicketPDF(ticket);

    // Set response headers
    res.setHeader('Content-Type', 'application/pdf');
    res.setHeader('Content-Disposition', `inline; filename="TICKET_${uid}.pdf"`);
    res.setHeader('Content-Length', pdfBuffer.length);

    res.send(pdfBuffer);

  } catch (error) {
    console.error("PDF generation error:", error);
    res.status(500).json({ error: "Failed to generate PDF" });
  }
});

// Short link page for ticket viewing
app.get("/t/:uid", async (req, res) => {
  try {
    const { uid } = req.params;
    
    const [ticket] = await db
      .select({
        uid: tickets.uid,
        buyer_name: tickets.buyer_name,
        paid: tickets.paid,
        event_name: events.name,
        event_venue: events.venue,
        event_date: events.date_from
      })
      .from(tickets)
      .leftJoin(events, eq(tickets.event_id, events.id))
      .where(eq(tickets.uid, uid));

    if (!ticket) {
      return res.status(404).send(`
        <html>
          <head><title>Ticket Not Found</title></head>
          <body style="font-family: system-ui; text-align: center; padding: 2rem;">
            <h1>Ticket Not Found</h1>
            <p>The ticket you're looking for doesn't exist.</p>
          </body>
        </html>
      `);
    }

    if (!ticket.paid) {
      return res.status(403).send(`
        <html>
          <head><title>Ticket Unpaid</title></head>
          <body style="font-family: system-ui; text-align: center; padding: 2rem;">
            <h1>Ticket Not Paid</h1>
            <p>This ticket hasn't been paid for yet.</p>
          </body>
        </html>
      `);
    }

    const formattedDate = formatEventDate(new Date(ticket.event_date));

    res.send(`
      <!DOCTYPE html>
      <html>
        <head>
          <title>${ticket.event_name} - Ticket</title>
          <meta name="viewport" content="width=device-width, initial-scale=1">
          <style>
            body { 
              font-family: system-ui, -apple-system, sans-serif; 
              margin: 0; 
              padding: 2rem; 
              background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
              min-height: 100vh;
              color: white;
            }
            .ticket-card {
              background: rgba(255, 255, 255, 0.1);
              backdrop-filter: blur(10px);
              border-radius: 16px;
              padding: 2rem;
              max-width: 400px;
              margin: 0 auto;
              text-align: center;
              border: 1px solid rgba(255, 255, 255, 0.2);
            }
            h1 { margin: 0 0 1rem 0; font-size: 1.5rem; }
            .details { margin: 1.5rem 0; text-align: left; }
            .detail { margin: 0.5rem 0; }
            .label { opacity: 0.8; font-size: 0.9rem; }
            .value { font-weight: 600; }
            .download-btn {
              background: #10b981;
              color: white;
              border: none;
              padding: 1rem 2rem;
              border-radius: 12px;
              font-size: 1.1rem;
              font-weight: 600;
              cursor: pointer;
              text-decoration: none;
              display: inline-block;
              margin-top: 1rem;
            }
            .download-btn:hover {
              background: #059669;
            }
          </style>
        </head>
        <body>
          <div class="ticket-card">
            <h1>🎫 ${ticket.event_name}</h1>
            <div class="details">
              <div class="detail">
                <div class="label">Attendee</div>
                <div class="value">${ticket.buyer_name}</div>
              </div>
              <div class="detail">
                <div class="label">Venue</div>
                <div class="value">${ticket.event_venue}</div>
              </div>
              <div class="detail">
                <div class="label">Date</div>
                <div class="value">${formattedDate}</div>
              </div>
              <div class="detail">
                <div class="label">Ticket ID</div>
                <div class="value">${generateShortTicketId(uid)}</div>
              </div>
            </div>
            <a href="/api/tickets/${uid}.pdf" class="download-btn">
              📱 Download PDF Ticket
            </a>
          </div>
        </body>
      </html>
    `);

  } catch (error) {
    console.error("Ticket page error:", error);
    res.status(500).send("Internal server error");
  }
});

// Email ticket functionality (called after successful payment)
const emailTicketPDF = async (ticketData: any, recipientEmail: string) => {
  try {
    const pdfBuffer = await generateTicketPDF(ticketData);
    
    const transporter = nodemailer.createTransporter({
      host: process.env.BREVO_SMTP_HOST || 'smtp-relay.brevo.com',
      port: 587,
      secure: false,
      auth: {
        user: process.env.BREVO_SMTP_USER || 'your-brevo-email',
        pass: process.env.BREVO_SMTP_PASSWORD
      }
    });

    const shortLink = `${process.env.BASE_URL || 'https://uliza.app'}/t/${ticketData.uid}`;

    await transporter.sendMail({
      from: process.env.SMTP_FROM || 'tickets@uliza.co.ke',
      to: recipientEmail,
      subject: `🎫 Your ticket for ${ticketData.event.name}`,
      html: `
        <div style="font-family: system-ui, sans-serif; max-width: 600px; margin: 0 auto;">
          <h1 style="color: #1f2937;">Your Ticket is Ready! 🎉</h1>
          
          <div style="background: #f9fafb; padding: 1.5rem; border-radius: 12px; margin: 1rem 0;">
            <h2 style="margin: 0 0 1rem 0; color: #374151;">${ticketData.event.name}</h2>
            <p style="margin: 0.5rem 0; color: #6b7280;">
              <strong>Date:</strong> ${formatEventDate(new Date(ticketData.event.date_from))}
            </p>
            <p style="margin: 0.5rem 0; color: #6b7280;">
              <strong>Venue:</strong> ${ticketData.event.venue}
            </p>
            <p style="margin: 0.5rem 0; color: #6b7280;">
              <strong>Attendee:</strong> ${ticketData.buyer_name}
            </p>
          </div>

          <div style="text-align: center; margin: 2rem 0;">
            <a href="${shortLink}" 
               style="background: #10b981; color: white; padding: 12px 24px; 
                      border-radius: 8px; text-decoration: none; font-weight: 600;">
              📱 View Your Ticket
            </a>
          </div>

          <p style="color: #6b7280; font-size: 0.9rem;">
            Your ticket is attached as a PDF. You can also view it online or download it anytime using the link above.
          </p>

          <hr style="border: none; border-top: 1px solid #e5e7eb; margin: 2rem 0;">
          
          <p style="color: #9ca3af; font-size: 0.8rem;">
            Need help? Contact us at support@uliza.co.ke or +254 700 000 000
          </p>
        </div>
      `,
      attachments: [
        {
          filename: `TICKET_${ticketData.uid}.pdf`,
          content: pdfBuffer,
          contentType: 'application/pdf'
        }
      ]
    });

    console.log(`Ticket PDF emailed to ${recipientEmail} for ticket ${ticketData.uid}`);

  } catch (error) {
    console.error("Email ticket error:", error);
    // Don't throw - log error but don't fail the payment process
  }
};

// QR Code verification endpoint for scanners
app.post("/api/tickets/verify", async (req, res) => {
  try {
    const { qrData } = req.body;
    
    let qrPayload;
    try {
      qrPayload = JSON.parse(qrData);
    } catch (error) {
      return res.status(400).json({ error: "Invalid QR code format" });
    }

    const { uid, eventId, sig } = qrPayload;

    if (!uid || !eventId || !sig) {
      return res.status(400).json({ error: "Missing required QR data" });
    }

    // Verify HMAC signature
    const expectedSig = generateTicketSignature(uid, eventId);
    if (sig !== expectedSig) {
      return res.status(400).json({ 
        error: "Invalid ticket signature",
        status: "INVALID"
      });
    }

    // Fetch and verify ticket
    const [ticket] = await db
      .select({
        uid: tickets.uid,
        buyer_name: tickets.buyer_name,
        event_id: tickets.event_id,
        status: tickets.status,
        paid: tickets.paid,
        event_name: events.name,
        event_date: events.date_from
      })
      .from(tickets)
      .leftJoin(events, eq(tickets.event_id, events.id))
      .where(and(
        eq(tickets.uid, uid),
        eq(tickets.event_id, eventId)
      ));

    if (!ticket) {
      return res.status(404).json({ 
        error: "Ticket not found",
        status: "NOT_FOUND"
      });
    }

    if (!ticket.paid) {
      return res.status(403).json({ 
        error: "Ticket not paid",
        status: "UNPAID"
      });
    }

    if (ticket.status === 'scanned') {
      return res.status(409).json({ 
        error: "Ticket already used",
        status: "ALREADY_SCANNED",
        ticket: {
          attendee: ticket.buyer_name,
          event: ticket.event_name,
          scanned_at: new Date().toISOString()
        }
      });
    }

    // Mark ticket as scanned
    await db
      .update(tickets)
      .set({ 
        status: 'scanned',
        scanned_at: new Date()
      })
      .where(eq(tickets.uid, uid));

    res.json({
      success: true,
      status: "VALID",
      ticket: {
        uid: ticket.uid,
        attendee: ticket.buyer_name,
        event: ticket.event_name,
        event_date: ticket.event_date,
        scanned_at: new Date().toISOString()
      }
    });

  } catch (error) {
    console.error("Ticket verification error:", error);
    res.status(500).json({ error: "Verification failed" });
  }
});
```

### Required Dependencies for PDF Generation

Add these to your `package.json`:

```json
{
  "dependencies": {
    "pdfkit": "^0.13.0",
    "qrcode": "^1.5.3",
    "date-fns": "^2.30.0",
    "date-fns-tz": "^2.0.0",
    "nodemailer": "^6.9.0"
  },
  "devDependencies": {
    "@types/pdfkit": "^0.12.12",
    "@types/qrcode": "^1.5.2",
    "@types/nodemailer": "^6.4.14"
  }
}
```

### Environment Variables for PDF System

Add to your `.env`:

```bash
# HMAC secret for ticket security (use a strong random string)
HMAC_SECRET=your-256-bit-secret-key-here

# Base URL for short links
BASE_URL=https://your-domain.com

# SMTP configuration for ticket emails
SMTP_FROM=tickets@uliza.co.ke
BREVO_SMTP_HOST=smtp-relay.brevo.com
BREVO_SMTP_USER=your-brevo-email
BREVO_SMTP_PASSWORD=your-brevo-password
```

### Integration with Payment Success

Update your payment success handler to send ticket emails:

```typescript
// After successful payment processing
if (paymentSuccess) {
  // Send ticket PDFs via email
  for (const ticket of orderTickets) {
    await emailTicketPDF(ticket, ticket.buyer_email);
  }
}
```

### Guest Checkout Ticket Linking System

Add these API routes to `server/routes.ts` for linking guest checkout tickets to user accounts:

```typescript
// Link guest checkout tickets to user account (call after signup/login)
app.post("/api/link-guest-tickets", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userId = req.user?.id;
    const userEmail = req.user?.email;

    if (!userId || !userEmail) {
      return res.status(401).json({ error: "Authentication required" });
    }

    // Find guest checkouts for this user's email
    const guestCheckouts = await db
      .select()
      .from(guest_checkouts)
      .where(and(
        eq(guest_checkouts.email, userEmail),
        isNull(guest_checkouts.linked_user_id)
      ));

    let totalLinkedTickets = 0;

    for (const guestCheckout of guestCheckouts) {
      const ticketIds = JSON.parse(guestCheckout.ticket_ids);
      
      // Update all tickets to link them to the user
      const result = await db
        .update(tickets)
        .set({ 
          user_id: userId,
          is_guest_purchase: false,
          linked_at: new Date()
        })
        .where(and(
          inArray(tickets.id, ticketIds),
          isNull(tickets.user_id) // Only link unlinked tickets
        ));

      // Mark guest checkout as linked
      await db
        .update(guest_checkouts)
        .set({ 
          linked_user_id: userId,
          linked_at: new Date()
        })
        .where(eq(guest_checkouts.id, guestCheckout.id));

      totalLinkedTickets += ticketIds.length;
    }

    res.json({
      success: true,
      linked_tickets: totalLinkedTickets,
      message: `Successfully linked ${totalLinkedTickets} tickets to your account`
    });

  } catch (error) {
    console.error("Guest ticket linking error:", error);
    res.status(500).json({ error: "Failed to link guest tickets" });
  }
});

// Get user's tickets (including linked guest purchases) for dashboard
app.get("/api/my-tickets", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userId = req.user?.id;

    if (!userId) {
      return res.status(401).json({ error: "Authentication required" });
    }

    const userTickets = await db
      .select({
        id: tickets.id,
        uid: tickets.uid,
        event_name: events.name,
        event_slug: events.slug,
        event_date: events.date_from,
        venue: events.venue,
        buyer_name: tickets.buyer_name,
        quantity: tickets.quantity,
        total_paid_kes: tickets.total_paid_kes,
        status: tickets.status,
        created_at: tickets.created_at,
        is_guest_purchase: tickets.is_guest_purchase,
        linked_at: tickets.linked_at,
        event_image: events.image_url,
      })
      .from(tickets)
      .leftJoin(events, eq(tickets.event_id, events.id))
      .where(eq(tickets.user_id, userId))
      .orderBy(desc(tickets.created_at));

    res.json(userTickets);

  } catch (error) {
    console.error("Get user tickets error:", error);
    res.status(500).json({ error: "Failed to fetch user tickets" });
  }
});
```

### Payment Request System for Organizers

Add these API routes for organizer payment requests:

```typescript
// Create payment request for organizer
app.post("/api/payment-requests", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userId = req.user?.id;
    const userRole = req.user?.role;

    if (userRole !== 'organizer') {
      return res.status(403).json({ error: "Only organizers can create payment requests" });
    }

    const { event_id, bank_details, notes } = req.body;

    // Calculate payment details for the event
    const [event] = await db
      .select()
      .from(events)
      .where(and(
        eq(events.id, event_id),
        eq(events.organizer_id, userId)
      ));

    if (!event) {
      return res.status(404).json({ error: "Event not found or not owned by organizer" });
    }

    // Get ticket sales for this event
    const ticketSales = await db
      .select({
        total_tickets: count(tickets.id),
        total_revenue: sum(tickets.total_paid_kes),
      })
      .from(tickets)
      .where(and(
        eq(tickets.event_id, event_id),
        eq(tickets.paid, true)
      ));

    const tickets_sold = ticketSales[0]?.total_tickets || 0;
    const total_revenue = ticketSales[0]?.total_revenue || 0;
    
    if (tickets_sold === 0) {
      return res.status(400).json({ error: "No paid tickets found for this event" });
    }

    // Calculate platform fee (3.5%) and net amount
    const platform_fee_kes = Math.round(total_revenue * 0.035);
    const net_amount_kes = total_revenue - platform_fee_kes;

    // Check if payment request already exists for this event
    const [existingRequest] = await db
      .select()
      .from(payment_requests)
      .where(eq(payment_requests.event_id, event_id));

    if (existingRequest) {
      return res.status(400).json({ error: "Payment request already exists for this event" });
    }

    // Create payment request
    const [paymentRequest] = await db
      .insert(payment_requests)
      .values({
        organizer_id: userId,
        event_id,
        amount_kes: total_revenue,
        tickets_sold,
        platform_fee_kes,
        net_amount_kes,
        bank_details: JSON.stringify(bank_details),
        notes,
        status: 'pending'
      })
      .returning();

    res.json({
      success: true,
      payment_request: paymentRequest,
      message: "Payment request created successfully"
    });

  } catch (error) {
    console.error("Payment request creation error:", error);
    res.status(500).json({ error: "Failed to create payment request" });
  }
});

// Get organizer's payment requests
app.get("/api/my-payment-requests", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userId = req.user?.id;
    const userRole = req.user?.role;

    if (userRole !== 'organizer') {
      return res.status(403).json({ error: "Only organizers can view payment requests" });
    }

    const paymentRequests = await db
      .select({
        id: payment_requests.id,
        event_name: events.name,
        event_slug: events.slug,
        amount_kes: payment_requests.amount_kes,
        tickets_sold: payment_requests.tickets_sold,
        platform_fee_kes: payment_requests.platform_fee_kes,
        net_amount_kes: payment_requests.net_amount_kes,
        status: payment_requests.status,
        requested_at: payment_requests.requested_at,
        processed_at: payment_requests.processed_at,
        notes: payment_requests.notes,
      })
      .from(payment_requests)
      .leftJoin(events, eq(payment_requests.event_id, events.id))
      .where(eq(payment_requests.organizer_id, userId))
      .orderBy(desc(payment_requests.requested_at));

    res.json(paymentRequests);

  } catch (error) {
    console.error("Get payment requests error:", error);
    res.status(500).json({ error: "Failed to fetch payment requests" });
  }
});

// Admin: Get all payment requests
app.get("/api/admin/payment-requests", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userRole = req.user?.role;

    if (userRole !== 'admin') {
      return res.status(403).json({ error: "Admin access required" });
    }

    const paymentRequests = await db
      .select({
        id: payment_requests.id,
        organizer_name: users.name,
        organizer_email: users.email,
        event_name: events.name,
        event_slug: events.slug,
        amount_kes: payment_requests.amount_kes,
        tickets_sold: payment_requests.tickets_sold,
        platform_fee_kes: payment_requests.platform_fee_kes,
        net_amount_kes: payment_requests.net_amount_kes,
        status: payment_requests.status,
        requested_at: payment_requests.requested_at,
        processed_at: payment_requests.processed_at,
        notes: payment_requests.notes,
        bank_details: payment_requests.bank_details,
      })
      .from(payment_requests)
      .leftJoin(events, eq(payment_requests.event_id, events.id))
      .leftJoin(users, eq(payment_requests.organizer_id, users.id))
      .orderBy(desc(payment_requests.requested_at));

    res.json(paymentRequests);

  } catch (error) {
    console.error("Get admin payment requests error:", error);
    res.status(500).json({ error: "Failed to fetch payment requests" });
  }
});

// Admin: Approve/reject payment request
app.patch("/api/admin/payment-requests/:id", authenticateToken, async (req: AuthRequest, res) => {
  try {
    const userRole = req.user?.role;
    const userId = req.user?.id;

    if (userRole !== 'admin') {
      return res.status(403).json({ error: "Admin access required" });
    }

    const { id } = req.params;
    const { status, notes } = req.body; // status: 'approved', 'paid', 'rejected'

    if (!['approved', 'paid', 'rejected'].includes(status)) {
      return res.status(400).json({ error: "Invalid status. Must be 'approved', 'paid', or 'rejected'" });
    }

    const [updatedRequest] = await db
      .update(payment_requests)
      .set({
        status,
        processed_at: new Date(),
        processed_by: userId,
        notes: notes || payment_requests.notes
      })
      .where(eq(payment_requests.id, id))
      .returning();

    if (!updatedRequest) {
      return res.status(404).json({ error: "Payment request not found" });
    }

    res.json({
      success: true,
      payment_request: updatedRequest,
      message: `Payment request ${status} successfully`
    });

  } catch (error) {
    console.error("Payment request update error:", error);
    res.status(500).json({ error: "Failed to update payment request" });
  }
});
```

### User Dashboard with Linked Tickets

Create `client/src/pages/UserDashboard.tsx`:

```typescript
import React, { useEffect } from 'react';
import { motion } from 'framer-motion';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { useToast } from '@/hooks/use-toast';
import { Link } from 'wouter';
import { 
  ArrowLeft,
  Ticket,
  Calendar,
  MapPin,
  Download,
  Clock,
  CheckCircle,
  LinkIcon
} from 'lucide-react';

export function UserDashboard() {
  const { toast } = useToast();
  const queryClient = useQueryClient();

  // Fetch user's tickets
  const { data: tickets, isLoading } = useQuery({
    queryKey: ['/api/my-tickets'],
  });

  // Link guest tickets mutation
  const linkGuestTickets = useMutation({
    mutationFn: async () => {
      const token = localStorage.getItem('auth_token');
      const response = await fetch('/api/link-guest-tickets', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
        },
      });
      return response.json();
    },
    onSuccess: (data) => {
      if (data.success && data.linked_tickets > 0) {
        toast({
          title: '🎫 Tickets Linked!',
          description: data.message,
        });
        queryClient.invalidateQueries({ queryKey: ['/api/my-tickets'] });
      }
    },
  });

  // Auto-link guest tickets when dashboard loads
  useEffect(() => {
    linkGuestTickets.mutate();
  }, []);

  if (isLoading) {
    return (
      <div className="min-h-screen bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500 flex items-center justify-center">
        <div className="text-white text-center">
          <div className="animate-spin w-8 h-8 border-2 border-white border-t-transparent rounded-full mx-auto mb-4"></div>
          <p>Loading your tickets...</p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-pink-500 via-red-500 to-yellow-500">
      {/* Header with back button */}
      <div className="sticky top-0 z-50 bg-white/10 backdrop-blur-md border-b border-white/20 px-4 py-3">
        <div className="flex items-center justify-between max-w-7xl mx-auto">
          <Link href="/">
            <Button size="sm" variant="ghost" className="text-white hover:bg-white/20">
              <ArrowLeft className="w-4 h-4 mr-1" />
              Back to Events
            </Button>
          </Link>
          <h1 className="text-lg font-bold text-white">My Tickets</h1>
          <div></div>
        </div>
      </div>

      <div className="p-4 md:p-6 max-w-7xl mx-auto">
        <motion.div
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          transition={{ duration: 0.6 }}
        >
          {/* Stats Cards */}
          <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-8">
            <Card className="bg-white/10 backdrop-blur-sm border-white/20">
              <CardContent className="p-4 text-center">
                <Ticket className="w-8 h-8 text-white mx-auto mb-2" />
                <p className="text-2xl font-bold text-white">{tickets?.length || 0}</p>
                <p className="text-white/70 text-sm">Total Tickets</p>
              </CardContent>
            </Card>
            
            <Card className="bg-white/10 backdrop-blur-sm border-white/20">
              <CardContent className="p-4 text-center">
                <CheckCircle className="w-8 h-8 text-green-400 mx-auto mb-2" />
                <p className="text-2xl font-bold text-white">
                  {tickets?.filter(t => t.status === 'unused').length || 0}
                </p>
                <p className="text-white/70 text-sm">Valid Tickets</p>
              </CardContent>
            </Card>

            <Card className="bg-white/10 backdrop-blur-sm border-white/20">
              <CardContent className="p-4 text-center">
                <LinkIcon className="w-8 h-8 text-blue-400 mx-auto mb-2" />
                <p className="text-2xl font-bold text-white">
                  {tickets?.filter(t => t.is_guest_purchase === false && t.linked_at).length || 0}
                </p>
                <p className="text-white/70 text-sm">Linked from Guest</p>
              </CardContent>
            </Card>
          </div>

          {/* Tickets List */}
          <div className="space-y-4">
            {tickets?.length === 0 ? (
              <Card className="bg-white/10 backdrop-blur-sm border-white/20">
                <CardContent className="p-8 text-center">
                  <Ticket className="w-16 h-16 text-white/50 mx-auto mb-4" />
                  <h3 className="text-xl font-bold text-white mb-2">No Tickets Yet</h3>
                  <p className="text-white/70 mb-4">
                    You haven't purchased any tickets yet. Browse events to get started!
                  </p>
                  <Link href="/">
                    <Button className="bg-gradient-to-r from-pink-500 to-yellow-500 hover:from-pink-600 hover:to-yellow-600">
                      Browse Events
                    </Button>
                  </Link>
                </CardContent>
              </Card>
            ) : (
              tickets?.map((ticket) => (
                <motion.div
                  key={ticket.id}
                  initial={{ opacity: 0, y: 20 }}
                  animate={{ opacity: 1, y: 0 }}
                  transition={{ duration: 0.3 }}
                >
                  <Card className="bg-white/10 backdrop-blur-sm border-white/20 hover:bg-white/15 transition-all">
                    <CardContent className="p-6">
                      <div className="flex flex-col md:flex-row md:items-center justify-between space-y-4 md:space-y-0">
                        <div className="flex-1">
                          <div className="flex items-start justify-between mb-3">
                            <div>
                              <h3 className="text-xl font-bold text-white mb-1">
                                {ticket.event_name}
                              </h3>
                              {ticket.linked_at && (
                                <Badge variant="outline" className="bg-blue-500/20 border-blue-400 text-blue-200 mb-2">
                                  <LinkIcon className="w-3 h-3 mr-1" />
                                  Linked from Guest Purchase
                                </Badge>
                              )}
                            </div>
                            <Badge 
                              variant={ticket.status === 'unused' ? 'default' : 'secondary'}
                              className={ticket.status === 'unused' 
                                ? 'bg-green-500/20 border-green-400 text-green-200' 
                                : 'bg-gray-500/20 border-gray-400 text-gray-200'
                              }
                            >
                              {ticket.status === 'unused' ? 'Valid' : 'Used'}
                            </Badge>
                          </div>

                          <div className="space-y-2 text-white/80">
                            <div className="flex items-center">
                              <Calendar className="w-4 h-4 mr-2" />
                              <span>{new Date(ticket.event_date).toLocaleDateString()}</span>
                            </div>
                            <div className="flex items-center">
                              <MapPin className="w-4 h-4 mr-2" />
                              <span>{ticket.venue}</span>
                            </div>
                            <div className="flex items-center">
                              <Ticket className="w-4 h-4 mr-2" />
                              <span>Ticket #{ticket.uid}</span>
                            </div>
                          </div>

                          <div className="mt-3 flex items-center justify-between">
                            <div className="text-white">
                              <span className="text-sm text-white/70">Total Paid: </span>
                              <span className="font-bold">KES {ticket.total_paid_kes}</span>
                            </div>
                            {ticket.linked_at && (
                              <div className="text-xs text-blue-200">
                                Linked: {new Date(ticket.linked_at).toLocaleDateString()}
                              </div>
                            )}
                          </div>
                        </div>

                        <div className="flex flex-col space-y-2 md:ml-6">
                          <Button 
                            size="sm" 
                            className="bg-white/20 hover:bg-white/30 border border-white/30"
                            onClick={() => {
                              // Generate and download QR code for ticket
                              window.open(`/api/tickets/${ticket.id}/qr`, '_blank');
                            }}
                          >
                            <Download className="w-4 h-4 mr-1" />
                            Download QR
                          </Button>
                          
                          <Link href={`/events/${ticket.event_slug}`}>
                            <Button size="sm" variant="outline" className="w-full border-white/30 text-white hover:bg-white/20">
                              View Event
                            </Button>
                          </Link>
                        </div>
                      </div>
                    </CardContent>
                  </Card>
                </motion.div>
              ))
            )}
          </div>
        </motion.div>
      </div>
    </div>
  );
}
```

### Organizer Payment Request UI Integration

Add this payment request section to your existing `OrganizerDashboard.tsx` component:

```typescript
// Add these imports to the top of OrganizerDashboard.tsx
import { 
  DollarSign, 
  Clock, 
  CheckCircle, 
  XCircle, 
  CreditCard,
  AlertCircle 
} from 'lucide-react';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Label } from '@/components/ui/label';

// Add this state and mutations to the OrganizerDashboard component:
const [selectedEventForPayment, setSelectedEventForPayment] = useState<string>('');
const [bankDetails, setBankDetails] = useState({
  account_name: '',
  account_number: '',
  bank_name: '',
  bank_code: ''
});
const [paymentNotes, setPaymentNotes] = useState('');

// Fetch organizer's payment requests
const { data: paymentRequests, isLoading: paymentRequestsLoading } = useQuery({
  queryKey: ['/api/my-payment-requests'],
});

// Create payment request mutation
const createPaymentRequest = useMutation({
  mutationFn: async ({ event_id, bank_details, notes }: any) => {
    const token = localStorage.getItem('auth_token');
    const response = await fetch('/api/payment-requests', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
      },
      body: JSON.stringify({ event_id, bank_details, notes }),
    });
    return response.json();
  },
  onSuccess: (data) => {
    if (data.success) {
      toast({
        title: '💰 Payment Request Created',
        description: data.message,
      });
      queryClient.invalidateQueries({ queryKey: ['/api/my-payment-requests'] });
      setSelectedEventForPayment('');
      setBankDetails({ account_name: '', account_number: '', bank_name: '', bank_code: '' });
      setPaymentNotes('');
    } else {
      toast({
        title: 'Request Failed',
        description: data.error || 'Failed to create payment request',
        variant: 'destructive',
      });
    }
  },
});

const handleCreatePaymentRequest = () => {
  if (!selectedEventForPayment || !bankDetails.account_name || !bankDetails.account_number) {
    toast({
      title: 'Missing Information',
      description: 'Please select an event and provide bank details',
      variant: 'destructive',
    });
    return;
  }

  createPaymentRequest.mutate({
    event_id: selectedEventForPayment,
    bank_details: bankDetails,
    notes: paymentNotes,
  });
};

// Add this JSX section after the events section and before the closing div:

{/* Payment Requests Section */}
<motion.div
  initial={{ opacity: 0, y: 20 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.6, delay: 0.4 }}
  className="mb-8"
>
  <Card className="bg-white/10 backdrop-blur-sm border-white/20">
    <CardHeader>
      <CardTitle className="text-white flex items-center">
        <DollarSign className="w-6 h-6 mr-2" />
        Payment Requests
      </CardTitle>
      <CardDescription className="text-white/70">
        Request payments for your event ticket sales
      </CardDescription>
    </CardHeader>
    <CardContent className="space-y-6">
      {/* Create Payment Request Form */}
      <div className="bg-white/5 rounded-lg p-4">
        <h3 className="text-white font-semibold mb-4">Create Payment Request</h3>
        
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
          <div>
            <Label className="text-white/80">Select Event</Label>
            <select
              value={selectedEventForPayment}
              onChange={(e) => setSelectedEventForPayment(e.target.value)}
              className="w-full p-2 rounded bg-white/10 border border-white/20 text-white"
            >
              <option value="">Choose an event</option>
              {organizerEvents?.map((event) => (
                <option key={event.id} value={event.id} className="bg-gray-800">
                  {event.name}
                </option>
              ))}
            </select>
          </div>
          
          <div>
            <Label className="text-white/80">Account Name</Label>
            <Input
              value={bankDetails.account_name}
              onChange={(e) => setBankDetails({...bankDetails, account_name: e.target.value})}
              placeholder="Full account name"
              className="bg-white/10 border-white/20 text-white"
            />
          </div>
        </div>

        <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
          <div>
            <Label className="text-white/80">Account Number</Label>
            <Input
              value={bankDetails.account_number}
              onChange={(e) => setBankDetails({...bankDetails, account_number: e.target.value})}
              placeholder="Account number"
              className="bg-white/10 border-white/20 text-white"
            />
          </div>
          
          <div>
            <Label className="text-white/80">Bank Name</Label>
            <Input
              value={bankDetails.bank_name}
              onChange={(e) => setBankDetails({...bankDetails, bank_name: e.target.value})}
              placeholder="Bank name"
              className="bg-white/10 border-white/20 text-white"
            />
          </div>
        </div>

        <div className="mb-4">
          <Label className="text-white/80">Notes (Optional)</Label>
          <Textarea
            value={paymentNotes}
            onChange={(e) => setPaymentNotes(e.target.value)}
            placeholder="Additional notes about your payment request"
            className="bg-white/10 border-white/20 text-white"
            rows={3}
          />
        </div>

        <Button
          onClick={handleCreatePaymentRequest}
          disabled={createPaymentRequest.isPending}
          className="bg-green-500 hover:bg-green-600"
        >
          {createPaymentRequest.isPending ? (
            <>
              <div className="animate-spin w-4 h-4 border-2 border-white border-t-transparent rounded-full mr-2"></div>
              Creating...
            </>
          ) : (
            <>
              <CreditCard className="w-4 h-4 mr-2" />
              Create Payment Request
            </>
          )}
        </Button>
      </div>

      {/* Payment Requests List */}
      <div>
        <h3 className="text-white font-semibold mb-4">Your Payment Requests</h3>
        
        {paymentRequestsLoading ? (
          <div className="text-center text-white/70 py-8">
            <div className="animate-spin w-6 h-6 border-2 border-white border-t-transparent rounded-full mx-auto mb-2"></div>
            Loading payment requests...
          </div>
        ) : paymentRequests?.length === 0 ? (
          <div className="text-center text-white/70 py-8">
            <DollarSign className="w-12 h-12 text-white/30 mx-auto mb-2" />
            <p>No payment requests yet</p>
            <p className="text-sm">Create your first payment request above</p>
          </div>
        ) : (
          <div className="space-y-3">
            {paymentRequests?.map((request) => (
              <div
                key={request.id}
                className="bg-white/5 rounded-lg p-4 border border-white/10"
              >
                <div className="flex flex-col md:flex-row md:items-center justify-between space-y-2 md:space-y-0">
                  <div className="flex-1">
                    <h4 className="text-white font-medium">{request.event_name}</h4>
                    <div className="text-sm text-white/70 space-y-1">
                      <p>{request.tickets_sold} tickets sold</p>
                      <p>Gross: KES {request.amount_kes?.toLocaleString()}</p>
                      <p>Platform Fee: KES {request.platform_fee_kes?.toLocaleString()}</p>
                      <p className="font-medium text-white">Net: KES {request.net_amount_kes?.toLocaleString()}</p>
                    </div>
                  </div>
                  
                  <div className="flex items-center space-x-3">
                    <Badge
                      variant="outline"
                      className={
                        request.status === 'pending'
                          ? 'bg-yellow-500/20 border-yellow-400 text-yellow-200'
                          : request.status === 'approved'
                          ? 'bg-blue-500/20 border-blue-400 text-blue-200'
                          : request.status === 'paid'
                          ? 'bg-green-500/20 border-green-400 text-green-200'
                          : 'bg-red-500/20 border-red-400 text-red-200'
                      }
                    >
                      {request.status === 'pending' && <Clock className="w-3 h-3 mr-1" />}
                      {request.status === 'approved' && <CheckCircle className="w-3 h-3 mr-1" />}
                      {request.status === 'paid' && <CheckCircle className="w-3 h-3 mr-1" />}
                      {request.status === 'rejected' && <XCircle className="w-3 h-3 mr-1" />}
                      {request.status.charAt(0).toUpperCase() + request.status.slice(1)}
                    </Badge>
                    
                    <div className="text-xs text-white/50">
                      {new Date(request.requested_at).toLocaleDateString()}
                    </div>
                  </div>
                </div>
                
                {request.notes && (
                  <div className="mt-2 text-sm text-white/60 italic">
                    "{request.notes}"
                  </div>
                )}
              </div>
            ))}
          </div>
        )}
      </div>
    </CardContent>
  </Card>
</motion.div>
```

### Admin Payment Request Management

Add this section to the existing `AdminDashboard.tsx` for managing organizer payment requests:

```typescript
// Add these imports to AdminDashboard.tsx
import { DollarSign, Clock, CheckCircle, XCircle, User, Calendar, CreditCard } from 'lucide-react';

// Add these states and mutations to AdminDashboard component:
const [selectedPaymentStatus, setSelectedPaymentStatus] = useState<string>('');
const [adminNotes, setAdminNotes] = useState('');
const [selectedPaymentRequest, setSelectedPaymentRequest] = useState<any>(null);

// Fetch all payment requests (admin view)
const { data: allPaymentRequests, isLoading: paymentRequestsLoading } = useQuery({
  queryKey: ['/api/admin/payment-requests'],
});

// Update payment request status mutation
const updatePaymentRequest = useMutation({
  mutationFn: async ({ id, status, notes }: any) => {
    const token = localStorage.getItem('auth_token');
    const response = await fetch(`/api/admin/payment-requests/${id}`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
      },
      body: JSON.stringify({ status, notes }),
    });
    return response.json();
  },
  onSuccess: (data) => {
    if (data.success) {
      toast({
        title: '✅ Payment Request Updated',
        description: data.message,
      });
      queryClient.invalidateQueries({ queryKey: ['/api/admin/payment-requests'] });
      setSelectedPaymentRequest(null);
      setSelectedPaymentStatus('');
      setAdminNotes('');
    } else {
      toast({
        title: 'Update Failed',
        description: data.error || 'Failed to update payment request',
        variant: 'destructive',
      });
    }
  },
});

// Add this JSX section to the AdminDashboard component (after the user approval section):

{/* Payment Requests Management */}
<motion.div
  initial={{ opacity: 0, y: 20 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.6, delay: 0.5 }}
  className="mb-8"
>
  <Card className="bg-white/10 backdrop-blur-sm border-white/20">
    <CardHeader>
      <CardTitle className="text-white flex items-center">
        <DollarSign className="w-6 h-6 mr-2" />
        Payment Request Management
      </CardTitle>
      <CardDescription className="text-white/70">
        Review and process organizer payment requests
      </CardDescription>
    </CardHeader>
    <CardContent>
      {paymentRequestsLoading ? (
        <div className="text-center text-white/70 py-8">
          <div className="animate-spin w-6 h-6 border-2 border-white border-t-transparent rounded-full mx-auto mb-2"></div>
          Loading payment requests...
        </div>
      ) : allPaymentRequests?.length === 0 ? (
        <div className="text-center text-white/70 py-8">
          <DollarSign className="w-12 h-12 text-white/30 mx-auto mb-2" />
          <p>No payment requests found</p>
        </div>
      ) : (
        <div className="space-y-4">
          {allPaymentRequests?.map((request) => (
            <div
              key={request.id}
              className="bg-white/5 rounded-lg p-4 border border-white/10"
            >
              <div className="flex flex-col lg:flex-row lg:items-start justify-between space-y-4 lg:space-y-0">
                <div className="flex-1 space-y-3">
                  <div>
                    <h4 className="text-white font-medium text-lg">{request.event_name}</h4>
                    <div className="flex items-center text-sm text-white/70 mt-1">
                      <User className="w-4 h-4 mr-1" />
                      {request.organizer_name} ({request.organizer_email})
                    </div>
                  </div>

                  <div className="grid grid-cols-2 md:grid-cols-4 gap-4 text-sm">
                    <div>
                      <p className="text-white/60">Tickets Sold</p>
                      <p className="text-white font-medium">{request.tickets_sold}</p>
                    </div>
                    <div>
                      <p className="text-white/60">Gross Revenue</p>
                      <p className="text-white font-medium">KES {request.amount_kes?.toLocaleString()}</p>
                    </div>
                    <div>
                      <p className="text-white/60">Platform Fee (3.5%)</p>
                      <p className="text-white font-medium">KES {request.platform_fee_kes?.toLocaleString()}</p>
                    </div>
                    <div>
                      <p className="text-white/60">Net Amount</p>
                      <p className="text-green-300 font-bold">KES {request.net_amount_kes?.toLocaleString()}</p>
                    </div>
                  </div>

                  {request.bank_details && (
                    <div className="bg-white/5 rounded p-3">
                      <p className="text-white/60 text-sm mb-2">Bank Details:</p>
                      {(() => {
                        try {
                          const details = JSON.parse(request.bank_details);
                          return (
                            <div className="text-sm text-white/80 space-y-1">
                              <p><strong>Account Name:</strong> {details.account_name}</p>
                              <p><strong>Account Number:</strong> {details.account_number}</p>
                              <p><strong>Bank:</strong> {details.bank_name}</p>
                            </div>
                          );
                        } catch {
                          return <p className="text-white/60 text-sm">Bank details unavailable</p>;
                        }
                      })()}
                    </div>
                  )}

                  {request.notes && (
                    <div className="text-sm text-white/70">
                      <strong>Organizer Notes:</strong> {request.notes}
                    </div>
                  )}
                </div>

                <div className="flex flex-col items-end space-y-3">
                  <Badge
                    variant="outline"
                    className={
                      request.status === 'pending'
                        ? 'bg-yellow-500/20 border-yellow-400 text-yellow-200'
                        : request.status === 'approved'
                        ? 'bg-blue-500/20 border-blue-400 text-blue-200'
                        : request.status === 'paid'
                        ? 'bg-green-500/20 border-green-400 text-green-200'
                        : 'bg-red-500/20 border-red-400 text-red-200'
                    }
                  >
                    {request.status === 'pending' && <Clock className="w-3 h-3 mr-1" />}
                    {request.status === 'approved' && <CheckCircle className="w-3 h-3 mr-1" />}
                    {request.status === 'paid' && <CreditCard className="w-3 h-3 mr-1" />}
                    {request.status === 'rejected' && <XCircle className="w-3 h-3 mr-1" />}
                    {request.status.charAt(0).toUpperCase() + request.status.slice(1)}
                  </Badge>

                  <div className="text-xs text-white/50 text-right">
                    <p>Requested: {new Date(request.requested_at).toLocaleDateString()}</p>
                    {request.processed_at && (
                      <p>Processed: {new Date(request.processed_at).toLocaleDateString()}</p>
                    )}
                  </div>

                  {request.status === 'pending' && (
                    <div className="flex space-x-2">
                      <Button
                        size="sm"
                        onClick={() => {
                          updatePaymentRequest.mutate({
                            id: request.id,
                            status: 'approved',
                            notes: 'Payment approved by admin'
                          });
                        }}
                        disabled={updatePaymentRequest.isPending}
                        className="bg-blue-500 hover:bg-blue-600"
                      >
                        <CheckCircle className="w-4 h-4 mr-1" />
                        Approve
                      </Button>
                      <Button
                        size="sm"
                        onClick={() => {
                          updatePaymentRequest.mutate({
                            id: request.id,
                            status: 'rejected',
                            notes: 'Payment request rejected'
                          });
                        }}
                        disabled={updatePaymentRequest.isPending}
                        variant="destructive"
                      >
                        <XCircle className="w-4 h-4 mr-1" />
                        Reject
                      </Button>
                    </div>
                  )}

                  {request.status === 'approved' && (
                    <Button
                      size="sm"
                      onClick={() => {
                        updatePaymentRequest.mutate({
                          id: request.id,
                          status: 'paid',
                          notes: 'Payment completed'
                        });
                      }}
                      disabled={updatePaymentRequest.isPending}
                      className="bg-green-500 hover:bg-green-600"
                    >
                      <CreditCard className="w-4 h-4 mr-1" />
                      Mark as Paid
                    </Button>
                  )}
                </div>
              </div>
            </div>
          ))}
        </div>
      )}
    </CardContent>
  </Card>
</motion.div>
```

## 📋 Step 7: Final Environment Variables Setup

## 📋 Step 7: Environment Variables Setup

Create these environment variables in Replit Secrets:

```
DATABASE_URL=postgresql://username:password@host/database
JWT_SECRET=your-super-secret-jwt-key-here
BREVO_SMTP_PASSWORD=your-brevo-smtp-password
CODEREADR_API_KEY=your-codereadr-api-key
PAYSTACK_SECRET_KEY=sk_test_your-paystack-secret-key
NODE_ENV=development
PORT=5000
```

## 📋 Step 8: Development Workflow

### Package.json Scripts
```json
{
  "scripts": {
    "dev": "NODE_ENV=development tsx server/index.ts",
    "build": "tsc && vite build",
    "start": "NODE_ENV=production node dist/server/index.js",
    "db:push": "drizzle-kit push:pg",
    "db:generate": "drizzle-kit generate:pg"
  }
}
```

### Vite Configuration
Create `vite.config.ts`:
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './client/src'),
      '@shared': path.resolve(__dirname, './shared'),
      '@assets': path.resolve(__dirname, './attached_assets'),
    },
  },
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
      },
    },
  },
});
```

## 📋 Step 9: Testing and Deployment

### Test the Complete System
1. **Start the development server**: `npm run dev`
2. **Test guest checkout** with Paystack test cards
3. **Verify email delivery** with real email addresses
4. **Test QR code scanning** with CodeREADr app
5. **Check admin dashboard** functionality
6. **Test box office sales** with different payment methods

### Deploy to Production
1. **Set production environment variables** in Replit Secrets
2. **Run database migrations**: `npm run db:push`
3. **Test with production payment keys**
4. **Monitor server logs** for any startup errors
5. **Verify all integrations** are working correctly

## 🎯 Success Checklist

✅ **Database**: PostgreSQL with all tables created  
✅ **Authentication**: Multi-role JWT system working  
✅ **Payments**: Paystack integration processing real transactions  
✅ **Tickets**: PDF generation with QR codes  
✅ **Email**: Automated ticket delivery via Brevo  
✅ **Scanning**: CodeREADr integration for validation  
✅ **Box Office**: Physical sales interface  
✅ **Analytics**: Real-time dashboard with revenue tracking  
✅ **Waiting Rooms**: High-traffic management  
✅ **Deployment**: Robust error handling and validation  

This complete guide will create a production-ready African event ticketing platform with all the advanced features, integrations, and deployment optimizations needed for real-world use.