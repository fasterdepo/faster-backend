# Implementation Plan: Documentation Automation

## Overview

This implementation plan converts the BANKERCHANGER documentation system from manual maintenance to automated generation. The approach starts with foundational infrastructure (validating existing OpenAPI spec and configuring type generation), then adds TSDoc coverage to services, generates schema documentation, sets up Storybook, and finally deploys a versioned documentation site with search functionality.

---

## Tasks

- [ ] 1. Set up documentation infrastructure and establish OpenAPI validation
  - [ ] 1.1 Install documentation generation tools and dependencies
    - Install `openapi-typescript`, `typedoc`, `typedoc-plugin-missing-exports`, `swagger-parser` as dev dependencies
    - Install Storybook packages: `@storybook/react@latest`, `@storybook/cli@latest`
    - Update `package.json` scripts with documentation generation commands
    - _Requirements: Infrastructure setup for all documentation pipelines_

  - [ ] 1.2 Enhance OpenAPI specification with complete JSDoc patterns
    - Review and standardize all existing endpoints in `src/api/controllers/*.ts` with JSDoc `@swagger` comments
    - Ensure every public controller method has JSDoc with operation summary, parameters, responses
    - Add missing error responses (400, 401, 403, 500) to all endpoint documentation
    - Test OpenAPI spec validation with `swagger-parser`
    - _Requirements: OpenAPI spec structure and JSDoc patterns_

  - [ ] 1.3 Validate existing OpenAPI spec and fix any compliance issues
    - Run `scripts/validate-openapi.js` and fix any reported issues
    - Ensure all schema references are valid
    - Verify security schemes are correctly defined
    - Generate `openapi.json` from existing `openapi.yaml` for next steps
    - _Requirements: OpenAPI specification validation_

  - [ ] 1.4 Create npm scripts for documentation generation
    - Add `npm run docs:generate` - runs all documentation generation steps
    - Add `npm run docs:types` - generates TypeScript types from OpenAPI
    - Add `npm run docs:schema` - generates schema documentation
    - Add `npm run docs:build` - builds complete documentation site
    - Add `npm run docs:dev` - watches for changes and regenerates
    - _Requirements: Scripted documentation generation_

- [ ] 2. Implement TypeScript type generation pipeline from OpenAPI spec
  - [ ] 2.1 Configure and test OpenAPI TypeScript code generation
    - Create `openapi-typescript.config.ts` configuration file
    - Configure to generate types at `src/types/api.generated.ts`
    - Run generation and verify TypeScript compilation succeeds
    - Verify generated types include all schemas, parameters, and response objects
    - _Requirements: TypeScript type generation from OpenAPI_

  - [ ] 2.2 Create type import utilities for controllers
    - Add TypeScript utility types for request/response typing in controllers
    - Create helper types that combine OpenAPI types with Express types (Request, Response)
    - Add documentation showing how to use generated types in controller methods
    - Update at least 3 controller methods to use generated types for compilation verification
    - _Requirements: Type-safe controller implementation_

  - [ ] 2.3 Update Postman collection generation to reference OpenAPI types
    - Enhance existing `scripts/generate-postman.js` to include generated types
    - Add environment variables template to Postman collection
    - Include pre-request and post-request scripts for token management
    - Test collection import in Postman
    - _Requirements: Postman collection synchronization with OpenAPI_

- [ ] 3. Add TSDoc coverage to service layer and core business logic
  - [ ] 3.1 Add TSDoc comments to all public service functions
    - Add comprehensive JSDoc/TSDoc to all public functions in `src/services/*.ts` (if exists) or business logic modules
    - Follow TSDoc format with @param, @returns, @throws, @example, @remarks for at least 20 public functions
    - Include examples for complex functions like market resolution, bet calculation, Oracle consensus
    - _Requirements: TSDoc documentation for all public functions_

  - [ ] 3.2 Add TSDoc comments to all utility and helper functions
    - Document utility functions in `src/utils/*.ts`
    - Document DTO validation functions and type guards
    - Include usage examples and error conditions
    - _Requirements: Public function documentation completeness_

- [ ] 4. Implement schema documentation generation from Drizzle ORM
  - [ ] 4.1 Create Drizzle schema parser and documentation generator
    - Build script to parse `src/db/schema.ts` and extract all table definitions
    - Extract column names, types, constraints, indexes, foreign keys
    - Generate Markdown table documentation for each table
    - Include descriptions, default values, and constraints
    - Create `docs/schema/index.md` with table overview and relationships
    - _Requirements: Schema documentation generation from Drizzle ORM_

  - [ ] 4.2 Generate HTML schema reference with relationships diagram
    - Convert Markdown schema docs to HTML with navigation
    - Create relationship diagram showing foreign key relationships between tables
    - Add search functionality for schema tables and columns
    - Generate at `docs/schema/index.html`
    - _Requirements: Interactive schema documentation_

- [ ] 5. Configure TypeDoc for API reference generation
  - [ ] 5.1 Create and configure typedoc.json
    - Create `typedoc.json` with configuration for entry points, output directory, theme
    - Configure to exclude private/internal APIs
    - Enable search in comments and source code links
    - Set output to `docs/typedoc/`
    - _Requirements: TypeDoc configuration setup_

  - [ ] 5.2 Generate TypeDoc HTML and verify compilation
    - Run `typedoc` to generate HTML documentation site
    - Verify documentation is generated at `docs/typedoc/index.html`
    - Check that module hierarchy, class documentation, and function signatures are correct
    - Test navigation and search functionality in generated docs
    - _Requirements: TypeDoc HTML documentation generation_

- [ ] 6. Set up Storybook for component documentation (frontend-specific)
  - [ ] 6.1 Initialize and configure Storybook
    - Run `npx storybook init` in frontend directory
    - Configure Storybook main.ts with React addon settings
    - Set up preview.ts for global decorators and theme
    - Configure paths alias for component imports
    - _Requirements: Storybook setup for React components_

  - [ ] 6.2 Create story templates and document existing components
    - Create story template following CSF 3 format with autodocs
    - Add stories for at least 5 key React components
    - Include argTypes for component props documentation
    - Test Storybook build creates static site at `frontend/storybook-static/`
    - _Requirements: Component documentation with Storybook_

- [ ] 7. Build and deploy unified documentation site with versioning
  - [ ] 7.1 Create documentation build and deployment script
    - Create `scripts/build-docs.sh` that orchestrates all generation steps
    - Combine TypeDoc, schema docs, API reference, and Storybook into single site structure
    - Organize by version: `docs/v{version}/api/`, `docs/v{version}/schema/`, etc.
    - Create version index and navigation
    - Build search index covering all documentation pages
    - _Requirements: Unified documentation site building_

  - [ ] 7.2 Set up GitHub Pages deployment configuration
    - Configure `gh-pages` branch for documentation hosting
    - Add GitHub Actions workflow for automatic doc builds on release
    - Create `.github/workflows/docs.yml` to trigger on version tags
    - Set up GitHub Pages to serve from `docs/` directory or `gh-pages` branch
    - Verify documentation is deployed and accessible
    - _Requirements: Documentation site deployment strategy_

  - [ ] 7.3 Implement full-text search for documentation
    - Generate search index JSON from all documentation pages
    - Integrate search library (Lunr.js or MiniSearch) for client-side search
    - Add search interface to documentation site home page
    - Test search returns correct results from all sections (API, schema, guides)
    - _Requirements: Documentation site search and navigation_

- [ ] 8. Create Architecture Decision Records (ADRs) for key technical decisions
  - [ ] 8.1 Create ADR template and document existing decisions
    - Create `docs/adr/template.md` for future ADRs
    - Create `docs/adr/0001-stellar-based-settlement.md` documenting payment architecture
    - Create `docs/adr/0002-oracle-consensus.md` documenting Oracle resolution mechanism
    - Create `docs/adr/index.md` listing all ADRs with links
    - _Requirements: Architecture Decision Records_

- [ ] 9. Create integration tests for documentation generation pipeline
  - [ ] 9.1 Write tests for JSDoc extraction and OpenAPI generation
    - Create test controllers with various JSDoc formats
    - Verify extracted OpenAPI operations match expected structure
    - Test error cases: missing tags, invalid schema refs, malformed YAML
    - _Requirements: OpenAPI generation validation_

  - [ ] 9.2 Write tests for schema documentation generation
    - Create sample Drizzle table definitions
    - Parse and verify generated documentation
    - Test index and constraint extraction
    - Verify relationship detection between tables
    - _Requirements: Schema documentation validation_

  - [ ] 9.3 Write tests for TypeScript type generation
    - Verify types compile without errors
    - Check field names and types match OpenAPI schema
    - Test edge cases: nullable fields, enums, array types
    - _Requirements: Type generation validation_

- [ ] 10. Documentation quality and deployment verification
  - [ ] 10.1 Run documentation quality checks
    - Run documentation coverage report (% of public functions documented)
    - Validate all internal documentation links work
    - Check for broken external links
    - Generate and review warning/error report
    - _Requirements: Documentation coverage and link validation_

  - [ ] 10.2 Verify complete documentation build and deployment
    - Run full `npm run docs:build` pipeline successfully
    - Verify all artifacts generated: OpenAPI types, schema docs, TypeDoc site, Storybook
    - Test deployed documentation site loads correctly
    - Verify search functionality works
    - Check version navigation works across releases
    - _Requirements: Complete documentation system verification_

---

## Notes

- Tasks marked with `*` are optional and can be skipped for MVP; all tasks here are core requirements
- Each task builds incrementally on previous steps
- Documentation generation integrates with existing tooling (swagger-jsdoc, swagger-ui-express)
- Testing is embedded within implementation tasks
- Deployment uses GitHub Pages with version management
- Search functionality is essential for discoverability
- Versioning strategy ensures previous documentation remains accessible

