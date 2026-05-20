# UI Coding Standards

## Component Library

**Only shadcn/ui components may be used for UI in this project.** No custom components should be created. No exceptions to this rule.

- Install components via the shadcn CLI: `npx shadcn@latest add <component>`
- All installed components live in `components/ui/`
- Import directly from their file: `import { Button } from "@/components/ui/button"`
- Compose pages and features by combining shadcn components — do not wrap them in custom abstractions

## Date Formatting

Use **date-fns** for all date formatting. No other date library should be used.

### Required Format

Dates must be displayed as: **ordinal day + abbreviated month + full year**

| Date       | Formatted Output |
| ---------- | ---------------- |
| 2025-09-01 | 1st Sep 2025     |
| 2025-08-02 | 2nd Aug 2025     |
| 2026-01-03 | 3rd Jan 2026     |
| 2024-06-04 | 4th Jun 2024     |

### Implementation

```ts
import { format } from "date-fns";

function formatDate(date: Date): string {
  const day = date.getDate();
  const ordinal =
    day % 100 >= 11 && day % 100 <= 13
      ? "th"
      : (["th", "st", "nd", "rd"][Math.min(day % 10, 3)] ?? "th");

  return `${day}${ordinal} ${format(date, "MMM yyyy")}`;
}
```
