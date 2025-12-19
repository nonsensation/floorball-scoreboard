<script lang="ts">
    import { Saisonmanager as SM } from "floorball-saisonmanager";
    import { onMount } from "svelte";

    const API_BASE = "https://saisonmanager.de/api/v2";

    // State
    let gameId = $state<number | null>(null);
    let manualGameId = $state("");
    let gameData = $state<any>(null);
    let isLoading = $state(false);
    let error = $state<string | null>(null);
    let headerIsSticky = $state(false);

    // League table & scorer data
    let leagueTable = $state<any[]>([]);
    let isGroupedTable = $state(false);
    let groupedTable = $state<Record<string, any[]> | null>(null);
    let scorerData = $state<any[]>([]);
    let selectedPlayer = $state<any>(null);
    let scheduleData = $state<any[]>([]);

    // Team logos (resolved URLs)
    let homeTeamLogoUrl = $state<string | null>(null);
    let guestTeamLogoUrl = $state<string | null>(null);

    // Team color configuration
    let homeTeamColors = $state({
        primary: '#3B82F6', secondary: '#60A5FA', nameColor: '#60A5FA', scoreColor: '#93C5FD',
        additionalColors: [] as string[]
    });
    let guestTeamColors = $state({
        primary: '#10B981', secondary: '#34D399', nameColor: '#34D399', scoreColor: '#6EE7B7',
        additionalColors: [] as string[]
    });

    let headerSentinelRef = $state<HTMLElement | null>(null);

    function parseGameId(): number | null {
        const hash = window.location.hash;
        const match = hash.match(/[?&]gameId=(\d+)/);
        return match ? parseInt(match[1], 10) : null;
    }

    async function resolveLogoUrl(relativePath: string): Promise<string | null> {
        if (!relativePath) return null;
        return `https://saisonmanager.de${relativePath}`;
    }

    async function loadGame(id: number) {
        isLoading = true;
        error = null;
        gameData = null;
        homeTeamLogoUrl = null;
        guestTeamLogoUrl = null;
        leagueTable = [];
        groupedTable = null;
        scheduleData = [];
        try {
            const res = await fetch(`${API_BASE}/games/${id}.json`);
            if (!res.ok) throw new Error(`Failed to load game ${id}`);
            gameData = await res.json();
            
            if (gameData.home_team_logo) homeTeamLogoUrl = await resolveLogoUrl(gameData.home_team_logo);
            if (gameData.guest_team_logo) guestTeamLogoUrl = await resolveLogoUrl(gameData.guest_team_logo);
            
            if (gameData.league_id) {
                loadLeagueTable(gameData.league_id);
                loadScorerData(gameData.league_id);
                loadSchedule(gameData.league_id);
            }
        } catch (e) {
            console.error("Failed to load game", e);
            error = e instanceof Error ? e.message : "Unknown error";
        } finally {
            isLoading = false;
        }
    }

    async function loadLeagueTable(leagueId: number) {
        try {
            let res = await fetch(`${API_BASE}/leagues/${leagueId}/grouped_table.json`);
            if (res.ok) {
                const data = await res.json();
                const keys = Object.keys(data);
                if (keys.length > 0 && Array.isArray(data[keys[0]]) && data[keys[0]].length > 0) {
                    groupedTable = data;
                    isGroupedTable = true;
                    return;
                }
            }
        } catch (e) {}
        
        try {
            const res = await fetch(`${API_BASE}/leagues/${leagueId}/table.json`);
            if (res.ok) {
                const data = await res.json();
                if (Array.isArray(data) && data.length > 0) {
                    leagueTable = data;
                    isGroupedTable = false;
                }
            }
        } catch (e) {
            console.error("Failed to load league table", e);
        }
    }

    async function loadScorerData(leagueId: number) {
        try {
            const res = await fetch(`${API_BASE}/leagues/${leagueId}/scorer.json`);
            if (res.ok) scorerData = await res.json();
        } catch (e) {
            console.error("Failed to load scorer data", e);
        }
    }

    async function loadSchedule(leagueId: number) {
        try {
            const res = await fetch(`${API_BASE}/leagues/${leagueId}/schedule.json`);
            if (res.ok) scheduleData = await res.json();
        } catch (e) {
            console.error("Failed to load schedule", e);
        }
    }

    function getTeamForm(teamName: string, currentGameId: number): { result: 'W' | 'D' | 'L'; score: string; opponent: string; date: string; gameId: number }[] {
        if (!scheduleData.length) return [];
        const teamGames = scheduleData
            .filter((g: any) => g.ended && g.game_id !== currentGameId && 
                (g.home_team_name === teamName || g.guest_team_name === teamName))
            .sort((a: any, b: any) => new Date(b.date).getTime() - new Date(a.date).getTime())
            .slice(0, 10);
        
        return teamGames.map((g: any) => {
            const isHome = g.home_team_name === teamName;
            const teamGoals = isHome ? g.result.home_goals : g.result.guest_goals;
            const oppGoals = isHome ? g.result.guest_goals : g.result.home_goals;
            const opponent = isHome ? g.guest_team_name : g.home_team_name;
            let result: 'W' | 'D' | 'L' = 'D';
            if (teamGoals > oppGoals) result = 'W';
            else if (teamGoals < oppGoals) result = 'L';
            return { result, score: `${teamGoals}:${oppGoals}`, opponent, date: g.date, gameId: g.game_id };
        }).reverse();
    }

    function navigateToGame(gameId: number) {
        window.location.hash = `#/Match?gameId=${gameId}`;
    }

    function handleManualLoad() {
        const id = parseInt(manualGameId, 10);
        if (id && !isNaN(id)) {
            gameId = id;
            window.location.hash = `#/Match?gameId=${id}`;
            loadGame(id);
        }
    }

    function getPlayerName(roster: any[], trikotNumber: number): string {
        if (!roster) return `#${trikotNumber}`;
        const player = roster.find((p: any) => p.trikot_number === trikotNumber);
        return player ? `${player.player_firstname} ${player.player_name}` : `#${trikotNumber}`;
    }

    function isStartingPlayer(team: 'home' | 'guest', playerId: string | number): boolean {
        const starting = gameData?.starting_players?.[team];
        if (!starting) return false;
        return starting.some((p: any) => p.player_id && p.player_id.toString() === playerId.toString());
    }

    function parsePenaltyMinutes(penaltyTypeString: string): number {
        if (!penaltyTypeString) return 2;
        if (penaltyTypeString.includes('+')) {
            const parts = penaltyTypeString.match(/(\d+)\+(\d+)/);
            if (parts) return parseInt(parts[1]) + parseInt(parts[2]);
        }
        const match = penaltyTypeString.match(/(\d+)/);
        return match ? parseInt(match[1]) : 2;
    }

    function getPlayerGameStats(team: 'home' | 'guest', trikotNumber: number): { goals: number; assists: number; penalties: number; penaltyMinutes: number } {
        if (!gameData?.events) return { goals: 0, assists: 0, penalties: 0, penaltyMinutes: 0 };
        let goals = 0, assists = 0, penalties = 0, penaltyMinutes = 0;
        for (const e of gameData.events) {
            if (e.event_team !== team) continue;
            if (e.event_type === 'goal' && e.number === trikotNumber) goals++;
            if (e.event_type === 'goal' && e.assist === trikotNumber) assists++;
            if (e.event_type === 'penalty' && e.number === trikotNumber) {
                penalties++;
                penaltyMinutes += parsePenaltyMinutes(e.penalty_type_string);
            }
        }
        return { goals, assists, penalties, penaltyMinutes };
    }

    function getPlayerSeasonStats(playerId: number): any | null {
        return scorerData.find((s: any) => s.player_id === playerId) ?? null;
    }

    function calculateSeasonPenaltyMinutes(stats: any): number {
        if (!stats) return 0;
        return (
            (stats.penalty_2 ?? 0) * 2 +
            (stats.penalty_2and2 ?? 0) * 4 +
            (stats.penalty_5 ?? 0) * 5 +
            (stats.penalty_10 ?? 0) * 10
        );
    }

    function selectPlayer(player: any) {
        selectedPlayer = selectedPlayer?.player_id === player.player_id ? null : player;
    }

    function getIngameStatus(): { label: string; color: string } {
        const status = gameData?.ingame_status;
        const statusMap: Record<string, { label: string; color: string }> = {
            'period1': { label: '1st Period', color: 'bg-info' },
            'period2': { label: '2nd Period', color: 'bg-info' },
            'period3': { label: '3rd Period', color: 'bg-info' },
            'pause1': { label: '1st Pause', color: 'bg-warning' },
            'pause2': { label: '2nd Pause', color: 'bg-warning' },
            'pause_et': { label: 'OT Pause', color: 'bg-warning' },
            'pause_ps': { label: 'PS Pause', color: 'bg-warning' },
            'extratime': { label: 'Extra Time', color: 'bg-accent-secondary' },
            'penalty_shots': { label: 'Penalty Shots', color: 'bg-danger' },
        };
        return statusMap[status] ?? { label: gameData?.ended ? 'Final' : 'Scheduled', color: gameData?.ended ? 'bg-text-muted' : 'bg-success' };
    }

    function getEventStyle(eventType: string): { icon: string; bg: string; text: string } {
        switch (eventType) {
            case 'goal': return { icon: '⚽', bg: 'bg-success-surface', text: 'text-success' };
            case 'penalty': return { icon: '🟨', bg: 'bg-warning-surface', text: 'text-warning' };
            case 'timeout': return { icon: '⏱️', bg: 'bg-bg-secondary', text: 'text-text-muted' };
            default: return { icon: '📋', bg: 'bg-info-surface', text: 'text-info' };
        }
    }

    function getEventsGroupedByPeriod(): { period: number; events: any[] }[] {
        if (!gameData?.events?.length) return [];
        const sorted = [...gameData.events].sort((a, b) => a.sortkey.localeCompare(b.sortkey));
        const groups: { period: number; events: any[] }[] = [];
        let currentPeriod = -1;
        for (const event of sorted) {
            if (event.period !== currentPeriod) {
                currentPeriod = event.period;
                groups.push({ period: currentPeriod, events: [] });
            }
            groups[groups.length - 1].events.push(event);
        }
        return groups;
    }

    function addColor(team: 'home' | 'guest') {
        if (team === 'home') homeTeamColors.additionalColors = [...homeTeamColors.additionalColors, '#808080'];
        else guestTeamColors.additionalColors = [...guestTeamColors.additionalColors, '#808080'];
    }
    function removeColor(team: 'home' | 'guest', index: number) {
        if (team === 'home') homeTeamColors.additionalColors = homeTeamColors.additionalColors.filter((_, i) => i !== index);
        else guestTeamColors.additionalColors = guestTeamColors.additionalColors.filter((_, i) => i !== index);
    }

    // Persistence Keys
    const STORAGE_KEY_SETTINGS = "uni_hockey_match_settings";

    function updateColorState(target: any, source: any) {
        if (!source) return;
        target.primary = source.primary ?? target.primary;
        target.secondary = source.secondary ?? target.secondary;
        target.nameColor = source.nameColor ?? target.nameColor;
        target.scoreColor = source.scoreColor ?? target.scoreColor;
        if (Array.isArray(source.additionalColors)) {
            target.additionalColors = [...source.additionalColors];
        }
    }

    function loadSettings() {
        try {
            const stored = localStorage.getItem(STORAGE_KEY_SETTINGS);
            console.log("Loading settings from LS:", stored);
            if (stored) {
                const settings = JSON.parse(stored);
                updateColorState(homeTeamColors, settings.homeTeamColors);
                updateColorState(guestTeamColors, settings.guestTeamColors);
            }
        } catch (e) {
            console.error("Failed to load settings", e);
        }
    }

    function saveSettings() {
        try {
            const settings = {
                homeTeamColors: $state.snapshot(homeTeamColors),
                guestTeamColors: $state.snapshot(guestTeamColors),
                lastGameId: gameId
            };
            console.log("Saving settings to LS:", settings);
            localStorage.setItem(STORAGE_KEY_SETTINGS, JSON.stringify(settings));
        } catch (e) {
            console.error("Failed to save settings", e);
        }
    }


    
    // Color Utils
    function adjustColor(hex: string, amount: number): string {
        const usePound = hex.startsWith('#');
        let color = usePound ? hex.slice(1) : hex;
        const num = parseInt(color, 16);
        let r = (num >> 16) + amount;
        let g = ((num >> 8) & 0x00FF) + amount;
        let b = (num & 0x0000FF) + amount;

        r = Math.max(Math.min(255, r), 0);
        g = Math.max(Math.min(255, g), 0);
        b = Math.max(Math.min(255, b), 0);

        return (usePound ? '#' : '') + (g | (b << 8) | (r << 16)).toString(16).padStart(6, '0');
    }

    function updatePrimaryColor(team: 'home' | 'guest', newColor: string) {
        const colors = team === 'home' ? homeTeamColors : guestTeamColors;
        colors.primary = newColor;
        // Previously we auto-calculated secondary/name/score colors here.
        // User reported this as an issue (unwanted side effects), so we now ONLY update primary.
        // If the user wants to match them, they can manually set them.
    }

    let isInitialized = false;

    onMount(() => {
        loadSettings();
        isInitialized = true;
        
        const id = parseGameId();
        if (id) { gameId = id; manualGameId = id.toString(); loadGame(id); }
        const handleHashChange = () => {
            const newId = parseGameId();
            if (newId && newId !== gameId) { gameId = newId; manualGameId = newId.toString(); loadGame(newId); }
        };
        window.addEventListener('hashchange', handleHashChange);

        const observer = new IntersectionObserver(
            (entries) => {
                headerIsSticky = !entries[0].isIntersecting;
            },
            { threshold: 0, rootMargin: '0px' }
        );
        
        if (headerSentinelRef) observer.observe(headerSentinelRef);

        return () => {
            window.removeEventListener('hashchange', handleHashChange);
            observer.disconnect();
        };
    });

    $effect(() => {
        if (!isInitialized) return;

        if (headerSentinelRef) {
            const observer = new IntersectionObserver(
                (entries) => {
                    headerIsSticky = !entries[0].isIntersecting;
                },
                { threshold: 0, rootMargin: '0px' }
            );
            observer.observe(headerSentinelRef);
            return () => observer.disconnect();
        }
    });

    // Reactively save settings when they change
    $effect(() => {
        if (!isInitialized) return;

        // Force deep dependency tracking by serializing the state
        // This ensures the effect re-runs whenever ANY nested property changes
        JSON.stringify(homeTeamColors);
        JSON.stringify(guestTeamColors);
        // Also track gameId
        const _id = gameId;
        
        saveSettings();
    });
</script>

<div class="w-full min-h-screen bg-bg-primary bg-[radial-gradient(ellipse_at_top,_var(--tw-gradient-stops))] from-gradient-from via-gradient-via to-gradient-to transition-colors duration-300">
    <!-- Sentinel element -->
    <div bind:this={headerSentinelRef} class="h-0"></div>
    
    <!-- Sticky Score Header -->
    {#if gameData}
        {@const ingameStatus = getIngameStatus()}
        <div class="sticky top-0 z-50 glass border-b border-border-main transition-all duration-300 {headerIsSticky ? 'py-3' : 'py-5 sm:py-8'}">
            <div class="px-4 sm:px-8">
                <div class="flex items-center justify-center gap-3 sm:gap-8">
                    <!-- Home Team -->
                    <div class="flex items-center gap-3 sm:gap-4 flex-1 justify-end min-w-0">
                        <span class="font-bold text-right transition-all {headerIsSticky ? 'text-sm sm:text-lg truncate' : 'text-lg sm:text-2xl leading-tight'} text-text-main">{gameData.home_team_name}</span>
                        {#if homeTeamLogoUrl}<img src={homeTeamLogoUrl} alt="" class="object-contain shrink-0 transition-all drop-shadow-lg {headerIsSticky ? 'w-10 h-10 sm:w-12 sm:h-12' : 'w-14 h-14 sm:w-20 sm:h-20'}" />{/if}
                    </div>
                    <!-- Score -->
                    <div class="flex flex-col items-center shrink-0 px-3 sm:px-6">
                        <div class="font-black font-mono flex items-center gap-2 sm:gap-3 transition-all {headerIsSticky ? 'text-3xl sm:text-5xl' : 'text-5xl sm:text-7xl'} tracking-tight">
                            <span class="text-text-main">{gameData.result?.home_goals ?? '-'}</span>
                            <span class="text-text-muted">:</span>
                            <span class="text-text-main">{gameData.result?.guest_goals ?? '-'}</span>
                        </div>
                        <span class="{ingameStatus.color} text-text-inverse font-bold px-4 py-1 rounded-full uppercase tracking-wider transition-all shadow-lg {headerIsSticky ? 'text-[10px]' : 'text-xs'}">{ingameStatus.label}</span>
                        {#if !headerIsSticky && gameData.result?.home_goals_period}
                            <div class="mt-3 flex flex-wrap justify-center gap-2 text-xs sm:text-sm font-mono">
                                {#each gameData.result.home_goals_period as hg, i (i)}
                                    {#if gameData.result.guest_goals_period[i] !== undefined && (hg > 0 || gameData.result.guest_goals_period[i] > 0 || i < 3)}
                                        <span class="bg-bg-secondary backdrop-blur px-3 py-1.5 rounded-lg text-text-muted border border-border-main">{i === 3 ? 'OT' : `P${i+1}`}: <span class="text-text-main font-bold">{hg}-{gameData.result.guest_goals_period[i]}</span></span>
                                    {/if}
                                {/each}
                            </div>
                        {/if}
                    </div>
                    <!-- Guest Team -->
                    <div class="flex items-center gap-3 sm:gap-4 flex-1 justify-start min-w-0">
                        {#if guestTeamLogoUrl}<img src={guestTeamLogoUrl} alt="" class="object-contain shrink-0 transition-all drop-shadow-lg {headerIsSticky ? 'w-10 h-10 sm:w-12 sm:h-12' : 'w-14 h-14 sm:w-20 sm:h-20'}" />{/if}
                        <span class="font-bold transition-all {headerIsSticky ? 'text-sm sm:text-lg truncate' : 'text-lg sm:text-2xl leading-tight'} text-text-main">{gameData.guest_team_name}</span>
                    </div>
                </div>
            </div>
        </div>
    {/if}

    <!-- Navigation Header -->
    <header class="w-full px-4 sm:px-8 py-4 bg-bg-card backdrop-blur border-b border-border-main transition-colors">
        <div class="flex flex-col sm:flex-row items-start sm:items-center justify-between gap-4">
            <div class="flex items-center gap-4">
                <h1 class="text-xl sm:text-2xl font-bold text-text-main tracking-tight">Match <span class="text-text-muted">Details</span></h1>
                {#if gameId}<span class="text-text-muted text-xs font-mono bg-bg-secondary px-2 py-1 rounded">#{gameId}</span>{/if}
            </div>
            <div class="flex gap-3 items-center w-full sm:w-auto">
                <input type="text" bind:value={manualGameId} placeholder="Game ID" class="flex-1 sm:flex-none sm:w-32 px-4 py-2.5 bg-bg-card border border-border-main rounded-xl font-mono text-sm text-text-main placeholder-text-muted focus:outline-none focus:ring-2 focus:ring-accent-primary/50 focus:border-transparent transition-all" onkeydown={(e) => e.key === 'Enter' && handleManualLoad()} />
                <button onclick={handleManualLoad} disabled={isLoading} class="px-5 py-2.5 bg-text-main text-text-inverse font-semibold text-sm rounded-xl hover:opacity-90 disabled:opacity-50 transition-all shadow-md">{isLoading ? '...' : 'Load'}</button>
                <a href="#/MatchSelect" class="text-sm font-medium text-text-muted hover:text-text-main transition-colors hidden sm:block">← Back</a>
            </div>
        </div>
    </header>

    <div class="w-full px-4 sm:px-8 py-6">
        {#if isLoading}
            <div class="flex flex-col items-center justify-center p-16 sm:p-24 glass rounded-3xl">
                <div class="relative">
                    <div class="w-16 h-16 rounded-full border-4 border-accent-primary/20 border-t-accent-primary animate-spin"></div>
                    <div class="absolute inset-0 w-16 h-16 rounded-full border-4 border-transparent border-b-info animate-spin animation-delay-150"></div>
                </div>
                <p class="text-text-muted text-base mt-6 font-medium">Loading match data...</p>
            </div>
        {:else if error}
            <div class="p-10 text-center glass rounded-3xl border border-danger/20">
                <div class="text-5xl mb-4">⚠️</div>
                <p class="text-danger text-lg font-medium">{error}</p>
            </div>
        {:else if !gameId}
            <div class="p-16 text-center bg-bg-card backdrop-blur rounded-3xl border border-border-main shadow-sm">
                <div class="text-7xl mb-6 animate-bounce">🏒</div>
                <p class="text-text-muted text-lg">Enter a Game ID above to get started</p>
            </div>
        {:else if gameData}
            <!-- Team Form Timelines -->
            {#if scheduleData.length > 0}
                {@const homeForm = getTeamForm(gameData.home_team_name, gameData.game_id)}
                {@const guestForm = getTeamForm(gameData.guest_team_name, gameData.game_id)}
                <div class="grid grid-cols-1 lg:grid-cols-2 gap-4 mb-6">
                    {#each [[gameData.home_team_name, homeForm, homeTeamColors], [gameData.guest_team_name, guestForm, guestTeamColors]] as [teamName, form, colors]}
                        <div class="bg-bg-card backdrop-blur rounded-2xl p-4 sm:p-5 border border-border-main hover:border-border-hover transition-all shadow-sm">
                            <div class="flex items-center gap-3 mb-4">
                                <h4 class="font-bold text-sm text-text-main">Recent Form</h4>
                                <span class="text-xs text-text-muted">{teamName}</span>
                            </div>
                            {#if form.length}
                                <div class="relative h-24 flex gap-1 items-center mt-2 mb-2">
                                    <div class="absolute left-0 right-0 top-1/2 h-px bg-border-main z-0"></div>
                                    {#each form as game, i (i)}
                                        <button 
                                            type="button"
                                            onclick={() => navigateToGame(game.gameId)} 
                                            class="relative flex-1 h-full flex flex-col items-center justify-center group cursor-pointer hover:bg-bg-card-hover rounded transition-colors z-10" 
                                            title="{game.score} vs {game.opponent} ({game.date})"
                                        >
                                            {#if game.result === 'W'}
                                                <div class="absolute bottom-1/2 w-[calc(100%-4px)] h-8 bg-success/80 hover:bg-success rounded-t border border-glass-border flex items-center justify-center transition-all origin-bottom group-hover:scale-y-110 shadow-lg">
                                                    <span class="text-[9px] font-bold text-text-inverse">{game.score}</span>
                                                </div>
                                            {:else if game.result === 'D'}
                                                <div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 px-1.5 h-5 bg-text-muted rounded-full border border-glass-border flex items-center justify-center transition-all shadow-lg z-20 whitespace-nowrap">
                                                    <span class="text-[9px] font-bold text-text-inverse">{game.score}</span>
                                                </div>
                                            {:else}
                                                <div class="absolute top-1/2 w-[calc(100%-4px)] h-8 bg-danger/80 hover:bg-danger rounded-b border border-glass-border flex items-center justify-center transition-all origin-top group-hover:scale-y-110 shadow-lg">
                                                    <span class="text-[9px] font-bold text-text-inverse">{game.score}</span>
                                                </div>
                                            {/if}
                                            <div class="hidden group-hover:block absolute bottom-full left-1/2 -translate-x-1/2 mb-2 px-3 py-2 bg-glass-dark-bg backdrop-blur text-text-inverse text-[10px] rounded-lg whitespace-nowrap z-50 border border-glass-border shadow-xl">
                                                <div class="font-bold mb-0.5">{game.result} • {game.score}</div>
                                                <div class="opacity-60">{game.opponent}</div>
                                                <div class="opacity-40 text-[9px] mt-1">{game.date}</div>
                                            </div>
                                        </button>
                                    {/each}
                                </div>
                            {:else}
                                <p class="text-xs text-text-muted">No recent games</p>
                            {/if}
                        </div>
                    {/each}
                </div>
            {/if}

            <!-- Info Row -->
            <div class="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-5 gap-3 mb-6">
                {#each [['Venue', gameData.arena_name], ['League', gameData.league_name ?? gameData.league_short_name], ['Game Day', gameData.game_day?.title], ['Referees', gameData.nominated_referees], ['Date/Time', `${gameData.date} • ${gameData.start_time}`]] as [label, value]}
                    <div class="bg-bg-card backdrop-blur rounded-xl p-3 sm:p-4 border border-border-main hover:border-border-hover transition-all group shadow-sm {label === 'Date/Time' ? 'col-span-2 sm:col-span-1' : ''}">
                        <div class="text-[10px] sm:text-xs font-bold text-text-accent uppercase tracking-wider mb-1">{label}</div>
                        <div class="font-semibold text-sm sm:text-base text-text-main truncate group-hover:text-text-main transition-colors">{value ?? '-'}</div>
                    </div>
                {/each}
            </div>

            <!-- Color Config -->
            <details class="mb-6 bg-bg-card backdrop-blur rounded-2xl border border-border-main overflow-hidden group shadow-sm">
                <summary class="px-5 py-4 text-sm font-semibold text-text-muted cursor-pointer hover:text-text-main transition-colors flex items-center gap-2">
                    <span class="text-lg">🎨</span> Team Colors
                    <span class="ml-auto text-text-muted group-open:rotate-180 transition-transform">▼</span>
                </summary>
                <div class="p-5 border-t border-border-main grid grid-cols-1 md:grid-cols-2 gap-6">
                    {#each [['home', homeTeamColors], ['guest', guestTeamColors]] as [team, colors]}
                        <div class="space-y-4">
                            <h4 class="font-bold text-text-main">{team === 'home' ? 'Home' : 'Guest'} Team</h4>
                            <div class="grid grid-cols-4 gap-3">
                                <label class="text-xs text-text-main">Primary<input type="color" bind:value={colors.primary} class="w-full h-10 rounded-lg cursor-pointer border border-border-main bg-bg-card" /></label>
                                <label class="text-xs text-text-main">Secondary<input type="color" bind:value={colors.secondary} class="w-full h-10 rounded-lg cursor-pointer border border-border-main bg-bg-card" /></label>
                                <label class="text-xs text-text-main">Name<input type="color" bind:value={colors.nameColor} class="w-full h-10 rounded-lg cursor-pointer border border-border-main bg-bg-card" /></label>
                                <label class="text-xs text-text-main">Score<input type="color" bind:value={colors.scoreColor} class="w-full h-10 rounded-lg cursor-pointer border border-border-main bg-bg-card" /></label>
                                {#each colors.additionalColors as c, i (i)}
                                    <label class="text-xs text-text-main relative">Extra {i+1}<input type="color" bind:value={colors.additionalColors[i]} class="w-full h-10 rounded-lg cursor-pointer border border-border-main bg-bg-card" /><button onclick={() => removeColor(team, i)} class="absolute -top-1 -right-1 w-5 h-5 bg-danger text-text-inverse rounded-full text-xs font-bold hover:opacity-80 transition-colors">×</button></label>
                                {/each}
                            </div>
                            <button onclick={() => addColor(team)} class="text-sm text-text-accent hover:opacity-80 transition-colors">+ Add Color</button>
                        </div>
                    {/each}
                </div>
            </details>

            <!-- Rosters -->
            <div class="grid grid-cols-1 lg:grid-cols-2 gap-4 mb-6">
                {#each [['home', gameData.players?.home, homeTeamColors, homeTeamLogoUrl], ['guest', gameData.players?.guest, guestTeamColors, guestTeamLogoUrl]] as [team, roster, colors, logo]}
                    <div class="bg-bg-card backdrop-blur rounded-2xl border border-border-main overflow-hidden shadow-sm">
                        <div class="px-5 py-4 border-b border-border-main flex items-center gap-4 bg-bg-card">
                            {#if logo}<img src={logo} alt="" class="w-10 h-10 object-contain drop-shadow-lg" />{/if}
                            <h3 class="font-bold text-lg truncate text-text-main">{team === 'home' ? gameData.home_team_name : gameData.guest_team_name}</h3>
                            <span class="text-sm text-text-muted ml-auto bg-bg-secondary px-3 py-1 rounded-full border border-border-main">{roster?.length ?? 0}</span>
                        </div>
                        <div class="p-3 sm:p-4">
                            {#if roster?.length}
                                <div class="space-y-1">
                                    {#each roster as player, i (i)}
                                        {@const stats = getPlayerGameStats(team, player.trikot_number)}
                                        {@const isStarter = isStartingPlayer(team, player.player_id)}
                                        {@const isSelected = selectedPlayer?.player_id === player.player_id}
                                        {@const seasonStats = isSelected ? getPlayerSeasonStats(player.player_id) : null}
                                        {@const seasonPenaltyMins = seasonStats ? calculateSeasonPenaltyMinutes(seasonStats) : 0}
                                        <button 
                                            type="button" 
                                            onclick={() => selectPlayer(player)} 
                                            class="player-card w-full flex flex-col text-left p-3 rounded-lg transition-all cursor-pointer {isSelected ? 'bg-bg-secondary' : 'hover:bg-bg-card-hover'}"
                                        >
                                            <div class="flex items-center gap-3">
                                                <div class="w-8 h-8 rounded-lg flex items-center justify-center font-semibold text-sm text-text-main shrink-0 bg-bg-secondary">{player.trikot_number}</div>
                                                <span class="font-medium text-sm sm:text-base text-text-main truncate flex-1">{player.player_firstname} {player.player_name}</span>
                                                {#if player.captain}<span class="text-[10px] font-semibold text-warning bg-warning-surface px-1.5 py-0.5 rounded shrink-0">C</span>{/if}
                                                {#if player.goalkeeper}<span class="text-[10px] font-semibold text-accent-secondary bg-accent-secondary-surface px-1.5 py-0.5 rounded shrink-0">GK</span>{/if}
                                                {#if isStarter}<span class="text-[10px] font-semibold text-success bg-success-surface px-2 py-0.5 rounded shrink-0">Starting</span>{/if}
                                            </div>
                                            <div class="flex gap-2 mt-2 pl-11">
                                                {#if stats.goals > 0}<span class="bg-success-surface text-success px-2 py-0.5 rounded font-semibold text-xs">⚽ {stats.goals}</span>{/if}
                                                {#if stats.assists > 0}<span class="bg-info-surface text-info px-2 py-0.5 rounded font-semibold text-xs">🅰️ {stats.assists}</span>{/if}
                                                {#if stats.penaltyMinutes > 0}<span class="bg-warning-surface text-warning px-2 py-0.5 rounded font-semibold text-xs">🟨 {stats.penaltyMinutes}m</span>{/if}
                                            </div>
                                            {#if isSelected && seasonStats}
                                                <div class="mt-3 p-3 bg-bg-card rounded-lg text-xs ml-11 border border-border-main">
                                                    <div class="font-semibold text-text-muted mb-2">Season Stats ({seasonStats.games ?? 0} games)</div>
                                                    <div class="grid grid-cols-4 gap-2 text-center">
                                                        <div><div class="font-bold text-lg text-success">{seasonStats.goals ?? 0}</div><div class="text-[10px] text-text-muted">Goals</div></div>
                                                        <div><div class="font-bold text-lg text-info">{seasonStats.assists ?? 0}</div><div class="text-[10px] text-text-muted">Assists</div></div>
                                                        <div><div class="font-bold text-lg text-text-main">{(seasonStats.goals ?? 0) + (seasonStats.assists ?? 0)}</div><div class="text-[10px] text-text-muted">Points</div></div>
                                                        <div><div class="font-bold text-lg text-warning">{seasonPenaltyMins}</div><div class="text-[10px] text-text-muted">Pen.Min</div></div>
                                                    </div>
                                                </div>
                                            {/if}
                                        </button>
                                    {/each}
                                </div>
                            {:else}
                                <p class="text-text-muted text-center py-8">No roster available</p>
                            {/if}
                        </div>
                    </div>
                {/each}
            </div>

            <!-- Timeline Events -->
            <div class="bg-bg-card backdrop-blur rounded-2xl border border-border-main overflow-hidden mb-6 shadow-sm">
                <div class="px-5 py-4 border-b border-border-main flex items-center justify-between bg-gradient-to-r from-accent-surface to-transparent">
                    <h3 class="font-bold text-lg text-text-main">Game Timeline</h3>
                    <span class="text-sm text-text-muted bg-bg-secondary px-3 py-1 rounded-full border border-border-main">{gameData.events?.length ?? 0} events</span>
                </div>
                <div class="p-4 sm:p-6">
                    {#if gameData.events?.length}
                        {@const periodGroups = getEventsGroupedByPeriod()}
                        <div class="relative">
                            <div class="absolute left-1/2 top-0 bottom-0 w-px bg-border-main -translate-x-1/2 z-0"></div>
                            
                            <div class="space-y-2">
                                {#each periodGroups as group, groupIdx (groupIdx)}
                                    <div class="relative z-10 flex items-center justify-center py-4">
                                        <div class="flex-1 h-px bg-border-main"></div>
                                        <span class="px-5 py-2 bg-bg-secondary text-text-main font-semibold text-sm rounded-full mx-4 border border-border-main shadow-sm">
                                            {group.period === 4 ? 'Overtime' : `Period ${group.period}`}
                                        </span>
                                        <div class="flex-1 h-px bg-border-main"></div>
                                    </div>
                                    
                                    <div class="space-y-4">
                                        {#each group.events as event, i (i)}
                                            {@const style = getEventStyle(event.event_type)}
                                            {@const isHome = event.event_team === 'home'}
                                            {@const roster = isHome ? gameData.players?.home : gameData.players?.guest}
                                            {@const teamColor = isHome ? homeTeamColors.primary : guestTeamColors.primary}
                                            {@const teamLogo = isHome ? homeTeamLogoUrl : guestTeamLogoUrl}
                                            {@const isGoal = event.event_type === 'goal'}
                                            
                                            <div class="relative flex items-stretch {isHome ? 'flex-row' : 'flex-row-reverse'}">
                                                <div class="w-[calc(50%-2rem)] sm:w-[calc(50%-3rem)] bg-bg-card backdrop-blur rounded-xl p-3 sm:p-4 border border-border-main hover:border-border-hover transition-all shadow-sm">
                                                    <div class="flex items-center gap-2 sm:gap-3 mb-2 {isHome ? '' : 'flex-row-reverse'}">
                                                        <span class="text-xl sm:text-2xl">{style.icon}</span>
                                                        {#if teamLogo}
                                                            <img src={teamLogo} alt="" class="w-5 h-5 sm:w-6 sm:h-6 object-contain drop-shadow" />
                                                        {:else}
                                                            <span class="font-bold text-xs uppercase" style="color: {teamColor}">{isHome ? 'HOME' : 'GUEST'}</span>
                                                        {/if}
                                                        <span class="font-bold text-xs sm:text-sm text-text-main capitalize tracking-wide">{event.event_type}</span>
                                                        {#if isGoal && event.home_goals !== undefined}
                                                            <span class="font-mono font-black text-base sm:text-xl text-text-main {isHome ? 'ml-auto' : 'mr-auto'}">{event.home_goals}:{event.guest_goals}</span>
                                                        {/if}
                                                    </div>
                                                    <div class="text-xs sm:text-sm text-text-muted {isHome ? '' : 'text-right'}">
                                                        {#if event.number}
                                                            <span class="font-mono bg-bg-secondary px-1.5 py-0.5 rounded text-xs text-text-muted">#{event.number}</span>
                                                            <span class="font-semibold ml-1 text-text-main">{getPlayerName(roster, event.number)}</span>
                                                        {/if}
                                                        {#if event.assist}
                                                            <span class="text-text-muted ml-2 text-xs">(A: #{event.assist} {getPlayerName(roster, event.assist)})</span>
                                                        {/if}
                                                        {#if event.penalty_reason_string}
                                                            <span class="text-text-muted block mt-1 text-xs">{event.penalty_type_string} - {event.penalty_reason_string}</span>
                                                        {/if}
                                                    </div>
                                                </div>
                                                
                                                <div class="w-16 sm:w-24 flex items-center justify-center shrink-0 relative z-10">
                                                    <div class="bg-bg-secondary border border-border-main rounded-full px-3 sm:px-4 py-1 sm:py-1.5 font-mono text-[10px] sm:text-sm font-medium text-text-muted whitespace-nowrap shadow-sm">
                                                        {event.time}
                                                    </div>
                                                </div>
                                                
                                                <div class="w-[calc(50%-2rem)] sm:w-[calc(50%-3rem)]"></div>
                                            </div>
                                        {/each}
                                    </div>
                                {/each}
                            </div>
                        </div>
                    {:else}
                        <p class="text-text-muted text-center py-12">No events recorded</p>
                    {/if}
                </div>
            </div>

            <!-- League Table -->
            {#if leagueTable.length > 0 || (groupedTable && Object.keys(groupedTable).length > 0)}
                <div class="bg-bg-card backdrop-blur rounded-2xl border border-border-main overflow-hidden mb-6 shadow-sm">
                    <div class="px-5 py-4 border-b border-border-main bg-gradient-to-r from-accent-secondary-surface to-transparent">
                        <h3 class="font-bold text-lg text-text-main">League Table</h3>
                    </div>
                    <div class="p-3 sm:p-5 overflow-x-auto">
                        {#if isGroupedTable && groupedTable}
                            {#each Object.entries(groupedTable) as [groupName, teams]}
                                <h4 class="font-bold text-sm text-text-muted mb-3 mt-5 first:mt-0">{groupName}</h4>
                                {@render tableRows(teams)}
                            {/each}
                        {:else}
                            {@render tableRows(leagueTable)}
                        {/if}
                    </div>
                </div>
            {/if}

            <!-- Debug -->
            <details class="bg-bg-card backdrop-blur rounded-2xl border border-border-main overflow-hidden shadow-sm">
                <summary class="px-5 py-3 text-text-muted text-sm cursor-pointer hover:text-text-main transition-colors">🔍 Raw JSON Data</summary>
                <pre class="p-5 text-xs text-text-muted overflow-x-auto max-h-64 border-t border-border-main font-mono bg-bg-secondary">{JSON.stringify(gameData, null, 2)}</pre>
            </details>
        {/if}
    </div>
</div>

{#snippet tableRows(teams)}
    <table class="w-full text-xs sm:text-sm">
        <thead>
            <tr class="text-left text-text-muted text-[10px] sm:text-xs uppercase tracking-wider">
                <th class="p-2 sm:p-3">#</th>
                <th class="p-2 sm:p-3">Team</th>
                <th class="p-2 sm:p-3 text-center hidden sm:table-cell">GP</th>
                <th class="p-2 sm:p-3 text-center">W</th>
                <th class="p-2 sm:p-3 text-center hidden sm:table-cell">D</th>
                <th class="p-2 sm:p-3 text-center">L</th>
                <th class="p-2 sm:p-3 text-center hidden md:table-cell">GF</th>
                <th class="p-2 sm:p-3 text-center hidden md:table-cell">GA</th>
                <th class="p-2 sm:p-3 text-center">GD</th>
                <th class="p-2 sm:p-3 text-center font-bold">Pts</th>
            </tr>
        </thead>
        <tbody>
            {#each teams as team, i (i)}
                {@const isMatchTeam = team.team_id === gameData.home_team_id || team.team_id === gameData.guest_team_id}
                {@const isHome = team.team_id === gameData.home_team_id}
                <tr class="border-t border-border-subtle transition-colors {isMatchTeam ? 'bg-bg-secondary font-medium' : 'hover:bg-bg-card-hover'}">
                    <td class="p-2 sm:p-3 font-bold text-text-muted">{team.position ?? i + 1}</td>
                    <td class="p-2 sm:p-3">
                        <div class="flex items-center gap-2">
                            {#if team.team_logo}<img src="https://saisonmanager.de{team.team_logo}" alt="" class="w-5 h-5 object-contain shrink-0 drop-shadow" />{/if}
                            <span class="font-medium truncate max-w-[100px] sm:max-w-none {isMatchTeam ? 'font-bold text-text-main' : 'text-text-muted'}">{team.team_name}</span>
                        </div>
                    </td>
                    <td class="p-2 sm:p-3 text-center text-text-muted hidden sm:table-cell">{team.games}</td>
                    <td class="p-2 sm:p-3 text-center text-success">{team.won}</td>
                    <td class="p-2 sm:p-3 text-center text-text-muted hidden sm:table-cell">{team.draw}</td>
                    <td class="p-2 sm:p-3 text-center text-danger">{team.lost}</td>
                    <td class="p-2 sm:p-3 text-center text-text-muted hidden md:table-cell">{team.goals_scored}</td>
                    <td class="p-2 sm:p-3 text-center text-text-muted hidden md:table-cell">{team.goals_received}</td>
                    <td class="p-2 sm:p-3 text-center font-mono font-medium text-text-muted">{team.goals_diff > 0 ? '+' : ''}{team.goals_diff}</td>
                    <td class="p-2 sm:p-3 text-center font-black text-base text-text-main">{team.points}</td>
                </tr>
            {/each}
        </tbody>
    </table>
{/snippet}

<style>
    @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800;900&display=swap');
    
    :global(body) {
        font-family: 'Inter', system-ui, sans-serif;
    }
    
    .score-glow {
        text-shadow: 0 0 40px var(--glow-color, currentColor);
    }
    
    .player-card {
        border: 1px solid transparent;
    }
    .player-card:hover {
        border-color: var(--border-hover);
    }
    
    .animation-delay-150 {
        animation-delay: 150ms;
    }
</style>
