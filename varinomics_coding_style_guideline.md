# Varinomics Coding Style Guideline

This document defines the coding style standard for Varinomics software products.
It is the canonical baseline for new code, code reviews, and product-specific style
profiles.

## 1. Scope

- This guideline applies to production code, tests, tools, build scripts, examples,
  and documentation snippets unless a section says otherwise.
- Product-specific guidance may add stricter rules, but it must not contradict this
  document.
- Language-specific sections refine the general rules. When a general rule and a
  language-specific rule both apply, the more specific rule wins.

## 2. Requirement Language

- `must` means mandatory.
- `should` means the default expectation. Deviations require a concrete reason.
- `may` means allowed but optional.
- When an example and a rule disagree, the rule wins.

## 3. Universal Rules

### 3.1 Code quality goals

- Code must optimize for correctness, leverage, maintainability, and scanability.
- Readability matters, but it is not the only axis. Do not force verbosity or
  repetition solely to make each line locally obvious.
- Prefer concise abstractions, helpers, templates, and reusable mechanisms when
  they remove real duplication or encode an important invariant.
- Dense or advanced code is acceptable when it captures the right abstraction and
  is documented well enough for future maintenance.
- Names must describe intent, not incidental implementation details.
- Abbreviations should be used only when they are well-established in the domain.

### 3.2 Consistency and local coherence

- Match the established conventions of the surrounding module unless this guideline
  explicitly requires a change.
- Preserve intentional alignment and layout when it improves scanability.
- Style-only refactors are allowed when they improve consistency, enforce this
  guideline, or materially improve visual scanability.
- When practical, keep pure beautification passes separate from behavioral changes.

### 3.3 Product evolution and obsolete code

- Do not keep code paths solely to preserve obsolete behavior unless a product
  explicitly requires that compatibility.
- If an old path exists only because of previous behavior and is no longer needed,
  remove it instead of carrying parallel implementations.
- Prefer changing the current code to the desired behavior over layering
  compatibility scaffolding on top of it.
- Contracts and APIs may be changed freely when that is the right design decision,
  unless compatibility requirements are stated explicitly.
- If changing a contract breaks owned consumers, fix those consumers next rather
  than weakening the contract change to avoid breakage.
- Compiler-visible breakage is often useful because it identifies every affected
  consumer immediately.
- Do not add compatibility shims, adapter overloads, legacy parameters, or dual
  code paths merely to avoid changing callers unless compatibility support is an
  explicit requirement.

### 3.4 Structure and control flow

- Prefer explicit control flow over compact but opaque constructs.
- Prefer early exits for invalid or completed cases when they reduce nesting.
- Each block should have one clear responsibility.
- Remove superseded or duplicate code paths instead of keeping parallel ways to do
  the same thing.

### 3.5 Architecture and integration bias

- Prefer fixing defects at the source of ownership when you control that source,
  rather than adding local workarounds downstream.
- Avoid polling as a default design choice. Prefer event-driven or signal-driven
  designs. Use polling only when there is no viable alternative.
- Avoid hysteresis-based behavior unless simpler and more deterministic approaches
  have been exhausted.
- Strongly discourage introducing new environment-variable dependencies for normal
  product behavior, configuration, discovery, or control flow when a clear
  alternative exists.
- Prefer explicit configuration through code, arguments, config files, or other
  local and inspectable mechanisms.
- Use environment variables only when they are genuinely the right integration
  boundary or when no practical alternative exists.

### 3.6 Comments and documentation

- Comments must explain intent, constraints, or non-obvious behavior.
- Comments must not restate code that is already obvious from the implementation.
- Public interfaces and non-trivial internals should have concise documentation.
- Examples in docs, tests, comments, and documentation snippets must use
  realistic, context-appropriate values and names.
- Do not use `42` as a generic example value, placeholder, filler constant, or
  joke. It is tired, unserious, and unacceptable in product-owned material
  unless the subject matter literally requires that value.
- Do not use meme values, wink/nod references, or joke placeholders when a real
  domain value would communicate the example better.
- Do not describe product-owned code or behavior as `v1`, `v2`, `legacy`,
  `old`, or similar in comments, docs, or user-facing messages unless that
  label names a real external contract, protocol version, or release boundary.

### 3.7 Tooling and enforcement

- Use automated formatting and linting where available, but tool output does not
  override this guideline.
- If a formatter cannot express a rule in this document, follow the document.
- Do not leave pure LF/CRLF line-ending churn in a change set.
- If a file differs only by line-ending normalization and no real content has
  changed, revert that noise immediately.
- Do not present line-ending-only diffs as meaningful work.

## 4. Language-Specific Profiles

- Each language used in a Varinomics product should have a profile section or a
  product-specific addendum.
- A profile may define naming, layout, API, and tooling rules for that language.
- A product may reference this document plus a small local addendum instead of
  maintaining a separate full style guide.

## 5. C++ Profile

This section defines the default C++ style for Varinomics products.

### 5.1 Naming

- Identifiers must use `snake_case` unless a rule below says otherwise.
- Identifiers must not start or end with `_`.
- File names must be lowercase `snake_case`.
- Names copied directly from an external API, protocol, file format, wire
  contract, callback signature, or inherited interface may keep the external
  spelling when mirroring that foreign name improves clarity or is required by
  the interface.
- Product-owned local variables, parameters, and members should use this
  guideline's naming rules even inside adapter code. Do not propagate a foreign
  naming style to adjacent local declarations merely because nearby library code
  uses it.

Good:

```cpp
int samples_per_pixel;
void set_viewport_size(QSize size);
```

Bad:

```cpp
int SamplesPerPixel;
int _samples;
```

### 5.2 Type names

- Passive data structs must use lowercase names with a `_t` suffix and be declared
  as `struct`.
- A type may use `_t` only when it is a plain data carrier with no custom
  constructor, destructor, inheritance, virtual behavior, or ownership-heavy
  members such as `std::vector`, `QString`, or `std::function`.
- Behavioral or owning types must use `Capitalized_words`.
- Established product or library prefixes keep their documented capitalization.

Good:

```cpp
struct rect_vertex_t {
    glm::vec4 color;
    glm::vec4 rect;
};

class Plot_renderer
{
    // ...
};
```

### 5.3 Members, statics, and constants

- Instance members must use the `m_` prefix.
- Static data members must use the `s_` prefix.
- Namespace-scope compile-time constants must use the `k_` prefix, be
  `constexpr`, and use `snake_case`.

```cpp
class Plot_renderer
{
public:
    void set_viewport_size(const QSize& size) { m_viewport_size = size; }

private:
    static inline int s_msaa_samples = 8;
    QSize m_viewport_size{};
};

constexpr float k_grid_line_half_px = 0.7f;
```

### 5.4 Namespaces

- Do not use `using namespace` at file scope or class scope.
- A `using namespace` directive may be used inside a function body when it removes
  heavy repetition and the scope remains local.
- Namespace aliases and type aliases at file scope, namespace scope, or class
  scope are allowed when they materially reduce repetition and improve
  scanability.
- Prefer a short local alias for verbose namespaces such as `std::filesystem`
  rather than repeating the fully qualified name throughout a file.
- Prefer explicit qualifiers or a short local namespace alias for long names.

```cpp
namespace algo = vnm::plot::algo;
namespace fs = std::filesystem;
```

### 5.5 Files and includes

- In a `.cpp` file, the first include must be the corresponding header.
- Project headers come next.
- Third-party and system headers come after project headers.
- STL headers come last.
- Do not rely on transitive includes.
- Use `#include "..."` for project headers and `#include <...>` for external
  headers.

```cpp
#include "vnm_plot_renderer.h"
#include "vnm_plot.h"
#include "vnm_plot_layout_calculator.h"
#include <glatter/glatter.h>
#include <gtc/matrix_transform.hpp>
#include <QOpenGLShaderProgram>
#include <vector>
```

### 5.6 Braces and layout

- Function, class, and struct opening braces must go on the next line.
- Lambda and control-statement opening braces must go on the same line when the
  condition fits on one line.
- If an `if`, `for`, or `while` condition wraps across lines, put the opening
  brace on the next line.
- `else`, `catch`, and `while` for a `do` loop must start on their own line.
- `else if` must be formatted as `else` followed by `if` on the next line.
- Do not write `else if` on one physical line.
- Do not write `} else`, `} catch`, or `} while` on one physical line.
- A short `if` body may stay on one line when it is isolated and clearly easier to
  scan that way.
- Do not compress a repetitive `if`/`else` ladder into one-line bodies when the
  result reads like a run-on chain. In that case, prefer full multi-line blocks
  even if each body contains only one short statement.

Good:

```cpp
void do_render()
{
    if (ready) {
        render();
    }
    else
    if (recover()) {
        render();
    }
}

if (blah1 && blah2 &&
    blah3 && blah4)
{
    render();
}
```

### 5.7 Short one-line forms

- Trivially short functions may stay on one line.
- A short consecutive block of trivial pure helper functions should remain on one
  line when each helper fits cleanly and the group reads as one scannable block.
- Intentional no-op functions may use `{}` on one line.
- Consecutive short helpers may be aligned on one line when that improves scanning.
- Preserve an aligned block of consecutive trivial one-line helpers when the
  alignment improves comparison.
- Do not expand a coherent one-line helper block into multi-line bodies unless
  another rule clearly requires it.
- Single-line control statements may be used only for a tight, consecutive group of
  short statements. Do not use the one-line form for an isolated guard.

```cpp
double Port_assembly::target_vctrl_voltage() { return m_target_vctrl_voltage; }
void install_unhandled_exception_handler() {}

static constexpr int frameWidth()        noexcept { return 1; }
static constexpr int horizontalPadding() noexcept { return 6; }
static constexpr int iconSpacing()       noexcept { return 4; }

if (value < vmin) { vmin = value; }
if (value > vmax) { vmax = value; }
```

### 5.8 Wrapped parameter lists, template lists, and initializer lists

- When a function declaration, definition, or call wraps, choose a deliberate
  wrapped form rather than drifting into an accidental partial wrap.
- Two common wrapped forms are allowed: a compact line split or an argument
  split after `(`.
- A compact line split keeps simple leading arguments on the first line and
  moves a long or structured trailing argument or suffix to a following line.
- An argument split breaks immediately after `(` and places one argument per
  line or in small scannable groups.
- Do not use a hanging-indent style where later parameters are lined up under the
  first parameter or under text from the first line.
- Wrapped parameters may stay compact on one continuation line when the types and
  names remain easy to read.
- In repetitive blocks of similar calls, a compact line split may be preferable
  when it keeps the block tight without harming scanability.
- If the parameter types are visually dense or hard to scan, put one parameter on
  each following line.
- If a wrapped parameter list contains pointer-heavy, reference-heavy, or
  otherwise visually dense parameter types, prefer one parameter per line over a
  compressed continuation line even when the compressed form would fit.
- If one wrapped argument is itself visually dense or internally structured, such
  as a flag bitmask, stream chain, boolean chain, or nested conditional
  expression, do not compress that argument onto the same continuation line as
  preceding simple arguments.
- When one argument is an operator chain, preserve that argument's own visual
  structure even if the compressed form would still fit under the line-length
  limit.
- Vertical alignment of types and names across wrapped parameter lines is allowed
  when it materially improves scanability.
- For dense wrapped signatures, aligned type and name columns are preferred when
  the alignment makes the parameter list read as a clear table.
- Do not recompress a wrapped parameter list into a dense continuation line
  merely because the result fits under the line-length limit.
- The closing `)` may either align with the start of the function statement or
  stay on its own final parameter line. Use one of those two forms consistently
  within the local block.
- Apply the same rule to wrapped template parameter lists and template argument
  lists.
- If a template list wraps, the preferred form is to break immediately after
  `<`.
- Do not use a hanging-indent style where later template parameters or arguments
  are lined up under the first argument or under text from the first line.
- Wrapped template lists may stay compact on one continuation line when the
  grouped arguments remain easy to scan.
- If the template arguments are visually dense or semantically separate, prefer
  one argument per line or small scannable groups per line.
- The closing `>` may stay on the final argument line or on its own line. Use
  one of those two forms consistently within the local block.
- Constructor initializer lists must start after a blank line.
- Put `:` on its own line, aligned with the function declaration or definition.
- Put each initializer on its own following line, indented one level.
- Place commas at the ends of initializer lines. Do not use a leading-comma
  layout.

```cpp
void update_and_write(
    const std::vector<double>& values,
    const std::vector<bool>& filter = {});

static std::filesystem::path resolve_font_path(
    const font_source_t& source, const std::filesystem::path& font_root, Pdf_font font);

static std::filesystem::path resolve_font_path(
    const font_source_t& source,
    const std::filesystem::path& font_root,
    Pdf_font font);

static std::filesystem::path resolve_font_path(
    const font_source_t&         source,
    const std::filesystem::path& font_root,
    Pdf_font                     font);

static std::filesystem::path resolve_font_path(
    const font_source_t& source,
    const std::filesystem::path& font_root,
    Pdf_font font
);

void update_geometry_node(
    QQuickWindow*               window,
    QSGNode*                    parent,
    QSGGeometryNode*&           node,
    const std::vector<QPointF>& points,
    QSGGeometry::DrawingMode    mode,
    const QColor&               color);

const auto glyph_runs = line.glyphRuns(
    0,
    line.textLength(),
    QTextLayout::RetrieveGlyphIndexes   |
    QTextLayout::RetrieveGlyphPositions |
    QTextLayout::RetrieveStringIndexes);

Example(
    QObject* parent,
    Helper helper)
:
    QObject(parent),
    m_helper(std::move(helper))
{}

using block_t = std::variant<
    paragraph_block_t,
    heading_block_t,
    list_block_t,
    code_block_t,
    table_block_t,
    page_break_block_t>;

using some_type = some_template_type<
    group1_param1_t, group1_param2_t, group1_param3_t,
    group2_param1_t, group2_param2_t, group2_param3_t>;

using some_type = some_template_type<
    group1_param1_t, group1_param2_t, group1_param3_t,
    group2_param1_t, group2_param2_t, group2_param3_t
>;
```

Not allowed:

```cpp
static std::filesystem::path resolve_font_path(const font_source_t& source,
                                               const std::filesystem::path& font_root,
                                               Pdf_font font);

const auto glyph_runs = line.glyphRuns(0, line.textLength(),
    QTextLayout::RetrieveGlyphIndexes | QTextLayout::RetrieveGlyphPositions | QTextLayout::RetrieveStringIndexes);

void update_geometry_node(QQuickWindow* window, QSGNode* parent, QSGGeometryNode*& node,
    const std::vector<QPointF>& points, QSGGeometry::DrawingMode mode, const QColor& color);

using block_t = std::variant<paragraph_block_t,
                             heading_block_t,
                             list_block_t,
                             code_block_t,
                             table_block_t,
                             page_break_block_t>;
```

### 5.9 Control blocks and switches

- Every `if`, `else`, `for`, `while`, `do`, and `catch` block must use braces.
- `case` labels must be indented one level under `switch`.
- This indentation rule also applies to compact one-line `case ...: return ...;`
  and `case ...: break;` forms.
- Do not place `case` labels flush with the `switch` body indentation.
- For short pure dispatch switches, compact one-line `case ...: return ...;`
  forms are preferred when the arms fit cleanly and remain easy to scan.
- When every arm is a short direct `return` or `break`, keep the switch in that
  compact form even if an auto-formatter would expand each arm into a multi-line
  block.
- Consecutive compact `case` lines may be vertically aligned when that improves
  scanability.
- When consecutive compact `case ...: return ...;` lines are aligned, pad shorter
  labels so the first token after `:` starts in the same column.
- A short fallthrough label may remain on its own line immediately above a compact
  `default: return ...;` when that is the clearest spelling.
- Every `switch` must include a `default`.

```cpp
if (count == 0) {
    return;
}

switch (style) {
    case Display_style::DOTS: return dots;
    case Display_style::AREA: return area;
    case Display_style::LINE: return line;
    default:                  return line;
}

switch (level) {
    case 1:  return body_size * 0.65;
    case 2:  return body_size * 0.55;
    case 3:  return body_size * 0.45;
    case 4:  return body_size * 0.40;
    default: return body_size * 0.35;
}

switch (style) {
    case Inline_style::BOLD:        return Pdf_font::BOLD;
    case Inline_style::ITALIC:      return Pdf_font::ITALIC;
    case Inline_style::BOLD_ITALIC: return Pdf_font::BOLD_ITALIC;
    case Inline_style::CODE:        return Pdf_font::MONO;
    case Inline_style::NORMAL:
    default:                        return Pdf_font::REGULAR;
}

switch (eff) {
    case FontQuality::QualityDefault:         return QFont::PreferDefault;
    case FontQuality::QualityNonAntialiased:  return QFont::NoAntialias;
    case FontQuality::QualityAntialiased:
    case FontQuality::QualityLcdOptimized:    return QFont::PreferAntialias;
    default:                                  return QFont::PreferDefault;
}
```

Not allowed:

```cpp
switch (style) {
case Display_style::DOTS: return dots;
case Display_style::AREA: return area;
default: return line;
}

switch (eff) {
    case FontQuality::QualityDefault:
        return QFont::PreferDefault;
    case FontQuality::QualityNonAntialiased:
        return QFont::NoAntialias;
    case FontQuality::QualityAntialiased:
    case FontQuality::QualityLcdOptimized:
        return QFont::PreferAntialias;
    default:
        return QFont::PreferDefault;
}

if (ready) {
    render();
}
else {
    recover();
}

if (ready) {
    render();
}
else
if (fallback) {
    recover();
}

if ((c & 0x80) == 0x00) {
    len = 1;
}
else
if ((c & 0xE0) == 0xC0) {
    len = 2;
}
else
if ((c & 0xF0) == 0xE0) {
    len = 3;
}
else
if ((c & 0xF8) == 0xF0) {
    len = 4;
}
```

### 5.10 Spacing and formatting

- Readability, not line packing, governs line length.
- Treat roughly 100 columns as a warning signal: consider whether splitting the
  line would improve scanability, especially when it is much longer than its
  neighbors.
- Treat 120 columns as a strong limit, not as a formatting goal.
- Prefer shorter lines when they expose structure more clearly, even if a denser
  form would still fit under 120 columns.
- Once a line grows past roughly 80 columns, consider whether folding it would
  improve scanability.
- Past roughly 100 columns, a line should usually remain that long only when the
  longer form is clearly the best-scanning layout.
- Exceed 120 columns only when the result is clearly easier to read than the
  wrapped form.
- Use one space after commas and around binary operators.
- Bind `*` and `&` to the type: `Type* name`, `Type& name`.
- Do not leave trailing whitespace.
- Every file must end with a newline.
- Intentional vertical alignment is allowed and encouraged for short groups of
  related lines when it makes comparison faster to read.
- Preserve existing intentional alignment in short, semantically related, stable
  blocks when it improves comparison or makes the block faster to scan.
- Do not de-align a short coherent block merely to normalize whitespace if the
  alignment does not conflict with another rule and still reads cleanly.
- For compact reset, initialization, or mapping blocks with repeated `=`
  assignments, aligned `=` is preferred when the variables form a clear related
  group and the alignment materially improves scanability.
- The same rule applies to short aligned declaration groups. When several nearby
  declarations share the same type-and-initializer shape, preserving aligned
  names and `=` columns is preferred when it makes the block easier to compare.
- Use common sense when aligning declaration groups. Prefer alignment when the
  lines are similarly shaped, the extra padding is modest, and the block reads
  as one coherent table.
- Do not force a short or simple declaration to align under a much longer or
  denser neighboring declaration when that would add large empty gaps or make
  the simpler line look visually subordinate to the longer one.
- Alignment should help comparison within a coherent group, not make unrelated
  declarations look artificially uniform.
- Do not apply alignment mechanically across large, unstable, or weakly related
  blocks.
- For wrapped expression chains such as stream insertion or other repeated
  operators, continue on the next line with a normal continuation indent by
  default.
- Do not recompress a wrapped call or expression merely because the recompressed
  form still fits under 120 columns.
- When a wrapped braced initializer contains a nested constructor or call as one
  of its elements, keep that nested expression visually structured instead of
  collapsing it into a dense continuation line.
- The same rule applies when a nested constructor or call is passed as one
  argument to an outer call: preserve the inner expression's structure when that
  makes the element boundaries or argument roles easier to scan.
- Inside compact brace-init blocks, treat each structured element as a visual
  unit. Do not flatten a previously clear nested `QRectF(...)`, `QPointF(...)`,
  or similar constructor just because the flattened form still fits.
- For wrapped ternary expressions, make the `?` and `:` structure visually clear.
  If one arm is substantially longer or denser than the condition, put the
  branches on their own continuation lines instead of letting a long arm dangle
  off the condition line.
- When a wrapped ternary arm is itself a structured call or constructor, format
  that arm as a subordinate block under the `?` or `:` so the conditional shape
  stays primary.
- Do not flatten a ternary back onto one line merely because it now fits under
  the line-length limit if the wrapped form makes the condition and branches
  easier to scan.
- The base expression may stand on its own line only when the chain is highly
  repetitive and the operands form a uniform vertical list.
- Do not use deep hanging indentation that lines continuation operators up under
  a token from the previous line.

```cpp
void ensure_tex(GLuint& tex, int& cur_size);
const char* message;

const auto code   = ws->closeCode();
const auto reason = ws->closeReason();

width  = function(something_1, something_else_2);
height = function(something_2, something_else_1);

device        = nullptr;
painter       = nullptr;
device_owned  = false;
painter_owned = false;
capture_only  = false;

const qreal dpr         = std::max<qreal>(1.0, window->effectiveDevicePixelRatio());
const QRectF normalized = rect.normalized();
const qreal left        = std::floor(normalized.left()  * dpr) / dpr;
const qreal top         = std::floor(normalized.top()   * dpr) / dpr;
const qreal right       = std::ceil(normalized.right()  * dpr) / dpr;
const qreal bottom      = std::ceil(normalized.bottom() * dpr) / dpr;

const qreal dpr     = std::max<qreal>(1.0, window->effectiveDevicePixelRatio());
const qreal pixel   = physical_pixel_size(window);
const int width_px  = std::max(1, static_cast<int>(std::round(whole_rect.width()  * dpr)));
const int height_px = std::max(1, static_cast<int>(std::round(whole_rect.height() * dpr)));

image_stream
    << "<< /Type /XObject /Subtype /Image /Width " << image.width_px()
    << " /Height " << image.height_px() << " /ColorSpace "
    << (image.color_components() == 1 ? "/DeviceGray" : "/DeviceRGB")
    << " /BitsPerComponent 8";

const auto glyph_runs = line.glyphRuns(
    0,
    line.textLength(),
    QTextLayout::RetrieveGlyphIndexes   |
    QTextLayout::RetrieveGlyphPositions |
    QTextLayout::RetrieveStringIndexes);

rects.push_back({
    QRectF(
        snapped.left() + static_cast<qreal>(pixel) * physical_pixel,
        snapped.top(),
        physical_pixel,
        physical_pixel),
    color,
});

return wid
    ? PRectangle(
          window(wid)->x(),                         window(wid)->y(),
          window(wid)->x() + window(wid)->width(), window(wid)->y() + window(wid)->height())
    : PRectangle(0, 0, 1000, 1000);

const qreal dpr = item.window()
    ? std::max<qreal>(1.0, item.window()->effectiveDevicePixelRatio())
    : 1.0;
```

Not allowed:

```cpp
const int height_px = std::max(1, static_cast<int>(std::round(whole_rect.height() * dpr)));
int idx             = 0;
```

### 5.11 Macros

- Keep multi-line macro continuations visually tidy.
- Either place each trailing `\` one space after the final token or align the `\`
  characters to the same column within that macro.
- Prefer the minimal-spacing form by default: place the trailing `\` one space
  after the final token unless wider alignment within that macro clearly
  improves scanability.
- Do not pad a tidy macro out with large horizontal whitespace merely to push
  the trailing `\` toward a far-right column.
- Use one style per macro.

```cpp
#define LOG_AND_RETURN(value)  \
    do {                       \
        LOG_INFO(value);       \
        return (value);        \
    }                          \
    while (false)
```

### 5.12 Functions and APIs

- Function names must use `snake_case`.
- Read-only member functions must be marked `const`.
- Variables must be declared in the narrowest practical scope.
- Prefer `auto` when the type is obvious from the initializer and the declaration
  becomes easier to read.
- Prefer free helpers in an anonymous namespace for translation-unit-local symbols.

```cpp
namespace {
constexpr double k_eps = 1e-6;
}
```

### 5.13 Structs, classes, and ownership boundaries

- Use plain data structs for passive aggregates and result bundles.
- Use classes for behavioral or owning types.
- Large classes with heavy private dependencies should use PIMPL.
- Within a class, order sections as `public`, `protected`, then `private`.
- Access labels must be flush with the class body's baseline indentation. Do not
  indent `public:`, `protected:`, or `private:` one extra level under the class.
- Apply the same layout rule to Qt section labels such as `signals:`,
  `public slots:`, `protected slots:`, and `private slots:`.
- Group members by subsystem or responsibility rather than by declaration kind alone.

```cpp
struct frame_layout_result_t {
    double usable_width;
    double usable_height;
};

class Plot_renderer
{
public:
    explicit Plot_renderer(const Plot* owner);
    ~Plot_renderer();

private:
    struct Impl;
    std::unique_ptr<Impl> d;
};
```

### 5.14 Constants, literals, and enums

- Compile-time constants must use `constexpr` whenever possible.
- Numeric literals should use explicit suffixes where type matters.
- Enum type names follow class-style capitalization.
- Enum values must use `SCREAMING_SNAKE_CASE`.

```cpp
constexpr int k_msaa_samples = 8;
constexpr float k_pixel_snap = 0.5f;

enum class Display_style
{
    DOTS,
    AREA,
    LINE,
};
```

### 5.15 Error handling and logging

- Use assertions for invariants and programmer errors.
- Use structured logging for non-fatal diagnostics.
- Choose log levels deliberately and consistently.
- `error` means a real failure state. It is potentially fatal and may require
  user action.
- `warning` means something is wrong, but the program can continue, possibly in
  a degraded or risky state.
- `info` means useful user-facing events or status updates.
- `external` means user-facing output originating from third-party tools or
  processes.
- `debug` means developer-facing diagnostics and troubleshooting detail.
- `error`, `warning`, `info`, and `external` are user-facing levels and their
  messages must be clear, well-formed, and non-cryptic.
- Avoid noisy logging in hot paths.
- Exception-based and status-return-based error handling are both acceptable; each
  module should choose deliberately and remain internally consistent.

### 5.16 Qt and OpenGL profile

- Resource creation and cleanup must follow the relevant Qt thread or render-lifecycle
  rules.
- Validate ranges and prerequisites before issuing draw calls.
- Keep OpenGL state changes local and leave the state machine in a known condition.
- Prefer bounded draws and batching when the rendering path allows it.

### 5.17 Comments and API documentation

- Comments should state what matters and why it matters.
- Public APIs and non-trivial internals should use concise Doxygen-style comments
  when that materially improves discoverability.

```cpp
/**
 * @brief Flush batched rectangles to GPU and draw them in one call.
 * @param pmv Combined projection-model-view matrix.
 */
void flush_rects_buffer(const glm::mat4& pmv);
```

### 5.18 Interfacing with C and C-style APIs

- API conventions win when a C or system API has a documented local style.
- C-style casts are acceptable where they are idiomatic, direct, and semantically
  adequate for the target API or conversion.
- Do not mechanically replace C-style casts with C++ casts just to satisfy
  convention.
- Use a C++ cast when the more specific cast category materially improves safety,
  communicates intent better, or avoids ambiguity.
- Do not churn cast style, sentinel values, or null style without a concrete gain in
  correctness, safety, clarity, or consistency within the module.

## 6. C and C-Style API Profile

This section defines the default style for designing C-callable APIs and any
C-style API surface, including ABI-stable headers that ship to external
consumers and shared-object exports. It applies whether the implementation
language is C or C++.

The rules below are about API *design*. Section 5.18 covers consuming C-style
APIs from C++ code; the two are independent.

These rules exist because a C-style signature is the only documentation
available to a caller who reads it cold. Direction, role, and failure model
must be visible in the prototype itself, without requiring the reader to
trace back to a producing call or a separate doc block.

### 6.1 Parameter ordering

- Inputs must come first. Outputs must come last.
- When a function takes an indexable container and an index that selects
  within it, the container comes first and the index comes second.
- Do not invert the container/index order. The container-first form matches
  every well-known C and C++ accessor convention (`arr[i]`, `vec.at(i)`,
  `PyList_GetItem(list, i)`, and so on) and is what every reader expects.
  Inverting it costs surprise and buys nothing semantic.

### 6.2 Parameter naming

- Input parameters must use the `in_` prefix.
- Output parameters must use the `out_` prefix.
- A parameter name must describe what the value *is*, not what role it played
  in a previous call. Do not name an input `result` because it was the output
  of the producer that built it. Name it after the thing it carries.
- An optional output should retain the local convention's nullability suffix
  (such as `_or_null`) so callers can read direction and optionality from the
  signature alone.
- Ownership semantics (borrowed, transferred-in, transferred-out) are an
  orthogonal axis from data direction. They belong in the per-function
  doc block and are not encoded in the parameter name.

```c
/* Good */
gln_status_t gln_backend_open_fints(
    const gln_fints_config_t*   in_config,
    gln_state_store_t*          in_state_store,
    gln_continuation_store_t*   in_cont_store_or_null,
    gln_secret_t*               in_pin,
    gln_backend_t**             out_backend,
    gln_error_t*                out_err_or_null);

const gln_fints_account_t* gln_account_list_at(
    const gln_account_list_t*   in_list,
    size_t                      in_index);
```

```c
/* Bad: input named after the producer's output role; no direction prefix */
const gln_fints_account_t* gln_account_list_at(
    const gln_account_list_t* result,
    size_t                    index);
```

### 6.3 Return convention

- A function that can fail must return a status code. Its result must be
  delivered through an `out_` parameter, not through the return value.
- A function with no meaningful status to report must return `void`. Do not
  return a value that callers will systematically ignore (a status that is
  always success, an `int` that is always zero, an unused enum).
- A function may return its result directly only when it meets the
  triviality test in 6.4. This is the only sanctioned exception to the
  status-return rule.

```c
/* Good: produces a handle, can fail at runtime */
gln_status_t gln_backend_open_fints(
    /* ... */
    gln_backend_t** out_backend,
    gln_error_t*    out_err_or_null);

/* Good: pure accessor, trivial under 6.4 */
size_t gln_account_list_count(const gln_account_list_t* in_list);

/* Good: no meaningful status to report */
void gln_account_list_destroy(gln_account_list_t* in_list);
```

```c
/* Bad: returns the produced handle directly even though it allocates and can fail */
gln_backend_t* gln_backend_open_fints(/* ... */);

/* Bad: returns a status that callers can never act on */
int gln_account_list_destroy(gln_account_list_t* in_list);
```

### 6.4 Trivial functions and the const-method test

A function may return its result directly only when *all* of the following
hold:

- It does not mutate any input or any caller-visible state. If it were a C++
  member function, marking it `const` would be honest.
- It performs no I/O.
- It performs no allocation that the caller is responsible for releasing.
- Its only possible failure modes are predictable, locally checkable input
  validation: null pointer, out-of-range index, missing optional field.
  These may be encoded as null or sentinel returns and require no separate
  status channel.

A function that does not meet all four conditions is not trivial under this
rule, even if its body is short.

- A function that allocates is not trivial, even if allocation almost never
  fails.
- A function that performs I/O is not trivial, even if the I/O usually
  succeeds.
- A function whose failure modes include domain-level conditions ("the bank
  rejected the request", "the file is malformed", "the token expired") is
  not trivial, regardless of how short its body is.

Trivial functions in this sense are typically `_count`, `_at`, `_size`,
getters over a materialized result handle, and pure conversions between
compile-time-known representations.

## 7. Review Checklist

- Names are descriptive and follow the applicable naming rules.
- The change is focused and does not include unrelated formatting churn.
- Control flow is explicit and fully braced.
- Includes are ordered correctly and are not relying on transitive includes.
- Comments explain intent or constraints instead of restating code.
- Constants, types, and ownership boundaries are clear.
- Module-local consistency is preserved.
- C-style API surfaces follow the parameter ordering, naming, return
  convention, and triviality rules in section 6.
