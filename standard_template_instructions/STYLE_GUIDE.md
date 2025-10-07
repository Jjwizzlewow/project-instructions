# Frontend Style Guide

This style guide defines the baseline visual and structural rules for building the frontend of this application. It is designed for use by AI coding assistants and developers. The goal is to produce consistent, modern, clean UI code without inventing new design systems or applying custom styles unless explicitly requested.

The generated UI should be:

- Easy to read and navigate
- Built with modern, accessible components
- Flexible — changes like “make this darker,” “make it wider,” or “add a column” should require only minimal Tailwind utility adjustments

## Component System

- Use `shadcn/ui` components for all UI primitives (e.g., Button, Input, Dialog)
- Only install needed components into `components/ui/`
- Do not use third-party component libraries (e.g., MUI, Ant, Chakra)
- Use `lucide-react` for all icons

## Styling System

- Use [Tailwind CSS] utility classes for all layout, color, and spacing
- Do not use raw CSS, CSS-in-JS, or inline styles
- All styling should be editable via Tailwind class names

## Layout

- Pages should use a responsive layout with a max-width container:
  - `max-w-7xl mx-auto px-4 sm:px-6 lg:px-8`
- Use `flex` or `grid` for layout structure
- Use Tailwind spacing utilities (`gap-*`, `space-*`, `px-*`, `py-*`) for all spacing
- Use consistent vertical spacing between page sections (`space-y-8` or similar)

## Colors

- Use Tailwind’s default color palette
- Do not define custom colors unless explicitly requested
- Example defaults:
  - Primary action: `orange-500` / `orange-600`
  - Secondary elements: `gray-100` through `gray-800`
  - Accent (optional): `blue-500`

## Typography

- Use Tailwind text utilities for font size and weight
- Recommended defaults:
  - Section titles: `text-2xl font-semibold`
  - Subheadings: `text-xl font-medium`
  - Body text: `text-base`
  - Muted/secondary: `text-sm text-muted-foreground`

## Reusability

- Place all reusable UI primitives in `components/ui/`
- Place domain-specific or application-specific components in `components/`
- Component files should be named clearly and match their component names (e.g., `UserCard.tsx`, `DashboardHeader.tsx`)
- Do not create new UI primitives if an existing shadcn/ui component can be used

## Interactivity

- Use loading indicators (e.g., skeletons, `isLoading` state on buttons)
- Use dialogs for confirmation or interruption flows
- Use toasts or alerts for user feedback after actions
- Use accessible, keyboard-navigable components at all times
- All feedback components (e.g., `Toast`, `Dialog`) should come from shadcn/ui

## Customization

- All components should be styled using Tailwind utility classes that can be easily modified
- Avoid hardcoded values that would make it difficult to adjust spacing, size, color, or layout
- Do not introduce new design systems, tokens, themes, or configuration files unless explicitly instructed

## Dark Mode

- Support Tailwind's `dark:` modifier for all components
- Do not build separate themes unless requested
- All components must remain legible and consistent in both light and dark mode

## Do Not

- Do not use inline styles
- Do not use global or scoped CSS files
- Do not import third-party UI libraries unless explicitly told to
- Do not define custom color schemes or spacing systems
- Do not over-design — keep things clean, minimal, and composable

## Accessibility Checklist

- All interactive elements are reachable via keyboard (Tab/Shift+Tab) and have a visible focus state.
- Use semantic HTML elements (button, nav, main, header, footer, ul/li, label + input) over generic divs.
- Every form control has an associated `<label>` or `aria-label`.
- Buttons are real `<button>` elements, not clickable divs.
- Provide `aria-live="polite"` for toast/alert regions.
- Maintain sufficient color contrast (WCAG AA). Prefer Tailwind defaults; avoid low-contrast combinations.
- Do not disable outlines globally; style focus using Tailwind (e.g., focus classes on shadcn components).

## Summary

Use modern defaults and Tailwind conventions. Keep everything predictable and easy to change. Only use the tools listed in this guide. Do not introduce complexity unless instructed. This style guide provides a shared foundation for consistent, maintainable frontend code across all projects.
