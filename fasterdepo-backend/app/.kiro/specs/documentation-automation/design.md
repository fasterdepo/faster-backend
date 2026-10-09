# Design Document: Documentation Automation

## Overview

The Documentation Automation feature creates a comprehensive, automated documentation pipeline for the BANKERCHANGER API and codebase. This system extracts and generates documentation from multiple sources—JSDoc comments, Drizzle ORM schemas, TypeScript types, and React components—creating a unified documentation site with OpenAPI/Swagger specs, TypeScript types, Postman collections, TSDoc coverage, Storybook components, and an HTML documentation site with version history.

The primary goal is to reduce manual documentation maintenance by automating generation from code comments, type definitions, and database schema, while ensuring all public functions and API endpoints are well-documented and discoverable.

### Key Goals

1. **Reduce manual documentation overhead** by auto-generating from code
2. **Ensure API documentation stays in sync** with implementation
3. **Generate type-safe TypeScript interfaces** from OpenAPI specs
4. **Improve developer onboarding** with comprehensive, discoverable docs
5. **Maintain documentation versioning** alongside code releases
6. **Increase TSDoc coverage** for all public functions

---

## Architecture

### High-Level Pipeline

```
Source Code & Schemas
  ├── JSDoc Comments in Controllers → OpenAPI Spec
  ├── OpenAPI Spec → TypeScript Types (via openapi-typescript)
  ├── Drizzle ORM Schema → Schema Documentation
  ├── TSDoc in Services → TypeDoc HTML Documentation
  └── React Components → Storybook

Generated Artifacts
  ├── openapi.yaml / openapi.json
  ├── Generated TypeScript types (*.generated.ts)
  ├── Postman Collection
  ├── Schema Documentation (HTML/Markdown)
  ├── TypeDoc HTML Site
  ├── Storybook Static Site
  └── HTML Documentation Site (built from TypeDoc + Storybook + ADRs)

Deployment
  └── GitHub Pages / Netlify (with version history)
```

### Documentation Generation Flow

```
1. Extract OpenAPI from JSDoc
   └─> Validate OpenAPI spec
   └─> Generate TypeScript types
   └─> Generate Postman collection

2. Generate Schema Documentation
   └─> Parse Drizzle schema.ts
   └─> Generate HTML/Markdown tables
   └─> Include validation rules

3. Generate TypeDoc HTML
   └─> Parse all public functions with TSDoc
   └─> Generate searchable HTML site
   └─> Create navigation and indices

4. Integrate Storybook
   └─> Stories for React components
   └─> Live component preview
   └─> Props documentation

5. Build Documentation Site
   └─> Combine all artifacts
   └─> Add search functionality
   └─> Implement version history
   └─> Deploy to hosting

6. Create ADRs
   └─> Document architectural decisions
   └─> Track design rationale
```

---

## Components and Interfaces

### 1. OpenAPI Spec Generation from JSDoc

**Source**: JSDoc comments in controllers
**Output**: `openapi.yaml`, `openapi.json`

#### JSDoc Pattern

Controllers should follow this pattern for documenting endpoints:

```typescript
/**
 * @swagger
 * /api/markets:
 *   get:
 *     summary: List all markets
 *     description: Retrieve a paginated list of all available prediction markets
 *     operationId: listMarkets
 *     tags:
 *       - Markets
 *     parameters:
 *       - name: status
 *         in: query
 *         schema: { type: string, enum: [open, locked, resolved] }
 *       - name: page
 *         in: query
 *         schema: { type: integer, default: 1 }
 *     responses:
 *       '200':
 *         description: List of markets
 *         content:
 *           application/json:
 *             schema:
 *               type: array
 *               items: { $ref: '#/components/schemas/Market' }
 *       '400':
 *         description: Invalid query parameters
 */
async function listMarkets(req: Request, res: Response) {
  // Implementation
}
```

**Tool**: `swagger-jsdoc` already configured
**Validation**: `scripts/validate-openapi.js` ensures spec compliance

---

### 2. TypeScript Type Generation Pipeline

**Input**: `openapi.json`
**Output**: `src/types/api.generated.ts`
**Tool**: `openapi-typescript`

#### Integration Points

```typescript
// Generated file: src/types/api.generated.ts
export interface Market {
  id: number;
  market_id: string;
  contract_address: string;
  // ... all fields from OpenAPI schema
}

export interface ListMarketsQuery {
  status?: 'open' | 'locked' | 'resolved';
  page?: number;
}

export interface ListMarketsResponse {
  data: Market[];
  pagination: Pagination;
}
```

**Usage in Controllers**:
```typescript
import { ListMarketsQuery, ListMarketsResponse } from '@/types/api.generated';

async function listMarkets(
  req: Request<{}, {}, {}, ListMarketsQuery>,
  res: Response<ListMarketsResponse>
): Promise<void> {
  // Implementation with type safety
}
```

**Generation Command**:
```bash
openapi-typescript openapi.json -o src/types/api.generated.ts
```

---

### 3. Schema Documentation Generation

**Input**: `src/db/schema.ts`
**Output**: `docs/schema/index.md` and `docs/schema/*.html`

#### Schema Parser

Parses Drizzle ORM schema and generates:

```markdown
# Database Schema Reference

## Markets Table

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | SERIAL | PRIMARY KEY | Unique market identifier |
| market_id | TEXT | UNIQUE, NOT NULL | Stellar contract market ID |
| contract_address | TEXT | NOT NULL | Stellar contract address |
| status | TEXT | DEFAULT 'open' | Market lifecycle state |
| outcome | TEXT | NULLABLE | Fighter A, B, Draw, or No Contest |
| fee_bps | INTEGER | DEFAULT 200 | Platform fee in basis points |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp |

### Indexes

- `markets_market_id_idx`: UNIQUE on market_id
- `markets_status_idx`: INDEX on status
- `markets_scheduled_at_idx`: INDEX on scheduled_at

### Constraints

- Foreign Key: `market_id` references from bets table
```

#### Implementation

- Parse `pgTable()` definitions from schema.ts
- Extract column definitions, types, constraints
- Generate Markdown and HTML documentation
- Include relationships and indexes

---

### 4. TSDoc and TypeDoc Configuration

**Goal**: 100% documentation of public functions with searchable HTML

#### TSDoc Standards for Public Functions

All public functions in services and controllers should follow TSDoc:

```typescript
/**
 * Retrieves all active markets matching the specified criteria.
 *
 * @param status - Filter by market status (open, locked, resolved)
 * @param page - Page number for pagination (default: 1)
 * @param limit - Results per page (default: 20, max: 100)
 * @returns Promise resolving to an array of Market objects
 * @throws {BadRequestError} If query parameters are invalid
 * @throws {InternalServerError} If database query fails
 *
 * @example
 * ```typescript
 * const markets = await listMarkets({ status: 'open', page: 1 });
 * console.log(markets.length); // Number of open markets
 * ```
 *
 * @remarks
 * - Results are cached for 60 seconds
 * - Requires no authentication
 * - Rate limited to 60 requests/min per IP
 *
 * @see {@link MarketController} for endpoint routing
 */
export async function listMarkets(
  status?: 'open' | 'locked' | 'resolved',
  page: number = 1,
  limit: number = 20,
): Promise<Market[]> {
  // Implementation
}
```

#### TypeDoc Configuration

**File**: `typedoc.json`

```json
{
  "entryPoints": ["src/"],
  "out": "docs/typedoc",
  "theme": "default",
  "includeVersion": true,
  "gitRevision": "main",
  "plugins": ["typedoc-plugin-missing-exports"],
  "name": "BANKERCHANGER API Reference",
  "readme": "README.md",
  "excludePrivate": true,
  "excludeInternal": true,
  "excludeExternals": false,
  "hideGenerator": false,
  "disableSources": false,
  "searchInComments": true,
  "skipErrorChecking": false
}
```

**Outputs**:
- HTML documentation site at `docs/typedoc/index.html`
- Searchable interface with source code references
- Module hierarchy and dependency graph

---

### 5. Storybook Setup for React Components

**Status**: Not yet implemented; this design covers setup

**Directory Structure**:
```
frontend/
  src/
    components/
      Button/
        Button.tsx          # Component
        Button.stories.tsx  # Storybook story
        Button.test.tsx     # Unit tests
  .storybook/
    main.ts               # Storybook config
    preview.ts            # Global decorators
```

**Story Example**:
```typescript
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  tags: ['autodocs'],
  argTypes: {
    onClick: { action: 'clicked' },
    disabled: { control: 'boolean' },
    size: { control: 'inline-radio', options: ['sm', 'md', 'lg'] },
  },
};

export default meta;

export const Primary: StoryObj<typeof Button> = {
  args: {
    children: 'Click me',
    variant: 'primary',
  },
};

export const Disabled: StoryObj<typeof Button> = {
  args: {
    children: 'Disabled',
    disabled: true,
  },
};
```

**Storybook Build Output**:
- Static site at `frontend/storybook-static/`
- Embedded in documentation site

---

### 6. Architecture Decision Records (ADRs)

**Directory**: `docs/adr/`

#### ADR Template

**File**: `docs/adr/0001-payment-settlement.md`

```markdown
# ADR-0001: Stellar-Based Payment Settlement

## Status
Accepted

## Context
BANKERCHANGER needs a payment mechanism for bet settlements that is:
- Decentralized and trustless
- Fast and low-cost
- Supports multiple currencies
- Compliant with financial regulations

## Decision
Use Stellar blockchain for all bet settlements via smart contracts.

## Rationale
- Stellar Foundation's primary use case is financial payments
- Native multi-asset support
- Efficient consensus protocol (~5s finality)
- Established regulatory relationships

## Consequences

### Positive
- Trustless settlement without custodian
- Clear audit trail on immutable ledger
- Sub-second confirmation potential

### Negative
- Dependency on Stellar infrastructure
- Network fees (though minimal)
- Complexity in error recovery

## Alternatives Considered
1. Ethereum - Higher gas fees, slower
2. Polygon - Good but less established in fintech

## Related Decisions
- ADR-0002: Oracle Consensus for Match Outcomes
```

**Index File**: `docs/adr/index.md` lists all ADRs

---

### 7. Postman Collection Generation

**Input**: `openapi.json`
**Output**: `postman/BANKERCHANGER.postman_collection.json`

**Existing Script**: `scripts/generate-postman.js`

**Enhancement**: Add environment variables and pre/post request scripts

```json
{
  "info": {
    "name": "BANKERCHANGER API",
    "version": "0.2.0"
  },
  "auth": {
    "type": "bearer",
    "bearer": [{ "key": "token", "value": "{{access_token}}", "type": "string" }]
  },
  "item": [
    {
      "name": "Markets",
      "item": [
        {
          "name": "List Markets",
          "request": {
            "method": "GET",
            "url": { "raw": "{{base_url}}/api/markets?status=open", "host": ["{{base_url}}"] },
            "header": [{ "key": "Authorization", "value": "Bearer {{access_token}}" }]
          }
        }
      ]
    }
  ]
}
```

---

### 8. Documentation Site Deployment Strategy

**Platform**: GitHub Pages (with Netlify as fallback)

#### Structure

```
docs/
  ├── index.html              # Main landing page
  ├── v0.2.0/                 # Version-specific docs
  │   ├── api/                # Swagger UI + TypeScript types
  │   ├── schema/             # Database schema reference
  │   ├── guide/              # Getting started, examples
  │   ├── adr/                # Architecture decision records
  │   └── components/         # Storybook (iframe or link)
  ├── v0.1.0/                 # Previous version (archived)
  ├── latest/                 # Symlink to current version
  └── search-index.json       # Full-text search index
```

#### Build Process

1. Extract version from `package.json`
2. Generate all documentation artifacts
3. Build Storybook static site
4. Combine into `docs/` directory
5. Generate search index
6. Build version-aware navigation

#### Versioning

- New release triggers documentation build
- Previous versions archived but accessible
- Search index includes all versions
- Latest version pinned in navigation

---

## Data Models

### Documentation Metadata

```typescript
interface DocumentationConfig {
  version: string;
  title: string;
  description: string;
  buildDate: Date;
  apiVersion: string;
  schemaVersion: number;
  searchEnabled: boolean;
}

interface DocumentedFunction {
  name: string;
  filePath: string;
  signature: string;
  description: string;
  parameters: DocumentedParameter[];
  returnType: string;
  examples: string[];
  throws: ThrownError[];
  relatedFunctions: string[];
}

interface DocumentedParameter {
  name: string;
  type: string;
  description: string;
  optional: boolean;
  default?: unknown;
}

interface SchemaTable {
  name: string;
  description: string;
  columns: SchemaColumn[];
  indexes: SchemaIndex[];
  constraints: SchemaConstraint[];
}

interface SchemaColumn {
  name: string;
  type: string;
  nullable: boolean;
  default?: string;
  description: string;
}
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: OpenAPI Type Round-Trip

*For any* valid OpenAPI schema, generating TypeScript types and then re-serializing the types back to JSON should produce an equivalent schema structure.

**Validates: OpenAPI type generation consistency**

### Property 2: JSDoc Extraction Preserves Structure

*For any* controller method with valid JSDoc annotations, the extracted OpenAPI operation should include all parameters, request body schema, and response schemas specified in the JSDoc comment.

**Validates: JSDoc-to-OpenAPI extraction accuracy**

### Property 3: Schema Documentation Completeness

*For any* Drizzle table definition, the generated schema documentation should include all columns, indexes, constraints, and relationships present in the table definition.

**Validates: Schema documentation covers entire schema**

### Property 4: TSDoc Coverage Detection

*For any* public function in the codebase, the documentation generator should either produce a TSDoc entry or flag it as missing documentation.

**Validates: Documentation coverage tracking**

### Property 5: Postman Collection Synchronization

*For any* endpoint in the OpenAPI spec, the generated Postman collection should include an equivalent request with matching parameters, authentication, and response schemas.

**Validates: Postman collection completeness**

---

## Error Handling

### Generation Errors

| Error | Cause | Recovery |
|-------|-------|----------|
| Invalid OpenAPI YAML | Malformed JSDoc or YAML syntax | Validate with `swagger-parser`, provide error line number |
| Schema Parse Failure | Invalid Drizzle table syntax | Log table name and line number, continue with other tables |
| Missing Exports | Function not exported | Flag and list missing exports per file |
| Type Generation Failure | OpenAPI schema incompatibility | Log schema fragment, fall back to `any` type |
| Build Timeout | Large codebase or slow I/O | Implement parallel processing and progress reporting |

### Deployment Errors

| Error | Cause | Recovery |
|-------|-------|----------|
| GitHub Pages build fails | Invalid HTML or large files | Validate HTML before push, compress assets |
| Version conflict | Same version published twice | Check version in package.json before build |
| Search index too large | Many pages/versions | Implement incremental indexing, compress |
| Dependency issues | Peer dependency mismatch | Pin exact versions, validate with npm audit |

### Validation

- **OpenAPI Validation**: `swagger-parser` before generation
- **HTML Validation**: W3C HTML validator on built docs
- **Link Validation**: Check internal and external links
- **Search Index Validation**: Verify all pages are indexed

---

## Testing Strategy

Documentation generation is infrastructure-focused with deterministic outputs. PBT is not appropriate here; instead, we use:

### Unit Tests (Example-Based)

1. **JSDoc Extraction Tests**
   - Test controllers with various JSDoc formats
   - Verify correct OpenAPI operation generation
   - Test edge cases (missing tags, invalid schema refs)

2. **Schema Documentation Tests**
   - Parse sample Drizzle tables
   - Verify generated Markdown/HTML tables
   - Check index and constraint documentation

3. **Type Generation Tests**
   - Generate types from OpenAPI specs
   - Verify TypeScript compilation
   - Check interface names and field types

4. **Postman Collection Tests**
   - Verify collection structure
   - Check endpoint parameters
   - Validate authentication setup

### Integration Tests

1. **Full Pipeline Tests**
   - Run entire documentation build
   - Verify all artifacts are generated
   - Check cross-references between artifacts

2. **Storybook Tests**
   - Verify stories are valid
   - Check component props documentation
   - Test story rendering

3. **Deployment Tests**
   - Test GitHub Pages deployment
   - Verify versioning structure
   - Check search index functionality

### Quality Checks

1. **Documentation Coverage**
   - Report % of public functions documented
   - List undocumented exports
   - Flag functions with incomplete JSDoc

2. **Link Validation**
   - Check all internal links work
   - Verify external links are reachable
   - Report broken references

3. **Search Functionality**
   - Test search index completeness
   - Verify search results accuracy
   - Performance test with large indices

---

## Implementation Considerations

### Incremental Generation

- Generate only changed files for faster builds
- Cache parsed schemas and type definitions
- Use file watchers in development mode

### Performance Optimization

- Parallelize schema parsing and generation
- Implement progressive site building
- Compress documentation artifacts

### Maintainability

- Version all documentation tools
- Document JSDoc and ADR formats
- Provide templates and examples
- Create CI/CD checks for documentation quality

### Tooling

**Already Installed**:
- `swagger-jsdoc` (existing)
- `swagger-ui-express` (existing)

**To Install**:
- `openapi-typescript` - Generate types from OpenAPI
- `typedoc` - Generate API reference
- `@storybook/react` - Component documentation
- `@storybook/cli` - Storybook setup
- `yaml` - YAML parsing (already installed)

---

## Related Decisions

- **ADR-0001**: Stellar-based settlement architecture
- **ADR-0002**: Oracle consensus for match outcomes
- Decision to use Drizzle ORM (type-safe database queries)
- Decision to maintain Postman collections for API testing

