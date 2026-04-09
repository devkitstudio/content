## Execution: Managing Utility-First Variants

When adopting Tailwind (Utility-First), the biggest architectural challenge is managing component variants (e.g., Primary, Secondary, Outline buttons) without polluting the markup with massive string concatenations.

**Architectural Rule:** Manage state and variants using JavaScript, not CSS abstractions.

```typescript
// The Standard Component Variant Architecture (using 'cva' and 'clsx'/'tailwind-merge')
import { cva, type VariantProps } from "class-variance-authority";
import { cn } from "@/lib/utils"; // A utility wrapping clsx and tailwind-merge

// 1. Define the design system contract in TypeScript
const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2",
  {
    variants: {
      variant: {
        default: "bg-blue-600 text-white hover:bg-blue-700",
        destructive: "bg-red-500 text-white hover:bg-red-600",
        outline: "border border-gray-200 bg-transparent hover:bg-gray-100 text-gray-900",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
);

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement>, VariantProps<typeof buttonVariants> {}

// 2. The Component Execution
export function Button({ className, variant, size, ...props }: ButtonProps) {
  return (
    <button
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  );
}
```
