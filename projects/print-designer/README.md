# 🖨️ Print Designer — Enterprise Project Structure

> **React 19 · TypeScript · Vite · Fabric.js 6 · Zustand · Tailwind CSS**

A web-based print designer canvas SPA mirroring the w2p-printerp-pod designer. Customers use this tool to create designs on top of product templates with full support for multi-side products, variable data printing, AI artwork generation, and print-ready submission.

---

## Testing Strategy

**Co-located unit tests** — every source file has a sibling `.test.ts` / `.test.tsx` right next to it.
**Centralised test infrastructure** — `src/__tests__/` holds only shared setup, mocks, fixtures, and E2E specs.

| Layer | Location | Runner |
|---|---|---|
| Unit tests (components) | `ComponentName.test.tsx` beside source | Vitest + Testing Library |
| Unit tests (hooks/utils) | `hookName.test.ts` beside source | Vitest |
| Unit tests (store slices) | `slice-name.test.ts` beside source | Vitest |
| Unit tests (services/API) | `service-name.test.ts` beside source | Vitest + MSW |
| Test infrastructure | `src/__tests__/setup.ts`, `test-utils.tsx`, `mocks/` | — |
| E2E tests | `src/__tests__/e2e/*.spec.ts` | Playwright |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | React 19 + TypeScript 5.x (strict) |
| Build Tool | Vite 6 |
| Canvas Engine | Fabric.js 6 |
| State Management | Zustand + Immer |
| GraphQL | Apollo Client + GraphQL Code Generator |
| Styling | Tailwind CSS 4 |
| Routing | React Router v7 |
| i18n | i18next |
| Unit Testing | Vitest + Testing Library |
| E2E Testing | Playwright |
| Linting | ESLint (flat config) + Prettier |
| CI/CD | GitHub Actions |
| Containerisation | Docker + nginx |

---

## Directory Structure

```
print-designer/
│
├── .husky/                                    # Git hooks
│   ├── pre-commit                             # Runs lint-staged before each commit
│   └── commit-msg                             # Enforces conventional commit format
│
├── .github/                                   # CI/CD & repo config
│   ├── workflows/
│   │   ├── ci.yml                             # PR checks — lint, typecheck, unit tests, build
│   │   ├── deploy-staging.yml                 # Auto-deploy to staging on merge to develop
│   │   ├── deploy-production.yml              # Manual approval deploy to production on main
│   │   └── lighthouse.yml                     # Performance budget checks via Lighthouse CI
│   ├── CODEOWNERS                             # Defines ownership per directory for PR reviews
│   └── pull_request_template.md               # Standardised PR description template
│
├── public/                                    # Static assets
│   ├── favicon.ico                            # App favicon
│   ├── manifest.json                          # PWA manifest for installable web app
│   └── robots.txt                             # Search engine crawl directives
│
├── src/
│   │
│   ├── app/                                   # App shell
│   │   ├── App.tsx                            # Root component — providers, error boundary, router
│   │   ├── App.test.tsx                       # Tests provider composition, error boundary, routes
│   │   ├── routes.tsx                         # React Router v7 route definitions (lazy-loaded)
│   │   ├── routes.test.tsx                    # Tests route matching, lazy loading, redirects
│   │   ├── providers.tsx                      # Composes all context providers
│   │   ├── global-error-boundary.tsx          # Top-level error boundary with crash reporter
│   │   └── global-error-boundary.test.tsx     # Tests error catching, fallback UI, crash report
│   │
│   ├── pages/                                 # Route-level page components
│   │   ├── designer/
│   │   │   ├── DesignerPage.tsx               # Main designer page — canvas + toolbar orchestrator
│   │   │   └── DesignerPage.test.tsx          # Tests designer page lifecycle, layout rendering
│   │   ├── my-designs/
│   │   │   ├── MyDesignsPage.tsx              # Grid/list of saved designs with search & filters
│   │   │   └── MyDesignsPage.test.tsx         # Tests listing, filtering, search, empty state
│   │   ├── submission/
│   │   │   ├── SubmissionPage.tsx             # Print-ready submission review, preflight, order
│   │   │   └── SubmissionPage.test.tsx        # Tests preflight validation, proof preview, submit
│   │   └── not-found/
│   │       ├── NotFoundPage.tsx               # 404 page with navigation back to designer
│   │       └── NotFoundPage.test.tsx          # Tests 404 render and navigation link
│   │
│   ├── features/                              # Feature modules (self-contained)
│   │   │
│   │   ├── canvas/                            # [CORE] Fabric.js canvas engine
│   │   │   ├── components/
│   │   │   │   ├── DesignCanvas.tsx           # Fabric.js canvas wrapper — init, resize, DPI
│   │   │   │   ├── DesignCanvas.test.tsx      # Tests canvas init, resize, DPI scaling, cleanup
│   │   │   │   ├── CanvasWorkspace.tsx        # Artboard container with drop zone, shortcuts
│   │   │   │   ├── CanvasWorkspace.test.tsx   # Tests drop zone, shortcut binding, layout
│   │   │   │   ├── CanvasOverlay.tsx          # Rulers, gridlines, safe-zone & bleed SVG overlay
│   │   │   │   ├── CanvasOverlay.test.tsx     # Tests guide positions, unit switching
│   │   │   │   ├── ObjectControls.tsx         # Custom control handles (rotate, resize, lock)
│   │   │   │   ├── ObjectControls.test.tsx    # Tests control visibility, actions, lock state
│   │   │   │   ├── ContextMenu.tsx            # Right-click menu: copy, paste, align, order, lock
│   │   │   │   └── ContextMenu.test.tsx       # Tests menu items, disabled states, handlers
│   │   │   ├── hooks/
│   │   │   │   ├── useCanvas.ts              # Canvas instance via context; init/dispose lifecycle
│   │   │   │   ├── useCanvas.test.ts          # Tests context provision, init/dispose lifecycle
│   │   │   │   ├── useCanvasEvents.ts        # Binds Fabric.js events to store actions
│   │   │   │   ├── useCanvasEvents.test.ts    # Tests event binding, dispatch, cleanup
│   │   │   │   ├── useZoomPan.ts             # Scroll/pinch zoom, spacebar pan, fit-to-screen
│   │   │   │   ├── useZoomPan.test.ts         # Tests zoom limits, pan bounds, fit calculation
│   │   │   │   ├── useKeyboardShortcuts.ts   # Global & canvas-scoped shortcuts
│   │   │   │   ├── useKeyboardShortcuts.test.ts # Tests shortcut map, modifiers, scope isolation
│   │   │   │   ├── useCanvasSync.ts          # Two-way sync: Fabric.js ↔ Zustand store
│   │   │   │   └── useCanvasSync.test.ts      # Tests bidirectional sync, conflict, debounce
│   │   │   ├── utils/
│   │   │   │   ├── canvas-factory.ts         # Creates pre-configured Fabric.js instances
│   │   │   │   ├── canvas-factory.test.ts     # Tests factory config, defaults, static vs interactive
│   │   │   │   ├── coordinate-utils.ts       # px↔mm↔in conversion, DPI-aware mapping
│   │   │   │   ├── coordinate-utils.test.ts   # Tests unit conversions, DPI scaling, edge cases
│   │   │   │   ├── object-serializer.ts      # Custom toJSON/fromJSON for all object types
│   │   │   │   ├── object-serializer.test.ts  # Tests round-trip serialization per object type
│   │   │   │   ├── render-pipeline.ts        # Offscreen 300 DPI render for print export
│   │   │   │   ├── render-pipeline.test.ts    # Tests output dimensions, DPI, memory cleanup
│   │   │   │   ├── hit-testing.ts            # Hit-test for irregular shapes & groups
│   │   │   │   └── hit-testing.test.ts        # Tests hit detection: shapes, groups, rotation
│   │   │   └── constants.ts                  # Canvas defaults: DPI, zoom, grid snap, min size
│   │   │
│   │   ├── multi-side/                        # [CORE] Multi-side product canvas
│   │   │   ├── components/
│   │   │   │   ├── SideNavigator.tsx          # Thumbnail strip of all product sides
│   │   │   │   ├── SideNavigator.test.tsx     # Tests side switching, thumbnails, active state
│   │   │   │   ├── SidePreview.tsx            # Live thumbnail via offscreen canvas
│   │   │   │   └── SidePreview.test.tsx       # Tests offscreen render, preview update cycle
│   │   │   ├── hooks/
│   │   │   │   ├── useProductSides.ts         # Side definitions, active side state
│   │   │   │   └── useProductSides.test.ts    # Tests side CRUD, switching, state persistence
│   │   │   └── types.ts                      # ProductSide, SideConfig, SideCanvasState
│   │   │
│   │   ├── templates/                         # Template system
│   │   │   ├── components/
│   │   │   │   ├── TemplateGallery.tsx        # Browsable/searchable template gallery
│   │   │   │   ├── TemplateGallery.test.tsx   # Tests search, categories, loading states
│   │   │   │   ├── TemplatePreviewCard.tsx    # Template thumbnail with apply button
│   │   │   │   ├── TemplatePreviewCard.test.tsx # Tests card render, apply action, hover
│   │   │   │   ├── TemplateLayerLock.tsx      # Locked vs editable layer indicator
│   │   │   │   └── TemplateLayerLock.test.tsx # Tests lock icon, edit restriction, unlock
│   │   │   ├── hooks/
│   │   │   │   ├── useTemplates.ts            # Fetch, apply, manage locked layers
│   │   │   │   └── useTemplates.test.ts       # Tests fetch, apply, lock enforcement, errors
│   │   │   └── services/
│   │   │       ├── template-api.ts            # API client for template CRUD
│   │   │       ├── template-api.test.ts       # Tests API calls, payloads, errors (MSW)
│   │   │       ├── template-parser.ts         # Template JSON → Fabric.js objects
│   │   │       └── template-parser.test.ts    # Tests parsing, object creation, lock metadata
│   │   │
│   │   ├── text-tool/                         # Text tool
│   │   │   ├── components/
│   │   │   │   ├── TextToolbar.tsx            # Font, size, weight, alignment, spacing toolbar
│   │   │   │   ├── TextToolbar.test.tsx       # Tests rendering, font change, alignment toggle
│   │   │   │   ├── FontPicker.tsx             # Searchable font browser with live preview
│   │   │   │   ├── FontPicker.test.tsx        # Tests search, selection, preview, lazy load
│   │   │   │   ├── TextEffectsPanel.tsx       # Curved, outline, shadow, glow, gradient
│   │   │   │   ├── TextEffectsPanel.test.tsx  # Tests effect toggle, sliders, preview update
│   │   │   │   ├── InlineTextEditor.tsx       # On-canvas editing with cursor, selection, IME
│   │   │   │   └── InlineTextEditor.test.tsx  # Tests text input, cursor, selection, IME
│   │   │   ├── hooks/
│   │   │   │   ├── useTextTool.ts             # Add/edit text, apply formatting
│   │   │   │   ├── useTextTool.test.ts        # Tests creation, formatting, edit mode
│   │   │   │   ├── useFontLoader.ts           # Async font loading with FontFace API
│   │   │   │   └── useFontLoader.test.ts      # Tests load, cache hit, failure fallback
│   │   │   └── utils/
│   │   │       ├── text-effects.ts            # Arc/wave text, outlines, gradients renderer
│   │   │       ├── text-effects.test.ts       # Tests arc calc, outline render, gradient stops
│   │   │       ├── font-registry.ts           # Font catalog with metadata
│   │   │       └── font-registry.test.ts      # Tests lookup, category filter, weight check
│   │   │
│   │   ├── image-upload/                      # Image upload
│   │   │   ├── components/
│   │   │   │   ├── ImageUploader.tsx          # Drag-drop / file-picker with progress
│   │   │   │   ├── ImageUploader.test.tsx     # Tests drag-drop, validation, progress, error
│   │   │   │   ├── ImageCropper.tsx           # Crop/rotate modal
│   │   │   │   ├── ImageCropper.test.tsx      # Tests crop area, rotation, aspect lock
│   │   │   │   ├── ImageFilters.tsx           # Brightness, contrast, saturation controls
│   │   │   │   └── ImageFilters.test.tsx      # Tests filter sliders, reset, live preview
│   │   │   ├── hooks/
│   │   │   │   ├── useImageUpload.ts          # Validation, compression, CDN upload, place
│   │   │   │   └── useImageUpload.test.ts     # Tests validation, compression, upload mock
│   │   │   └── utils/
│   │   │       ├── image-processor.ts         # Resize, EXIF rotation, format conversion
│   │   │       ├── image-processor.test.ts    # Tests resize output, EXIF, format convert
│   │   │       ├── dpi-checker.ts             # Effective DPI at print size; threshold warn
│   │   │       └── dpi-checker.test.ts        # Tests DPI calc at various sizes, warnings
│   │   │
│   │   ├── clipart/                           # Clipart browser
│   │   │   ├── components/
│   │   │   │   ├── ClipartBrowser.tsx         # Category tree + search with virtual grid
│   │   │   │   ├── ClipartBrowser.test.tsx    # Tests categories, search, virtual scroll
│   │   │   │   ├── ClipartCard.tsx            # Lazy thumbnail with drag/click-to-add
│   │   │   │   └── ClipartCard.test.tsx       # Tests lazy load, drag data, click action
│   │   │   ├── hooks/
│   │   │   │   ├── useClipart.ts              # Paginated fetch, category, search, scroll
│   │   │   │   └── useClipart.test.ts         # Tests pagination, filter, search debounce
│   │   │   └── services/
│   │   │       ├── clipart-api.ts             # API for search, categories, SVG/PNG
│   │   │       └── clipart-api.test.ts        # Tests endpoints, query params (MSW)
│   │   │
│   │   ├── backgrounds/                       # Background images
│   │   │   ├── components/
│   │   │   │   ├── BackgroundPanel.tsx        # Solid, gradient, pattern, image picker
│   │   │   │   ├── BackgroundPanel.test.tsx   # Tests tab switching, color/gradient apply
│   │   │   │   ├── BackgroundImageBrowser.tsx # Browse/search background images
│   │   │   │   └── BackgroundImageBrowser.test.tsx # Tests grid, search, selection, preview
│   │   │   └── hooks/
│   │   │       ├── useBackground.ts           # Set background, tiling & fit modes
│   │   │       └── useBackground.test.ts      # Tests solid/gradient/image, fit modes
│   │   │
│   │   ├── color-picker/                      # Universal color picker
│   │   │   ├── components/
│   │   │   │   ├── ColorPicker.tsx            # Wheel, sliders, hex/RGB/CMYK, swatches
│   │   │   │   ├── ColorPicker.test.tsx       # Tests wheel, sliders, hex input parsing
│   │   │   │   ├── SwatchPalette.tsx          # Preset palettes, recent, brand colours
│   │   │   │   ├── SwatchPalette.test.tsx     # Tests palette render, recent tracking
│   │   │   │   ├── GradientEditor.tsx         # Linear/radial gradient with stop editor
│   │   │   │   └── GradientEditor.test.tsx    # Tests stop add/remove, drag, type toggle
│   │   │   ├── hooks/
│   │   │   │   ├── useColorPicker.ts          # Color state, format conversion, CMYK
│   │   │   │   └── useColorPicker.test.ts     # Tests state updates, format sync, CMYK
│   │   │   └── utils/
│   │   │       ├── color-conversions.ts       # RGB↔CMYK↔HSL↔HEX with gamut warnings
│   │   │       └── color-conversions.test.ts  # Tests all conversions, edges, gamut
│   │   │
│   │   ├── shapes/                            # Shapes library
│   │   │   ├── components/
│   │   │   │   ├── ShapesPanel.tsx            # Shape grid: basic, arrows, callouts, SVG
│   │   │   │   ├── ShapesPanel.test.tsx       # Tests grid render, categories, insert
│   │   │   │   ├── ShapeProperties.tsx        # Fill, stroke, radius, opacity controls
│   │   │   │   └── ShapeProperties.test.tsx   # Tests controls, value sync with object
│   │   │   ├── hooks/
│   │   │   │   ├── useShapes.ts               # Insert, edit path, stroke↔fill convert
│   │   │   │   └── useShapes.test.ts          # Tests insertion, path edit, conversion
│   │   │   └── data/
│   │   │       └── shape-library.ts           # SVG paths for all built-in shapes
│   │   │
│   │   ├── name-number/                       # Name & Number (sports personalisation)
│   │   │   ├── components/
│   │   │   │   ├── NameNumberPanel.tsx        # Bulk name/number entry grid
│   │   │   │   ├── NameNumberPanel.test.tsx   # Tests grid input, row CRUD, validation
│   │   │   │   ├── NameNumberPreview.tsx      # Live preview cycling through roster
│   │   │   │   ├── NameNumberPreview.test.tsx # Tests cycling, canvas update, navigation
│   │   │   │   ├── RosterImport.tsx           # CSV/Excel upload with column mapping
│   │   │   │   └── RosterImport.test.tsx      # Tests upload, column detect, mapping
│   │   │   ├── hooks/
│   │   │   │   ├── useNameNumber.ts           # Roster state, canvas binding, batch export
│   │   │   │   └── useNameNumber.test.ts      # Tests roster CRUD, binding, export trigger
│   │   │   └── types.ts                      # RosterEntry, NameNumberConfig, PlacementRule
│   │   │
│   │   ├── vdp/                               # Variable Data Printing
│   │   │   ├── components/
│   │   │   │   ├── VdpPanel.tsx               # CSV upload, field mapping, placeholders
│   │   │   │   ├── VdpPanel.test.tsx          # Tests upload flow, field list, placeholder
│   │   │   │   ├── VdpFieldMapper.tsx         # Map CSV columns → canvas placeholders
│   │   │   │   ├── VdpFieldMapper.test.tsx    # Tests dropdown, mapping, unmapped warn
│   │   │   │   ├── VdpRecordNavigator.tsx     # Paginate records with live preview
│   │   │   │   ├── VdpRecordNavigator.test.tsx # Tests prev/next, count, canvas swap
│   │   │   │   ├── VdpBatchPreview.tsx        # Thumbnail grid of all records
│   │   │   │   └── VdpBatchPreview.test.tsx   # Tests grid render, thumbnails, scroll
│   │   │   ├── hooks/
│   │   │   │   ├── useVdp.ts                  # CSV parse, record iteration, substitution
│   │   │   │   └── useVdp.test.ts             # Tests parsing, nav, placeholder resolution
│   │   │   └── utils/
│   │   │       ├── csv-parser.ts              # CSV parser: encoding, delimiter sniffing
│   │   │       ├── csv-parser.test.ts         # Tests delimiters, encoding, malformed rows
│   │   │       ├── placeholder-engine.ts      # Resolves {{field}} per record
│   │   │       └── placeholder-engine.test.ts # Tests matching, nested, missing fallback
│   │   │
│   │   ├── rulers-grid/                       # Ruler & gridlines
│   │   │   ├── components/
│   │   │   │   ├── Ruler.tsx                  # H/V rulers with tick marks, unit labels
│   │   │   │   ├── Ruler.test.tsx             # Tests tick render, unit switch, zoom scale
│   │   │   │   ├── GridOverlay.tsx            # Grid with snap, adjustable spacing
│   │   │   │   ├── GridOverlay.test.tsx       # Tests grid lines, spacing, snap logic
│   │   │   │   ├── GuideLines.tsx             # Draggable guide lines from rulers
│   │   │   │   └── GuideLines.test.tsx        # Tests guide create, drag, delete
│   │   │   └── hooks/
│   │   │       ├── useRulers.ts               # Unit toggle, guide management, snap
│   │   │       └── useRulers.test.ts          # Tests unit toggle, guide CRUD, snap calc
│   │   │
│   │   ├── history/                           # [CORE] Undo/Redo
│   │   │   ├── hooks/
│   │   │   │   ├── useHistory.ts              # Undo/redo stack with debounced snapshots
│   │   │   │   └── useHistory.test.ts         # Tests traversal, max depth, debounce
│   │   │   └── utils/
│   │   │       ├── history-manager.ts         # Command pattern: diffs, coalesce, stack
│   │   │       ├── history-manager.test.ts    # Tests diff record, coalesce, stack limits
│   │   │       ├── state-differ.ts            # JSON-patch diffing for efficient history
│   │   │       └── state-differ.test.ts       # Tests diff gen, patch apply, round-trip
│   │   │
│   │   ├── submission/                        # Print-ready submission
│   │   │   ├── components/
│   │   │   │   ├── PreflightChecks.tsx        # Validates DPI, bleed, overflow, colour, size
│   │   │   │   ├── PreflightChecks.test.tsx   # Tests each rule pass/fail, warning, block
│   │   │   │   ├── SubmissionReview.tsx        # Proof preview with approve/reject per side
│   │   │   │   ├── SubmissionReview.test.tsx   # Tests proof render, approve/reject, state
│   │   │   │   ├── PrintJobForm.tsx           # Quantity, material, finishing options
│   │   │   │   └── PrintJobForm.test.tsx      # Tests validation, selection, submit payload
│   │   │   ├── hooks/
│   │   │   │   ├── useSubmission.ts           # Preflight, PDF trigger, API submission
│   │   │   │   └── useSubmission.test.ts      # Tests preflight, PDF, API, error handling
│   │   │   └── services/
│   │   │       ├── submission-api.ts           # POST print job to designer-backend
│   │   │       ├── submission-api.test.ts      # Tests POST payload, auth, retry, errors
│   │   │       ├── pdf-exporter.ts            # 300 DPI PDF with bleed marks, CMYK
│   │   │       └── pdf-exporter.test.ts       # Tests page count, DPI, bleeds, fonts
│   │   │
│   │   ├── my-designs/                        # My Designs (save/load)
│   │   │   ├── components/
│   │   │   │   ├── DesignGrid.tsx             # Responsive grid of saved designs
│   │   │   │   ├── DesignGrid.test.tsx        # Tests grid layout, responsive, actions
│   │   │   │   ├── DesignCard.tsx             # Card: load, duplicate, rename, delete
│   │   │   │   ├── DesignCard.test.tsx        # Tests card actions, rename input, confirm
│   │   │   │   ├── AutoSaveIndicator.tsx      # Auto-save status display
│   │   │   │   └── AutoSaveIndicator.test.tsx # Tests status transitions, conflict warn
│   │   │   ├── hooks/
│   │   │   │   ├── useMyDesigns.ts            # Design CRUD, auto-save, conflict resolution
│   │   │   │   ├── useMyDesigns.test.ts       # Tests CRUD, optimistic update, conflict
│   │   │   │   ├── useAutoSave.ts             # Periodic & on-change save, dirty tracking
│   │   │   │   └── useAutoSave.test.ts        # Tests periodic trigger, dirty, debounce
│   │   │   └── services/
│   │   │       ├── designs-api.ts             # REST client for design endpoints
│   │   │       └── designs-api.test.ts        # Tests all REST endpoints, pagination (MSW)
│   │   │
│   │   ├── ai/                                # AI artwork & background removal
│   │   │   ├── components/
│   │   │   │   ├── AiArtworkPanel.tsx         # Prompt, style selector, preview, place
│   │   │   │   ├── AiArtworkPanel.test.tsx    # Tests prompt, style, generation, place
│   │   │   │   ├── AiTextArtPanel.tsx         # Text-to-art: word art, calligraphy
│   │   │   │   ├── AiTextArtPanel.test.tsx    # Tests text input, style, generation
│   │   │   │   ├── RemoveBackgroundButton.tsx # One-click BG removal, before/after
│   │   │   │   ├── RemoveBackgroundButton.test.tsx # Tests disabled, loading, toggle
│   │   │   │   ├── AiGenerationProgress.tsx   # Progress/spinner for AI tasks
│   │   │   │   └── AiGenerationProgress.test.tsx # Tests progress %, cancel, timeout
│   │   │   ├── hooks/
│   │   │   │   ├── useAiGeneration.ts         # AI generation trigger, polling, result
│   │   │   │   ├── useAiGeneration.test.ts    # Tests trigger, poll cycle, result, retry
│   │   │   │   ├── useRemoveBackground.ts     # BG removal endpoint, canvas replace
│   │   │   │   └── useRemoveBackground.test.ts # Tests send, swap, failure rollback
│   │   │   └── services/
│   │   │       ├── ai-api.ts                  # API for AI generation & BG removal
│   │   │       └── ai-api.test.ts             # Tests generation POST, polling GET (MSW)
│   │   │
│   │   └── toolbar/                           # Toolbar & panels
│   │       └── components/
│   │           ├── MainToolbar.tsx             # Top toolbar: product, undo, zoom, save
│   │           ├── MainToolbar.test.tsx        # Tests button states, undo disabled, zoom
│   │           ├── LeftToolPanel.tsx           # Left sidebar: tool icons
│   │           ├── LeftToolPanel.test.tsx      # Tests selection, active highlight, panel
│   │           ├── RightPropertiesPanel.tsx    # Right sidebar: context-aware properties
│   │           ├── RightPropertiesPanel.test.tsx # Tests panel switch per type, empty
│   │           ├── BottomStatusBar.tsx         # Status: zoom %, position, size, memory
│   │           ├── BottomStatusBar.test.tsx    # Tests data display, formatting, update
│   │           ├── FloatingToolbar.tsx         # Object-anchored floating toolbar
│   │           └── FloatingToolbar.test.tsx    # Tests positioning, visibility, actions
│   │
│   ├── shared/                                # Shared components & hooks
│   │   ├── components/
│   │   │   ├── ui/
│   │   │   │   ├── Button.tsx                # Base button: primary, secondary, ghost, icon
│   │   │   │   ├── Button.test.tsx            # Tests variants, disabled, loading, click
│   │   │   │   ├── Input.tsx                 # Text input with label, error, slots
│   │   │   │   ├── Input.test.tsx             # Tests change, error display, prefix/suffix
│   │   │   │   ├── Select.tsx                # Custom dropdown with search, multi-select
│   │   │   │   ├── Select.test.tsx            # Tests open/close, search, single/multi
│   │   │   │   ├── Slider.tsx                # Range slider with min/max, step, dual-thumb
│   │   │   │   ├── Slider.test.tsx            # Tests drag, step snap, bounds, dual-thumb
│   │   │   │   ├── Modal.tsx                 # Accessible modal with focus trap, ESC
│   │   │   │   ├── Modal.test.tsx             # Tests focus trap, ESC, overlay click, a11y
│   │   │   │   ├── Popover.tsx               # Floating popover via @floating-ui
│   │   │   │   ├── Popover.test.tsx           # Tests trigger, placement, dismiss
│   │   │   │   ├── Tooltip.tsx               # Accessible tooltip with delay, placement
│   │   │   │   ├── Tooltip.test.tsx           # Tests hover delay, placement, keyboard
│   │   │   │   ├── Tabs.tsx                  # Tab container with lazy render, keyboard
│   │   │   │   ├── Tabs.test.tsx              # Tests switching, lazy render, arrow keys
│   │   │   │   ├── VirtualList.tsx           # Virtualised list for large collections
│   │   │   │   ├── VirtualList.test.tsx       # Tests visible window, scroll, recycling
│   │   │   │   ├── Spinner.tsx               # Loading spinner with size variants
│   │   │   │   ├── Spinner.test.tsx           # Tests size variants, a11y role
│   │   │   │   ├── Toast.tsx                 # Toast notification stack
│   │   │   │   ├── Toast.test.tsx             # Tests show/dismiss, timeout, stack, action
│   │   │   │   ├── ConfirmDialog.tsx         # Confirm/cancel for destructive actions
│   │   │   │   └── ConfirmDialog.test.tsx     # Tests confirm/cancel callbacks, focus
│   │   │   ├── layout/
│   │   │   │   ├── PanelContainer.tsx        # Resizable panel shell for sidebars
│   │   │   │   ├── PanelContainer.test.tsx    # Tests resize drag, min/max, collapse
│   │   │   │   ├── SplitPane.tsx             # Draggable split pane
│   │   │   │   ├── SplitPane.test.tsx         # Tests divider drag, proportion, orient
│   │   │   │   ├── ScrollArea.tsx            # Custom scrollbar area
│   │   │   │   └── ScrollArea.test.tsx        # Tests scroll pos, thumb, overlay mode
│   │   │   └── feedback/
│   │   │       ├── ErrorBoundary.tsx          # Error boundary with retry, fallback
│   │   │       ├── ErrorBoundary.test.tsx      # Tests catch, fallback render, retry
│   │   │       ├── EmptyState.tsx             # Empty state with action prompt
│   │   │       ├── EmptyState.test.tsx         # Tests illustration, action, message
│   │   │       ├── ProgressBar.tsx            # Determinate/indeterminate progress
│   │   │       └── ProgressBar.test.tsx        # Tests percentage, indeterminate, a11y
│   │   └── hooks/
│   │       ├── useDebounce.ts                # Debounced value for search, auto-save
│   │       ├── useDebounce.test.ts            # Tests delay, value update, cancel
│   │       ├── useThrottle.ts                # Throttled callback for scroll/resize
│   │       ├── useThrottle.test.ts            # Tests throttle window, edge, cancel
│   │       ├── useMediaQuery.ts              # Responsive breakpoint detection
│   │       ├── useMediaQuery.test.ts          # Tests breakpoint match, resize, SSR
│   │       ├── useClickOutside.ts            # Click outside detection for popups
│   │       ├── useClickOutside.test.ts        # Tests outside click, inside ignore
│   │       ├── useDragAndDrop.ts             # HTML5 DnD with preview, drop zone
│   │       ├── useDragAndDrop.test.ts         # Tests drag data, drop accept/reject
│   │       ├── useLocalStorage.ts            # Typed localStorage with JSON
│   │       ├── useLocalStorage.test.ts        # Tests read/write, parse, default, SSR
│   │       ├── useEventListener.ts           # Safe event listener with cleanup
│   │       └── useEventListener.test.ts       # Tests attach, cleanup, ref change
│   │
│   ├── store/                                 # Global state (Zustand)
│   │   ├── index.ts                          # Combined store, re-exports slices
│   │   ├── slices/
│   │   │   ├── canvas-slice.ts               # Objects, selection, zoom, pan
│   │   │   ├── canvas-slice.test.ts           # Tests object CRUD, selection, zoom, pan
│   │   │   ├── product-slice.ts              # Product config: sides, dimensions, bleed
│   │   │   ├── product-slice.test.ts          # Tests product load, dimension, bleed
│   │   │   ├── history-slice.ts              # Undo/redo stack state and actions
│   │   │   ├── history-slice.test.ts          # Tests push/undo/redo, overflow, clear
│   │   │   ├── ui-slice.ts                   # Active tool, panels, modals, sidebar
│   │   │   ├── ui-slice.test.ts               # Tests tool switch, panel toggle, modal
│   │   │   ├── design-slice.ts               # Design metadata: id, name, dirty, saved
│   │   │   ├── design-slice.test.ts           # Tests metadata update, dirty, timestamp
│   │   │   ├── vdp-slice.ts                  # CSV data, current record, field mappings
│   │   │   ├── vdp-slice.test.ts              # Tests CSV load, record nav, mappings
│   │   │   ├── roster-slice.ts               # Name & number roster entries
│   │   │   └── roster-slice.test.ts           # Tests roster CRUD, bulk import, validate
│   │   └── middleware/
│   │       ├── logger.ts                     # Redux-devtools action logger
│   │       ├── logger.test.ts                 # Tests log format, filter, dev-only
│   │       ├── persist.ts                    # IndexedDB persistence for crash recovery
│   │       ├── persist.test.ts                # Tests write/read, migration, corruption
│   │       ├── immer.ts                      # Immer middleware for immutable updates
│   │       └── immer.test.ts                  # Tests draft mutation, new ref, nested
│   │
│   ├── apollo/                                # GraphQL client
│   │   └── client.ts                          # Apollo Client configuration (singleton)
│   │
│   ├── graphql/                               # GraphQL layer
│   │   ├── GraphQlProvider.tsx                # ApolloProvider wrapper ("use client")
│   │   ├── GraphQlProvider.test.tsx           # Tests provider renders, client injection
│   │   ├── generated/
│   │   │   └── graphql.ts                    # Auto-generated types & hooks (do not edit)
│   │   └── operation/
│   │       ├── template.graphql              # Template queries (list, detail, categories)
│   │       ├── design.graphql                # Design CRUD mutations, auto-save
│   │       ├── clipart.graphql               # Clipart search, categories
│   │       ├── product.graphql               # Product config, sides, dimensions
│   │       ├── submission.graphql            # Submit print job, preflight status
│   │       ├── ai.graphql                    # AI generation, polling, BG removal
│   │       └── user.graphql                  # User profile, permissions
│   │
│   ├── services/                              # Application services
│   │   ├── api/
│   │   │   ├── http-client.ts                # Axios/ky with auth interceptor, retry (REST fallback)
│   │   │   ├── http-client.test.ts            # Tests base URL, auth header, retry, timeout
│   │   │   ├── api-types.ts                  # Shared API TypeScript types
│   │   │   ├── error-handler.ts              # API error mapping to user messages
│   │   │   └── error-handler.test.ts          # Tests status→message, network, unknown
│   │   ├── auth/
│   │   │   ├── auth-service.ts               # JWT management, refresh, session
│   │   │   ├── auth-service.test.ts           # Tests token store, refresh, expiry, logout
│   │   │   ├── auth-context.tsx              # Auth context with login state, permissions
│   │   │   └── auth-context.test.tsx          # Tests context, login/logout, permissions
│   │   ├── cdn/
│   │   │   ├── upload-service.ts             # Presigned URL upload to S3/CDN
│   │   │   └── upload-service.test.ts         # Tests presigned fetch, upload, retry
│   │   ├── analytics/
│   │   │   ├── analytics-service.ts          # Event tracking, conversion funnel
│   │   │   ├── analytics-service.test.ts      # Tests event dispatch, payload, batching
│   │   │   ├── performance-monitor.ts        # Canvas FPS, memory, render time
│   │   │   └── performance-monitor.test.ts    # Tests FPS calc, threshold, sample rate
│   │   └── websocket/
│   │       ├── ws-client.ts                  # WebSocket for real-time sync (future)
│   │       └── ws-client.test.ts              # Tests connect, reconnect, heartbeat
│   │
│   ├── lib/                                   # Third-party wrappers
│   │   ├── fabric/
│   │   │   ├── fabric-extensions.ts          # Custom types: ArcText, QRCode, Barcode
│   │   │   ├── fabric-extensions.test.ts      # Tests creation, serialization, render
│   │   │   ├── fabric-filters.ts             # Custom image filters
│   │   │   ├── fabric-filters.test.ts         # Tests apply, parameter range, compose
│   │   │   ├── fabric-controls.ts            # Custom controls: icons, hover, touch
│   │   │   └── fabric-controls.test.ts        # Tests render, hit area, cursor style
│   │   ├── pdf/
│   │   │   ├── pdf-generator.ts              # PDF via pdf-lib; fonts, transparency
│   │   │   └── pdf-generator.test.ts          # Tests page create, font embed, output
│   │   └── i18n/
│   │       ├── i18n.ts                       # i18next init, language detection
│   │       └── locales/
│   │           ├── en.json                   # English translations
│   │           └── es.json                   # Spanish translations
│   │
│   ├── types/                                 # Global TypeScript types
│   │   ├── canvas.types.ts                   # CanvasObject, CanvasState, ObjectType
│   │   ├── product.types.ts                  # Product, ProductSide, Dimensions, Bleed
│   │   ├── design.types.ts                   # Design, DesignMeta, DesignVersion
│   │   ├── template.types.ts                 # Template, TemplateLayer, LockConfig
│   │   ├── vdp.types.ts                      # VdpRecord, FieldMapping, PlaceholderDef
│   │   ├── user.types.ts                     # User, Session, Permissions
│   │   └── api.types.ts                      # ApiResponse<T>, PaginatedResult<T>
│   │
│   ├── styles/                                # Global styles
│   │   ├── globals.css                       # CSS reset, custom properties, typography
│   │   ├── tokens.css                        # Design tokens: colours, spacing, shadows
│   │   ├── animations.css                    # Keyframes: fade, slide, pulse, skeleton
│   │   └── tailwind.css                      # Tailwind directives
│   │
│   ├── config/                                # App configuration
│   │   ├── env.ts                            # Typed env vars with zod validation
│   │   ├── env.test.ts                        # Tests validation, missing var, coercion
│   │   ├── feature-flags.ts                  # Feature flag definitions & overrides
│   │   ├── feature-flags.test.ts              # Tests defaults, override, reset, nested
│   │   ├── print-presets.ts                  # Print product presets: sizes, DPI, bleed
│   │   └── shortcuts.ts                      # Keyboard shortcut definitions
│   │
│   ├── __tests__/                             # Shared test infrastructure (NOT unit tests)
│   │   ├── setup.ts                          # Vitest global setup: DOM, canvas mock, MSW
│   │   ├── test-utils.tsx                    # Custom render with providers, factories
│   │   ├── mocks/
│   │   │   ├── handlers.ts                   # MSW request handlers for all APIs
│   │   │   ├── server.ts                     # MSW server instance
│   │   │   ├── canvas-mock.ts                # Fabric.js canvas mock for unit tests
│   │   │   └── fixtures/
│   │   │       ├── products.ts               # Sample product configs for tests
│   │   │       ├── designs.ts                # Sample design JSON for tests
│   │   │       └── templates.ts              # Sample template data for tests
│   │   └── e2e/                              # End-to-end tests (Playwright)
│   │       ├── designer-flow.spec.ts         # Full designer: load, add text, save, submit
│   │       ├── vdp-flow.spec.ts              # VDP: upload CSV, map, preview, submit
│   │       └── template-flow.spec.ts         # Template: load, edit unlocked, save
│   │
│   ├── main.tsx                              # Entry: ReactDOM.createRoot, strict mode
│   └── vite-env.d.ts                         # Vite client types
│
├── package.json                               # Dependencies, scripts
├── codegen.ts                                 # GraphQL Code Generator config
├── tsconfig.json                              # TS strict, path aliases (@/), ESNext
├── tsconfig.app.json                          # App-specific TS config
├── tsconfig.node.json                         # Node scripts TS config
├── vite.config.ts                             # Vite: React plugin, aliases, proxy, chunks
├── vitest.config.ts                           # Vitest: jsdom, coverage thresholds, setup
├── playwright.config.ts                       # Playwright: browsers, base URL, screenshots
├── tailwind.config.ts                         # Tailwind: custom theme, designer utilities
├── postcss.config.js                          # PostCSS: Tailwind, autoprefixer
├── eslint.config.js                           # ESLint flat: ts-eslint, react, imports, a11y
├── .prettierrc                                # Prettier: single quotes, trailing commas
├── .env.example                               # Environment variable template
├── .env.development                           # Dev defaults (local API, debug flags)
├── .env.production                            # Production (CDN URLs, analytics keys)
├── .gitignore                                 # Ignore: node_modules, dist, .env, coverage
├── Dockerfile                                 # Multi-stage: install → build → nginx
├── docker-compose.yml                         # Local dev: app + mock API + Redis
├── nginx.conf                                 # Production nginx: gzip, cache, SPA, security
├── README.md                                  # Project overview, setup, architecture
├── ARCHITECTURE.md                            # ADRs, data flow, module boundaries
└── CONTRIBUTING.md                            # Coding standards, PR process, branches
```

---

## Architecture Principles

### 1. Feature-Sliced Design

Each feature module is self-contained with its own components, hooks, services, utils, types, and **tests**. This enables independent development by different teams and clean code-splitting at feature boundaries.

### 2. Core vs Feature Separation

Three modules are tagged as **CORE** because every other feature depends on them: `canvas/`, `multi-side/`, and `history/`. Feature modules are isolated and can be lazy-loaded independently.

### 3. Co-located Testing (Hybrid Approach)

| What | Where | Why |
|---|---|---|
| Unit tests | Sibling `.test.ts` next to source | Impossible to miss; moves/deletes with source; PR shows both |
| Test setup | `__tests__/setup.ts` | Global Vitest config, DOM mocks, MSW server start |
| Custom render | `__tests__/test-utils.tsx` | Wraps component in all providers for testing |
| API mocks | `__tests__/mocks/handlers.ts` | MSW handlers shared across all service tests |
| Test fixtures | `__tests__/mocks/fixtures/` | Shared sample data (products, designs, templates) |
| E2E tests | `__tests__/e2e/*.spec.ts` | Cross-feature flows don't belong to one module |

### 4. State Architecture

| Concern | Solution |
|---|---|
| Canvas object state | Fabric.js internal ↔ synced to Zustand via `useCanvasSync` |
| Application UI state | Zustand slices (ui-slice, design-slice) |
| Server/async state | Apollo Client (templates, clipart, designs, submissions GraphQL API) |
| Ephemeral UI state | React local state (useState) |
| Crash recovery | IndexedDB persistence middleware on Zustand |

### 5. Data Flow

```
User Interaction
       ↓
Canvas Event (Fabric.js)  →  useCanvasEvents  →  Zustand Store (canvas-slice)
       ↓                                              ↓
Object Controls                                  History Manager (diff snapshot)
       ↓                                              ↓
Properties Panel  ←  RightPropertiesPanel  ←  Store subscription
       ↓
Auto-Save (debounced)  →  designs-api  →  designer-backend
```

### 6. Print Pipeline

```
Canvas State (Fabric.js JSON)
       ↓
Preflight Checks (DPI, bleed, overflow)
       ↓
Offscreen Render (300 DPI per side)
       ↓
PDF Generation (pdf-lib, CMYK conversion)
       ↓
Submission API → designer-backend → Print Queue
```

---

## Key Dependencies

| Package | Purpose |
|---|---|
| `fabric` (v6) | Canvas rendering engine |
| `zustand` | Lightweight state management |
| `@apollo/client` | GraphQL client with caching & React hooks |
| `graphql` | GraphQL query language |
| `@graphql-codegen/cli` | Auto-generate types & hooks from `.graphql` |
| `react-router` (v7) | Client-side routing |
| `@floating-ui/react` | Popover / tooltip positioning |
| `react-image-crop` | Image cropping |
| `pdf-lib` | Client-side PDF generation |
| `papaparse` | CSV parsing for VDP |
| `i18next` | Internationalisation |
| `zod` | Runtime schema validation |
| `immer` | Immutable state updates |
| `msw` | API mocking for tests |

---

## Scripts

```bash
# Development
npm run dev              # Start Vite dev server with HMR
npm run dev:mock         # Dev server with MSW mock API
npm run codegen:watch    # Auto-regenerate GraphQL types on .graphql change

# GraphQL
npm run codegen          # Generate types & hooks from GraphQL schema

# Build
npm run build            # Production build with type checking
npm run preview          # Preview production build locally

# Quality
npm run lint             # ESLint check
npm run lint:fix         # ESLint auto-fix
npm run typecheck        # TypeScript compiler check (no emit)
npm run format           # Prettier format

# Testing
npm run test             # Vitest watch mode
npm run test:ci          # Vitest single run with coverage
npm run test:e2e         # Playwright end-to-end tests

# Docker
docker compose up        # Full local environment
docker compose up --build # Rebuild and start
```

---

## Environment Variables

```env
# .env.example
VITE_API_BASE_URL=https://api.designer.example.com
VITE_GRAPHQL_URL=https://api.designer.example.com/graphql
VITE_CDN_URL=https://cdn.designer.example.com
VITE_WS_URL=wss://ws.designer.example.com
VITE_AI_ENABLED=true
VITE_MAX_UPLOAD_SIZE_MB=50
VITE_ANALYTICS_KEY=
VITE_SENTRY_DSN=
VITE_FEATURE_VDP=true
VITE_FEATURE_NAME_NUMBER=true
VITE_FEATURE_AI_ARTWORK=true
VITE_FEATURE_REMOVE_BG=true
```
