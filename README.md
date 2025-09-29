<p align="center">
  <h1>⚡️ Express.js Best Practices & Architecture Guide</h1>
</p>

## 🚀 Why Choose Express.js?

Express.js remains the **de facto standard** for Node.js web applications due to its:

- 🎯 **Minimalist Philosophy**: Provides essential web application features without unnecessary bloat
- 🌟 **Mature Ecosystem**: Extensive middleware library and community support
- ⚡ **Performance**: Lightweight and fast, ideal for high-throughput applications
- 🔧 **Flexibility**: Unopinionated framework allowing architectural freedom
- 📚 **Learning Curve**: Gentle introduction to backend development concepts

### 🔷 Why TypeScript for Large Projects?

For **enterprise-grade applications**, TypeScript is essential:

- 🛡️ **Type Safety**: Prevents runtime errors through compile-time checking
- 💡 **Better IDE Support**: Enhanced autocomplete, refactoring, and navigation
- 👥 **Team Collaboration**: Self-documenting code with clear interfaces
- 🔄 **Maintainability**: Easier refactoring and code evolution
- 🐛 **Developer Experience**: Improved debugging and error messages

> **Note**: While TypeScript adds complexity initially, it becomes invaluable as projects scale beyond 10k+ lines of code.

---

## 🏗️ Project Structure

### 🏛️ Core Architecture

```
src/
├── 📁 modules/                 # Feature-based modules
│   ├── users/
│   ├── auth/
│   ├── products/
│   └── orders/
├── 📁 integrations/           # External service integrations
│   ├── gcp/               # Google Cloud Platform services
│   ├── aws/               # Amazon Web Services
│   ├── payments/          # Payment processors
│   └── ai/                # AI service integrations
├── 📁 shared/                # Shared utilities and types
│   ├── types/
│   ├── utils/
│   ├── middleware/
│   └── constants/
├── 📁 config/                # Configuration files
└── 📁 app.ts                 # Application entry point
```

### 📋 Module Structure Pattern

Each module follows a consistent structure:

```
src/modules/users/
├── index.ts              # Module exports
├── users.controller.ts   # HTTP request handlers
├── users.service.ts      # Business logic
├── users.dto.ts          # Data Transfer Objects (Zod schemas)
├── users.bo.ts           # Business Objects/Models
├── users.types.ts        # TypeScript interfaces
└── users.routes.ts       # Route definitions
```

---

## 📚 Module Examples

### 👤 Users Module

```typescript
import { z } from "zod";

export const CreateUserDto = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  firstName: z.string().min(1),
  lastName: z.string().min(1),
});

export const UpdateUserDto = CreateUserDto.partial();

export type CreateUserRequest = z.infer<typeof CreateUserDto>;
export type UpdateUserRequest = z.infer<typeof UpdateUserDto>;
```

```typescript
import { Request, Response } from "express";
import { UserService } from "./users.service";
import { CreateUserDto } from "./users.dto";

export class UserController {
  constructor(private userService: UserService) {}

  async createUser(req: Request, res: Response) {
    const userData = CreateUserDto.parse(req.body);
    const user = await this.userService.createUser(userData);
    res.status(201).json(user);
  }

  async getUsers(req: Request, res: Response) {
    const users = await this.userService.getUsers();
    res.json(users);
  }
}
```

### 🔐 Authentication Module

```typescript
import jwt from "jsonwebtoken";
import argon2 from "argon2";

export class AuthService {
  async hashPassword(password: string): Promise<string> {
    return argon2.hash(password);
  }

  async verifyPassword(hash: string, password: string): Promise<boolean> {
    return argon2.verify(hash, password);
  }

  generateToken(userId: string): string {
    return jwt.sign({ userId }, process.env.JWT_SECRET!, { expiresIn: "7d" });
  }
}
```

---

## 🔌 Integration Patterns

### ☁️ Google Cloud Platform

```typescript
import { CloudTasksClient } from "@google-cloud/tasks";

export class CloudTasksService {
  private client = new CloudTasksClient();

  async scheduleTask(queuePath: string, payload: any, delaySeconds?: number) {
    const task = {
      httpRequest: {
        httpMethod: "POST",
        url: process.env.TASK_HANDLER_URL,
        body: Buffer.from(JSON.stringify(payload)),
      },
    };

    if (delaySeconds) {
      task.scheduleTime = {
        seconds: Date.now() / 1000 + delaySeconds,
      };
    }

    return this.client.createTask({ parent: queuePath, task });
  }
}
```

> **Why Cloud Tasks over Cron?**
>
> - 📈 **Scalability**: Automatic scaling based on queue depth
> - 🔒 **Reliability**: Built-in retry mechanisms and dead letter queues
> - 📊 **Monitoring**: Integrated with Google Cloud monitoring
> - 💰 **Cost-Effective**: Pay per task execution

### 🟠 AWS Integration

```typescript
import { SQSClient, SendMessageCommand } from "@aws-sdk/client-sqs";

export class SQSService {
  private client = new SQSClient({ region: process.env.AWS_REGION });

  async sendMessage(queueUrl: string, messageBody: any) {
    const command = new SendMessageCommand({
      QueueUrl: queueUrl,
      MessageBody: JSON.stringify(messageBody),
    });

    return this.client.send(command);
  }
}
```

### 💳 Payment Integrations

```typescript
import Stripe from "stripe";

export class StripeService {
  private stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

  async createPaymentIntent(amount: number, currency: string = "usd") {
    return this.stripe.paymentIntents.create({
      amount: amount * 100, // Convert to cents
      currency,
      automatic_payment_methods: { enabled: true },
    });
  }
}
```

```typescript
import paypal from "@paypal/checkout-server-sdk";

export class PayPalService {
  private client: paypal.core.PayPalHttpClient;

  constructor() {
    const environment =
      process.env.NODE_ENV === "production"
        ? new paypal.core.LiveEnvironment(
            process.env.PAYPAL_CLIENT_ID!,
            process.env.PAYPAL_CLIENT_SECRET!
          )
        : new paypal.core.SandboxEnvironment(
            process.env.PAYPAL_CLIENT_ID!,
            process.env.PAYPAL_CLIENT_SECRET!
          );

    this.client = new paypal.core.PayPalHttpClient(environment);
  }
}
```

### 🤖 AI Integrations

```typescript
import OpenAI from "openai";

export class OpenAIService {
  private client = new OpenAI({
    apiKey: process.env.OPENAI_API_KEY,
  });

  async generateCompletion(prompt: string, maxTokens: number = 150) {
    const response = await this.client.chat.completions.create({
      model: "gpt-4",
      messages: [{ role: "user", content: prompt }],
      max_tokens: maxTokens,
    });

    return response.choices[0]?.message?.content;
  }
}
```

---

## 🗄️ Database Recommendations

### 🗃️ SQL Databases

**Prisma** _(Recommended for TypeScript projects)_

```typescript
// prisma/schema.prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  firstName String
  lastName  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

**Drizzle** _(Lightweight alternative)_

```typescript
import { pgTable, varchar, timestamp } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: varchar("id").primaryKey(),
  email: varchar("email").notNull().unique(),
  firstName: varchar("first_name").notNull(),
  lastName: varchar("last_name").notNull(),
  createdAt: timestamp("created_at").defaultNow(),
});
```

### 🍃 MongoDB

**Mongoose** _(Recommended for MongoDB)_

```typescript
import mongoose from "mongoose";

const userSchema = new mongoose.Schema(
  {
    email: { type: String, required: true, unique: true },
    firstName: { type: String, required: true },
    lastName: { type: String, required: true },
  },
  { timestamps: true }
);

export const User = mongoose.model("User", userSchema);
```

---

## 🛠️ Tooling & Development

### 🎨 Code Quality - Biome

```json
{
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2
  }
}
```

> **Why Biome?**
>
> - ⚡ **Performance**: 100x faster than ESLint
> - 🎯 **Zero Configuration**: Works out of the box
> - 🦀 **Rust-Based**: Extremely fast linting and formatting
> - 🔧 **All-in-One**: Combines linting, formatting, and more

### 🔐 Authentication Strategies

**🔑 JWT Implementation**

```typescript
import jwt from "jsonwebtoken";
import { Request, Response, NextFunction } from "express";

export const authenticateToken = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const token = req.headers.authorization?.split(" ")[1];

  if (!token) {
    return res.status(401).json({ error: "Access token required" });
  }

  jwt.verify(token, process.env.JWT_SECRET!, (err, user) => {
    if (err) return res.status(403).json({ error: "Invalid token" });
    req.user = user;
    next();
  });
};
```

**🎫 Passport.js Alternative**

```typescript
import passport from "passport";
import { Strategy as JwtStrategy, ExtractJwt } from "passport-jwt";
```

### 🔒 Password Hashing

**🛡️ Argon2** _(Recommended - Winner of password hashing competition)_

```typescript
import argon2 from "argon2";

// More secure, modern algorithm
const hash = await argon2.hash(password);
const isValid = await argon2.verify(hash, password);
```

**🔐 bcryptjs** _(Alternative - Well established)_

```typescript
import bcrypt from "bcryptjs";

// Traditional, widely adopted
const hash = await bcrypt.hash(password, 12);
const isValid = await bcrypt.compare(password, hash);
```

---

## 🏛️ Key Architectural Notes

### 📁 **Module Organization**

- 🎯 Each feature is self-contained within its module
- 🔄 Clear separation of concerns (Controller → Service → Repository)
- 📝 Consistent naming conventions across all modules

### 🔌 **Integration Strategy**

- ☁️ Cloud services over self-hosted solutions for scalability
- 🔄 Provider-agnostic interfaces for easy switching
- 🌍 Environment-based configuration for different deployment targets

### 🛡️ **Type Safety**

- ✅ Zod for runtime validation and TypeScript type generation
- 🔒 Strict TypeScript configuration in enterprise environments
- 📋 Clear interfaces between modules and external services

### ⚡ **Performance Considerations**

- 🚀 Use Cloud Tasks/SQS for background processing instead of blocking operations
- 📊 Implement proper database indexing strategies
- 💾 Cache frequently accessed data with Redis or similar

### 🔒 **Security Best Practices**

- 🔐 Always hash passwords with modern algorithms (Argon2 preferred)
- 🎫 Implement proper JWT token management with refresh tokens
- 🔑 Use environment variables for sensitive configuration
- ✅ Validate all inputs with Zod schemas

---

_This architecture scales from small prototypes to enterprise applications while maintaining code quality and developer productivity._
