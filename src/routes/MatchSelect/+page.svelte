
<script lang="ts">
    import { Saisonmanager as SM } from "floorball-saisonmanager";
    import { onMount, tick } from "svelte";
    import { goto } from "$app/navigation";


    const API_BASE = "https://saisonmanager.de/api/v2";
    const STORAGE_KEY = "saisonmanager_selection";

    // State using Svelte 5 runes
    let gameOperations = $state<SM.GameOperation[]>([]);
    let openGameOperationId = $state<number | null>(null);
    let openLeagueId = $state<number | null>(null);
    let openGameDay = $state<number | null>(null);

    let leagues = $state<Record<number, SM.League[]>>({});
    let schedules = $state<Record<number, SM.Game[]>>({});
    let isLoadingInit = $state(true);

    // Derived: Group games by game_day
    function getGamesByDay(leagueId: number): Map<number, SM.Game[]> {
        const games = schedules[leagueId] || [];
        const grouped = new Map<number, SM.Game[]>();
        for (const game of games) {
            const day = game.game_day ?? 0;
            if (!grouped.has(day)) {
                grouped.set(day, []);
            }
            grouped.get(day)!.push(game);
        }
        return grouped;
    }

    // Find the first game day with an unplayed match
    function findFirstUnplayedGameDay(leagueId: number): number | null {
        const gamesByDay = getGamesByDay(leagueId);
        for (const [day, games] of gamesByDay) {
            if (games.some(g => !g.result)) {
                return day;
            }
        }
        return null;
    }

    // Persist selection to localStorage
    function saveSelection() {
        const selection = {
            gameOperationId: openGameOperationId,
            leagueId: openLeagueId,
            gameDay: openGameDay
        };
        localStorage.setItem(STORAGE_KEY, JSON.stringify(selection));
    }

    // Load selection from localStorage
    function loadSelection() {
        try {
            const stored = localStorage.getItem(STORAGE_KEY);
            if (stored) {
                return JSON.parse(stored);
            }
        } catch (e) {
            console.error("Failed to load selection", e);
        }
        return null;
    }

    // Fetch initial data
    async function loadInit() {
        isLoadingInit = true;
        try {
            const res = await fetch(`${API_BASE}/init.json`);
            const data = await res.json();
            gameOperations = data.game_operations || [];
        } catch (e) {
            console.error("Failed to load init.json", e);
        } finally {
            isLoadingInit = false;
        }
    }

    async function loadLeagues(gameOperationId: number) {
        if (leagues[gameOperationId]) return;
        try {
            const res = await fetch(`${API_BASE}/game_operations/${gameOperationId}/leagues.json`);
            const data = await res.json();
            leagues = { ...leagues, [gameOperationId]: data };
        } catch (e) {
            console.error(`Failed to load leagues for ${gameOperationId}`, e);
        }
    }

    async function loadSchedule(leagueId: number) {
        if (schedules[leagueId]) return;
        try {
            const res = await fetch(`${API_BASE}/leagues/${leagueId}/schedule.json`);
            const data = await res.json();
            schedules = { ...schedules, [leagueId]: data };
            
            // Auto-open first unplayed game day
            await tick();
            const firstUnplayed = findFirstUnplayedGameDay(leagueId);
            if (firstUnplayed !== null) {
                openGameDay = firstUnplayed;
                // Scroll to the game day after a short delay
                setTimeout(() => {
                    const element = document.getElementById(`gameday-${leagueId}-${firstUnplayed}`);
                    element?.scrollIntoView({ behavior: "smooth", block: "center" });
                }, 100);
            }
        } catch (e) {
            console.error(`Failed to load schedule for ${leagueId}`, e);
        }
    }

    onMount(async () => {
        await loadInit();
        
        // Restore previous selection
        const selection = loadSelection();
        if (selection) {
            if (selection.gameOperationId) {
                openGameOperationId = selection.gameOperationId;
                await loadLeagues(selection.gameOperationId);
                
                if (selection.leagueId) {
                    openLeagueId = selection.leagueId;
                    await loadSchedule(selection.leagueId);
                    
                    if (selection.gameDay !== null) {
                        openGameDay = selection.gameDay;
                    }
                }
            }
        }
    });

    function handleGOToggle(id: number, isOpen: boolean) {
        if (isOpen) {
            openGameOperationId = id;
            openLeagueId = null;
            openGameDay = null;
            loadLeagues(id);
        } else if (openGameOperationId === id) {
            openGameOperationId = null;
        }
        saveSelection();
    }

    function handleLeagueToggle(id: number, isOpen: boolean) {
        if (isOpen) {
            openLeagueId = id;
            openGameDay = null;
            loadSchedule(id);
        } else if (openLeagueId === id) {
            openLeagueId = null;
        }
        saveSelection();
    }

    function handleGameDayToggle(day: number, isOpen: boolean) {
        if (isOpen) {
            openGameDay = day;
        } else if (openGameDay === day) {
            openGameDay = null;
        }
        saveSelection();
    }

    function selectGame(game: SM.Game, gameIndex: number) {
        const gameSelection = {
            gameOperationId: openGameOperationId,
            leagueId: openLeagueId,
            gameDay: openGameDay,
            gameIndex: gameIndex
        };
        localStorage.setItem(STORAGE_KEY + "_game", JSON.stringify(gameSelection));
    }
</script>

<div class="w-full min-h-screen bg-primary bg-[radial-gradient(ellipse_at_top,_var(--tw-gradient-stops))] from-gradient-from via-gradient-via to-gradient-to transition-colors duration-300 p-4 md:p-8">
    <div class="max-w-4xl mx-auto space-y-6">
        <header class="mb-8 flex flex-col md:flex-row md:items-end justify-between gap-4">
            <div>
                <h1 class="text-4xl font-extrabold text-main tracking-tight">
                    Floorball <span class="text-accent-primary">Saisonmanager</span>
                </h1>
                <p class="text-muted mt-2">Explore game operations, leagues, and schedules.</p>
            </div>
            <a href="#/" class="text-sm font-medium text-text-muted hover:text-accent-primary transition-colors flex items-center gap-1 group">
                <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 transform group-hover:-translate-x-1 transition-transform" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 19l-7-7m0 0l7-7m-7 7h18" />
                </svg>
                Back to Home
            </a>
        </header>

        <div class="space-y-4">
            {#if isLoadingInit}
                <div class="flex flex-col items-center justify-center p-12 bg-card backdrop-blur rounded-2xl shadow-sm border border-border-main">
                    <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-accent-primary mb-4"></div>
                    <p class="text-muted font-medium text-lg">Warming up...</p>
                </div>
            {:else if gameOperations.length === 0}
                <div class="p-8 text-center bg-danger-surface rounded-2xl border border-danger/20">
                    <p class="text-danger font-medium">No game operations found.</p>
                </div>
            {:else}
                {#each gameOperations as go (go.id)}
                    <details 
                        name="game-operations"
                        class="group bg-card backdrop-blur rounded-2xl shadow-sm border border-border-main overflow-hidden transition-all duration-300 open:bg-card-hover"
                        open={openGameOperationId === go.id}
                        ontoggle={(e) => handleGOToggle(go.id, (e.target as HTMLDetailsElement).open)}
                    >
                        <summary class="flex items-center justify-between p-5 cursor-pointer hover:bg-card-hover list-none select-none transition-colors">
                            <div class="flex items-center gap-4">
                                <div class="w-10 h-10 rounded-xl bg-accent-surface flex items-center justify-center text-accent-primary shadow-inner group-hover:scale-110 transition-transform border border-border-main">
                                    <span class="font-black text-xs">GO</span>
                                </div>
                                <span class="font-bold text-xl text-main">{go.name}</span>
                            </div>
                            <div class="transform transition-all duration-300 bg-bg-secondary p-2 rounded-full group-open:rotate-180 group-open:bg-accent-primary group-open:text-white">
                                <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                                    <path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
                                </svg>
                            </div>
                        </summary>

                        <div class="px-5 pb-5 pt-2 border-t border-border-main bg-bg-secondary/30 space-y-3">
                            {#if leagues[go.id]}
                                {#each leagues[go.id] as league (league.id)}
                                    <details 
                                        name="leagues-{go.id}"
                                        class="group/league bg-card rounded-xl border border-border-main overflow-hidden transition-all duration-200 hover:border-border-hover shadow-sm"
                                        open={openLeagueId === league.id}
                                        ontoggle={(e) => handleLeagueToggle(league.id, (e.target as HTMLDetailsElement).open)}
                                    >
                                        <summary class="flex items-center justify-between p-4 cursor-pointer hover:bg-accent-surface transition-colors list-none select-none">
                                            <div class="flex items-center gap-3">
                                                <div class="w-8 h-8 rounded-lg bg-success-surface flex items-center justify-center text-success group-hover/league:bg-success-surface transition-colors border border-success/20">
                                                    <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" viewBox="0 0 20 20" fill="currentColor">
                                                        <path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.07 3.292a1 1 0 00.95.69h3.462c.969 0 1.371 1.24.588 1.81l-2.8 2.034a1 1 0 00-.364 1.118l1.07 3.292c.3.921-.755 1.688-1.54 1.118l-2.8-2.034a1 1 0 00-1.175 0l-2.8 2.034c-.784.57-1.838-.197-1.539-1.118l1.07-3.292a1 1 0 00-.364-1.118L2.98 8.72c-.783-.57-.38-1.81.588-1.81h3.461a1 1 0 00.951-.69l1.07-3.292z" />
                                                    </svg>
                                                </div>
                                                <span class="font-semibold text-main">{league.name}</span>
                                            </div>
                                            <div class="flex items-center gap-2">
                                                <span class="text-[10px] font-bold text-muted bg-bg-secondary px-2 py-0.5 rounded border border-border-main uppercase">LEAGUE</span>
                                                <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 text-muted group-open/league:rotate-180 transition-transform" viewBox="0 0 20 20" fill="currentColor">
                                                    <path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
                                                </svg>
                                            </div>
                                        </summary>

                                        <div class="px-4 pb-4 pt-2 border-t border-border-main bg-bg-secondary/50 space-y-2">
                                            {#if schedules[league.id]}
                                                {@const gamesByDay = getGamesByDay(league.id)}
                                                {#each [...gamesByDay.entries()] as [day, games], dayIndex (day)}
                                                    <details
                                                        id="gameday-{league.id}-{day}"
                                                        name="gamedays-{league.id}"
                                                        class="group/day bg-card rounded-lg border border-border-main overflow-hidden shadow-sm"
                                                        open={openGameDay === day}
                                                        ontoggle={(e) => handleGameDayToggle(day, (e.target as HTMLDetailsElement).open)}
                                                    >
                                                        <summary class="flex items-center justify-between p-3 cursor-pointer hover:bg-warning-surface list-none select-none transition-colors">
                                                            <div class="flex items-center gap-2">
                                                                <div class="w-7 h-7 rounded-md bg-warning-surface flex items-center justify-center text-warning font-black text-xs border border-warning/20">
                                                                    {day}
                                                                </div>
                                                                <span class="font-semibold text-sm text-muted">Game Day {day}</span>
                                                                <span class="text-[10px] text-muted bg-bg-secondary px-1.5 py-0.5 rounded border border-border-main">{games.length} {games.length === 1 ? 'match' : 'matches'}</span>
                                                            </div>
                                                            <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 text-muted group-open/day:rotate-180 transition-transform" viewBox="0 0 20 20" fill="currentColor">
                                                                <path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
                                                            </svg>
                                                        </summary>

                                                        <div class="px-3 pb-3 pt-1 border-t border-border-main space-y-2">
                                                            {#each games as game, i (i)}
                                                                <button
                                                                    type="button"
                                                                    onclick={() => {
                                                                        selectGame(game, i)
                                                                        goto(`#/Match?gameId=${game.game_id}`)
                                                                    }}
                                                                    class="w-full flex flex-col md:flex-row md:items-center justify-between p-4 bg-card rounded-xl border border-border-main shadow-sm hover:shadow-md hover:border-border-hover hover:bg-card-hover transition-all group/game cursor-pointer text-left"
                                                                >
                                                                    <div class="flex items-center gap-4 mb-2 md:mb-0">
                                                                        <div class="flex flex-col text-center min-w-[85px] bg-bg-secondary p-2 rounded-lg border border-border-main">
                                                                            <span class="text-[10px] uppercase font-black text-muted tracking-wider font-mono">{game.date}</span>
                                                                            <span class="text-sm font-black text-main">{game.time}</span>
                                                                        </div>
                                                                        <div class="flex flex-col">
                                                                            <div class="flex items-center gap-2">
                                                                                <span class="text-base font-bold text-main">{game.home_team_name}</span>
                                                                                <span class="text-[10px] font-black text-muted italic uppercase">VS</span>
                                                                                <span class="text-base font-bold text-main">{game.guest_team_name}</span>
                                                                            </div>
                                                                            <span class="text-[10px] text-muted font-medium">Match #{i + 1}</span>
                                                                        </div>
                                                                    </div>
                                                                    
                                                                    <div class="flex items-center justify-center px-4 py-2 bg-accent-surface rounded-lg min-w-[100px] border border-border-main">
                                                                        {#if game.result}
                                                                            <span class="text-xl font-black text-accent-primary font-mono tracking-tighter">
                                                                                {game.result.home_goals}<span class="text-muted mx-1">:</span>{game.result.guest_goals}
                                                                            </span>
                                                                        {:else}
                                                                            <span class="text-[10px] font-extrabold text-accent-primary uppercase tracking-widest">Scheduled</span>
                                                                        {/if}
                                                                    </div>
                                                                </button>
                                                            {/each}
                                                        </div>
                                                    </details>
                                                {/each}
                                            {:else}
                                                <div class="flex flex-col items-center justify-center p-12 bg-card rounded-xl my-2 border border-dashed border-border-main">
                                                    <div class="flex gap-1.5 items-center">
                                                        <div class="animate-bounce h-2 w-2 bg-accent-primary rounded-full [animation-duration:1s]"></div>
                                                        <div class="animate-bounce h-2 w-2 bg-accent-primary rounded-full [animation-duration:1s] [animation-delay:0.2s]"></div>
                                                        <div class="animate-bounce h-2 w-2 bg-accent-primary rounded-full [animation-duration:1s] [animation-delay:0.4s]"></div>
                                                    </div>
                                                    <span class="text-xs font-bold text-muted mt-4 uppercase tracking-widest">Loading games...</span>
                                                </div>
                                            {/if}
                                        </div>
                                    </details>
                                {/each}
                            {:else}
                                <div class="flex flex-col items-center justify-center p-16 bg-card rounded-2xl border border-dashed border-border-main shadow-inner">
                                    <div class="relative">
                                        <div class="animate-ping absolute inset-0 rounded-full bg-accent-primary/40 opacity-20"></div>
                                        <div class="animate-bounce rounded-full h-10 w-10 border-4 border-accent-surface border-t-accent-primary"></div>
                                    </div>
                                    <p class="text-muted text-xs font-black uppercase tracking-[0.2em] mt-8">Fetching leagues</p>
                                </div>
                            {/if}
                        </div>
                    </details>
                {/each}
            {/if}
        </div>
    </div>
</div>

<style>
    /* Remove default details arrow */
    summary::-webkit-details-marker {
        display: none;
    }
    summary {
        list-style: none;
    }
</style>
