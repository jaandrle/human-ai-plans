# Migration from React Router to TanStack Router

This guide documents the process of migrating from React Router to TanStack Router in this project.

See the [TanStack Router](https://tanstack.com/router) documentation for more information.

## Oficial Guide
See [Migration from React Router Checklist | TanStack Router React Docs]
(https://tanstack.com/router/latest/docs/framework/react/installation/migrate-from-react-router).

**_If your UI is blank, open the console, and you will probably have some errors that read something along the lines of
`cannot use 'useNavigate' outside of context` . This means there are React Router api’s that are still imported and
referenced that you need to find and remove. The easiest way to make sure you find all React Router imports is to
uninstall `react-router-dom` and then you should get typescript errors in your files. Then you will know what to change
to a `@tanstack/react-router` import._**

Here is the [example repo]
(https://github.com/Benanna2019/SickFitsForEveryone/tree/migrate-to-tanstack/router/React-Router)

- [x] Install Router - `npm i @tanstack/react-router` (see [detailed installation guide]
(https://tanstack.com/router/latest/docs/framework/react/how-to/install.md))
- [x] **Optional:** Uninstall React Router to get TypeScript errors on imports.
	- At this point I don’t know if you can do a gradual migration, but it seems likely you could have multiple router
		providers, not desirable.
	- The api’s between React Router and TanStack Router are very similar and could most likely be handled in a sprint
		cycle or two if that is your companies way of doing things.
- [x] Create Routes for each existing React Router route we have
- [x] Create root route
- [x] Create router instance
- [x] Add global module in main.tsx
- [x] Remove any React Router (`createBrowserRouter` or `BrowserRouter`), `Routes`, and `Route` Components from main.tsx
- [x] **Optional:** Refactor `render` function for custom setup/providers - The repo referenced above has an example -
	This was necessary in the case of Supertokens. Supertoken has a specific setup with React Router and a different
	setup with all other React implementations
- [x] Set RouterProvider and pass it the router as the prop
- [x] Replace all instances of React Router `Link` component with `@tanstack/react-router` `Link` component
	- [x] Add `to` prop with literal path
	- [x] Add `params` prop, where necessary with params like so `params={{ orderId: order.id }}`
- [x] Replace all instances of React Router `useNavigate` hook with `@tanstack/react-router` `useNavigate` hook
	- [x] Set `to` property and `params` property where needed
- [x] Replace any React Router `Outlet`'s with the `@tanstack/react-router` equivalent
- [x] If you are using `useSearchParams` hook from React Router, move the search params default value to the
	validateSearch property on a Route definition.
	- [x] Instead of using the `useSearchParams` hook, use `@tanstack/react-router` `Link`'s search property to update the
		search params state
	- [x] To read search params you can do something like the following
		- `const { page } = useSearch({ from: productPage.fullPath })`
- [x] If using React Router’s `useParams` hook, update the import to be from `@tanstack/react-router` and set the `from`
	property to the literal path name where you want to read the params object from
	- So say we have a route with the path name `orders/$orderid`.
	- In the `useParams` hook we would set up our hook like so: `const params = useParams({ from: "/orders/$orderId" })`
	- Then wherever we wanted to access the order id we would get it off of the params object `params.orderId`

## Concrete Steps
With *gemini-cli* help.

### Dependency Changes

The first step was to update the dependencies in `package.json`.

#### Removed Dependencies

- `react-router-dom`

#### Added Dependencies

```terminal
npm install @tanstack/router-plugin --save-dev
npm install @tanstack/react-router @tanstack/react-router-devtools --save
```
…(optional) refact dependencies `~major.minor` to allow only patch updates.

### Configuration Changes

Next, the configuration files were updated to integrate TanStack Router.

#### `vite.config.ts`

The `@tanstack/router-plugin/vite` was added to the `plugins` array in `vite.config.ts`. This plugin is responsible for
generating the route tree based on the file structure. **Must be added before `react()`.**
_(Optional) use `app-*` files for routing and others for colocating logic, see [Frontent pages (URL
endpoints)](../src/app/README.md)._

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tsconfigPaths from "vite-tsconfig-paths";
import { tanstackRouter } from "@tanstack/router-plugin/vite";

export default defineConfig({
	// ...
	plugins: [
		tanstackRouter({
			target: "react",
			autoCodeSplitting: true,
			routesDirectory: "./src/app",
			routeFilePrefix: "app-",
			routeFileIgnorePrefix: "",
			generatedRouteTree: "./routeTree.gen.ts",
			quoteStyle: "double",
			semicolons: true,
		}),
		react(),
		tsconfigPaths(),
		// ...
	],
});
```

#### `tsconfig.node.json` (may not need)

Needed to properly find types for build time Vite.
The `moduleResolution` in `tsconfig.node.json` was changed from `Node` to `nodenext`.

```json
{
	"compilerOptions": {
		"composite": true,
		"module": "ESNext",
		"moduleResolution": "nodenext",
		"allowSyntheticDefaultImports": true,
		"forceConsistentCasingInFileNames": true,
		"strictBindCallApply": true,
		"verbatimModuleSyntax": true
	},
	"include": ["vite.config.ts", "bs"]
}
```

### Application Entry Point

The application entry point, `src/index.tsx`, was updated to use the new router.

```typescript
import { createRoot } from "react-dom/client";
import { StrictMode } from "react";
import { ToastContainer } from "react-toastify";
import "react-toastify/dist/ReactToastify.css";
import { GlobalStyle } from "./ui/globals";
import { RouterProvider, createRouter, createHashHistory } from "@tanstack/react-router";
import { routeTree } from "./routeTree.gen";

const history = createHashHistory();
const router = createRouter({
	routeTree, history,
	defaultPreload: "intent",
	defaultViewTransition: true,
});

declare module "@tanstack/react-router" {
	interface Register {
		router: typeof router;
	}
}
createRoot(document.getElementById("root") as HTMLElement).render(
	<StrictMode>
		<GlobalStyle />
		<RouterProvider router={router} />
		<ToastContainer
			position="bottom-center"
			autoClose={5000}
			hideProgressBar={false}
			newestOnTop={false}
			closeOnClick
			pauseOnFocusLoss
			draggable
			pauseOnHover
		/>
	</StrictMode>,
);
```

### Route Migration

With TanStack Router, routes are defined as files in the `src/app` directory. Each route is a file that exports a
`Route` component created with `createFileRoute`.

#### Root Route

The root route is defined in `src/app/app-__root.tsx`. It uses `createRootRoute` and defines the root layout of the
application, including the `Outlet` for nested routes and the `TanStackRouterDevtools`.

```typescript
import { createRootRoute, Outlet, Link } from "@tanstack/react-router";
// ...

export const Route = createRootRoute({
	component: Root,
	notFoundComponent: () => (
		<>
			<h1>Not Found</h1>
			<Link to="/">GO HOME</Link>
		</>
	),
});

function Root() {
	// ...
	return (
		<>
			<Outlet />
			{import.meta.env.DEV && <TanStackRouterDevtools />}
		</>
	);
}
```

#### Index Route

The index route is defined in `src/app/app-index.tsx`. It corresponds to the `/` path.

```typescript
import { createFileRoute, useNavigate } from "@tanstack/react-router";
// ...

export const Route = createFileRoute("/")({
	component: Page,
});

export function Page() {
	// ...
}
```

#### Dynamic Routes

Dynamic routes are created by using the `$` prefix in the filename. For example, the route for `/dashboard/$ip` is
defined in `src/app/app-dashboard/app-$ip.tsx`.

The `useParams` hook can be used to access the dynamic parameters from the URL.

```typescript
import { createFileRoute } from "@tanstack/react-router";
// ...

export const Route = createFileRoute("/dashboard/$ip")({
	component: Page,
});

export function Page() {
	const { ip } = Route.useParams();
	// ...
}
```

#### Navigation

The `useNavigate` hook provides a function to navigate to a different route.

```typescript
import { useNavigate } from "@tanstack/react-router";

// ...
const navigate = useNavigate();
navigate({ to: "/dashboard/$ip", params: { ip } });
// ...
```

### Component Changes

#### `Link` Component

The `Link` component is now imported from `@tanstack/react-router` instead of `react-router-dom`.

#### `useOutletContext`

The `useOutletContext` hook from `react-router-dom` is no longer needed. Data can be passed to nested routes using
TanStack Router's loader functions and context.

### Generated Route Tree

The `routeTree.gen.ts` file is automatically generated by the TanStack Router plugin. It contains the route tree based
on the file structure in the `src/app` directory. This file should not be edited manually.
