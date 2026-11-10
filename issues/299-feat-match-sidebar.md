# Issue #299: feat: Match the sidebar to the approved Five Panel design

Source issue: https://github.com/five-panel/five-panel/issues/299

## TL;DR

Match the sidebar to the approved design on desktop, tablet, and mobile. Use a white background in the light theme, with a thin gray divider and a soft gray selected row. This replaces the cream background in the original issue; the user approved white during discussion. The main content already has a white background and stays unchanged.

Approved canvas views: [desktop](https://five-pane-dev-server.tail7d89bf.ts.net/c/J88FjcHO5C?frame=PH61td720u), [mobile](https://five-pane-dev-server.tail7d89bf.ts.net/c/J88FjcHO5C?frame=8eRu3salR8), and [tablet](https://five-pane-dev-server.tail7d89bf.ts.net/c/J88FjcHO5C?frame=CjQ-cWEXaX).

## What will change

**Code impact: Medium.** The work stays in the frontend, but covers three screen sizes, desktop collapse, and existing keyboard and permission behavior.

- Add the new sidebar presentation: company header, top controls, grouped pages, quiet footer, and desktop collapse state. Use Inter within the sidebar, matching the approved spacing and icons.
- Make focused changes to `frontend/src/layouts/MainLayout.vue` for sizing, responsive behavior, the header, and footer. Update `frontend/src/components/SidebarNavigation.vue` for group styling, folding, and page selection.
- Reuse `authStore`, `userViewsStore`, `ViewService`, and `frontend/src/services/sidebarNavigationTree.ts`. Keep the current placement-based tree, ordering, destinations, and permissions. Reuse the existing theme system and keep dark mode readable.
- Search, notifications, AI/automations, trash, and company switching appear disabled, with a clear indication that they are coming later. Show the current company name and icon. No new backend services are needed.

## How it would work

1. **Existing code we reuse:** load the current company's sections and saved pages through the current navigation services. Keep labels and routes from that data; the sample company names and pages in the canvas are examples.
2. **New presentation:** show the company header, top controls, expandable groups, indented links, and footer. Selecting a page uses its existing destination and active state. Long navigation lists scroll while the company header and footer remain reachable.
3. **Existing actions we keep:** “Añadir página” opens the current add-existing-saved-view flow; “Nuevo espacio” opens section creation. Keep move and remove actions and their current superuser restriction. Settings retains its existing settings destination, theme choices, sign-out, and sign-out-all actions. Disabled future controls do not navigate or perform an action; the client-portal example does not create a new destination.
4. **New responsive sizing:** below 768 px, a menu button opens the sidebar at 100% of the screen width. From 768 to 1023 px, it opens as a 320 px panel over a dimmed page; the table keeps its original width underneath. From 1024 px, show the fixed 220 px desktop sidebar. Its collapse control hides the page tree in a narrow strip with a clear expand button.
5. **Existing behavior we extend:** the phone and tablet menus close through the close button, Escape, or successful navigation; tablet also closes through the backdrop. Keep keyboard focus inside the open menu, make the page behind it inactive, and return focus to the menu button when closed. Keep loading, empty, error, and retry states. Use touch targets of at least 44 px in the mobile and tablet menus.

## Done when

- Desktop, tablet, and phone screenshots match the approved white designs, including company identity, grouping, selection, footer, and the 220 px desktop width. Dark mode remains readable.
- At 390 px and 320 px phone widths, the menu fills the width without horizontal overflow. At 768 px, the tablet panel is 320 px wide and opening it does not shrink the table.
- Desktop collapse and expand work. Long menus scroll without hiding footer actions, and groups fold and reopen.
- Real company pages keep their placements, order, routes, and active state. Existing add, move, remove, settings, theme, and sign-out actions retain their permissions and behavior.
- Phone and tablet opening, closing, navigation, Escape, focus containment, and focus return work. Loading, empty, error, and retry states remain usable.
- All five future controls are visibly disabled and perform no action. Focused navigation, layout accessibility, and theme checks pass, along with the frontend build; content screens remain unchanged.

## Out of scope

- Building search, notifications, AI/automations, trash, company switching, or a client portal. Their real behavior belongs to later work.
- Changing tables or other content screens, global fonts, or the application's overall theme.
- Changing backend data, APIs, navigation placements, or permissions; hard-coding the sample company's menu.
- Adding new page creation flows, drag-and-drop ordering, or other navigation-management features.
