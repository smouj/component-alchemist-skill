---
name: Component Alchemist
description: Transforms UI mockups, wireframes, and design systems into reusable, framework-agnostic component specifications and token definitions
version: "2.1.0"
author: OpenClaw Design Team
tags: [ui, components, specs, design, transformation, figma, tokens]
maintainer: design-team@openclaw.io
---

# Component Alchemist

## Purpose

Component Alchemist bridges the gap between design artifacts and frontend implementation by automatically extracting, analyzing, and transforming UI mockups into structured component specifications. It serves three primary real-world use cases:

1. **Design-to-Code Handoff**: Convert Figma/Sketch/PSD files into detailed component specs with exact measurements, colors, typography scales, and spacing systems that developers can directly implement without reverse-engineering.

2. **Design System Extraction**: Analyze 50-100+ screenshots or mockups to discover implicit component patterns, extract consistent design tokens (colors, fonts, spacing, shadows, border radii), and generate a unified design system definition.

3. **Spec Validation & Comparison**: Compare implemented components (via visual regression or DOM inspection) against original designs to identify deviations, missing states, or accessibility violations.

## Scope

### Core Commands

```bash
# Analyze a single mockup and extract component structure
component-alchemist analyze \
  --input ./designs/figma-export/signup-flow.fig \
  --output ./specs/signup-components.json \
  --format figma-json \
  --detect-states \
  --extract-tokens \
  --framework react

# Batch process a directory of screenshots to discover component patterns
component-alchemist discover \
  --screens ./marketing-screenshots/ \
  --min-similarity 0.85 \
  --output ./patterns/discovered-components.json \
  --group-by layout

# Extract design tokens from multiple sources into unified system
component-alchemist tokens extract \
  --sources ./designs/brand-guidelines.json,./screens/*.png \
  --format design-tokens \
  --output ./design-system/tokens.css \
  --include colors,typography,spacing,shadows,borders \
  --normalize \
  --deduplicate

# Validate component implementation against source design
component-alchemist validate \
  --spec ./specs/button-spec.json \
  --component ./src/components/Button.tsx \
  --mode visual \
  --pixel-threshold 2 \
  --report ./reports/button-validation.html

# Generate framework-specific component code from spec
component-alchemist generate \
  --spec ./specs/card-spec.json \
  --target vue \
  --output ./src/components/generated/ \
  --typescript \
  --with-tests \
  --with-storybook \
  --accessibility a11y-specs.json

# Merge multiple specs into coherent design system
component-alchemist merge \
  --specs ./specs/*.json \
  --output ./design-system/complete.json \
  --resolve-conflicts prefer-latest \
  --dedupe-components \
  --validate
```

### Configuration Commands

```bash
# Configure design system rules and constraints
component-alchemist config design-system \
  --name "OpenClaw Design System" \
  --primary-font "Inter" \
  --spacing-scale "4-8-12-16-24-32-48" \
  --color-palette ./brand-colors.json \
  --border-radius "4,8,12,full" \
  --breakpoints "sm:640,md:768,lg:1024,xl:1280"

# Set up token extraction preferences
component-alchemist config tokens \
  --precision 2 \
  --include-unused false \
  --format css-custom-properties \
  --prefix --oclaw \
  --semantic-names true

# Define component state machine templates
component-alchemist config states \
  --template button "default,hover,active,disabled,focus,loading,error" \
  --template modal "closed,opening,open,closing" \
  --template input "idle,focus,error,disabled,success"
```

## Work Process

### 1. Input Analysis & Validation

The alchemist first validates the input artifact:

```bash
# Check if Figma file is accessible and parseable
component-alchemist check --input design.fig --format figma-json
# Returns: {"valid": true, "type": "figma-file", "pages": 3, "components": 42}
```

**REAL Steps:**
1. Verify file exists and has read permissions (`fs.access(inputPath, fs.constants.R_OK)`)
2. Detect format via file extension or magic bytes
3. Parse metadata (Figma: `document.frames`, Sketch: `pages`, PSD: `layers`, PNG: dimensions)
4. Extract canvas sizes and artboard boundaries
5. Build internal representation tree
6. Detect if file is encrypted/locked (Figma with restricted access)
7. Validate against minimum requirement thresholds:
   - At least 1 artboard/screen
   - Minimum 20px dimension for any element
   - No corrupted layer data

**Real Command Example:**
```bash
component-alchemist analyze \
  --input ~/Downloads/ProductLanding_V3.fig \
  --output ~/project/specs/landing.json \
  --format figma \
  --verbose
# Output:
# [✓] Valid Figma file detected (version 134)
# [✓] Found 7 artboards across 2 frames
# [✓] Detected 23 named components, 87 layers
# [✓] Extracted 312 colors, 4 text styles
# [✓] Analysis complete: ~/project/specs/landing.json (45KB)
```

### 2. Design Token Extraction

Extract all design tokens with deduplication:

```bash
component-alchemist tokens extract \
  --input ./designs/*.fig,./reference/screenshots/ \
  --output ./tokens/design-tokens.json \
  --include colors,typography,spacing,shadows,borders \
  --normalize \
  --deduplicate \
  --format design-tokens
```

**REAL Steps:**
1. **Color extraction**: Scan all fill properties, convert to sRGB, dedupe by hex/HSL within tolerance (default ΔE < 2.3)
   ```javascript
   // Example extracted tokens
   {
     "color": {
       "primary": {"value": "#3B82F6", "usage": 47, "sources": ["Button/primary", "Link/active"]},
       "error": {"value": "#EF4444", "usage": 12, "sources": ["Alert/error", "Input/error"]}
     }
   }
   ```

2. **Typography extraction**: Extract font-family, weight, size, line-height, letter-spacing
   ```json
   {
     "font": {
       "heading": {"family": "Inter", "weight": 700, "size": "32px", "lineHeight": 1.2},
       "body": {"family": "Inter", "weight": 400, "size": "16px", "lineHeight": 1.5}
     }
   }
   ```

3. **Spacing extraction**: Collect allAutoLayout padding/margin/width/height values, map to scale
   ```bash
   # If spacing values: 8, 16, 24, 32, 48, 64
   # Map to scale index: 1→8, 2→16, etc.
   # Output: tokens.spacing.scale = [8, 16, 24, 32, 48, 64]
   ```

4. **Shadow extraction**: Parse box-shadow properties, convert to standardized format
   ```json
   {
     "shadow": {
       "card": {"x": 0, "y": 4, "blur": 12, "color": "rgba(0,0,0,0.1)"},
       "elevated": {"x": 0, "y": 8, "blur": 24, "color": "rgba(0,0,0,0.15)"}
     }
   }
   ```

5. **Border radius**: Collect all corner radius values, cluster into discrete set
   - Values: 4, 8, 12, 9999 → rounds: ["sm", "md", "lg", "full"]

### 3. Component Detection & Classification

Identify reusable component instances:

```bash
component-alchemist detect \
  --input ./specs/landing.json \
  --min-instances 2 \
  --similarity-threshold 0.9 \
  --output ./components/detected.json
```

**REAL Detection Logic:**
1. **Visual similarity clustering**: Use perceptual hashing or feature matching
   - Compare AutoLayout frames with identical child counts
   - Match layer structure tree with Levenshtein distance
   - Check style consistency (same text styles, fills, borders)
   
2. **Named component recognition**: Respect Figma's "Component" types
   ```json
   {
     "type": "button",
     "name": "Primary Button",
     "instances": 12,
     "props": {
       "label": "text",
       "variant": ["primary", "secondary"],
       "size": ["sm", "md", "lg"]
     }
   }
   ```

3. **State detection**:
   - Identify layers with `:hover`, `:active`, `:disabled` suffixes
   - Detect opacity changes (0.5 for disabled)
   - Find color shifts (primary→muted for hover states)

4. **Slot/children identification**: Find AutoLayouts with varying content
   - Count child layers differing in text/image
   - Mark as slots if variance > 30% across instances

**Real Output:**
```json
{
  "components": [
    {
      "id": "btn_primary",
      "type": "button",
      "source": "DesignSystem/Components/Button/Primary",
      "instances": 23,
      "states": ["default", "hover", "active", "disabled"],
      "props": {
        "children": "slot:text",
        "variant": "enum:primary,secondary,ghost",
        "size": "enum:sm,md,lg",
        "disabled": "boolean"
      },
      "styles": {
        "background": "token:color.primary",
        "padding": "token:spacing[2] token:spacing[3]",
        "borderRadius": "token:radius.md",
        "fontSize": "token:font.body.size"
      }
    }
  ]
}
```

### 4. Specification Generation

Generate framework-agnostic specs:

```bash
component-alchemist spec generate \
  --components ./components/detected.json \
  --tokens ./tokens/design-tokens.json \
  --output ./specs/component-specs.json \
  --schema ./schemas/component-spec-v2.json
```

**Spec Structure (REAL)**:
```json
{
  "specVersion": "2.1",
  "component": "Button",
  "metadata": {
    "source": "Figma: Design System > Components > Buttons",
    "lastAnalyzed": "2024-01-15T14:30:00Z",
    "author": "Design Team",
    "framework": "agnostic"
  },
  "props": {
    "children": {
      "type": "slot",
      "description": "Button label or icon",
      "required": true
    },
    "variant": {
      "type": "enum",
      "values": ["primary", "secondary", "ghost"],
      "default": "primary"
    },
    "size": {
      "type": "enum",
      "values": ["sm", "md", "lg"],
      "default": "md"
    },
    "disabled": {
      "type": "boolean",
      "default": false
    },
    "onClick": {
      "type": "function",
      "description": "Click handler"
    }
  },
  "states": {
    "default": { "background": "token:color.primary", "color": "white" },
    "hover": { "background": "token:color.primary-dark" },
    "active": { "background": "token:color.primary-darker" },
    "disabled": { "background": "token:color.gray-200", "color": "token:color.gray-400" }
  },
  "variants": {
    "primary": { "background": "token:color.primary" },
    "secondary": { "background": "token:color.gray-100", "color": "token:color.gray-900" },
    "ghost": { "background": "transparent", "border": "1px solid token:color.gray-300" }
  },
  "accessibility": {
    "keyboardNavigable": true,
    "focusVisible": "ring:2px token:color.focus",
    "ariaPressed": "when toggle",
    "minTouchTarget": "44px"
  },
  "dependencies": {
    "iconLibrary": "@oclaw/icons",
    "designTokens": "./tokens/design-tokens.json"
  },
  "tests": {
    "required": ["a11y", "interaction", "responsive"],
    "snapshots": ["all-states", "all-variants", "sizes"]
  }
}
```

### 5. Code Generation (Optional)

Generate actual component code:

```bash
component-alchemist generate \
  --spec ./specs/button-spec.json \
  --target vue \
  --output ./src/components/Button/ \
  --typescript \
  --with-storybook \
  --with-tests \
  --accessibility ./a11y/patterns.json
```

**Generated File Structure (REAL)**:
```
./src/components/Button/
├── Button.vue              # Main component
├── Button.test.ts          # Unit tests with Vitest
├── Button.stories.ts       # Storybook stories (8 stories)
├── Button.spec.ts          # Component spec documentation
├── Button.mocks.ts         # Mock props for stories
└── index.ts               # Exports
```

**Generated Code Example (Button.vue)**:
```vue
<script setup lang="ts">
import { computed } from 'vue'
import { useTokens } from '@/composables/design-tokens'

interface Props {
  variant?: 'primary' | 'secondary' | 'ghost'
  size?: 'sm' | 'md' | 'lg'
  disabled?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  variant: 'primary',
  size: 'md',
  disabled: false
})

const emit = defineEmits<{
  click: [MouseEvent]
}>()

const tokens = useTokens()

const classes = computed(() => [
  'btn',
  `btn--${props.variant}`,
  `btn--${props.size}`,
  {
    'btn--disabled': props.disabled
  }
])
</script>

<template>
  <button
    :class="classes"
    :disabled="disabled"
    @click="emit('click', $event)"
  >
    <slot />
  </button>
</template>

<style scoped>
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-family: v-bind('tokens.font.body.family');
  border-radius: v-bind('tokens.radius.md');
  cursor: pointer;
  transition: all 0.2s ease;
}
.btn--primary {
  background: v-bind('tokens.color.primary');
  color: white;
}
.btn--primary:hover:not(:disabled) {
  background: v-bind('tokens.color.primary.dark');
}
.btn--sm { padding: v-bind('tokens.spacing[1]') v-bind('tokens.spacing[2]'); font-size: 14px; }
.btn--md { padding: v-bind('tokens.spacing[2]') v-bind('tokens.spacing[3]'); font-size: 16px; }
.btn--lg { padding: v-bind('tokens.spacing[3]') v-bind('tokens.spacing[4]'); font-size: 18px; }
.btn--disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
</style>
```

## Golden Rules

### Design Fidelity Rules
1. **Pixel-Perfect Preservation**: Extract dimensions to 0.1px precision; never round unless design shows rounded values
2. **State Exhaustiveness**: If design shows 3 button states, generate all 3 plus standard states (focus, disabled, loading, error)
3. **Token Normalization**: Consolidate values within ±2.5% into single token; log merge decisions
4. **Hierarchy Integrity**: Preserve parent-child relationships; AutoLayout → Flex/Grid mapping
5. **A11y Requirements**: Extract aria-* attributes from layer names (e.g., "Button[aria-pressed=true]")

### Technical Rules
1. **No Assumptions**: If design shows 8px border-radius, don't convert to "sm" unless explicitly mapping
2. **Source Priority**: Figma components > AutoLayouts > Groups > Individual layers
3. **Breakpoint Detection**: Only infer responsive breakpoints if design shows multiple screen variants
4. **Image Handling**: Extract export settings (format: 1x/2x, compression) from Figma export rules
5. **Code Safety**: Generated code must pass ESLint + TypeScript strict mode without modifications

### Process Rules
1. **Always Validate**: Run `validate-specs` after generation before committing
2. **Token First**: Extract tokens before detecting components; tokens feed into component styles
3. **Document Sources**: Every token/component must cite source (Figma frame: x,y, page)
4. **Idempotency**: Running same analysis twice produces identical JSON (sort keys, deterministic hashes)
5. **Fail Fast**: Abort if input file corrupted or missing required layers; don't generate partial specs

## Examples

### Example 1: Analyze Figma Button Component

**Input Command:**
```bash
component-alchemist analyze \
  --input ./designs/figma/Button_Components.fig \
  --output ./specs/buttons.json \
  --format figma-json \
  --detect-states \
  --extract-tokens \
  --framework react
```

**Figma Structure:**
```
Button (Component)
├── Primary (Component Set)
│   ├── Default (Instance)
│   ├── Hover (Instance)
│   ├── Active (Instance)
│   └── Disabled (Instance)
├── Secondary (Component Set)
└── Tertiary (Component Set)
```

**Real Output (specs/buttons.json):**
```json
{
  "specVersion": "2.1",
  "generatedAt": "2024-01-15T10:30:00Z",
  "source": "Button_Components.fig (Figma)",
  "tokens": {
    "color": {
      "button-primary-bg": {"value": "#3B82F6", "type": "solid"},
      "button-primary-hover": {"value": "#2563EB"},
      "button-primary-active": {"value": "#1D4ED8"},
      "button-text": {"value": "#FFFFFF"}
    },
    "spacing": {
      "button-padding-x": {"value": 16},
      "button-padding-y": {"value": 10}
    },
    "typography": {
      "button-font": {"family": "Inter", "weight": 600, "size": 14}
    }
  },
  "components": [
    {
      "name": "Button",
      "type": "component",
      "baseComponent": true,
      "variants": ["primary", "secondary"],
      "props": {
        "children": {"type": "slot", "required": true},
        "variant": {"type": "enum", "values": ["primary", "secondary", "tertiary"], "default": "primary"},
        "size": {"type": "enum", "values": ["sm", "md", "lg"], "default": "md"},
        "disabled": {"type": "boolean", "default": false},
        "fullWidth": {"type": "boolean", "default": false}
      },
      "states": {
        "default": {"background": "token:color.button-primary-bg"},
        "hover": {"background": "token:color.button-primary-hover"},
        "active": {"background": "token:color.button-primary-active"},
        "disabled": {"opacity": 0.5, "cursor": "not-allowed"}
      }
    }
  ]
}
```

**Generated React Output (Button.tsx snippet):**
```typescript
export const Button: React.FC<ButtonProps> = ({
  children,
  variant = 'primary',
  size = 'md',
  disabled = false,
  fullWidth = false,
  onClick
}) => {
  const tokens = useDesignTokens()
  
  return (
    <button
      className={cn(
        'btn',
        `btn--${variant}`,
        `btn--${size}`,
        fullWidth && 'btn--full-width',
        disabled && 'btn--disabled'
      )}
      disabled={disabled}
      onClick={onClick}
      style={{
        backgroundColor: variant === 'primary' 
          ? tokens.color['button-primary-bg'] 
          : variant === 'secondary'
          ? tokens.color.gray[100]
          : 'transparent'
      }}
    >
      {children}
    </button>
  )
}
```

### Example 2: Extract Tokens from Marketing Screenshots

**Input Command:**
```bash
component-alchemist tokens extract \
  --sources ./marketing-screenshots/*.png,./brand-guidelines.json \
  --output ./design-system/tokens.css \
  --include colors,typography,spacing \
  --normalize \
  --deduplicate \
  --format css-custom-properties
```

**Sources:**
- 47 PNG screenshots from marketing site
- Existing brand guidelines JSON

**Real Output (tokens.css):**
```css
:root {
  /* Colors - extracted from 43 unique instances, normalized */
  --oclaw-color-primary: #3B82F6;
  --oclaw-color-primary-hover: #2563EB;
  --oclaw-color-secondary: #10B981;
  --oclaw-color-background: #FFFFFF;
  --oclaw-color-surface: #F9FAFB;
  --oclaw-color-text-primary: #111827;
  --oclaw-color-text-secondary: #6B7280;
  --oclaw-color-border: #E5E7EB;
  
  /* Typography - 4 distinct text styles discovered */
  --oclaw-font-heading: 'Inter', -apple-system, system-ui, sans-serif;
  --oclaw-font-body: 'Inter', -apple-system, system-ui, sans-serif;
  --oclaw-font-mono: 'JetBrains Mono', 'Fira Code', monospace;
  
  --oclaw-text-xs: 0.75rem;    /* 12px */
  --oclaw-text-sm: 0.875rem;   /* 14px */
  --oclaw-text-base: 1rem;     /* 16px */
  --oclaw-text-lg: 1.125rem;   /* 18px */
  --oclaw-text-xl: 1.25rem;    /* 20px */
  --oclaw-text-2xl: 1.5rem;    /* 24px */
  --oclaw-text-4xl: 2.25rem;   /* 36px */
  
  /* Spacing - 8px base scale (1=8, 2=16, 3=24, 4=32, 5=40, 6=48) */
  --oclaw-space-1: 0.5rem;   /* 8px */
  --oclaw-space-2: 1rem;     /* 16px */
  --oclaw-space-3: 1.5rem;   /* 24px */
  --oclaw-space-4: 2rem;     /* 32px */
  --oclaw-space-5: 2.5rem;   /* 40px */
  --oclaw-space-6: 3rem;     /* 48px */
  
  /* Border Radius */
  --oclaw-radius-sm: 0.25rem;   /* 4px */
  --oclaw-radius-md: 0.5rem;    /* 8px */
  --oclaw-radius-lg: 0.75rem;   /* 12px */
  --oclaw-radius-full: 9999px;
}

/* Report: 147 unique values → 37 tokens (74% reduction) */
/* Deduplication: merged 23 near-identical colors (ΔE < 2.3) */
```

### Example 3: Discover Component Patterns in Wild

**Input Command:**
```bash
component-alchemist discover \
  --screens ./screenshots/v2-dashboard/ \
  --min-similarity 0.88 \
  --output ./patterns/dashboard-components.json \
  --group-by layout
```

**Screenshots:** 156 PNGs from dashboard v2 redesign

**Real Output:**
```json
{
  "analysis": {
    "totalScreens": 156,
    "uniqueLayouts": 12,
    "componentsDetected": 8
  },
  "patterns": [
    {
      "name": "MetricCard",
      "type": "container",
      "occurrences": 47,
      "confidence": 0.94,
      "structure": {
        "root": "AutoLayout",
        "children": [
          {"type": "Text", "role": "label", "style": "heading-sm"},
          {"type": "Text", "role": "value", "style": "heading-2xl"},
          {"type": "Icon", "role": "trend", "optional": true}
        ]
      },
      "props": {
        "label": {"type": "string", "required": true},
        "value": {"type": "number|string", "required": true},
        "trend": {"type": "enum:up,down,neutral", "required": false},
        "format": {"type": "enum:number,currency,percentage", "default": "number"}
      }
    },
    {
      "name": "DataTable",
      "type": "container",
      "occurrences": 3,
      "confidence": 0.91,
      "structure": {
        "root": "AutoLayout (vertical)",
        "children": [
          {"type": "Container", "role": "header", "children": ["Text:column1", "Text:column2", ...]},
          {"type": "Container", "role": "rows", "repeating": true}
        ]
      }
    }
  ]
}
```

**Actionable Insight:**
```bash
# Now generate specs for discovered components
component-alchemist spec generate \
  --components ./patterns/dashboard-components.json \
  --output ./specs/dashboard-specs.json
```

### Example 4: Validate Implementation Against Design

**Input Command:**
```bash
component-alchemist validate \
  --spec ./specs/modal-spec.json \
  --component ./src/components/Modal.vue \
  --mode visual \
  --pixel-threshold 2 \
  --report ./reports/modal-validation.html
```

**What Happens:**
1. Renders `Modal.vue` in headless browser with test props
2. Takes screenshot, compares to design reference (from spec's `referenceScreenshot` field)
3. Runs pixel diff algorithm (pixelmatch)
4. Highlights misalignments > 2px threshold
5. Generates interactive HTML report

**Real Report (validation summary):**
```
VALIDATION RESULTS: Modal.spec ↔ Modal.vue

✓ Layout structure matches (100%)
✓ Colors match (±0 ΔE)
✓ Typography matches (exact)
✗ Padding: left 24px vs design 16px (8px diff)
✗ Corner radius: 8px vs design 12px (4px diff)
✗ Shadow blur: 16px vs design 24px (8px diff)

Score: 87/100 (threshold: 95)

Issues blocking merge:
1. Padding mismatch in .modal--content (expected 16px, got 24px)
2. Border-radius should be 12px (currently 8px)
3. Box-shadow blur too low (expected 24px, got 16px)

Recommended fix:
  .modal--content { padding: 16px; }
  .modal { border-radius: 12px; }
  .modal__overlay { box-shadow: 0 8px 32px rgba(0,0,0,0.12); }
```

### Example 5: Generate Full Component Library

**Input Command:**
```bash
component-alchemist generate-all \
  --specs-dir ./specs/ \
  --tokens ./design-system/tokens.json \
  --target vue \
  --output ./src/components/generated/ \
  --typescript \
  --with-storybook \
  --with-tests \
  --accessibility ./a11y/patterns.json
```

**Process:**
1. Reads all `*.spec.json` files from `./specs/`
2. For each spec:
   - Validate against schema (`./schemas/component-spec-v2.json`)
   - Resolve token references
   - Generate component file
   - Generate test file (Vitest)
   - Generate Storybook stories (CSF3 format)
   - Generate a11y test suite (axe-core integration)
3. Write `index.ts` barrel export

**Real Output Structure:**
```
./src/components/generated/
├── index.ts                         # Export all components
├── tokens.css                       # Imports global tokens
├── Button/
│   ├── Button.vue
│   ├── Button.test.ts
│   ├── Button.stories.ts    (8 stories: default, variants, sizes, states)
│   ├── Button.spec.ts
│   └── mocks.ts
├── Card/
│   ├── Card.vue
│   ├── Card.test.ts
│   ├── Card.stories.ts      (5 stories: default, with-image, with-actions)
│   └── Card.a11y.ts         # axe-core tests
├── Modal/
│   ├── Modal.vue
│   ├── Modal.test.ts
│   └── Modal.stories.ts
└── shared/
    └── types.ts             # Common TypeScript interfaces
```

**Barrel Export (index.ts):**
```typescript
export { default as Button } from './Button/Button.vue'
export { default as Card } from './Card/Card.vue'
export { default as Modal } from './Modal/Modal.vue'
export type { ButtonProps } from './Button/Button.spec'
export type { CardProps } from './Card/Card.spec'
```

## Rollback Commands

### 1. Revert Generated Components
```bash
# If generated code was committed and needs rollback
component-alchemist rollback generate \
  --target-dir ./src/components/generated/ \
  --commit "revert: rollback generated components" \
  --preserve-overrides \
  --backup-to ./backup/generated-components-20240115/
```

**What it does:**
- Identifies files with generator comment header (`<!-- Generated by Component Alchemist -->`)
- Reverts changes via `git restore` but preserves manual edits in files NOT containing header
- Creates backup of all generated files with timestamp
- Commits rollback with descriptive message

### 2. Restore Previous Spec Version
```bash
# Spec file got corrupted; restore from git
component-alchemist rollback spec \
  --spec ./specs/button-spec.json \
  --to-commit abc1234 \
  --preview-diff
```

### 3. Clean Temporary Analysis Files
```bash
# Clear all .alchemist-cache, .tmp, intermediate exports
component-alchemist rollback clean \
  --path ./specs/ \
  --dry-run \
  --confirm
# Removes:
# - *.analysis.tmp.json
# - .alchemist-cache/
# - layers-parsed/
# - extracted-tokens-*
```

### 4. Revert Token Merges
```bash
# Tokens deduplication merged #3E8AC7 and #3E8BC8; want to split
component-alchemist tokens revert-merge \
  --tokens ./tokens/design-tokens.json \
  --pre-threshold 0.9 \
  --output ./tokens/restored.json
# Restores all pre-merge distinct values (increases token count)
```

### 5. Full Reset to Clean State
```bash
# Remove ALL generated output, start fresh
component-alchemist reset \
  --specs-dir ./specs/ \
  --components-dir ./src/components/generated/ \
  --tokens ./tokens/design-tokens.json \
  --confirm
```

**Interactive Confirmation:**
```
? This will permanently delete:
  - 47 spec files in ./specs/
  - 156 component files in ./src/components/generated/
  - 1 token file (372 tokens)
  Proceed? (Y/n)
```

## Dependencies & Requirements

### System Dependencies
```bash
# Required
- Node.js >= 18.0.0
- Python 3.9+ (for image processing)
- ImageMagick (for PNG/JPEG analysis)
- Ghostscript (for PDF layer extraction)
- Figma CLI (optional, for live Figma API access)

# Optional but recommended
- sharp (npm) - fast image processing
- pixelmatch (npm) - visual diff
- jimp (npm) - alternative image analysis
```

### NPM Dependencies (in package.json):
```json
{
  "dependencies": {
    "@figma/plugin-typings": "^1.82.0",
    "pixelmatch": "^5.3.0",
    "sharp": "^0.33.0",
    "color-convert": "^2.3.1",
    "lodash": "^4.17.21",
    "commander": "^11.1.0",
    "chalk": "^4.1.2",
    "ora": "^5.4.1",
    "jimp": "^0.22.10",
    "xml2js": "^0.6.2",
    "css-tree": "^2.3.1",
    "design-systems-utils": "^1.4.0"
  },
  "devDependencies": {
    "@types/pixelmatch": "^5.2.4",
    "@types/node": "^20.0.0",
    "vitest": "^1.0.0",
    "@storybook/vue3": "^7.6.0",
    "axe-core": "^4.8.0"
  }
}
```

### Design File Requirements
- **Figma**: .fig files or exported .json via Figma API (requires access token)
- **Sketch**: .sketch files (unencrypted)
- **PSD**: .psd files (layers must not be rasterized completely)
- **Screenshots**: PNG, JPEG, WebP (min 1024px width for accurate extraction)
- **DrawIO**: .drawio.xml exports (limited support)

### Access Requirements
```bash
# For Figma live API
export FIGMA_API_TOKEN="figd_xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
export FIGMA_FILE_KEY="figmaFileKeyFromURL"

# For design system integration
export DESIGN_SYSTEM_URL="https://design.system.openclaw.io"
export DESIGN_SYSTEM_TOKEN="ds_token_xxx"
```

## Environment Variables

```bash
# Core configuration
export COMPONENT_ALCHEMIST_HOME="${HOME}/.component-alchemist"
export COMPONENT_ALCHEMIST_CACHE_DIR="${COMPONENT_ALCHEMIST_HOME}/cache"
export COMPONENT_ALCHEMIST_SPEC_SCHEMA="${COMPONENT_ALCHEMIST_HOME}/schemas/component-spec-v2.json"

# Token extraction tuning
export ALCHEMIST_TOKEN_PRECISION=2              # Decimal places for numeric values
export ALCHEMIST_COLOR_TOLERANCE=2.3           # ΔE threshold for deduplication
export ALCHEMIST_SPACING_SCALE="4,8,12,16,24,32,48,64"
export ALCHEMIST_NORMALIZE_TOKENS=true

# Analysis behavior
export ALCHEMIST_MAX_PARALLEL_JOBS=4
export ALCHEMIST_MEMORY_LIMIT_MB=2048
export ALCHEMIST_VERBOSITY="info"  # quiet|info|verbose|debug

# Framework defaults
export ALCHEMIST_DEFAULT_FRAMEWORK="vue"
export ALCHEMIST_GENERATE_TESTS=true
export ALCHEMIST_GENERATE_STORYBOOK=true

# Validation
export ALCHEMIST_PIXEL_DIFF_THRESHOLD=2
export ALCHEMIST_VISUAL_VIEWPORT_WIDTH=1280
export ALCHEMIST_VISUAL_VIEWPORT_HEIGHT=800

# Output formatting
export ALCHEMIST_REPORT_FORMAT="html"  # html|json|markdown
export ALCHEMIST_USE_CSHARP_NAMES=false # camelCase vs PascalCase
```

## Verification Steps

### After Generation

1. **Spec Schema Validation**
```bash
component-alchemist validate-schema \
  --spec ./specs/button-spec.json \
  --schema ./schemas/component-spec-v2.json
# Should return: {"valid": true, "errors": []}
```

2. **Token Consistency Check**
```bash
component-alchemist tokens verify \
  --tokens ./tokens/design-tokens.json \
  --components ./specs/*.json
# Ensures all token references in specs exist in token file
```

3. **Generated Code Linting**
```bash
npm run lint -- --ext .vue,.ts ./src/components/generated/
# Expected: 0 errors, 0 warnings
```

4. **TypeScript Check**
```bash
npx tsc --noEmit ./src/components/generated/**/*.ts
# Expected: No type errors
```

5. **Visual Regression Test**
```bash
npm run test:visual -- --component=Button --mode=all-states
# Compare generated component screenshots against reference images
```

6. **A11y Audit**
```bash
npm run test:a11y -- --component=Button
# Expected: 0 violations
```

7. **Storybook Build**
```bash
npm run build-storybook -- --quiet
# Expected: Build succeeds without errors
```

### Integration Verification

```bash
# 1. Import generated component in app
# src/App.vue:
import { Button } from '@/components/generated'

# 2. Run app and visually inspect
npm run dev
# Open http://localhost:3000

# 3. Check bundle size impact
npm run build -- --report
# Verify generated components add <50KB gzipped

# 4. Test in isolation
npm run test:unit Button.test.ts
# Expected: All tests pass
```

## Troubleshooting

### Issue: "Failed to parse Figma file: unsupported format"

**Cause:** Figma file version > supported plugin API version.

**Fix:**
```bash
# Update Figma plugin to latest
npm update @figma/plugin-typings

# Or export as JSON from Figma manually
figma Dev Mode > Select file > Copy JSON > paste to file
component-alchemist analyze --input exported.json --format figma-json
```

### Issue: "Colors detected: 312 unique values (expected < 50)"

**Cause:** Design contains many unique colors (gradients, images, effects).

**Fix:**
```bash
# Filter only solid fills
component-alchemist tokens extract \
  --sources designs/ \
  --include colors \
  --color-types solid,gradient  # specify types to include
# Or manually inspect and add to denylist:
# echo "gradient*,image*,effect*" >> .alchemist/ignore-colors.txt
```

### Issue: "Component detection failed: similarity threshold too low"

**Cause:** Design has inconsistent AutoLayouts or layer naming.

**Fix:**
```bash
# Lower similarity threshold gradually
component-alchemist discover \
  --screens ./screens/ \
  --min-similarity 0.75  # from 0.88
# Or manually group components in Figma using proper Component Sets
```

### Issue: "Generated TypeScript errors: Property 'xxx' does not exist"

**Cause:** Spec prop type doesn't match framework expectations.

**Fix:**
```bash
# Regenerate with target framework explicitly set
component-alchemist generate \
  --spec ./specs/card.json \
  --target vue \
  --typescript strict

# Or patch the spec manually and re-run generation
component-alchemist generate \
  --spec ./specs/card-patched.json \
  --output ./src/components/
```

### Issue: "Visual validation: pixel diff exceeds threshold (87/100)"

**Cause:** Implementation deviates from design specs.

**Debug:**
```bash
# Generate detailed diff report with highlights
component-alchemist validate \
  --spec ./specs/button.json \
  --component ./src/components/Button.vue \
  --mode visual \
  --output ./reports/diff.html \
  --highlight-all

# Open report, inspect mismatched areas
# Common fixes:
# 1. Box-sizing: border-box missing
# 2. Font loading delay causing FOUT
# 3. Subpixel rounding differences (use transform: translateZ(0))
```

### Issue: "Token extraction: out of memory (MB limit exceeded)"

**Cause:** Processing large image directory (1000+ PNGs).

**Fix:**
```bash
# Increase memory limit
export ALCHEMIST_MEMORY_LIMIT_MB=4096

# Or process in batches
component-alchemist tokens extract \
  --sources "./screenshots/group1/*.png,./screenshots/group2/*.png" \
  --output ./tokens/partial-1.json
# Merge later:
component-alchemist tokens merge \
  --tokens ./tokens/partial-*.json \
  --output ./tokens/complete.json
```

### Issue: "State detection incomplete: missing hover/active states"

**Cause:** Figma file doesn't use component variants or naming convention.

**Fix:**
```bash
# Manually annotate in Figma naming:
# "Button/Primary:default"  "Button/Primary:hover"  "Button/Primary:active"

# Or run with relaxed rules and manually complete spec
component-alchemist analyze \
  --input designs.fig \
  --detect-states relaxed \
  --output specs/auto.json

# Then edit spec to add missing states manually
# Edit specs/auto.json, add:
# "states": {"default": {...}, "hover": {...}, "active": {...}}
```

### Issue: "Generate command failed: 'framework' not recognized"

**Cause:** Framework not in supported list.

**Supported frameworks:**
- `react` (with options: --typescript|--javascript)
- `vue` (with options: --vue-version 2|3, --typescript)
- `svelte` (with --svelte4)
- `angular` (with --standalone)
- `web-components` (with --stencil|--lit)

**Fix:**
```bash
# Use exact framework name
component-alchemist generate \
  --spec ./specs/component.json \
  --target vue \
  --vue-version 3 \
  --typescript
```

### Performance: Slow analysis on 50+ screen files

**Debug & Optimize:**
```bash
# 1. Check cache hit rate
ls .alchemist-cache/  # Should have cached thumbnails/hashes

# 2. Increase parallelism
export ALCHEMIST_MAX_PARALLEL_JOBS=8  # if CPU has 8+ cores

# 3. Skip re-analyzing unchanged files
component-alchemist analyze \
  --input ./screens/ \
  --skip-cache false  # use cache aggressively

# 4. Pre-generate thumbnails (faster similarity matching)
component-alchemist preheat \
  --screens ./screens/ \
  --size 256x256 \
  --output .alchemist-cache/thumbnails/
```

### Issue: "Rollback failed: no git repository"

**Cause:** Generated files not yet committed.

**Fix:**
```bash
# Don't use rollback; manual cleanup
rm -rf ./src/components/generated/
# Or if you want to preserve some manual edits:
component-alchemist rollback clean \
  --path ./src/components/generated/ \
  --dry-run  # see what will be deleted
# Then remove without commit:
component-alchemist reset --confirm
```

## Schema Reference

The Component Alchemist validates all specs against `./schemas/component-spec-v2.json`:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Component Specification",
  "version": "2.1",
  "required": ["specVersion", "component", "props", "states"],
  "properties": {
    "specVersion": {"type": "string", "enum": ["2.0", "2.1"]},
    "component": {"type": "string", "minLength": 1},
    "metadata": {
      "type": "object",
      "properties": {
        "source": {"type": "string"},
        "lastAnalyzed": {"type": "string", "format": "date-time"},
        "author": {"type": "string"},
        "framework": {"type": "string"}
      }
    },
    "props": {
      "type": "object",
      "patternProperties": {
        "^[a-zA-Z]+$": {
          "oneOf": [
            {"type": "object", "required": ["type"]},
            {"type": "string"}
          ]
        }
      }
    },
    "states": {"type": "object"},
    "variants": {"type": "object"},
    "accessibility": {"type": "object"},
    "dependencies": {"type": "object"},
    "tests": {"type": "object"}
  }
}
```

---
*Generated by Component Alchemist v2.1.0 • For schema updates, run: component-alchemist schema export --pretty*
```