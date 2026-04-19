# Varinomics Coding Style — LLM Addendum

## Purpose and audience

This document is a companion to `varinomics_coding_style_guideline.md`. It exists to communicate a layer of judgment-based formatting that sits on top of the rules in the main guide.

The main guide remains the canonical baseline. This addendum does two narrower things: it records refinements that are intended house style when they apply, and it shows heuristics for making contextual formatting decisions once the baseline rules have been satisfied. It is aimed primarily at LLMs that need to reproduce a house style across files they have never seen, and at humans who want to understand why a given block was formatted the way it was.

Three things to keep in mind while reading:

1. **This document mixes refinements and heuristics.** Some patterns below are intended house style when they apply; others are examples of judgment. When this addendum conflicts with an explicit rule in the main guide, the main guide wins.
2. **Formatting here is judgment, not mechanics.** A linter or auto-formatter will not reproduce this output faithfully. Convergence under a mechanical pass produces shallow, uniform-looking code; the patterns in this addendum deliberately reward human/LLM judgment of local structure.
3. **Finding "nothing to do" is a valid and desirable outcome.** Many of the rules below describe what to do *when* a block has certain properties. When it doesn't, leave it alone. Drift toward uniformity for its own sake is a failure mode, not a success.

## The core concept

Most of the patterns in this addendum are instances of a single idea: **when a block of code contains several consecutive lines with similar structural shape, align the corresponding syntactic positions so that the block reads as a visual table.** The table reveals structure that would otherwise be hidden by incidental whitespace, and makes scanning much faster.

"Similar structural shape" is a judgment call. The best answer to "are these lines similar enough to align?" is usually "read them side by side and look for the positions you would want your eye to jump between." Common answers include: `=` in consecutive assignments, operators (`&&`, `||`, `==`, `<`, `>`, `+`, `-`) in compound conditions, accessor pairs (`.left()`/`.right()`, `.top()`/`.bottom()`, `.width()`/`.height()`) across sibling lines, and the comma between the first and second argument of a repeated function call.

The corollary: when a block **doesn't** have similar shape, forcing alignment makes things worse, not better. If aligning adds large empty gaps, or forces unrelated declarations to look artificially uniform, don't do it.

---

## Pattern 1 — Aligned `=` in declaration and assignment groups

When you see two or more consecutive `type name = value;` lines with broadly similar shape, pad the names (and if necessary the types) so every `=` lands in the same column. **Pairs count.** Do not wait for three identical shapes before aligning — a simple two-line pair like `m_v_max = ... ; m_v_page = ... ;` gets the same treatment.

**Before:**

```cpp
const int bounded_style = std::clamp(style, 0, STYLE_MAX);
const Style& scintilla_style = vs.styles[static_cast<size_t>(bounded_style)];
const Style& default_style = vs.styles[StyleDefault];
```

**After:**

```cpp
const int bounded_style      = std::clamp(style, 0, STYLE_MAX);
const Style& scintilla_style = vs.styles[static_cast<size_t>(bounded_style)];
const Style& default_style   = vs.styles[StyleDefault];
```

**Pair version:**

```cpp
int v_new_page = nPage;
int v_new_max  = nMax - v_new_page + 1;
```

```cpp
scn.nmhdr.code = Notification::URIDropped;
scn.text       = uri;
```

```cpp
m_current_row = -1;
m_top_row     = 0;
```

**Cross-type groups are fine**, and the decision of whether to align them is **not about gap size — it is about whether the lines form a single coherent operation.** If a block of 3–5 lines computes one thing together — read one value, derive another from it, derive a third from both — align them even if some lines end up with 10+ characters of padding between name and `=`. If the lines just happen to be neighbors but are unrelated (configure something, read a property, declare an unrelated helper), leave them at their natural width no matter how close the types are.

**Do align — coherent operation with large gap:**

```cpp
const unsigned char uch       = text[i];
const unsigned int byte_count = UTF8BytesOfLead[uch];
const int code_units          = UTF16LengthFromUTF8ByteCount(byte_count);
qreal x_position              = tl.cursorToX(ui + code_units);
```

The four lines are one UTF-8 glyph-positioning calculation. `qreal` is 5 characters wide and `const unsigned char` is 19 characters wide, giving `x_position` 14 spaces of padding before the `=`. The alignment is still the right call because the *logical* unit is one block.

**Do align — short, similar, different types:**

```cpp
const int    selection_start = 7;
const int    selection_end   = 11;
const int    selection_line  = static_cast<int>(send(SCI_LINEFROMPOSITION, selection_start));
const double expected_left   = static_cast<double>(send(SCI_POINTXFROMPOSITION, 0, selection_start));
const double expected_right  = static_cast<double>(send(SCI_POINTXFROMPOSITION, 0, selection_end));
```

All five lines compute input coordinates for the same fixture check.

**Before:**

```cpp
const int selection_start = 7;
const int selection_end = 11;
const int selection_line = static_cast<int>(send(SCI_LINEFROMPOSITION, selection_start));
const double expected_left = static_cast<double>(send(SCI_POINTXFROMPOSITION, 0, selection_start));
const double expected_right = static_cast<double>(send(SCI_POINTXFROMPOSITION, 0, selection_end));
```

**After:**

```cpp
const int selection_start   = 7;
const int selection_end     = 11;
const int selection_line    = static_cast<int>(send(SCI_LINEFROMPOSITION, selection_start));
const double expected_left  = static_cast<double>(send(SCI_POINTXFROMPOSITION, 0, selection_start));
const double expected_right = static_cast<double>(send(SCI_POINTXFROMPOSITION, 0, selection_end));
```

Note the padding is on the **names**, not the types. The `=` column is `max(len("type name")) + 1` across all lines in the group.

**Assignment-block version:** the same rule applies to `obj.field = value;` / `map[key] = value;` / `target[i] = value;` groups:

```cpp
m_render_data->snapshot_dirty        = true;
m_render_data->static_content_dirty  = m_render_data->static_content_dirty || static_content_dirty;
m_render_data->style_sync_needed     = m_render_data->style_sync_needed || needs_style_sync;
m_render_data->scrolling_update      = m_render_data->scrolling_update || scrolling;
m_render_data->overlay_content_dirty = true;
```

**Struct/POD initialization block:**

```cpp
Capture_text_run out;
out.text                = run.utf8_text;
out.foreground          = qcolor_from_rgba(run.foreground_rgba);
out.x                   = run.x;
out.width               = run.width;
out.top                 = run.top;
out.bottom              = run.bottom;
out.baseline_y          = run.baseline_y;
out.style_id            = run.style_id;
out.direction           = direction_from_capture(run.direction);
out.is_represented_text = run.is_represented_text;
out.represented_as_blob = run.represented_as_blob;
```

**Counter-example — don't force alignment across unrelated declarations, regardless of gap size:**

```cpp
// Don't do this:
const QString        output_dir          = QFileInfo(...).absoluteFilePath();
const QStringList    selected_fixtures   = parser.values(fixture_option);
const auto           should_run_fixture  = [&](const QString& name) { ... };
```

These three declarations happen to be adjacent in the source, but they're three unrelated setup steps — resolving an output path, parsing a CLI flag, defining a filter predicate. They do not form one logical unit of computation, so aligning them produces a table that shows structure that isn't there. Leave them at their natural width.

The rule of thumb: ask "would I read these three lines as a single logical step, or as three separate things that happen to be near each other?" If it's three separate things, don't align them.

### Pattern 1b — `=`, `+=`, and `-=` share one vertical rail

When a block contains a mix of plain assignments and compound assignments (`+=`, `-=`, `*=`, `/=`), the `=` character in every one of them counts as the alignment anchor. Pad so all the `=` characters line up in the same column.

**Before:**

```cpp
const XYPOSITION halfStroke = fillStroke.stroke.width / 2.0f;
const XYPOSITION radius = rc.Height() / 2.0f - halfStroke;
PRectangle rcInner = rc;
rcInner.left += radius;
rcInner.right -= radius;
const XYPOSITION arc_height = rc.Height() - fillStroke.stroke.width;
```

**After:**

```cpp
const XYPOSITION halfStroke = fillStroke.stroke.width / 2.0f;
const XYPOSITION radius     = rc.Height() / 2.0f - halfStroke;
PRectangle rcInner          = rc;
rcInner.left               += radius;
rcInner.right              -= radius;
const XYPOSITION arc_height = rc.Height() - fillStroke.stroke.width;
```

The `=` of the const declarations, and the `=` inside the `+=` and `-=` operators, all land in the same column. The block reads as one coherent geometry setup even though the line shapes vary.

### Pattern 1c — Sign alignment on numeric literals

When an aligned group of initialisers contains a mix of negative and positive numeric literals, pad the positives with one leading space so the digits line up under the minus sign of the negatives.

**Before:**

```cpp
int               m_visible_rows = 5;
int               m_current_row  = -1;
int               m_top_row      = 0;
```

**After:**

```cpp
int               m_visible_rows =  5;
int               m_current_row  = -1;
int               m_top_row      =  0;
```

After the `=`, the first *digit* of each value sits in the same column. `5` and `0` get one extra leading space so they line up under the `1` of `-1`. This applies to any group of aligned short numeric initialisers with mixed signs, not just member initialisers.

### Pattern 1d — Pad type names inside `<>` brackets and function calls to align across rows

When an aligned group of declarations has varying type parameters inside template-like constructs — `static_cast<T>`, `std::vector<T>`, `std::make_unique<T>`, etc. — pad the shorter type *inside the angle brackets* so the closing `>` lines up across rows. The same principle extends to padding arguments **inside** a function call on one row so they align with corresponding arguments in a sibling row that has more arguments.

**Before:**

```cpp
const int    selection_line  = static_cast<int>(send(SCI_LINEFROMPOSITION, selection_start));
const double expected_left   = static_cast<double>(send(SCI_POINTXFROMPOSITION, 0, selection_start));
const double expected_right  = static_cast<double>(send(SCI_POINTXFROMPOSITION, 0, selection_end));
```

**After:**

```cpp
const int    selection_line  = static_cast<int   >(send(SCI_LINEFROMPOSITION,      selection_start));
const double expected_left   = static_cast<double>(send(SCI_POINTXFROMPOSITION, 0, selection_start));
const double expected_right  = static_cast<double>(send(SCI_POINTXFROMPOSITION, 0, selection_end));
```

Two alignments happen together here:

1. **Inside the cast brackets:** `int` is padded with three trailing spaces → `int   ` so the `>` of `static_cast<int   >` aligns with the `>` of `static_cast<double>` below it. The `double` is left at its natural width because it is the longest type in the group.

2. **Inside the inner `send(...)` call:** the top row's `send(SCI_LINEFROMPOSITION, selection_start)` has no middle `0,` argument, while the two rows below have one. The top row's comma after `SCI_LINEFROMPOSITION,` is padded with six spaces so its `selection_start` argument starts in the same column as the `selection_start` in the rows below (which sits after `, 0, `). The result is that `selection_start` stacks vertically across all three rows.

This pattern is what the earlier Pattern 1 rules call "coherent operation" alignment taken one level deeper: not only do the outer `=` and the outer call arguments line up, but the *inner* type parameters and inner function-call arguments also line up. The cost is a couple of unusual whitespace insertions; the payoff is that the three rows read as one 2D table where every syntactically-corresponding token is in its own column.

**When to do this:** only when the rows form a single coherent operation (see Pattern 1), the syntactic shape is parallel enough that the inner alignment is meaningful, and the resulting padding is modest. This kind of nested alignment may extend to omitted optional arguments and to inner callee names when that makes the omission or correspondence visually explicit.

**When to stop:** stop pushing the alignment inward once the added padding creates large gaps, fights a wrap decision, or stops exposing a meaningful relationship between the rows. The goal is to reveal structure, not to recursively align every token.

**Counter-example:** don't do this on three independent `static_cast<>` calls that happen to be near each other in source but compute unrelated things. The alignment is only useful when the reader's eye is already comparing the rows as variants of the same calculation.

---

## Pattern 2 — Wrapped calls and declarations: choose line split or argument split

When a declaration, definition, or call no longer reads cleanly on one line, choose one of two deliberate shapes:

1. **Line split.** Keep the simple leading part of the call on the first line and move a long or structured trailing argument or suffix to the next line.
2. **Argument split.** Break immediately after the opening `(` and put one argument per line, or small scannable groups per line.

The choice is contextual. The point is not to force every wrapped call into the same shape; it is to pick the shape that best exposes the structure of the local block.

**Line split is often better** when the call shape is repetitive, only one trailing argument is long, and forcing one-argument-per-line would add bulk without revealing anything new.

```cpp
ok &= check(nearly_equal(ind.rect.bottom(), expected_bottom), id,
    "indicator rect bottom must stay within the indicator line");
```

**Argument split is often better** when the arguments are numerous, dense, or intentionally varied, and the reader should compare them one by one.

```cpp
void sync_text_nodes_by_key(
    QQuickWindow*                              window,
    QSGNode*                                   parent,
    std::vector<Scene_graph_frame_text_node*>& nodes,
    const std::vector<Visual_line_frame>&      visual_lines,
    const QRectF&                              viewport);
```

The same applies to wrapped function calls:

```cpp
m_core->current_render_frame(
    nullptr,
    static_content_dirty,
    m_render_data->style_sync_needed && static_content_dirty,
    m_render_data->scrolling_update,
    capture_buffer_lines);
```

**Avoid accidental hybrids.** Opening after `(` and then keeping an arbitrary dense remainder on the next line usually loses the benefit of both shapes.

```cpp
// Avoid this simple-case hybrid:
ok &= check(
    nearly_equal(sel.rect.right(), expected_right), id,
    "selection rect right edge must match the selected text end");
```

**Short and simple calls stay on one line.** Do not wrap a call that already reads cleanly just because some nearby call had to wrap.

```cpp
sync_text_nodes_by_key(window, m_text_group, m_text_nodes, frame.visual_lines, frame.text_rect);
```

### Pattern 2b — Short sibling declarations may align after `(`

Short sibling declarations with short parameter lists should usually stay on one line. When a small coherent family of such declarations benefits from it, pad after `(` so the parameter text begins in the same column across the group.

```cpp
void DragEnter(const Point& point);
void DragMove( const Point& point);
```

This also applies when the parameter types are not textually identical but are still comparable enough that the group reads as one local family:

```cpp
void MousePress(  QMouseEvent* ev);
void MouseMove(   QMouseEvent* ev);
void MouseRelease(QMouseEvent* ev);
void Wheel(       QWheelEvent* ev);
```

Use this only within a small coherent sibling group. Do not extend it mechanically across unrelated declarations just because they are nearby.

---

## Pattern 3 — Accessor-pair alignment

`.left()`/`.right()`, `.top()`/`.bottom()`, `.width()`/`.height()`, `.x()`/`.y()` appear constantly in geometry code. When two or more consecutive lines use such a pair, pad the shorter accessor with a single space so the operator or comma after it lines up with the longer accessor's.

**Before:**

```cpp
ok &= check(sel.rect.left() >= frame.text_rect.left() - 2.0, id, "selection rect extends left of text area");
ok &= check(sel.rect.right() <= frame.text_rect.right() + 2.0, id, "selection rect extends right of text area");
```

**After:**

```cpp
ok &= check(sel.rect.left()  >= frame.text_rect.left()  - 2.0, id, "selection rect extends left of text area");
ok &= check(sel.rect.right() <= frame.text_rect.right() + 2.0, id, "selection rect extends right of text area");
```

The columns that now line up: the outer accessor (`.left()`/`.right()`), the operator (`>=`/`<=`), the inner accessor, the sign (`-`/`+`), and the constant.

**The rule also applies within a single wrapped compound condition**, not just across sibling lines:

**Before (not allowed):**

```cpp
if (ind.rect.bottom() > vl.clip_rect.top() - 2.0 && ind.rect.top() < vl.clip_rect.bottom() + 2.0 &&
    ind.rect.right() > vl.clip_rect.left() - 2.0 && ind.rect.left() < vl.clip_rect.right() + 2.0)
```

**After:**

```cpp
if (ind.rect.bottom() > vl.clip_rect.top()  - 2.0 && ind.rect.top()  < vl.clip_rect.bottom() + 2.0 &&
    ind.rect.right()  > vl.clip_rect.left() - 2.0 && ind.rect.left() < vl.clip_rect.right()  + 2.0)
```

Both the within-line pairs (`bottom()`/`right()` vs `top()`/`left()`) and the cross-line pairs (`bottom()` above `right()`) get padded so the `>` and `<` columns are visibly aligned.

**Coordinate lists follow the same rule:**

```cpp
return {
    QPointF(rect.left(),  rect.top()),
    QPointF(rect.right(), rect.top()),
    QPointF(rect.right(), rect.bottom()),
    QPointF(rect.left(),  rect.bottom()),
    QPointF(rect.left(),  rect.top()),
};
```

`.left(),` is padded by one space so its comma aligns with `.right(),` above and below.

---

## Pattern 4 — Binary expressions split at the top-level operator

When a single long wrapped line has a top-level binary operator (`-`, `+`, `*`, `/`, `&&`, `||`, `==`), split at that operator so both operands sit at the same indent. Inside each operand, apply accessor-pair alignment recursively.

**Before:**

```cpp
double v_overlap =
    std::min(a.clip_rect.bottom(), b.clip_rect.bottom()) - std::max(a.clip_rect.top(), b.clip_rect.top());
```

**After:**

```cpp
double v_overlap =
    std::min(a.clip_rect.bottom(), b.clip_rect.bottom()) -
    std::max(a.clip_rect.top(),    b.clip_rect.top());
```

Both `std::min(...)` and `std::max(...)` are at the same indent under the `=`, and inside each call, `.top()` has been padded so it lines up under `.bottom()`.

Another example:

```cpp
const long long key =
    (static_cast<long long>(line->key.document_line) << 32) |
    static_cast<unsigned int>(line->key.subline_index);
```

---

## Pattern 5 — Long boolean chains on separate lines with trailing operators

When an `&&` or `||` chain wraps, put each clause on its own line with the operator at the **end** of the line, and pad the clauses so the operators form a vertical rail.

**Before:**

```cpp
const bool scroll_position_changed = m_render_data->previous_first_visible_line < 0 ||
                                     m_render_data->previous_x_offset < 0 ||
                                     current_first_visible_line != m_render_data->previous_first_visible_line ||
                                     current_x_offset != m_render_data->previous_x_offset;
```

**After:**

```cpp
const bool scroll_position_changed =
    m_render_data->previous_first_visible_line < 0                           ||
    m_render_data->previous_x_offset           < 0                           ||
    current_first_visible_line != m_render_data->previous_first_visible_line ||
    current_x_offset           != m_render_data->previous_x_offset;
```

Notice three things:

1. The `const bool … =` sits alone on its own line; all the clauses are beneath it at the same indent.
2. Operators (`<`, `!=`) within each clause have been padded so they form a vertical column where the clauses share shape.
3. The trailing `||`s form a rail at the end of each line.

**For a plain `return` of such a chain**, the `return` can sit alone on its own line too:

```cpp
return
    hangul_jamo            ||
    hangul_compatible_jamo ||
    hangul_syllable        ||
    hangul_jamo_extended_a ||
    hangul_jamo_extended_b;
```

---

## Pattern 6 — Wrapped `if` conditions with brace on next line

When an `if`, `for`, or `while` condition wraps across lines, put the opening brace on its own next line. This is an explicit visual cue that the condition itself is multi-line.

**Before (not allowed):**

```cpp
if (caret.rect.bottom() > vl.clip_rect.top() - 1.0 && caret.rect.top() < vl.clip_rect.bottom() + 1.0) {
    overlaps_any = true;
    break;
}
```

**After:**

```cpp
if (caret.rect.bottom() > vl.clip_rect.top() - 1.0 &&
    caret.rect.top() < vl.clip_rect.bottom() + 1.0)
{
    overlaps_any = true;
    break;
}
```

---

## Pattern 7 — Stream chains: `<<` at the start of continuation lines

When a `<<` chain wraps, move `<<` to the **start** of the continuation lines and indent consistently. Never leave a dangling `<<` at the end of one line and another `<<` at the start of the next.

**Before:**

```cpp
qWarning().noquote() << "ScintillaQuick profiling started."
                     << " output_dir=" << m_profiling_state->output_directory
                     << " duration_seconds=" << duration_seconds;
```

**After:**

```cpp
qWarning().noquote()
    << "ScintillaQuick profiling started."
    << " output_dir=" << m_profiling_state->output_directory
    << " duration_seconds=" << duration_seconds;
```

---

## Pattern 8 — Wrapped ternaries as structured subordinate blocks

When a ternary wraps, make the `?` and `:` structure visually clear. The condition sits alone on its own line, and each branch is an indented continuation. Don't flatten the ternary back onto one line just because it would fit.

```cpp
const QColor base_color = guide.highlight
    ? QColor(192, 192, 192)
    : QColor(128, 128, 128);
```

When a ternary is itself a function argument, wrap the whole thing:

```cpp
Profiling_scope scope(
    m_profiling_state && m_profiling_state->active.load(std::memory_order_acquire)
        ? &m_profiling_state->update_polish
        : nullptr);
```

The outer call breaks after `(`, then the condition sits on its own line, and the `?`/`:` branches are at +4 indent under the condition.

**Nested ternary as function argument** (used, for example, when selecting among several colors):

```cpp
fill_rect(below_symbol,
    plus && primitive.fold_part == 4 ? colors.tail :
    plus                             ? colors.body :
                                       colors.head);
```

Each condition sits on its own line with the `:` separating it from the next condition. The final else-branch is indented to the value column.

---

## Pattern 9 — Compact pure-dispatch switches

When every arm of a `switch` is a short single-statement dispatch (a direct `return` or an assignment-plus-`break` pair that fits cleanly), collapse each arm to one line and align the columns. The compact form must be preserved across an auto-formatter pass if your tooling tries to expand it.

```cpp
switch (style) {
    case Display_style::DOTS: return dots;
    case Display_style::AREA: return area;
    case Display_style::LINE: return line;
    default:                  return line;
}
```

With column padding:

```cpp
switch (eff) {
    case FontQuality::QualityDefault:         return QFont::PreferDefault;
    case FontQuality::QualityNonAntialiased:  return QFont::NoAntialias;
    case FontQuality::QualityAntialiased:
    case FontQuality::QualityLcdOptimized:    return QFont::PreferAntialias;
    default:                                  return QFont::PreferDefault;
}
```

The `return` keyword in every arm starts in the same column.

**The compact form extends to `key = X; break;` arms** when every arm has that same two-statement shape:

```cpp
switch (event->key()) {
    case Qt::Key_Down:          key = SCK_DOWN;     break;
    case Qt::Key_Up:            key = SCK_UP;       break;
    case Qt::Key_Left:          key = SCK_LEFT;     break;
    case Qt::Key_Right:         key = SCK_RIGHT;    break;
    case Qt::Key_Home:          key = SCK_HOME;     break;
    case Qt::Key_End:           key = SCK_END;      break;
    case Qt::Key_PageUp:        key = SCK_PRIOR;    break;
    case Qt::Key_PageDown:      key = SCK_NEXT;     break;
    case Qt::Key_Backtab:       // fall through
    case Qt::Key_Tab:           key = SCK_TAB;      break;
    case Qt::Key_Enter:         // fall through
    case Qt::Key_Return:        key = SCK_RETURN;   break;
    case Qt::Key_Control:       key = 0;            break;
    case Qt::Key_Alt:           key = 0;            break;
    case Qt::Key_Shift:         key = 0;            break;
    case Qt::Key_Meta:          key = 0;            break;
    default:                    key = event->key(); break;
}
```

All three columns (label, assignment, `break;`) are aligned. Fall-through comments sit alone on their own line above the target arm.

**Counter-example:** when a switch has any arm with three or more statements, or with nested braces, or with different structural shape, leave the whole switch in the normal multi-line-per-arm form. Mixing compact and expanded arms in the same switch looks worse than picking one.

---

## Pattern 10 — Test-harness `check(cond, id, msg)` wrap rules

This is a specific pattern for test code where a `check()` (or similar) function takes a boolean condition, a fixture id, and a descriptive message.

**Compact form — if it fits on one line and reads cleanly, keep it compact:**

```cpp
ok &= check(sel.rect.width() > 0, id, "selection rect width must be > 0");
```

**Line split — if the message alone is what makes the call long, splitting after `id,` is often the best compact wrapped form:**

```cpp
ok &= check(nearly_equal(ind.rect.bottom(), expected_bottom), id,
    "indicator rect bottom must stay within the indicator line");
```

This is especially effective in repetitive assertion blocks where the call shape stays the same and the message text is the main thing that varies.

**Block consistency is heuristic, not absolute.** If one long message appears inside a tight block of otherwise similar `check()` calls, both of these can be reasonable:

1. Keep the short ones compact and split only the exceptional long line.
2. Use the same line-split shape across the whole coherent block when that makes the block read more uniformly.

```cpp
ok &= check(sel.main,            id, "selection must be main");
ok &= check(sel.valid,           id, "selection must be valid");
ok &= check(line != nullptr,     id, "selection must map to a line");
ok &= check(line == active_line, id,
    "selection must map to the currently active visual line in the test fixture");
```

and

```cpp
ok &= check(sel.is_main, id,
    "single-line selection must be marked as main");
ok &= check(sel.rect.width()  > 0, id,
    "selection rect width must be > 0");
ok &= check(sel.rect.height() > 0, id,
    "selection rect height must be > 0");
ok &= check(line != nullptr, id,
    "selection rect must map to a visual line");
ok &= check(line && line->key.document_line == selection_line, id,
    "selection rect must map to the selected document line");
ok &= check(nearly_equal(sel.rect.left(), expected_left), id,
    "selection rect left edge must match the selected text start");
ok &= check(nearly_equal(sel.rect.right(), expected_right), id,
    "selection rect right edge must match the selected text end");
```

The judgment call is what counts as one block and how exceptional the long line really is. If only one line is the exception, leaving only that line split is often fine. If several neighboring lines are close in shape and topic, using one compact wrapped shape across them can be better.

**Argument split — use it when the condition itself is long, structured, or intentionally varied enough that the reader should see its internal shape:**

```cpp
ok &= check(
        sel.rect.top() ==            frame.selection_primitives.front().rect.top() ||
        nearly_equal(sel.rect.top(), frame.selection_primitives.front().rect.top()),
    id, "multi-selection rects must sit on the same visual line");
```

Here the first argument is itself the structured block. In that case, opening after `(` is the right choice because it reveals the condition's own shape.

Inside the condition, the two `sel.rect.top()` subexpressions are padded so `==` lines up with `nearly_equal(sel.rect.top(),`, making the two alternative formulations visually comparable.

---

## Pattern 11 — Struct initializer arrays / dispatch tables

When you have a `{label, value}` initializer list with many rows (a test fixture table, a dispatch array, a keyword map), pad the first column so the second column starts in the same place. Add blank lines between semantic groups.

```cpp
struct
{
    const char* name;
    bool (*fn)();
}
fixtures[] = {

    // Phase 2 fixtures
    {"plain_ascii_short",                        test_plain_ascii_short},
    {"plain_ascii_long_wrap",                    test_plain_ascii_long_wrap},
    {"horizontal_scroll_resets_on_doc_switch",   test_horizontal_scroll_resets_on_doc_switch},
    {"caret_left_scrolls_to_long_previous_line", test_caret_left_scrolls_to_long_previous_line},
    {"mixed_styles_wrap",                        test_mixed_styles_wrap},
    {"tab_layout_default",                       test_tab_layout_default},
    {"tab_layout_nondefault",                    test_tab_layout_nondefault},

    // Phase 3: multi-selection, rectangular selection, current-line
    {"multi_selection",                          test_multi_selection},
    {"rectangular_selection",                    test_rectangular_selection},
    {"current_line_frame",                       test_current_line_frame},

    // Phase 4: expanded indicator styles
    {"squiggle_indicator",                       test_squiggle_indicator},
    {"box_indicator",                            test_box_indicator},
};
```

Two details worth noting:

1. The closing `}` of the anonymous struct is on its own line, separated from the variable name `fixtures[] = {`. Writing `} fixtures[] = {` in one line is allowed in smaller tables but splits cleanly when the table is large.
2. A blank line separates each semantic group, with a one-line `// Phase N: …` comment labelling each.

---

## Pattern 12 — Consecutive similar API calls get their arg columns aligned

When you see several consecutive calls to the same function with different first arguments (very common in test-setup code that configures an editor or state machine), pad the first argument so the second argument column starts in the same place.

**Before:**

```cpp
f.editor.send(SCI_INDICSETSTYLE, 1, INDIC_BOX);
f.editor.send(SCI_INDICSETFORE, 1, 0xFF0000);
f.editor.send(SCI_INDICSETUNDER, 1, 1);
f.editor.send(SCI_SETINDICATORCURRENT, 1);
f.editor.send(SCI_INDICATORFILLRANGE, 4, 6); // "around"
```

**After:**

```cpp
f.editor.send(SCI_INDICSETSTYLE,       1, INDIC_BOX);
f.editor.send(SCI_INDICSETFORE,        1, 0xFF0000);
f.editor.send(SCI_INDICSETUNDER,       1, 1);
f.editor.send(SCI_SETINDICATORCURRENT, 1);
f.editor.send(SCI_INDICATORFILLRANGE,  4, 6); // "around"
```

Five lines with a shared shape (`send(SCI_FOO, …)`), so the constant name becomes column 1 of a table and the remaining arguments sit in column 2.

---

## Pattern 13 — Nested point/rect constructor tables

When an initializer list or function call contains several structurally-identical nested constructors (like `QPointF(…)` or `QRectF(…)` rows inside a list), align the columns of those nested constructors. Pad shorter member-access expressions with a single space to match longer ones.

**Before:**

```cpp
points.insert(points.end(), {
    QPointF(cx - arm, cy),
    QPointF(cx + arm, cy),
    QPointF(cx, cy - arm),
    QPointF(cx, cy + arm),
});
```

**After:**

```cpp
points.insert(
    points.end(),
    {
        QPointF(cx - arm, cy),
        QPointF(cx + arm, cy),
        QPointF(cx,       cy - arm),
        QPointF(cx,       cy + arm),
    });
```

Three things happen:

1. The outer `.insert(…)` call is broken after `(` with one argument per line, since it now contains a multi-line initializer list.
2. Inside each `QPointF(…)` row, the first argument column is padded so the comma after the first argument lines up across all four rows.
3. The rows themselves form a readable 4-row table even though the first column has different shapes (`cx - arm`, `cx + arm`, `cx`, `cx`).

Another example — a four-row `append_rect_triangles` call:

```cpp
append_rect_triangles(triangles, QRectF(box.left(),          box.top(),            box.width(), pixel));
append_rect_triangles(triangles, QRectF(box.left(),          box.bottom() - pixel, box.width(), pixel));
append_rect_triangles(triangles, QRectF(box.left(),          box.top(),            pixel,       box.height()));
append_rect_triangles(triangles, QRectF(box.right() - pixel, box.top(),            pixel,       box.height()));
```

Each `QRectF(…)` row has its four arguments padded so the commas form four columns. The different first-argument shapes (`box.left()` vs `box.right() - pixel`) are padded to the same width so the second column lines up.

**Note:** this table exceeds the normal 120-column ceiling. It is accepted because the alignment reveals a clear 4-column table of rectangle coordinates that would be lost at 120 cols. See Pattern 15 for the rules on when >120 is acceptable.

---

## Pattern 14 — Repetitive helper call extraction

When the same complex wrapped expression appears several times in a row, introduce a short local alias or helper. This is not strictly a formatting rule — it is a readability refactor that often follows naturally from the "interrogate long lines" rule in Pattern 15, but it applies even when no single line is over the limit.

**Before:**

```cpp
cached_attributes.foreground = QColorFromColourRGBA(ColourRGBA(static_cast<int>(style.fore_rgba)));
cached_attributes.background = QColorFromColourRGBA(ColourRGBA(static_cast<int>(style.back_rgba)));
cached_attributes.selected   = QColorFromColourRGBA(ColourRGBA(static_cast<int>(style.selected_rgba)));
```

**After** — introduce a tiny helper in the anonymous namespace at the top of the file:

```cpp
// Convert a raw Scintilla capture rgba value to a QColor. Wraps the
// recurrent `QColorFromColourRGBA(ColourRGBA(static_cast<int>(X)))` triple
// used across every capture-to-frame conversion.
template <typename T>
QColor qcolor_from_rgba(T rgba)
{
    return QColorFromColourRGBA(ColourRGBA(static_cast<int>(rgba)));
}
```

and rewrite the three uses as:

```cpp
cached_attributes.foreground = qcolor_from_rgba(style.fore_rgba);
cached_attributes.background = qcolor_from_rgba(style.back_rgba);
cached_attributes.selected   = qcolor_from_rgba(style.selected_rgba);
```

Threshold rule of thumb: if a wrapped pattern appears four or more times in the same translation unit, a helper is probably worth it. At two or three occurrences, it is judgment; a lambda in the immediate enclosing function can be enough.

**A local alias is often enough:** when a multi-character receiver like `capture_margin_text.text` is referenced several times in a single loop body, `const auto& text_bytes = capture_margin_text.text;` lets the body read `text_bytes.data()` / `text_bytes.size()` instead.

**Hoisting a repeated call into a local temp** is the same pattern:

```cpp
// Before (one line, 135 cols):
points = make_rect_outline_as_lines(
    rect.adjusted(0.0, 0.0, -physical_pixel_size(window), -physical_pixel_size(window)));

// After:
const qreal pixel = physical_pixel_size(window);
points = make_rect_outline_as_lines(rect.adjusted(0.0, 0.0, -pixel, -pixel));
```

---

## Pattern 15 — The 120-column rule is a *diagnostic signal*, not a tolerance

The main guide treats 120 columns as a strong limit, not as a formatting goal. This addendum pushes the decision point earlier: **once a line is much longer than its neighbors, or moves past roughly 100 columns, ask whether the current shape is still the clearest one.** Readability wins. The threshold is a signal, not the point of the style.

The interrogation order is:

1. **Verbose local names.** `physical_pixel_size_in_logical_units` inside a small function is almost certainly too long for what it carries. Rename locally: `const qreal pixel = physical_pixel_size(window);`. The rename shortens every subsequent use and often drops the line under 120 by itself.
2. **Duplicated namespace qualifiers.** If `Scintilla::Internal::Something` appears three times in the same statement, a `using Item = ScintillaQuick_item;` or a `namespace sg = Scene_graph::Internal;` inside the function turns the duplication into a short alias.
3. **Repeated call chains.** `obj->a()->b()->c()` used several times begs for a cached `const auto& x = obj->a()->b()->c();` at the top of the block.
4. **Formatting-pass scope.** Small local renames, aliases, or cached subexpressions are fair game when they obviously improve scanability. Broader helper extraction or API refactoring is a separate cleanup choice, not something a formatting pass must invent.

If a line is the only conspicuously long line in a local block, splitting it is often right even when it is still below 120. If several nearby lines share the same strong structure and that structure reads best as a table, keeping one of them slightly over 120 can still be right.

Treat ~100 as "take a look," ~110 as "interrogate," and >120 as "justify with readability, not with habit."

---

## Pattern 16 — Preserve existing coherent structure when in doubt

The main guide repeatedly treats existing coherent formatting as evidence. This is the most important judgment rule in the whole addendum, because it is the one most likely to be violated by an LLM doing a mechanical pass.

If a block of code is already visually well-structured — it has intentional alignment, it has semantic grouping, it has a layout that makes scanning faster — the correct default is to **leave it alone**. Do not merge split lines just because the merged form still fits under 120. Do not de-align a coherent aligned block to "normalize whitespace." Do not expand compact one-line helpers into multi-line blocks because some formatter would prefer it that way.

The positive statement: a pass over a file that leaves most of it unchanged is a *successful* pass. Finding "nothing to do" in a given block is a valid outcome. Drift toward mechanical uniformity is a failure mode.

---

## Pattern 17 — Constructor initializer list with `:` on its own line

The main style guide already requires that constructor initializer list colons go on their own line for multi-member inits. This addendum clarifies: **the same form is used even for single-member initializer lists.**

The preferred shape is:

```cpp
Example(QObject* parent)
:
    QObject(parent)
{
    // ...
}
```

not

```cpp
Example(QObject* parent) : QObject(parent)
{
    // ...
}
```

The rationale: the `:` always sits on its own line at the outer-block indent, and the member initializers always sit one indentation level deeper than the `:`. That shape is the same whether there is one member init or ten, which means the visual rhythm of constructors is consistent across the codebase.

For multi-member cases the same form applies:

```cpp
Capture_frame_builder(Captured_frame& frame, bool capture_static_content)
:
    m_frame(frame),
    m_capture_static_content(capture_static_content)
{}
```

and for a wrapped constructor signature, the `:` still sits on its own line at the outer indent, below the closing `)` of the signature:

```cpp
Example(
    QObject* parent,
    Helper helper)
:
    QObject(parent),
    m_helper(std::move(helper))
{}
```

---

## Pattern 18 — Multi-literal string arguments break after the opening `(`

When the only argument to a function (typically `QStringLiteral`, `qDebug`, `qWarning`, `std::format`, `qFatal`) is two or more adjacent C++ string literals that together describe a single long message, break the call after the opening `(` and put each literal on its own continuation line. Chained `.arg(...)` calls (or other trailing method calls) follow at the natural continuation indent.

**Before:**

```cpp
lines << QStringLiteral("Wrapped line %1: The quick brown fox jumps over the lazy dog "
                        "and keeps running until the text spans multiple wrapped sublines "
                        "at the narrow visual-regression width %2.")
             .arg(i + 1, 5, 10, QChar('0'))
             .arg((i * 13) % 101);
```

**After:**

```cpp
lines << QStringLiteral(
    "Wrapped line %1: The quick brown fox jumps over the lazy dog "
    "and keeps running until the text spans multiple wrapped sublines "
    "at the narrow visual-regression width %2.")
        .arg(i + 1, 5, 10, QChar('0'))
        .arg((i * 13) % 101);
```

The literals are at the normal continuation indent under the opening `(`, and the `.arg(...)` chain is at +4 below that. The ugly "hanging indent under text from the first line" form (where later literals are lined up under `"Wrapped line %1…"`) is not allowed.

**Counter-example — if the two literals fit comfortably on one line as the joined form, prefer the joined form:**

```cpp
// Original:
qWarning("  [%s] document_line %d has %d margin primitives "
         "(expected 1 per doc line)",
    id, doc_line, count);

// After (one joined literal, fits on a single line):
qWarning("  [%s] document_line %d has %d margin primitives (expected 1 per doc line)",
    id, doc_line, count);
```

Multi-literal concatenation exists to keep a long string under the column limit. When the joined form fits, the concatenation is unnecessary and the single-literal form reads better.

---

## How to use this addendum

For LLMs performing a style pass:

1. Read the main guide first. The rules there always win.
2. When you encounter a block with 2+ lines of similar shape, check whether the patterns in this document apply. If they do, apply them. The decision is about whether the lines form a single coherent operation, not about gap size — a 14-character gap is acceptable when the logical unit is clear.
3. When you encounter a line over ~110 cols, go through the Pattern 15 interrogation sequence before wrapping.
4. When making a wrap decision inside a `check()` or similar assertion block, look at the *neighbors* in the same block. If they had to wrap, wrap uniformly. If they're all compact, stay compact.
5. Do not confuse "finished" with "edited every line." Most of every file should stay unchanged on most passes.

For humans reading the output of such a pass:

- Each edit should either match one of the patterns above, or reduce to one of the rules in the main guide, or be a semantic refactor (rename / cache / helper / alias) whose effect is obvious.
- If you see a diff that doesn't fit any of those categories, that's either a pattern this addendum should learn or a mistake — in either case, worth discussing.
