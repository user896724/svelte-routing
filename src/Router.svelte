<script>
import { getContext, setContext, onMount } from "svelte";
import { writable, derived } from "svelte/store";
import { LOCATION, ROUTER } from "./contexts.js";
import { globalHistory } from "./history.js";
import { pick, match, stripSlashes, combinePaths } from "./utils.js";

export let basepath = "/";
export let url = null;

let locationContext = getContext(LOCATION);
let routerContext = getContext(ROUTER);

let routes = writable([]);
let activeRoute = writable(null);
let hasActiveRoute = false; // Used in SSR to synchronously set that a Route is active.

// If locationContext is not set, this is the topmost Router in the tree.
// If the `url` prop is given we force the location to it.
let location =
	locationContext ||
	writable(url ? { pathname: url } : globalHistory.location);

// If routerContext is set, the routerBase of the parent Router
// will be the base for this Router's descendants.
// If routerContext is not set, the path and resolved uri will both
// have the value of the basepath prop.
let base = routerContext
	? routerContext.routerBase
	: writable({
			path: basepath,
			uri: basepath
		});

let routerBase = derived([base, activeRoute], ([base, activeRoute]) => {
	// If there is no activeRoute, the routerBase will be identical to the base.
	if (activeRoute === null) {
		return base;
	}

	let { path: basepath } = base;
	let { route, uri } = activeRoute;
	// Remove the potential /* or /*splatname from
	// the end of the child Routes relative paths.
	let path = route.default ? basepath : route.path.replace(/\*.*$/, "");

	return { path, uri };
});

function registerRoute(route) {
	let { path: basepath } = $base;
	let { path } = route;

	// We store the original path in the _path property so we can reuse
	// it when the basepath changes. The only thing that matters is that
	// the route reference is intact, so mutation is fine.
	route._path = path;
	route.path = combinePaths(basepath, path);

	if (typeof window === "undefined") {
		// In SSR we should set the activeRoute immediately if it is a match.
		// If there are more Routes being registered after a match is found,
		// we just skip them.
		if (hasActiveRoute) {
			return;
		}

		let matchingRoute = match(route, $location.pathname);
		if (matchingRoute) {
			activeRoute.set(matchingRoute);
			hasActiveRoute = true;
		}
	} else {
		routes.update(rs => {
			rs.push(route);
			return rs;
		});
	}
}

function unregisterRoute(route) {
	routes.update(rs => {
		let index = rs.indexOf(route);
		rs.splice(index, 1);
		return rs;
	});
}

// This reactive statement will update all the Routes' path when
// the basepath changes.
$: {
	let { path: basepath } = $base;
	routes.update(rs => {
		rs.forEach(r => (r.path = combinePaths(basepath, r._path)));
		return rs;
	});
}
// This reactive statement will be run when the Router is created
// when there are no Routes and then again the following tick, so it
// will not find an active Route in SSR and in the browser it will only
// pick an active Route after all Routes have been registered.
$: {
	let bestMatch = pick($routes, $location.pathname);
	activeRoute.set(bestMatch);
}

if (!locationContext) {
	// The topmost Router in the tree is responsible for updating
	// the location store and supplying it through context.
	onMount(() => {
		let unlisten = globalHistory.listen(history => {
			location.set(history.location);
		});

		return unlisten;
	});

	setContext(LOCATION, location);
}

setContext(ROUTER, {
	activeRoute,
	base,
	routerBase,
	registerRoute,
	unregisterRoute
});
</script>

<slot></slot>
