# Folk Language Syntax Highlighting Plan

## Overview

The Folk language is a hybrid language that combines Tcl syntax with embedded C code blocks. The syntax highlighting needs to handle:

1. **Tcl-like syntax** as the base language
2. **Embedded C code blocks** using the `c` namespace tools
3. **Mixed syntax patterns** where C code appears within Tcl constructs

## VS Code Implementation Approach

### TextMate Grammar Foundation
VS Code uses TextMate grammars for syntax highlighting, which work by:
- Breaking text into tokens using regular expressions
- Applying scopes to tokens for theming
- Supporting embedded languages through `include` patterns

### Embedded Language Strategy
Based on VS Code's embedded languages documentation, we'll use the **Language Services** approach:
- Define Tcl/Folk as the main language with embedded C regions
- Use TextMate grammar patterns to identify C code blocks
- Apply C syntax highlighting within identified regions

### Limitations of `embeddedLanguages`

**Cannot use `embeddedLanguages` for:**

1. **C Struct Definitions**: `c struct name { ... }`
   - The `{ ... }` contains C field definitions but with Folk-specific syntax
   - Field types and names follow C conventions but are parsed by Folk's C compiler
   - Need custom grammar patterns to highlight field types, names, and comments

2. **C Function Parameters**: `c proc name {param1 param2} returnType { ... }`
   - The parameter list `{param1 param2}` is Folk syntax, not C syntax
   - Parameters are space-separated, not comma-separated like C
   - Need custom patterns to highlight parameter types and names

3. **C Include Statements**: `c include <header.h>` or `c include "header.h"`
   - These are Folk commands that happen to look like C preprocessor directives
   - Need custom patterns to highlight the header names

4. **Mixed Tcl/C Expressions**: `$[expr { ... }]` within C code
   - Tcl expressions embedded within C code blocks
   - Need custom patterns to distinguish Tcl from C syntax

5. **C Substitution**: `csubst { ... }` and `[csubst { ... }]`
   - These are Tcl commands that contain C code
   - The C code within needs highlighting but the wrapper is Tcl

**Can use `embeddedLanguages` for:**

1. **Raw C Code Blocks**: `c code { ... }`
   - Pure C code that can be compiled directly
   - No Folk-specific syntax modifications

2. **C Function Bodies**: The body of `c proc` definitions
   - Once we've parsed the Folk-specific header, the function body is pure C

3. **C Comments and Preprocessor**: Within C code blocks
   - Standard C comments (`/* */`, `//`) and preprocessor directives

## Key Patterns Identified

### 1. C Namespace Usage
- `c create` - Creates a new C compiler instance
- `c proc` - Defines C functions callable from Tcl
- `c struct` - Defines C structs
- `c include` - Includes C headers
- `c code` - Embeds raw C code blocks
- `c compile` - Compiles the C code

### 2. Variable C Compiler Instances
- `camc` (camera compiler)
- `dc` (display compiler) 
- `kc` (keymap compiler)
- `cc` (general C compiler)

### 3. C Code Block Patterns
- `c code { ... }` - Raw C code blocks
- `c proc name {args} returnType { ... }` - C function definitions
- `c struct name { ... }` - C struct definitions
- `c include <header.h>` or `c include "header.h"` - Header includes

### 4. Tcl Integration Patterns
- `csubst { ... }` - C code substitution within Tcl
- `[csubst { ... }]` - Embedded C expressions in Tcl
- `$[expr { ... }]` - Tcl expressions within C code

## Implementation Strategy

### Phase 1: Basic Tcl Syntax Foundation
1. **Extend current grammar** to include full Tcl syntax
   - Variables (`$var`, `${var}`)
   - Lists and dictionaries
   - Control structures (`if`, `while`, `for`, `foreach`)
   - Procedures (`proc`)
   - Namespaces (`namespace eval`)
   - Comments (`#` and `# ...`)

2. **Add Tcl-specific patterns**
   - Command substitution `[command]`
   - Variable substitution `$variable`
   - String literals with proper escaping
   - List literals `{item1 item2}`

### Phase 2: C Namespace Recognition
1. **Identify C compiler instances**
   - Pattern: `[c create]` → `rename [c create] instanceName`
   - Common instances: `camc`, `dc`, `kc`, `cc`

2. **Highlight C namespace commands**
   - `instanceName proc` - C function definitions
   - `instanceName struct` - C struct definitions  
   - `instanceName include` - Header includes
   - `instanceName code` - Raw C code blocks
   - `instanceName compile` - Compilation command

### Phase 3: Embedded C Code Blocks
1. **C code blocks within `c code { ... }`**
   - Treat content as C syntax using embedded language approach
   - Handle C comments (`/* */` and `//`)
   - C keywords, types, operators
   - C preprocessor directives (`#define`, `#ifdef`, etc.)

2. **C function definitions**
   - `c proc name {args} returnType { body }`
   - Highlight function name, parameters, return type
   - Treat function body as C code

3. **C struct definitions**
   - `c struct name { fields }`
   - Highlight struct name and field definitions

### Phase 4: Mixed Syntax Patterns
1. **C substitution in Tcl**
   - `csubst { ... }` - C code within Tcl
   - `[csubst { ... }]` - C expressions in Tcl commands

2. **Tcl expressions in C**
   - `$[expr { ... }]` - Tcl expressions within C code
   - `$[if {condition} {expr1} {expr2}]` - Conditional Tcl expressions

### Phase 5: Advanced Patterns
1. **Header includes**
   - `c include <system.h>` - System headers
   - `c include "local.h"` - Local headers

2. **C preprocessor**
   - `#define`, `#ifdef`, `#ifndef`, `#endif`
   - `#include` within C code blocks

3. **C types and keywords**
   - Basic types: `int`, `char`, `float`, `double`, `void`
   - Modifiers: `const`, `static`, `extern`, `volatile`
   - Control flow: `if`, `else`, `for`, `while`, `switch`, `case`
   - Memory: `malloc`, `free`, `sizeof`

## Technical Implementation

### TextMate Grammar Structure
```json
{
  "scopeName": "source.folk",
  "patterns": [
    {"include": "#tcl-syntax"},
    {"include": "#c-namespace"},
    {"include": "#c-code-blocks"},
    {"include": "#mixed-syntax"}
  ],
  "repository": {
    "tcl-syntax": {
      "patterns": [
        {"include": "#tcl-comments"},
        {"include": "#tcl-strings"},
        {"include": "#tcl-variables"},
        {"include": "#tcl-commands"},
        {"include": "#tcl-control-structures"}
      ]
    },
    "c-namespace": {
      "patterns": [
        {"include": "#c-compiler-instance"},
        {"include": "#c-namespace-commands"}
      ]
    },
    "c-code-blocks": {
      "patterns": [
        {"include": "#c-proc-definition"},
        {"include": "#c-struct-definition"},
        {"include": "#c-code-block"},
        {"include": "#c-include"}
      ]
    },
    "embedded-c": {
      "patterns": [
        {"include": "source.c"}
      ]
    }
  }
}
```

### Key Grammar Patterns

#### C Code Block Detection (Uses embeddedLanguages)
```json
"c-code-block": {
  "begin": "(\\b\\w+)\\s+code\\s*\\{",
  "end": "\\}",
  "beginCaptures": {
    "1": {"name": "variable.other.folk"}
  },
  "contentName": "source.c",
  "patterns": [
    {"include": "source.c"}
  ]
}
```

#### C Function Definition (Mixed approach)
```json
"c-proc-definition": {
  "begin": "(\\b\\w+)\\s+proc\\s+(\\w+)\\s*\\{",
  "end": "\\}\\s*(\\w+)\\s*\\{",
  "beginCaptures": {
    "1": {"name": "variable.other.folk"},
    "2": {"name": "entity.name.function.c"}
  },
  "endCaptures": {
    "1": {"name": "storage.type.c"}
  },
  "patterns": [
    {
      "include": "#c-function-parameters"
    }
  ],
  "contentName": "source.c",
  "patterns": [
    {"include": "source.c"}
  ]
}
```

#### C Function Parameters (Custom pattern - no embeddedLanguages)
```json
"c-function-parameters": {
  "begin": "\\{",
  "end": "\\}",
  "contentName": "meta.function.parameters.folk",
  "patterns": [
    {
      "match": "(\\w+)\\s+(\\w+)",
      "captures": {
        "1": {"name": "storage.type.c"},
        "2": {"name": "variable.parameter.c"}
      }
    },
    {
      "match": "(\\w+\\*)\\s+(\\w+)",
      "captures": {
        "1": {"name": "storage.type.c"},
        "2": {"name": "variable.parameter.c"}
      }
    }
  ]
}
```

#### C Struct Definition (Custom pattern - no embeddedLanguages)
```json
"c-struct-definition": {
  "begin": "(\\b\\w+)\\s+struct\\s+(\\w+)\\s*\\{",
  "end": "\\}",
  "beginCaptures": {
    "1": {"name": "variable.other.folk"},
    "2": {"name": "entity.name.struct.c"}
  },
  "contentName": "meta.struct.fields.folk",
  "patterns": [
    {
      "match": "(\\w+)\\s+(\\w+);",
      "captures": {
        "1": {"name": "storage.type.c"},
        "2": {"name": "variable.other.member.c"}
      }
    },
    {
      "match": "(\\w+\\*)\\s+(\\w+);",
      "captures": {
        "1": {"name": "storage.type.c"},
        "2": {"name": "variable.other.member.c"}
      }
    },
    {
      "include": "#c-comments"
    }
  ]
}
```

#### C Include Statement (Custom pattern - no embeddedLanguages)
```json
"c-include": {
  "match": "(\\b\\w+)\\s+include\\s+(<[^>]+>|\"[^\"]+\")",
  "captures": {
    "1": {"name": "variable.other.folk"},
    "2": {"name": "string.quoted.other.c"}
  }
}
```

#### C Substitution (Custom pattern - no embeddedLanguages)
```json
"c-substitution": {
  "begin": "csubst\\s*\\{",
  "end": "\\}",
  "contentName": "source.c",
  "patterns": [
    {"include": "source.c"}
  ]
}
```

#### Tcl Expression in C (Custom pattern - no embeddedLanguages)
```json
"tcl-expression-in-c": {
  "begin": "\\$\\[",
  "end": "\\]",
  "contentName": "source.tcl",
  "patterns": [
    {"include": "source.tcl"}
  ]
}
```

### Scope Naming Convention
Following VS Code's scope naming conventions:
- `source.folk` - Main language scope
- `source.c` - Embedded C code
- `variable.other.folk` - C compiler instances
- `entity.name.function.c` - C function names
- `entity.name.struct.c` - C struct names
- `storage.type.c` - C return types
- `meta.function.parameters.c` - C function parameters

### Package.json Configuration
```json
{
  "contributes": {
    "languages": [{
      "id": "folk",
      "aliases": ["Folk", "folk"],
      "extensions": [".folk"],
      "configuration": "./language-configuration.json"
    }],
    "grammars": [{
      "language": "folk",
      "scopeName": "source.folk",
      "path": "./syntaxes/folk.tmLanguage.json",
      "embeddedLanguages": {
        "source.c": "c",
        "source.tcl": "tcl"
      }
    }]
  }
}
```

**Note**: The `embeddedLanguages` configuration only applies to:
- Pure C code blocks (`c code { ... }`)
- C function bodies (after parsing Folk-specific headers)
- Tcl expressions embedded in C (`$[expr { ... }]`)

All other Folk-specific C syntax (struct definitions, function parameters, includes) uses custom grammar patterns.

## Key Challenges & Solutions

### 1. Nested Syntax Highlighting
**Challenge**: C code within Tcl within C
**Solution**: Use TextMate's `include` patterns to embed C grammar within identified regions

### 2. Variable Scope Recognition
**Challenge**: Different C compiler instances (`camc`, `dc`, `kc`, `cc`)
**Solution**: Use regex patterns to capture variable names and apply consistent highlighting

### 3. Context Switching
**Challenge**: Knowing when to use Tcl vs C syntax
**Solution**: Define clear boundaries using `begin`/`end` patterns with proper scope names

### 4. Error Recovery
**Challenge**: Graceful handling of malformed code
**Solution**: Use non-greedy patterns and fallback to Tcl syntax for unmatched regions

### 5. Performance Considerations
**Challenge**: Complex regex patterns may impact performance
**Solution**: 
- Use simple, efficient patterns where possible
- Avoid deeply nested captures
- Test with large Folk files

## Success Criteria
- Most of `camera-usb.folk` should have good syntax highlighting
- Tcl syntax should be properly highlighted with appropriate scopes
- C code blocks should be recognized and highlighted using embedded C grammar
- C namespace commands should be distinct from regular Tcl commands
- Mixed syntax patterns should be handled appropriately
- Performance should remain acceptable for files up to 1000+ lines

## Testing Strategy

### Realistic Testing Approaches

**Manual Testing (Primary Method)**
- **Visual Inspection**: Open Folk files in VS Code and visually verify syntax highlighting
- **Theme Testing**: Test with different VS Code themes (Dark+, Light+, etc.)
- **File Testing**: Test with real Folk files like `camera-usb.folk`, `display.folk`
- **Edge Cases**: Test with malformed code, nested structures, and boundary conditions

**Simple Automated Testing (Optional)**
- **Grammar Validation**: Use TextMate grammar validators to check JSON syntax
- **Scope Inspector**: Use VS Code's built-in scope inspector to verify token scopes
- **Performance Testing**: Test with large files (1000+ lines) to ensure acceptable performance

**Not Realistic for This Project**
- **Unit Tests**: Would require complex test infrastructure to simulate VS Code's tokenization
- **Integration Tests**: Would need to mock VS Code's TextMate grammar engine
- **Automated Theme Testing**: Would require screenshot comparison and complex setup

### Testing Workflow

1. **Development Testing**
   - Press `F5` to launch extension in debug mode
   - Create test `.folk` files with various syntax patterns
   - Use VS Code's "Developer: Inspect Editor Tokens and Scopes" command
   - Verify scopes match expected patterns

2. **Real File Testing**
   - Test with actual Folk files from the project
   - Focus on `camera-usb.folk` as the primary test case
   - Test with files containing complex C/Tcl mixing patterns

3. **Theme Compatibility**
   - Test with popular themes: Dark+, Light+, Monokai, Solarized
   - Verify C code blocks are distinguishable from Tcl code
   - Ensure C compiler instances are properly highlighted

4. **Performance Validation**
   - Test with large Folk files (500+ lines)
   - Monitor VS Code performance during typing/editing
   - Ensure no noticeable lag in syntax highlighting

### Success Metrics
- **Visual Clarity**: C code blocks are clearly distinguishable from Tcl code
- **Scope Accuracy**: Token scopes match expected patterns (verified via scope inspector)
- **Theme Compatibility**: Syntax highlighting works well across different themes
- **Performance**: No noticeable performance impact on file editing
- **Real File Coverage**: `camera-usb.folk` has good syntax highlighting throughout

## Next Steps
1. **Phase 1**: Implement basic Tcl syntax with proper scope names
2. **Phase 2**: Add C namespace recognition patterns
3. **Phase 3**: Implement embedded C code block highlighting using `embeddedLanguages`
4. **Phase 4**: Add mixed syntax pattern support
5. **Phase 5**: Test and refine with real Folk files
6. **Phase 6**: Optimize performance and add comprehensive testing
