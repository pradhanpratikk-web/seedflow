# SeedFlow

SeedFlow generates realistic, reproducible test datasets for JavaScript applications. Define your own schemas, connect records with foreign keys, use ready-made e-commerce and social scenarios, and export datasets as JSON or CSV.

## Installation

```bash
npm install seedflow
```

SeedFlow requires Node.js 20 or newer and currently exposes an ES module API.

## Quick start

```js
import { createSeeder } from 'seedflow';

const seed = createSeeder({ seed: 12345 });
const users = seed.generate('user', 10); // built-in user schema
const data = seed.generateScenario({
  preset: 'ecommerce',
  counts: { users: 10, products: 20, orders: 30 }
});

console.log(data.orders[0]);
await seed.exportJSON(data, './output/store.json');
await seed.exportCSV(data.users, './output/users.csv');
```

## Custom schemas

`defineSchema()` validates a schema and registers it with that seeder. Supported types include `string`, `number`, `integer`, `boolean`, `email`, `uuid`, `fullName`, `firstName`, `lastName`, `city`, `address`, `date`, `company`, `phone`, `url`, and `word`. `enum`, `array`, and `custom` are also available.

Each seeder starts with a `user` schema containing `id`, `name`, `email`, `age`, `city`, and `isActive`. Define your own schema named `user` to replace it.

```js
const seed = createSeeder({ seed: 'nightly-build' });
seed.defineSchema({
  name: 'user',
  fields: {
    id: 'uuid',
    name: 'fullName',
    email: 'email',
    age: { type: 'integer', min: 18, max: 60 },
    nickname: { type: 'string', optional: true },
    tags: { type: 'array', of: 'word', minLength: 1, maxLength: 3 },
    role: { type: 'enum', values: ['member', 'admin'] },
    createdAt: { type: 'date', min: '2024-01-01', max: '2025-01-01' },
    greeting: { type: 'custom', generate: ({ row }) => `Hello ${row.name}` }
  }
});
const users = seed.generate('user', 10);
```

Optional fields are `null` 20% of the time by default; set `optionalProbability` between 0 and 1 to change that. Numeric bounds are inclusive for integers; decimal `number` values use two decimal places by default.

## Relationships

Generate the target dataset first, then define a `relation` field to pick a real target key. This keeps each foreign key tied to an existing record.

```js
seed.defineSchema({ name: 'post', fields: {
  id: 'uuid',
  authorId: { type: 'relation', target: 'user', key: 'id' },
  title: 'word'
} });
const posts = seed.generate('post', 25);
```

If the target schema has not generated records or its key is missing, generation throws an actionable error. `assertRelations(data, relations)` can verify relationships across multiple datasets.

## Deterministic generation

The same seed, schema definitions, and generation order produce the same records. Number and string seeds are supported. Different seeds produce different data. `reset()` clears generated datasets and restores the initial random state.

## Built-in presets

### E-commerce

```js
const store = seed.generateScenario({ preset: 'ecommerce', counts: {
  users: 20, products: 50, categories: 8, orders: 100, addresses: 20
} });
// store contains users, products, categories, orders, orderItems, and addresses
```

Each order has one or more order items. Orders point to generated users; order items point to generated orders and products. Products point to generated categories.

### Social media

```js
const community = seed.generateScenario({ preset: 'social', counts: {
  users: 25, posts: 80, comments: 120, likes: 200
} });
// community contains users, posts, comments, and likes
```

Posts, comments, and likes reference generated users and posts.

Preset counts default to useful sample sizes. For connected datasets, keep related target counts above zero whenever dependent records are requested.

## CLI

```bash
npx seedflow init
npx seedflow generate --preset ecommerce
npx seedflow generate --preset social --seed 12345 --output ./data
npx seedflow generate --config seedflow.config.js --output ./data
seedflow --help
```

`init` creates a starter `seedflow.config.js` without overwriting an existing file. Pass `--force` to replace it. A config default-exports `{ seed, preset, counts }` for a preset, or `{ seed, schemas, counts }` for custom schemas. For example:

```js
export default {
  seed: 123,
  schemas: [
    { name: 'user', fields: { id: 'uuid', name: 'fullName' } },
    { name: 'post', fields: { id: 'uuid', authorId: { type: 'relation', target: 'user' } } }
  ],
  counts: { user: 5, post: 12 }
};
```

CLI output is written as one JSON file per dataset and includes record totals and file paths.

## Export API

```js
await seed.exportJSON(data, './output/data.json');
await seed.exportCSV(data.users, './output/users.csv');
```

Both exporters create parent directories and report filesystem errors with the target path. CSV uses the union of record keys as columns, quotes and escapes commas, quotes, and newlines, and serializes nested values as JSON strings. The module also exports standalone `exportJSON(data, path)` and `exportCSV(rows, path)` functions.

## API reference

- `createSeeder({ seed })`: create an isolated seeded generator.
- `seed.defineSchema(schema)`: validate and register a custom schema.
- `seed.generate(name, count)`: append and return `count` records for a registered schema.
- `seed.generateScenario({ preset, counts })`: generate an e-commerce or social preset; without `preset`, generate the named custom schemas by count.
- `seed.getDataset(name)`: return a shallow copy of accumulated records.
- `seed.reset()`: clear datasets and restart the seeded random stream.
- `seed.exportJSON(data, path)` / `seed.exportCSV(rows, path)`: asynchronously write output files.
- `defineSchema(schema)`, `validateSchema(schema)`, `assertRelations(data, relations)`, `exportJSON(data, path)`, and `exportCSV(rows, path)` are also named exports.

## Errors and limitations

Invalid field types, schema shapes, ranges, counts, unknown schema names, missing relation targets, and file-writing failures throw errors. Presets use Faker.js for realistic values but their entity structures are intentionally small and are not database migrations. Preset relationships cannot be fulfilled when a requested dependent dataset has no possible target records. Custom relation targets must be generated before dependent records. SeedFlow currently supports ESM imports; synchronous CommonJS loading is not provided. The CLI config is trusted JavaScript and should only be loaded from files you control.

## Contributing

Install dependencies with `npm install`, run `npm test`, and open a pull request with focused changes and tests. Keep generators deterministic and validate public inputs.

## License

MIT. See [LICENSE](LICENSE).
