<script lang="ts">
	import './layout.css';
	import favicon from '$lib/assets/favicon.svg';
	import { onMount, type Snippet } from 'svelte';
    import { base } from '$app/paths';

	let { children }: { children: Snippet } = $props();
	let isDark = $state(false);
	let isMenuOpen = $state(false);
	let isPinned = $state(true);
	let isHovering = $state(false);

	onMount(() => {
		const savedTheme = localStorage.getItem('theme');
		// Default to light mode (only go dark if explicitly saved as dark)
		isDark = savedTheme === 'dark';

		const savedPin = localStorage.getItem('navbarPinned');
		isPinned = savedPin === null ? true : savedPin === 'true';
	});

	$effect(() => {
		if (isDark) {
			document.documentElement.classList.add('dark');
			localStorage.setItem('theme', 'dark');
		} else {
			document.documentElement.classList.remove('dark');
			localStorage.setItem('theme', 'light');
		}
	});

	function toggleTheme() {
		isDark = !isDark;
	}

	function togglePin() {
		isPinned = !isPinned;
		localStorage.setItem('navbarPinned', String(isPinned));
	}
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

<div class="min-h-screen bg-bg-primary text-text-main transition-colors duration-300">
	<!-- Trigger Zone for Auto-Show (only active when unpinned) -->
	{#if !isPinned}
		<!-- svelte-ignore a11y_no_static_element_interactions -->
		<div 
			class="fixed top-0 left-0 right-0 h-4 z-[101]"
			onmouseenter={() => isHovering = true}
		></div>
	{/if}

	<!-- svelte-ignore a11y_no_static_element_interactions -->
	<!-- svelte-ignore a11y_mouse_events_have_key_events -->
	<nav 
		class="fixed top-0 left-0 right-0 z-[100] h-16 px-4 sm:px-8 flex items-center justify-between border-b border-border-main bg-bg-card backdrop-blur-md transition-all duration-300 {isPinned || isHovering || isMenuOpen ? 'translate-y-0' : '-translate-y-full'}"
		onmouseenter={() => isHovering = true}
		onmouseleave={() => isHovering = false}
	>
		<!-- Logo / Brand -->
		<a href="/#/" class="flex items-center gap-2 font-bold text-lg hover:opacity-80 transition-opacity">
			<span class="text-2xl">🏒</span>
			<span>Uni Hockey</span>
		</a>

		<!-- Desktop Nav -->
		<div class="hidden sm:flex items-center gap-6 text-sm font-medium">
			<a href="{base}/#/MatchSelect" class="hover:text-text-accent transition-colors">Matches</a>
			<a href="{base}/#/Match" class="hover:text-text-accent transition-colors">Live Game</a>
			<a href="{base}/#/Settings" class="hover:text-text-accent transition-colors">Settings</a>
		</div>

		<!-- Actions -->
		<div class="flex items-center gap-4">
			<!-- Pin Toggle -->
			<button 
				onclick={togglePin}
				class="w-10 h-10 flex items-center justify-center rounded-full bg-bg-secondary hover:bg-bg-card-hover transition-colors cursor-pointer"
				aria-label={isPinned ? "Unpin Navbar" : "Pin Navbar"}
				title={isPinned ? "Unpin Navbar" : "Pin Navbar"}
			>
				{#if isPinned}
					<!-- Pinned (Show 'Hide' action) -->
					<svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="text-text-muted"><path d="M4 14.5L12 6.5l8 8"/><path d="M12 6.5V22"/></svg>
				{:else}
					<!-- Unpinned (Show 'Lock' indicator or generic State) -->
					<svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="text-text-muted"><path d="M4 9.5l8 8 8-8"/><path d="M12 17.5V2"/></svg>
				{/if}
			</button>

			<!-- Theme Toggle -->
			<button 
				onclick={toggleTheme}
				class="w-10 h-10 flex items-center justify-center rounded-full bg-bg-secondary hover:bg-bg-card-hover transition-colors cursor-pointer"
				aria-label="Toggle Dark Mode"
			>
				{#if isDark}
					<!-- Sun Icon -->
					<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="text-warning"><circle cx="12" cy="12" r="4"/><path d="M12 2v2"/><path d="M12 20v2"/><path d="m4.93 4.93 1.41 1.41"/><path d="m17.66 17.66 1.41 1.41"/><path d="M2 12h2"/><path d="M20 12h2"/><path d="m6.34 17.66-1.41 1.41"/><path d="m19.07 4.93-1.41 1.41"/></svg>
				{:else}
					<!-- Moon Icon -->
					<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="text-text-muted"><path d="M12 3a6 6 0 0 0 9 9 9 9 0 1 1-9-9Z"/></svg>
				{/if}
			</button>

			<!-- Mobile Menu Button -->
			<button class="sm:hidden p-2" onclick={() => isMenuOpen = !isMenuOpen}>
				<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="4" x2="20" y1="12" y2="12"/><line x1="4" x2="20" y1="6" y2="6"/><line x1="4" x2="20" y1="18" y2="18"/></svg>
			</button>
		</div>
	</nav>

	<!-- Mobile Menu -->
	{#if isMenuOpen}
		<div class="fixed inset-0 z-50 bg-bg-primary sm:hidden flex flex-col pt-20 px-8 gap-6 text-lg font-medium">
			<button class="absolute top-5 right-5 p-2" onclick={() => isMenuOpen = false}>✕</button>
			<a href="/#/MatchSelect" onclick={() => isMenuOpen = false}>Matches</a>
			<a href="/#/Match" onclick={() => isMenuOpen = false}>Live Game</a>
			<a href="/#/Settings" onclick={() => isMenuOpen = false}>Settings</a>
		</div>
	{/if}

	<main class="transition-[padding] duration-300 {isPinned ? 'pt-16' : 'pt-0'}">
		{@render children()}
	</main>
</div>

