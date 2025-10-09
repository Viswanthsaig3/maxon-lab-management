# Lab Management App: Migration Analysis for Next.js

This document provides a detailed functional analysis of the existing Create React App (CRA) application to guide its migration to Next.js.

## Table of Contents
1.  [Core Architecture & Concepts](#core-architecture--concepts)
2.  [Firebase Backend Configuration](#firebase-backend-configuration)
3.  [Global State & Contexts](#global-state--contexts)
4.  [Page-by-Page Functional Analysis](#page-by-page-functional-analysis)
    *   [Authentication (Login/Signup)](#authentication-loginsignup)
    *   [Dashboards (Admin, Employee)](#dashboards-admin-employee)
    *   [Task Management](#task-management)
    *   [CRM (Customer Relationship Management)](#crm-customer-relationship-management)
    *   [Capacity & Project Planning](#capacity--project-planning)
    *   [Inventory Management](#inventory-management)
    *   [Payroll & Attendance](#payroll--attendance)
    *   [RBAC (Role Management)](#rbac-role-management)
5.  [Migration Strategy Summary](#migration-strategy-summary)

---

## 1. Core Architecture & Concepts

The application is a comprehensive management system built on the MERN stack (React with TypeScript, Firebase).

*   **Frontend:** Create React App, TypeScript, Material-UI.
*   **Backend:** Firebase (Authentication, Firestore, Storage).
*   **Routing:** `react-router-dom` for client-side routing.
*   **State Management:** React Context API for global state (`AuthContext`, `RBACContext`, `NotificationContext`) and component-level state (`useState`, `useEffect`).
*   **Styling:** A mix of Material-UI's `styled` components, a custom theme, and CSS files.

### Next.js Migration Implications
*   The client-side routing will be replaced by Next.js's file-system-based App Router.
*   Data fetching logic will be moved from client-side `useEffect` hooks to Server Components, `getServerSideProps`, or client-side fetching libraries like SWR/TanStack Query for better performance and SEO.
*   The existing components can be largely reused, but they will need to be adapted to the Server/Client Component model.

---

## 2. Firebase Backend Configuration

### `firebase.json`
*   **`firestore`**: Points to `firestore.rules` and `firestore.indexes.json`. This configuration is critical and will be reused in the Next.js app. The security rules are the "backend" logic that protects data.
*   **`hosting`**: Configured for a Single Page Application (SPA). The `rewrites` rule directs all traffic to `index.html`, which is typical for a CRA.
    *   **Next.js Migration:** This section will be replaced by Next.js's own deployment configuration (e.g., for Vercel). The SPA rewrite is no longer needed as Next.js handles routing on the server.

### `firestore.rules` & `storage.rules`
*   These files define the security model. They control who can read, write, and delete data in Firestore and files in Firebase Storage.
*   The rules are granular, often checking `request.auth.uid` to ensure users can only access their own data, or checking a user's role from their document in the `users` collection.
*   **Next.js Migration:** These rules remain **critically important**. They are the primary line of defense for the database. The Next.js application, whether rendering on the server or client, will still be subject to these rules. No changes are needed here unless the data model is altered.

---

## 3. Global State & Contexts

The application relies heavily on three main contexts located in `src/contexts/`.

### `AuthContext.tsx`
*   **Purpose:** Manages the global authentication state. It uses Firebase's `onAuthStateChanged` listener to determine if a user is logged in.
*   **Functionality:**
    *   Holds `currentUser` (the Firebase User object) and `userProfile` (the corresponding user data from the Firestore `users` collection).
    *   Provides `login`, `logout`, and `signup` functions which wrap the `authService` calls.
    *   A `loading` state prevents rendering of the app until the auth state is resolved.
*   **Next.js Migration:**
    *   Authentication in Next.js needs to be handled on both the server and the client.
    *   The `AuthProvider` will need to be a Client Component (`'use client'`).
    *   To protect server-rendered pages, you'll need to check for an auth cookie or token in middleware (`middleware.ts`) or in Server Components. Libraries like `next-auth` or custom solutions with cookies are common.
    *   The core logic of `AuthContext` can be preserved for managing client-side state.

### `RBACContext.tsx`
*   **Purpose:** Manages Role-Based Access Control. This is the heart of the application's permission system.
*   **Functionality:**
    *   Fetches the current user's roles and aggregates all permissions associated with those roles from the `roles` collection in Firestore.
    *   Provides helper functions like `hasPermission`, `isAdmin`, etc., which are used by `ProtectedRoute.tsx` and throughout the app to conditionally render UI elements.
    *   Uses `subscribeToUserPermissions` for real-time updates to permissions.
*   **Next.js Migration:**
    *   This context will also need to be a Client Component.
    *   The logic for fetching permissions can remain the same.
    *   Protecting server-rendered pages/components will require fetching the user's role on the server to decide whether to render the page or redirect. This can be done in `layout.tsx` or page components.

### `NotificationContext.tsx`
*   **Purpose:** Provides real-time notifications to the user.
*   **Functionality:**
    *   Listens to the `notifications` collection in Firestore for documents where the `userId` matches the current user.
    *   Provides a list of notifications and a function to `markAsRead`.
*   **Next.js Migration:**
    *   This is a purely client-side feature. The context will be a Client Component and can be used as-is.

---

## 4. Page-by-Page Functional Analysis

### Authentication (`/login`, `/signup`)

*   **Files:** `pages/LoginPage.tsx`, `pages/SignupPage.tsx`
*   **Purpose:** User authentication.
*   **Backend Interaction:**
    *   `login`: Calls `signInWithEmailAndPassword` via `authService`.
    *   `signup`: Calls `createUserWithEmailAndPassword` and then creates a new document in the `users` collection in Firestore with the user's name and role.
*   **Cross-Page Impact:** Successful authentication updates the `AuthContext`, which grants access to the rest of the application via `PrivateRoute.tsx`.
*   **Next.js Migration:**
    *   These will become pages in the `app/(auth)` route group (e.g., `app/(auth)/login/page.tsx`).
    *   The forms will be Client Components. The submission logic can remain the same, but you might consider using Next.js Server Actions for form handling.

### Dashboards (`/dashboard`)

*   **Files:** `pages/AdminDashboard.tsx`, `pages/EmployeeDashboard.tsx`, `pages/TeamLeadDashboard.tsx`
*   **Purpose:** Provide a role-specific overview of the system.
*   **Backend Interaction:**
    *   **Admin:** Fetches all users (`getUsers`) and all tasks (`getTasks`) to compute aggregate statistics.
    *   **Employee:** Fetches tasks assigned to the current user (`getTasksForUser`).
*   **Cross-Page Impact:** These pages are read-only views that reflect the state of data from other modules (Tasks, Users, etc.).
*   **Next.js Migration:**
    *   These are excellent candidates for Server Components. Data can be fetched on the server before rendering, improving initial load time.
    *   The route could be `app/(dashboard)/dashboard/page.tsx`, with logic to determine which dashboard to render based on the user's role fetched on the server.

### Task Management (`/task-management`, `/my-tasks`)

*   **Files:** `pages/TaskManagementPage.tsx`, `pages/MyTasksPage.tsx`, `components/tasks/*`
*   **Purpose:** Core module for creating, assigning, and tracking tasks.
*   **Backend Interaction:**
    *   `TaskManagementPage` uses a real-time listener (`getTasksRealTime`) to display all tasks. Admins/Team Leads can create new tasks (`createTask`).
    *   `MyTasksPage` uses a real-time listener (`getTasksForUser`) for tasks assigned to the logged-in user.
    *   Status updates (`updateTaskStatus`) trigger notifications (`createNotification`).
*   **Cross-Page Impact:** A status update on `MyTasksPage` is instantly reflected on `TaskManagementPage` and can affect the stats on the `AdminDashboard`.
*   **Next.js Migration:**
    *   The main list/grid of tasks can be server-rendered for a fast initial load.
    *   Real-time updates can be layered on top on the client-side.
    *   The forms for creating/editing tasks will be Client Components.

### CRM (Customer Relationship Management) (`/crm`)

*   **Files:** `pages/CRMPage.tsx`, `components/crm/*`
*   **Purpose:** Manage clients, quotations, invoices, and business expenditures.
*   **Backend Interaction:**
    *   Heavy reliance on `clientService.ts` for all CRUD operations on clients, quotations, invoices, and expenditures.
    *   Integrates with `inventoryService.ts` when creating invoices to check stock.
    *   Uses `pdfGenerator.ts` (a client-side utility) to create PDF documents.
*   **Cross-Page Impact:** Creating an invoice from a quotation updates the quotation's status. Inventory used in an invoice will affect stock levels on the `InventoryPage`.
*   **Next.js Migration:**
    *   The main tables can be server-rendered.
    *   The complex forms (`ClientForm`, `InvoiceForm`, etc.) will be Client Components.
    *   PDF generation could be moved to a Next.js API route (`app/api/generate-pdf/route.ts`) to handle it on the server, which can be more robust.

### Capacity & Project Planning (`/project-planning`, `/capacity-management`)

*   **Files:** `pages/CapacityManagementPage.tsx`, `pages/ProjectPlanningPageNew.tsx`, `components/projects/*`
*   **Purpose:** A powerful planning tool. `CapacityManagementPage` is the "settings" area where admins define what resources (staff, inventory) are needed for different "Services". `ProjectPlanningPage` uses these definitions to automatically calculate the total resources required for a new project.
*   **Backend Interaction:**
    *   `capacityService.ts` is central here, managing `ServiceConfig` and `EmployeeType` documents.
    *   The `calculateProjectRequirements` function is a key piece of business logic. It fetches service configs, employee types, and inventory items to perform its calculation.
*   **Cross-Page Impact:** This is a highly interconnected module. Changes in `CapacityManagementPage` directly impact the output of the calculator in `ProjectPlanningPage`. A published project plan would then lead to the creation of tasks in the Task Management module.
*   **Next.js Migration:**
    *   The `calculateProjectRequirements` logic is a good candidate for an API route or a Server Action. This keeps the complex business logic on the server.
    *   The planning form would be a Client Component that calls this API/Action.
    *   The results page can be dynamically rendered on the server.

### Inventory Management (`/inventory`)

*   **Files:** `pages/InventoryPage.tsx`, `components/inventory/*`
*   **Purpose:** Track all physical and service-based items in the lab.
*   **Backend Interaction:**
    *   Uses `inventoryService.ts` for all CRUD operations.
    *   `recordStockMovement` uses a Firestore Transaction to ensure atomic updates to inventory quantities, which is crucial for data integrity.
*   **Cross-Page Impact:** Inventory levels are depleted when invoices are created in the CRM or when projects in the Project Planning module are executed.
*   **Next.js Migration:**
    *   The main inventory list can be server-rendered.
    *   Forms for adding/editing items and recording stock movements will be Client Components.

### Payroll & Attendance (`/payroll`, `/attendance`)

*   **Files:** `pages/PayrollPage.tsx`, `pages/AttendancePage.tsx`
*   **Purpose:** Manage employee attendance, leave, and generate payroll.
*   **Backend Interaction:**
    *   `AttendancePage` uses the browser's Geolocation and Camera APIs. It uploads photos to Firebase Storage (`uploadPhoto`) and records attendance data in Firestore (`checkIn`, `checkOut`).
    *   `PayrollPage` fetches attendance and leave data to calculate salaries (`calculateMonthlySalary`) and generate payslips (`PayslipGenerator`).
*   **Cross-Page Impact:** An employee's attendance record directly impacts their calculated salary on the `PayrollPage`.
*   **Next.js Migration:**
    *   `AttendancePage` must be a Client Component due to its use of browser-specific APIs.
    *   The payroll calculation logic in `PayslipGenerator` could be moved to a secure API route to protect sensitive salary information.

### RBAC (`/role-management`)

*   **Files:** `pages/RoleManagement.tsx`, `components/rbac/*`
*   **Purpose:** Admin interface for managing the Role-Based Access Control system.
*   **Backend Interaction:**
    *   Uses `rbacService.ts` to perform CRUD operations on the `roles` collection in Firestore.
*   **Cross-Page Impact:** Changes here have immediate, system-wide implications, as the `RBACContext` will fetch the new permissions and enforce them across the entire application.
*   **Next.js Migration:**
    *   This page can be server-rendered for admins. The forms for editing roles will be Client Components. The core RBAC logic remains the same.

---

## 5. Migration Strategy Summary

1.  **Setup Next.js Project:** Initialize a new Next.js project with the App Router.
2.  **Migrate Core Components & Styles:**
    *   Copy over the `components`, `types`, `utils`, and `theme.ts` files.
    *   Mark all components that use hooks (`useState`, `useEffect`, etc.) or event handlers as Client Components (`'use client'`).
    *   Set up the Material-UI `ThemeProvider` in the root `layout.tsx`.
3.  **Implement Authentication:**
    *   Create a new `app/(auth)` route group for login and signup pages.
    *   Implement a server-side authentication check (e.g., using cookies and middleware) to protect routes.
    *   Wrap the root layout with the `AuthProvider` and other global contexts.
4.  **Migrate Pages (Route by Route):**
    *   Start with the simplest pages (e.g., dashboards). Create corresponding routes in the `app` directory.
    *   Fetch data in Server Components where possible for initial page load.
    *   For pages requiring real-time data or heavy user interaction (like `TaskManagementPage`), fetch initial data on the server and then use client-side hooks to subscribe to real-time updates.
    *   Move complex, sensitive, or heavy business logic (e.g., `calculateProjectRequirements`, `calculateMonthlySalary`) into Next.js API Routes or Server Actions.
5.  **Refactor Services:** The `services` files can be largely reused. They will be called from both Server Components (for initial data) and Client Components (for mutations and real-time updates).
6.  **Testing:** Thoroughly test each migrated page, paying close attention to authentication, authorization (RBAC), and real-time data synchronization.

By following this detailed analysis, the migration from CRA to Next.js can be done systematically, leveraging the strengths of Next.js for improved performance, SEO, and developer experience while preserving the core business logic and real-time capabilities of the original application.
