---
knowledge_id: KA-SDK-007
version: "1.0.0"
status: approved
owner: Chief Platform Architect
phase: 2.0D.2.5A
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-PLATFORM
capability: platform-sdk
type: specification
depends_on:
  - id: KA-SDK-002
    reason: "Token and AST node sequences"
  - id: KA-SDK-003
    reason: "Algorithmic transforms on token streams"
---

# Parser SDK

## Lexer · Parser Combinators · AST · Grammar Definition · Error Recovery

---

## 1. Purpose

The Parser SDK provides the language-agnostic parsing infrastructure required
by Phase 2.2 (Universal Reverse Engineering) and the KQL Engine. It separates
grammar definition from parsing strategy, enabling the same grammar to be
parsed by recursive descent, PEG, or LL(k) strategies. The SDK is extensible
for any source language or domain-specific language.

---

## 2. Parsing Pipeline

```
Source Text
    │
    ▼ Lexer (tokenize)
Token Stream
    │
    ▼ Parser (grammar-driven)
Concrete Syntax Tree (CST)
    │
    ▼ ASTBuilder (flatten / simplify)
Abstract Syntax Tree (AST)
    │
    ▼ AST Visitors / Transformers
Transformed AST / Analysis Result
```

---

## 3. Interfaces

### 3.1 Token

```python
@dataclass(frozen=True)
class Token:
    type:   str         # Grammar-defined token type name
    value:  str         # Literal text matched
    pos:    int         # Byte offset in source
    line:   int
    col:    int
    length: int

@dataclass(frozen=True)
class TokenRule:
    name:    str
    pattern: str        # Regex pattern
    skip:    bool       # True for whitespace/comments
    priority: int       # Higher = matched first
```

### 3.2 Lexer

```python
class Lexer(Protocol):
    def tokenize(self, source: str) -> Sequence[Token]     # O(|source|)
    def tokenize_stream(self, source: str) -> Iterator[Token]  # lazy O(1) per call
    def add_rule(self, rule: TokenRule) -> None
    def remove_rule(self, name: str) -> None

class LexerFactory(Protocol):
    """Builds Lexer from grammar rules."""
    def from_rules(self, rules: Sequence[TokenRule]) -> Lexer: ...
    def from_grammar(self, grammar: "Grammar") -> Lexer: ...
```

### 3.3 Grammar

```python
@dataclass
class Grammar:
    name:        str
    version:     str
    rules:       Sequence[GrammarRule]
    start:       str           # Name of root production rule
    token_rules: Sequence[TokenRule]

@dataclass
class GrammarRule:
    name:        str           # Non-terminal name
    production:  str           # EBNF or PEG expression
    action:      str | None    # Optional semantic action name
    comment:     str | None

class GrammarBuilder(Protocol):
    def rule(self, name: str, production: str,
             action: str | None = None) -> "GrammarBuilder": ...
    def token(self, name: str, pattern: str,
              skip: bool = False, priority: int = 0) -> "GrammarBuilder": ...
    def start(self, rule_name: str) -> "GrammarBuilder": ...
    def build(self) -> Grammar: ...
```

### 3.4 CST Nodes

```python
@dataclass
class CSTNode:
    rule_name: str
    children:  Sequence["CSTNode | Token"]
    pos:       int
    end:       int

class CSTVisitor(Protocol):
    def visit(self, node: CSTNode) -> Any: ...
    def visit_token(self, token: Token) -> Any: ...
```

### 3.5 AST

```python
@dataclass
class ASTNode:
    kind:     str
    children: Sequence["ASTNode"]
    attrs:    ImmutableMap[str, Any]   # node-specific data
    pos:      int
    source:   str | None    # Original source text span

class ASTVisitor(Protocol[T]):
    """Typed visitor returning T from each node."""
    def visit(self, node: ASTNode) -> T: ...

class ASTTransformer(Protocol):
    """Produces a new AST from an existing one. Does not mutate."""
    def transform(self, node: ASTNode) -> ASTNode: ...

class ASTWalker(Protocol):
    """Walks AST in pre-order, post-order, or level-order."""
    def walk_pre(self, node: ASTNode) -> Iterator[ASTNode]: ...
    def walk_post(self, node: ASTNode) -> Iterator[ASTNode]: ...
    def walk_level(self, node: ASTNode) -> Iterator[ASTNode]: ...
```

### 3.6 Parser

```python
class Parser(Protocol):
    def parse(self, tokens: Sequence[Token]) -> ParseResult
    def parse_text(self, source: str) -> ParseResult  # Lexes then parses

@dataclass
class ParseResult:
    cst:    CSTNode | None
    ast:    ASTNode | None
    errors: Sequence[ParseError]
    ok:     bool                  # True if parse succeeded without fatal errors

@dataclass(frozen=True)
class ParseError:
    message: str
    pos:     int
    line:    int
    col:     int
    severity: str   # "error" | "warning" | "hint"
    recovery: str | None  # Description of recovery action taken
```

### 3.7 Parser Combinators

```python
class ParserCombinator(Protocol[T]):
    """Parser that produces T from a token stream."""
    def parse(self, stream: "TokenStream") -> T | ParseFailure: ...
    def __or__(self, other: "ParserCombinator[T]") -> "ParserCombinator[T]": ...  # ordered choice
    def __and__(self, other: "ParserCombinator[U]") -> "ParserCombinator[tuple[T,U]]": ...  # sequence
    def many(self) -> "ParserCombinator[Sequence[T]]": ...      # zero or more
    def many1(self) -> "ParserCombinator[Sequence[T]]": ...     # one or more
    def optional(self) -> "ParserCombinator[T | None]": ...
    def map(self, fn: Callable[[T], U]) -> "ParserCombinator[U]": ...

class TokenStream(Protocol):
    def peek(self) -> Token | None: ...
    def advance(self) -> Token: ...
    def pos(self) -> int: ...
    def save(self) -> int: ...           # Checkpoint for backtracking
    def restore(self, checkpoint: int) -> None: ...
```

### 3.8 Error Recovery

```python
class ErrorRecovery(Protocol):
    """Strategy for recovering from parse errors."""
    def recover(self, stream: "TokenStream",
                error: ParseError) -> "RecoveryResult": ...

class RecoveryResult:
    advanced_to: int      # Token position after recovery
    synthetic_node: ASTNode | None  # Placeholder node injected
    errors_emitted: Sequence[ParseError]

# Built-in recovery strategies:
# PanicModeRecovery     — skip tokens until synchronization token
# ErrorProductionRecovery — special grammar rules for common mistakes
# InsertionRecovery     — insert missing expected token
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-PSR-001 | `Lexer.tokenize("")` returns an empty sequence without error. |
| C-PSR-002 | Token positions are byte offsets from start of source, not character offsets. |
| C-PSR-003 | All tokens from a lexer are non-overlapping and cover the full input (minus skipped). |
| C-PSR-004 | `Parser.parse` always returns `ParseResult`; fatal errors are in `ParseResult.errors`. |
| C-PSR-005 | `AST` is immutable. Transformers return new nodes. |
| C-PSR-006 | `TokenRule` with higher priority is tried first; ties go in declaration order. |
| C-PSR-007 | `ParserCombinator.__or__` is an ordered choice: tries left first, backtracks on failure. |
| C-PSR-008 | `TokenStream.restore(save())` exactly reverses all advances since `save()`. |
| C-PSR-009 | `ASTWalker.walk_post` visits children before parent. |
| C-PSR-010 | `Grammar` with no start rule raises `GrammarError` on first `parse` call. |

---

## 5. Algorithm Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Lexer tokenize | O(|source|) | Regex DFA scan |
| Recursive Descent | O(|tokens| · k) | k = grammar rules |
| PEG packrat | O(|tokens| · G) | G = grammar size, memoized |
| LL(k) | O(|tokens| · k) | k = lookahead |
| AST walk (pre/post) | O(N) | N = AST nodes |
| AST walk (level) | O(N) | BFS order |
| Combinator sequence | O(T1 + T2) | sequential |
| Combinator choice | O(max(T1, T2)) | with backtracking |

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — token sequences, AST child lists
- Algorithms SDK (KA-SDK-003) — token stream transforms

---

## 7. Extension Points

```python
class ParserStrategy(Protocol):
    """Pluggable parsing algorithm (recursive descent, PEG, LL, Earley)."""
    def build(self, grammar: Grammar) -> Parser: ...

class SemanticAction(Protocol):
    """Attached to grammar rules; transforms CST subtree to AST node."""
    def execute(self, children: Sequence[ASTNode | Token]) -> ASTNode: ...

class LanguagePlugin(Protocol):
    """Bundles grammar + tokenizer + AST schema for a complete language."""
    def grammar(self) -> Grammar: ...
    def ast_schema(self) -> dict: ...
    def visitors(self) -> dict[str, ASTVisitor]: ...
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-PSR-001 | `tokenize("")` returns `[]` |
| V-PSR-002 | Token positions are monotonically increasing |
| V-PSR-003 | Sum of token lengths = source length minus skipped chars |
| V-PSR-004 | Parse of valid input: `result.ok == True` and `result.errors == []` |
| V-PSR-005 | Parse of invalid input: `result.ok == False` and at least one error |
| V-PSR-006 | `walk_post(root)` visits all leaves before interior nodes |
| V-PSR-007 | `restore(save())` after N advances returns to same position |
| V-PSR-008 | `many(p).parse` on empty input returns `[]` not failure |
| V-PSR-009 | Grammar without start rule raises `GrammarError` |
| V-PSR-010 | `ASTTransformer` returns new node; original is unchanged |
