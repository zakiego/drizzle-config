# Drizzle ORM Config

## PostgreSQL

```bash
pnpm add drizzle-orm postgres
pnpm add -D drizzle-kit
pnpm add zod tsx
```

### 1. Lib

`src/lib/env.ts`

```typescript
import { z } from "zod";

const schema = z.object({
  DATABASE_URL: z.string().url(),
  NODE_ENV: z.enum(["development", "production"]),
});

export type Env = z.infer<typeof schema>;

export const env = schema.parse(process.env);
```

### 2. Drizzle Config

`drizzle.config.ts`

```typescript
import { env } from "@/lib/env";
import type { Config } from "drizzle-kit";

export default {
  schema: "./src/db/*.ts",
  out: "./src/db/migrations",
  driver: "pg",
  dbCredentials: {
    connectionString: env.DATABASE_URL,
  },
} satisfies Config;
```

### 3. Database Schema

`src/db/schema.ts`

```typescript
import { integer, pgEnum, pgTable, serial, uniqueIndex, varchar } from 'drizzle-orm/pg-core';
// declaring enum in database
export const popularityEnum = pgEnum('popularity', ['unknown', 'known', 'popular']);
export const countries = pgTable('countries', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 256 }),
}, (countries) => {
  return {
    nameIndex: uniqueIndex('name_idx').on(countries.name),
  }
});
export const cities = pgTable('cities', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 256 }),
  countryId: integer('country_id').references(() => countries.id),
  popularity: popularityEnum('popularity'),
});
```

### 4. Database Index

`src/db/index.ts`

```typescript
import { cities, countries } from "@/db/schema";
import { env } from "@/lib/env";
import { PostgresJsDatabase, drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";

export const dbSchema = {
  countries,
  cities,
};

export const migrationClient = drizzle(postgres(env.DATABASE_URL, { max: 1 }));
const queryClient = postgres(env.DATABASE_URL, { max: 3 });

declare global {
  // biome-ignore lint/style/noVar: <explanation>
  var dbClient: PostgresJsDatabase<typeof dbSchema> | undefined;
}

// biome-ignore lint/suspicious/noRedeclare: <explanation>
let dbClient: PostgresJsDatabase<typeof dbSchema>;

if (env.NODE_ENV === "production") {
  const client = postgres(env.DATABASE_URL);

  dbClient = drizzle(client, {
    schema: dbSchema,
  });
} else {
  if (!global.dbClient) {
    const client = postgres(env.DATABASE_URL);

    global.dbClient = drizzle(client, {
      schema: dbSchema,
    });
  }

  dbClient = global.dbClient;
}

export { dbClient };
```

### 5. Database Migrations

`src/db/migrate.ts`

```typescript
import { migrate } from "drizzle-orm/postgres-js/migrator";
import { migrationClient } from "./index";

const main = async () => {
  try {
    await migrate(migrationClient, { migrationsFolder: "./src/db/migrations" });

    console.log("Migration success");
  } catch (error) {
    console.error(`Migration error: ${error}`);
  }
};

main().then(() => process.exit(0));
```

### 6. Scripts

`package.json`

```json
{
  "scripts": {
    "db:migrate": "tsx src/db/migrate.ts",
    "db:generate": "drizzle-kit generate:pg",
    "db:pull": "#drizzle-kit introspect:pg",
    "db:push": "#drizzle-kit push:pg",
    "db:rollback": "drizzle-kit drop",
    "db:check": "drizzle-kit check:pg",
    "db:studio": "drizzle-kit studio"
  }
}
```
