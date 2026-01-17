# ESLint Plugin: Declaration Order

## Project Overview

A custom ESLint plugin that enforces a specific declaration ordering pattern in TypeScript/JavaScript files, optimizing for **top-down readability** where high-level logic appears first and implementation details follow.

**Package Name**: `eslint-plugin-declaration-order`

**Rule Name**: `declaration-order/order`

---

## Rule Specification

### Ordering Requirements

Files must follow this exact order:

1. **Imports** - All `import` statements at the top
2. **Re-exports** - All `export { foo } from './bar'` statements
3. **Exported Types** - `export type`, `export interface`, `export enum`
4. **Private Types** - `type`, `interface`, `enum` (not exported)
5. **Exported Assignments** - `export const`, `export let`, `export var`
6. **Private Assignments** - `const`, `let`, `var` (not exported)
7. **Exported Functions** - `export function`, `export default function`
8. **Private Functions** - `function` declarations (not exported)

### Reverse Dependency Flow

Within each category (3-8), declarations must appear in **reverse dependency order**:

- **Dependents appear BEFORE their dependencies**
- High-level functions that call other functions appear first
- Low-level utility functions appear last

#### Example

```typescript
// ✅ CORRECT: main() appears before its dependencies
function main() {
  const data = fetchData()
  processData(data)
  render()
}

function fetchData() {
  return apiCall()
}

function processData(data) {
  return transform(data)
}

function render() {
  // ...
}

function apiCall() {
  // lowest level
}

function transform(data) {
  // lowest level
}
```

```typescript
// ❌ WRONG: dependencies appear before dependents
function apiCall() {
  // ...
}

function fetchData() {
  return apiCall()
}

function main() {
  const data = fetchData()
  // ...
}
```

### Special Cases

#### Multiple Dependents

If a declaration has multiple dependents, it should appear **after the last dependent**.

```typescript
// helper() is used by both foo() and bar()
// helper() appears after bar() (the last dependent)

function foo() {
  helper()
}

function bar() {
  helper()
}

function helper() {
  // appears after last dependent
}
```

#### Multiple Dependencies

When a declaration depends on multiple things, those dependencies should appear **in the order they are referenced** in the code.

```typescript
function process() {
  const a = getA()  // referenced first
  const b = getB()  // referenced second
  return combine(a, b)
}

function getA() { }
function getB() { }
```

#### Circular Dependencies

When circular dependencies are detected, fall back to **alphabetical order** for those declarations.

```typescript
// A depends on B, B depends on A
// Fallback: alphabetical

function alpha() {
  beta()
}

function beta() {
  alpha()
}
```

---

## Technical Architecture

### Core Components

#### 1. Declaration Classifier

**Purpose**: Categorize each top-level declaration

**Input**: AST node
**Output**: Category (0-8) and metadata

```typescript
interface DeclarationInfo {
  name: string
  node: ESTree.Node
  category: number // 0-8
  exported: boolean
  startLine: number
}
```

**Logic**:
- Check `node.type` (ImportDeclaration, ExportNamedDeclaration, etc.)
- Check if parent is an export
- Handle edge cases (default exports, namespace exports)

#### 2. Dependency Analyzer

**Purpose**: Build dependency graph between declarations

**Input**: Array of DeclarationInfo
**Output**: Dependency graph

```typescript
interface DependencyGraph {
  nodes: Map<string, GraphNode>
  edges: Map<string, Set<string>> // dependent -> dependencies
}

interface GraphNode {
  declaration: DeclarationInfo
  dependencies: Set<string>      // what this depends on
  dependents: Set<string>         // what depends on this
  orderedDependencies: string[]   // in reference order
}
```

**Algorithm**:
1. For each declaration, traverse its AST
2. Collect all `Identifier` references
3. Filter to only identifiers declared in the same file
4. Track order of first reference for each dependency
5. Build bidirectional edges (dependencies + dependents)

#### 3. Topological Sorter

**Purpose**: Sort declarations by reverse dependency order

**Input**: Array of DeclarationInfo in same category + DependencyGraph
**Output**: Sorted array

**Algorithm**:
```
function reverseTopologicalSort(declarations, graph):
  sorted = []
  visited = Set()
  visiting = Set() // for cycle detection

  function visit(decl, path):
    if decl in visited:
      return

    if decl in visiting:
      // Circular dependency detected
      handleCircularDependency(path)
      return

    visiting.add(decl)

    // Visit all DEPENDENTS first (this is the "reverse" part)
    for dependent in graph.getDependents(decl):
      if dependent not in visited:
        visit(dependent, path + [decl])

    visiting.remove(decl)
    visited.add(decl)
    sorted.append(decl)

  // Start with declarations that have the most dependents
  // or are entry points
  sortedByDependents = sortByDependentCount(declarations)

  for decl in sortedByDependents:
    visit(decl, [])

  return sorted
```

**Multiple Dependents Handling**:
- Track "last dependent line number" for each declaration
- Sort so dependencies appear after their last dependent

**Circular Dependency Handling**:
- Detect cycles using visiting set
- Extract strongly connected components (SCC)
- Sort SCC members alphabetically
- Continue with rest of graph

#### 4. Order Validator

**Purpose**: Compare actual order vs expected order and report violations

**Input**: Actual declarations, Expected order
**Output**: ESLint violations

**Logic**:
```typescript
for (let i = 0; i < actual.length; i++) {
  const actualDecl = actual[i]
  const expectedDecl = expected[i]

  if (actualDecl.name !== expectedDecl.name) {
    report({
      node: actualDecl.node,
      message: `'${actualDecl.name}' should appear after '${expectedDecl.name}'`,
      suggest: [
        {
          desc: 'Reorder declarations',
          fix: (fixer) => {
            // Generate fix to move declaration
          }
        }
      ]
    })
  }
}
```

---

## Implementation Plan

### Phase 1: Project Setup (1 hour)

- [ ] Create new repository `eslint-plugin-declaration-order`
- [ ] Initialize npm package
- [ ] Setup TypeScript configuration
- [ ] Add ESLint dependencies
- [ ] Create basic project structure:
  ```
  src/
    rules/
      order.ts
    utils/
      classifier.ts
      dependency-analyzer.ts
      topological-sort.ts
    index.ts
  tests/
    rules/
      order.test.ts
    fixtures/
  package.json
  tsconfig.json
  README.md
  ```

### Phase 2: Core Implementation (6-8 hours)

#### Step 1: Declaration Classifier (1 hour)
- [ ] Implement `classifyDeclaration(node)` function
- [ ] Handle all node types (Import, Export, Function, Variable, Type)
- [ ] Write unit tests for classification

#### Step 2: Dependency Analyzer (3 hours)
- [ ] Implement AST traversal to extract identifiers
- [ ] Build dependency graph
- [ ] Track reference order
- [ ] Filter to in-file declarations only
- [ ] Handle scoping correctly (don't count parameters as dependencies)
- [ ] Write unit tests for dependency extraction

#### Step 3: Topological Sorter (2-3 hours)
- [ ] Implement reverse topological sort
- [ ] Handle multiple dependents (last dependent rule)
- [ ] Detect circular dependencies
- [ ] Implement alphabetical fallback
- [ ] Write unit tests for sorting

#### Step 4: Rule Integration (1 hour)
- [ ] Create ESLint rule structure
- [ ] Integrate all components
- [ ] Generate error messages
- [ ] Write integration tests

### Phase 3: Testing (4-6 hours)

#### Unit Tests
- [ ] Classifier tests (15 test cases)
- [ ] Dependency analyzer tests (20 test cases)
- [ ] Topological sort tests (15 test cases)

#### Integration Tests
- [ ] Valid code examples (10 files)
- [ ] Invalid code examples (20 files)
- [ ] Edge cases:
  - Circular dependencies
  - Multiple dependents
  - Complex dependency chains
  - Mixed exports/imports
  - Default exports
  - Namespace imports
  - Type-only imports

#### Real-world Tests
- [ ] Test on actual TypeScript projects
- [ ] Verify performance on large files (1000+ lines)

### Phase 4: Auto-fix (Optional, 4-6 hours)

- [ ] Implement fixer that reorders declarations
- [ ] Preserve comments and whitespace
- [ ] Handle complex formatting cases
- [ ] Test auto-fix extensively

### Phase 5: Documentation (2-3 hours)

- [ ] Write comprehensive README
- [ ] Add code examples
- [ ] Document all edge cases
- [ ] Create migration guide
- [ ] Add contributing guidelines

### Phase 6: Publishing (1 hour)

- [ ] Setup npm publishing
- [ ] Create GitHub releases
- [ ] Add badges to README
- [ ] Setup CI/CD (GitHub Actions)
- [ ] Publish v1.0.0

---

## Testing Strategy

### Test Fixtures

#### Valid Examples

**Fixture 1: Basic Ordering**
```typescript
// imports
import { foo } from 'bar'

// re-exports
export { baz } from 'qux'

// exported types
export interface User {
  id: string
}

// private types
type UserId = string

// exported assignments
export const API_URL = 'https://api.com'

// private assignments
const MAX_RETRIES = 3

// exported functions
export function getUser() {
  return fetchUser()
}

// private functions
function fetchUser() {
  return apiCall()
}

function apiCall() {
  // lowest level
}
```

**Fixture 2: Reverse Dependency Flow**
```typescript
function main() {
  processData(fetchData())
}

function processData(data) {
  return transform(data)
}

function fetchData() {
  return fetch(API_URL)
}

function transform(data) {
  return data.map(item => item.value)
}
```

**Fixture 3: Multiple Dependents**
```typescript
function routeA() {
  helper()
}

function routeB() {
  helper()
}

function routeC() {
  helper()
}

// helper appears after last dependent
function helper() {
  return sharedLogic()
}

function sharedLogic() {
  // ...
}
```

#### Invalid Examples

**Fixture 1: Wrong Category Order**
```typescript
// ❌ function before type
function foo() {}

export interface Bar {}
```

**Fixture 2: Wrong Dependency Order**
```typescript
// ❌ dependency before dependent
function helper() {}

function main() {
  helper()
}
```

**Fixture 3: Multiple Dependents Wrong**
```typescript
// ❌ helper appears before last dependent
function foo() { helper() }

function helper() {}

function bar() { helper() }
```

---

## Performance Considerations

### Optimizations

1. **Caching**: Cache dependency graphs per file
2. **Early Exit**: Skip analysis if file has < 10 declarations
3. **Incremental**: Only re-analyze changed declarations
4. **Scope Filtering**: Only analyze top-level declarations

### Benchmarks

Target performance:
- Small files (<100 lines): < 10ms
- Medium files (100-500 lines): < 50ms
- Large files (500-1000 lines): < 200ms
- Very large files (>1000 lines): < 500ms

---

## Configuration Options

```json
{
  "rules": {
    "declaration-order/order": ["error", {
      "enforceCategories": true,
      "enforceDependencyOrder": true,
      "circularFallback": "alphabetical", // or "original"
      "ignoreDefaultExports": false
    }]
  }
}
```

---

## Publishing Checklist

### Pre-publish
- [ ] All tests passing
- [ ] 95%+ code coverage
- [ ] README complete
- [ ] CHANGELOG.md created
- [ ] LICENSE file (MIT)
- [ ] Keywords in package.json
- [ ] Repository field in package.json

### Package.json
```json
{
  "name": "eslint-plugin-declaration-order",
  "version": "1.0.0",
  "description": "ESLint plugin to enforce declaration ordering with reverse dependency flow",
  "keywords": [
    "eslint",
    "eslint-plugin",
    "declaration",
    "order",
    "sorting",
    "typescript",
    "dependency"
  ],
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "peerDependencies": {
    "eslint": ">=8.0.0"
  }
}
```

### NPM Publishing
```bash
npm run build
npm test
npm publish
```

### Post-publish
- [ ] Create GitHub release
- [ ] Tweet about it
- [ ] Submit to awesome-eslint list
- [ ] Create demo repository

---

## Future Enhancements

### v1.1
- [ ] Auto-fix support
- [ ] Configuration for custom category order
- [ ] Ignore patterns for specific declarations

### v1.2
- [ ] Support for class methods ordering
- [ ] Support for object property ordering
- [ ] Integration with prettier

### v2.0
- [ ] Visual dependency graph generation
- [ ] Web-based configuration UI
- [ ] IDE extensions (VS Code, WebStorm)

---

## Estimated Effort

**Total**: 20-25 hours

- Project setup: 1 hour
- Core implementation: 8 hours
- Testing: 6 hours
- Documentation: 3 hours
- Publishing: 1 hour
- Buffer for debugging: 5 hours

**Timeline**: 1-2 weeks (part-time)

---

## Success Criteria

- [ ] All test cases pass
- [ ] No false positives on real-world TypeScript projects
- [ ] Performance benchmarks met
- [ ] Documentation complete and clear
- [ ] Published to npm with >0 downloads
- [ ] Used in at least one production project (this project!)

---

## Open Questions

1. **Default exports**: Should `export default function main()` be category 7 or category 8?
2. **Arrow functions**: Should `const foo = () => {}` be category 5 (assignment) or 7 (function)?
3. **Class declarations**: New category or treat as functions?
4. **Namespace declarations**: New category or ignore?
5. **Type imports**: Should `import type { Foo }` affect dependency order?

---

## References

- [ESLint Plugin Guide](https://eslint.org/docs/latest/extend/plugins)
- [ESTree Spec](https://github.com/estree/estree)
- [Topological Sorting](https://en.wikipedia.org/wiki/Topological_sorting)
- [Tarjan's SCC Algorithm](https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm)
