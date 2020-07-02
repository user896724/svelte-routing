<script>
import { getContext, createEventDispatcher } from "svelte";
import { ROUTER, LOCATION } from "./contexts.js";
import { navigate } from "./history.js";
import { startsWith, resolve, shouldNavigate } from "./utils.js";

export let to = "#";
export let replace = false;
export let state = {};
export let component = null;

let { base } = getContext(ROUTER);
let location = getContext(LOCATION);
let dispatch = createEventDispatcher();

let href, isPartiallyCurrent, isCurrent, props;

$: href = to === "/" ? $base.uri : resolve(to, $base.uri);
$: isPartiallyCurrent = startsWith($location.pathname, href);
$: isCurrent = href === $location.pathname;
$: ariaCurrent = isCurrent ? "page" : undefined;

$: props = {
	location: $location,
	href,
	isPartiallyCurrent,
	isCurrent,
};

function onClick(event) {
	dispatch("click", event);

	if (shouldNavigate(event)) {
		event.preventDefault();
		// Don't push another entry to the history stack when the user
		// clicks on a Link to the page they are currently on.
		let shouldReplace = $location.pathname === href || replace;
		navigate(href, { state, replace: shouldReplace });
	}
}
</script>

{#if component}
	<svelte:component
		this={component}
		{...props}
		{...$$restProps}
		on:click={onClick}
	>
		<slot/>
	</svelte:component>
{:else}
	<a aria-current={ariaCurrent} on:click={onClick} {...props}>
		<slot></slot>
	</a>
{/if}

