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
        primary: '#3B82F6', secondary: '#60A5FA', nameColor: '#1E40AF', scoreColor: '#1E3A8A',
        additionalColors: [] as string[]
    });
    let guestTeamColors = $state({
        primary: '#10B981', secondary: '#34D399', nameColor: '#047857', scoreColor: '#064E3B',
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

    // Parse penalty minutes from penalty_type_string (e.g., "2 min", "2+2 min", "5 min", "10 min")
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

    // Calculate total penalty minutes from scorer.json penalty fields
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
            'period1': { label: '1st Period', color: 'bg-blue-500' },
            'period2': { label: '2nd Period', color: 'bg-blue-500' },
            'period3': { label: '3rd Period', color: 'bg-blue-500' },
            'pause1': { label: '1st Pause', color: 'bg-amber-500' },
            'pause2': { label: '2nd Pause', color: 'bg-amber-500' },
            'pause_et': { label: 'OT Pause', color: 'bg-amber-500' },
            'pause_ps': { label: 'PS Pause', color: 'bg-amber-500' },
            'extratime': { label: 'Extra Time', color: 'bg-purple-500' },
            'penalty_shots': { label: 'Penalty Shots', color: 'bg-red-500' },
        };
        return statusMap[status] ?? { label: gameData?.ended ? 'Final' : 'Scheduled', color: gameData?.ended ? 'bg-slate-500' : 'bg-green-500' };
    }

    function getEventStyle(eventType: string): { icon: string; bg: string; text: string } {
        switch (eventType) {
            case 'goal': return { icon: '🥅', bg: 'bg-emerald-100', text: 'text-emerald-700' };
            case 'penalty': return { icon: '🟨', bg: 'bg-amber-100', text: 'text-amber-700' };
            case 'timeout': return { icon: '⏱️', bg: 'bg-slate-200', text: 'text-slate-600' };
            default: return { icon: '📋', bg: 'bg-blue-100', text: 'text-blue-700' };
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

    onMount(() => {
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
</script>

<div class="w-full min-h-screen bg-slate-100">
    <!-- Sentinel element for detecting when header becomes sticky -->
    <div bind:this={headerSentinelRef} class="h-0"></div>
    
    <!-- Sticky Score Header - bigger when in view, smaller when sticky -->
    {#if gameData}
        {@const ingameStatus = getIngameStatus()}
        <div class="sticky top-0 z-50 bg-white/95 backdrop-blur-sm shadow-md border-b border-slate-300 transition-all duration-200 {headerIsSticky ? 'py-2' : 'py-4 sm:py-6'}">
            <div class="px-3 sm:px-6">
                <div class="flex items-center justify-center gap-2 sm:gap-6 max-w-5xl mx-auto">
                    <!-- Home Team -->
                    <div class="flex items-center gap-2 sm:gap-3 flex-1 justify-end min-w-0">
                        <span class="font-bold text-right transition-all {headerIsSticky ? 'text-sm sm:text-lg truncate' : 'text-base sm:text-xl leading-tight'}" style="color: {homeTeamColors.nameColor}">{gameData.home_team_name}</span>
                        {#if homeTeamLogoUrl}<img src={homeTeamLogoUrl} alt="" class="object-contain shrink-0 transition-all {headerIsSticky ? 'w-8 h-8 sm:w-10 sm:h-10' : 'w-12 h-12 sm:w-16 sm:h-16'}" />{/if}
                    </div>
                    <!-- Score -->
                    <div class="flex flex-col items-center shrink-0 px-2 sm:px-4">
                        <div class="font-black font-mono flex items-center gap-1 sm:gap-2 transition-all {headerIsSticky ? 'text-2xl sm:text-4xl' : 'text-4xl sm:text-6xl'}">
                            <span style="color: {homeTeamColors.scoreColor}">{gameData.result?.home_goals ?? '-'}</span>
                            <span class="text-slate-300">:</span>
                            <span style="color: {guestTeamColors.scoreColor}">{gameData.result?.guest_goals ?? '-'}</span>
                        </div>
                        <span class="{ingameStatus.color} text-white font-bold px-2 py-0.5 rounded-full uppercase transition-all {headerIsSticky ? 'text-[10px] sm:text-xs' : 'text-xs sm:text-sm'}">{ingameStatus.label}</span>
                        {#if !headerIsSticky && gameData.result?.home_goals_period}
                            <div class="mt-2 flex flex-wrap justify-center gap-1 sm:gap-2 text-xs sm:text-sm font-mono">
                                {#each gameData.result.home_goals_period as hg, i (i)}
                                    {#if gameData.result.guest_goals_period[i] !== undefined && (hg > 0 || gameData.result.guest_goals_period[i] > 0 || i < 3)}
                                        <span class="bg-slate-100 px-2 py-1 rounded">{i === 3 ? 'OT' : `P${i+1}`}: {hg}-{gameData.result.guest_goals_period[i]}</span>
                                    {/if}
                                {/each}
                            </div>
                        {/if}
                    </div>
                    <!-- Guest Team -->
                    <div class="flex items-center gap-2 sm:gap-3 flex-1 justify-start min-w-0">
                        {#if guestTeamLogoUrl}<img src={guestTeamLogoUrl} alt="" class="object-contain shrink-0 transition-all {headerIsSticky ? 'w-8 h-8 sm:w-10 sm:h-10' : 'w-12 h-12 sm:w-16 sm:h-16'}" />{/if}
                        <span class="font-bold transition-all {headerIsSticky ? 'text-sm sm:text-lg truncate' : 'text-base sm:text-xl leading-tight'}" style="color: {guestTeamColors.nameColor}">{gameData.guest_team_name}</span>
                    </div>
                </div>
            </div>
        </div>
    {/if}

    <!-- Navigation Header -->
    <header class="w-full px-4 py-3 bg-white border-b border-slate-300 shadow-sm">
        <div class="flex flex-col sm:flex-row items-start sm:items-center justify-between gap-3">
            <div class="flex items-center gap-3 w-full sm:w-auto">
                <h1 class="text-xl sm:text-2xl font-extrabold text-slate-900 whitespace-nowrap">Match <span class="text-blue-600">Details</span></h1>
                {#if gameId}<span class="text-slate-400 text-xs font-mono hidden sm:inline">#{gameId}</span>{/if}
            </div>
            <div class="flex gap-2 items-center w-full sm:w-auto">
                <input type="text" bind:value={manualGameId} placeholder="Game ID" class="flex-1 sm:flex-none sm:w-28 px-3 py-2 border border-slate-300 rounded-lg font-mono text-sm" onkeydown={(e) => e.key === 'Enter' && handleManualLoad()} />
                <button onclick={handleManualLoad} disabled={isLoading} class="px-4 py-2 bg-blue-600 text-white font-bold text-sm rounded-lg hover:bg-blue-700 disabled:opacity-50 whitespace-nowrap">{isLoading ? '...' : 'Load'}</button>
                <a href="#/MatchSelect" class="text-sm font-medium text-slate-500 hover:text-blue-600 whitespace-nowrap hidden sm:block">← Back</a>
            </div>
        </div>
    </header>

    <div class="w-full px-3 sm:px-6 py-4">
        {#if isLoading}
            <div class="flex flex-col items-center justify-center p-12 sm:p-20 bg-white rounded-2xl shadow border border-slate-300">
                <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-600 mb-4"></div>
                <p class="text-slate-400 text-base">Loading match data...</p>
            </div>
        {:else if error}
            <div class="p-8 text-center bg-red-50 rounded-2xl border border-slate-300">
                <p class="text-red-500 text-base font-medium">{error}</p>
            </div>
        {:else if !gameId}
            <div class="p-12 text-center bg-white rounded-2xl border border-slate-300 shadow">
                <div class="text-6xl mb-4">🏒</div>
                <p class="text-slate-500 text-base">Enter a Game ID above</p>
            </div>
        {:else if gameData}
            <!-- Team Form Timelines with scores and clickable -->
            {#if scheduleData.length > 0}
                {@const homeForm = getTeamForm(gameData.home_team_name, gameData.game_id)}
                {@const guestForm = getTeamForm(gameData.guest_team_name, gameData.game_id)}
                <div class="grid grid-cols-1 lg:grid-cols-2 gap-4 mb-4">
                    {#each [[gameData.home_team_name, homeForm, homeTeamColors], [gameData.guest_team_name, guestForm, guestTeamColors]] as [teamName, form, colors]}
                        <div class="bg-white rounded-xl border border-slate-300 p-3 sm:p-4">
                            <div class="flex items-center gap-2 mb-3">
                                <h4 class="font-bold text-sm" style="color: {colors.primary}">Recent Form</h4>
                                <span class="text-xs text-slate-400">{teamName}</span>
                            </div>
                            {#if form.length}
                                <div class="flex gap-1 items-end h-16">
                                    {#each form as game, i (i)}
                                        <button 
                                            type="button"
                                            onclick={() => navigateToGame(game.gameId)} 
                                            class="relative flex-1 flex flex-col items-center group cursor-pointer hover:scale-110 transition-transform" 
                                            title="{game.score} vs {game.opponent} ({game.date})"
                                        >
                                            <div class="w-full h-12 flex flex-col justify-center">
                                                {#if game.result === 'W'}
                                                    <div class="w-full h-8 rounded-t bg-emerald-500 self-start flex items-center justify-center">
                                                        <span class="text-[9px] font-bold text-white">{game.score}</span>
                                                    </div>
                                                {:else if game.result === 'D'}
                                                    <div class="w-full h-4 rounded bg-slate-400 self-center flex items-center justify-center">
                                                        <span class="text-[8px] font-bold text-white">{game.score}</span>
                                                    </div>
                                                {:else}
                                                    <div class="w-full h-8 rounded-b bg-red-500 self-end flex items-center justify-center">
                                                        <span class="text-[9px] font-bold text-white">{game.score}</span>
                                                    </div>
                                                {/if}
                                            </div>
                                            <span class="text-[10px] font-mono text-slate-400 group-hover:text-slate-700">{game.result}</span>
                                            <!-- Tooltip -->
                                            <div class="hidden group-hover:block absolute bottom-full left-1/2 -translate-x-1/2 mb-1 px-2 py-1 bg-slate-800 text-white text-[10px] rounded whitespace-nowrap z-10">
                                                {game.score} vs {game.opponent.length > 15 ? game.opponent.slice(0, 15) + '...' : game.opponent}
                                            </div>
                                        </button>
                                    {/each}
                                </div>
                            {:else}
                                <p class="text-xs text-slate-400">No recent games</p>
                            {/if}
                        </div>
                    {/each}
                </div>
            {/if}

            <!-- Info Row -->
            <div class="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-5 gap-2 sm:gap-3 mb-4">
                <div class="bg-white rounded-lg p-2 sm:p-3 border border-slate-300"><div class="text-[10px] sm:text-xs font-bold text-slate-400 uppercase">Venue</div><div class="font-semibold text-sm sm:text-base truncate">{gameData.arena_name ?? '-'}</div></div>
                <div class="bg-white rounded-lg p-2 sm:p-3 border border-slate-300"><div class="text-[10px] sm:text-xs font-bold text-slate-400 uppercase">League</div><div class="font-semibold text-sm sm:text-base truncate">{gameData.league_name ?? gameData.league_short_name ?? '-'}</div></div>
                <div class="bg-white rounded-lg p-2 sm:p-3 border border-slate-300"><div class="text-[10px] sm:text-xs font-bold text-slate-400 uppercase">Game Day</div><div class="font-semibold text-sm sm:text-base truncate">{gameData.game_day?.title ?? '-'}</div></div>
                <div class="bg-white rounded-lg p-2 sm:p-3 border border-slate-300"><div class="text-[10px] sm:text-xs font-bold text-slate-400 uppercase">Referees</div><div class="font-semibold text-sm sm:text-base truncate">{gameData.nominated_referees ?? '-'}</div></div>
                <div class="bg-white rounded-lg p-2 sm:p-3 border border-slate-300 col-span-2 sm:col-span-1"><div class="text-[10px] sm:text-xs font-bold text-slate-400 uppercase">Date/Time</div><div class="font-semibold text-sm sm:text-base">{gameData.date} • {gameData.start_time}</div></div>
            </div>

            <!-- Color Config -->
            <details class="mb-4 bg-white rounded-xl border border-slate-300 overflow-hidden">
                <summary class="px-4 py-3 text-sm font-semibold text-slate-600 cursor-pointer hover:bg-slate-50">🎨 Team Colors</summary>
                <div class="p-4 border-t border-slate-300 grid grid-cols-1 md:grid-cols-2 gap-6">
                    {#each [['home', homeTeamColors], ['guest', guestTeamColors]] as [team, colors]}
                        <div class="space-y-3">
                            <h4 class="font-bold" style="color: {colors.primary}">{team === 'home' ? 'Home' : 'Guest'} Team</h4>
                            <div class="grid grid-cols-4 gap-2">
                                <label class="text-xs text-slate-500">Primary<input type="color" bind:value={colors.primary} class="w-full h-10 rounded cursor-pointer border-0" /></label>
                                <label class="text-xs text-slate-500">Secondary<input type="color" bind:value={colors.secondary} class="w-full h-10 rounded cursor-pointer border-0" /></label>
                                <label class="text-xs text-slate-500">Name<input type="color" bind:value={colors.nameColor} class="w-full h-10 rounded cursor-pointer border-0" /></label>
                                <label class="text-xs text-slate-500">Score<input type="color" bind:value={colors.scoreColor} class="w-full h-10 rounded cursor-pointer border-0" /></label>
                                {#each colors.additionalColors as c, i (i)}
                                    <label class="text-xs text-slate-500 relative">Extra {i+1}<input type="color" bind:value={colors.additionalColors[i]} class="w-full h-10 rounded cursor-pointer border-0" /><button onclick={() => removeColor(team, i)} class="absolute top-0 right-0 text-red-500 font-bold">×</button></label>
                                {/each}
                            </div>
                            <button onclick={() => addColor(team)} class="text-sm text-blue-600 hover:underline">+ Add</button>
                        </div>
                    {/each}
                </div>
            </details>

            <!-- Rosters Side by Side -->
            <div class="grid grid-cols-1 lg:grid-cols-2 gap-4 mb-4">
                {#each [['home', gameData.players?.home, homeTeamColors, homeTeamLogoUrl], ['guest', gameData.players?.guest, guestTeamColors, guestTeamLogoUrl]] as [team, roster, colors, logo]}
                    <div class="bg-white rounded-xl border border-slate-300 overflow-hidden">
                        <div class="px-4 py-3 border-b border-slate-300 flex items-center gap-3" style="background: {colors.primary}20">
                            {#if logo}<img src={logo} alt="" class="w-8 h-8 object-contain" />{/if}
                            <h3 class="font-bold text-base sm:text-lg truncate" style="color: {colors.primary}">{team === 'home' ? gameData.home_team_name : gameData.guest_team_name}</h3>
                            <span class="text-sm text-slate-400 ml-auto shrink-0">{roster?.length ?? 0}</span>
                        </div>
                        <div class="p-2 sm:p-3">
                            {#if roster?.length}
                                <div class="grid grid-cols-1 sm:grid-cols-2 gap-1 sm:gap-2">
                                    {#each roster as player, i (i)}
                                        {@const stats = getPlayerGameStats(team, player.trikot_number)}
                                        {@const isStarter = isStartingPlayer(team, player.player_id)}
                                        {@const isSelected = selectedPlayer?.player_id === player.player_id}
                                        {@const seasonStats = isSelected ? getPlayerSeasonStats(player.player_id) : null}
                                        {@const seasonPenaltyMins = seasonStats ? calculateSeasonPenaltyMinutes(seasonStats) : 0}
                                        <button 
                                            type="button" 
                                            onclick={() => selectPlayer(player)} 
                                            class="player-card flex flex-col text-left p-2 rounded-lg transition-all cursor-pointer {isSelected ? 'bg-blue-50 shadow-md selected' : ''}"
                                        >
                                            <div class="flex items-center gap-2">
                                                <div class="w-7 h-7 rounded flex items-center justify-center font-bold text-sm text-white shrink-0" style="background: {player.goalkeeper ? '#8B5CF6' : colors.primary}">{player.trikot_number}</div>
                                                <span class="font-medium text-sm sm:text-base text-slate-800 truncate flex-1">{player.player_firstname} {player.player_name}</span>
                                                {#if player.captain}<span class="text-[10px] font-bold text-amber-600 bg-amber-50 px-1 rounded shrink-0">C</span>{/if}
                                                {#if player.goalkeeper}<span class="text-[10px] font-bold text-purple-600 bg-purple-50 px-1 rounded shrink-0">GK</span>{/if}
                                                {#if isStarter}<span class="text-[10px] font-bold text-green-700 bg-green-100 px-1.5 py-0.5 rounded shrink-0">Starting</span>{/if}
                                            </div>
                                            <!-- Game stats with padding -->
                                            <div class="flex gap-2 mt-1.5 pl-9">
                                                {#if stats.goals > 0}<span class="bg-emerald-100 text-emerald-700 px-2 py-0.5 rounded font-bold text-xs">🥅 {stats.goals}</span>{/if}
                                                {#if stats.assists > 0}<span class="bg-blue-100 text-blue-700 px-2 py-0.5 rounded font-bold text-xs">🅰️ {stats.assists}</span>{/if}
                                                {#if stats.penaltyMinutes > 0}<span class="bg-amber-100 text-amber-700 px-2 py-0.5 rounded font-bold text-xs">🟨 {stats.penaltyMinutes}min</span>{/if}
                                            </div>
                                            {#if isSelected && seasonStats}
                                                <div class="mt-2 p-2 bg-slate-100 rounded text-xs ml-9">
                                                    <div class="font-bold text-slate-600 mb-1">Season Stats ({seasonStats.games ?? 0} games):</div>
                                                    <div class="grid grid-cols-4 gap-1 text-center">
                                                        <div><div class="font-bold text-base text-emerald-600">{seasonStats.goals ?? 0}</div><div class="text-[10px] text-slate-400">Goals</div></div>
                                                        <div><div class="font-bold text-base text-blue-600">{seasonStats.assists ?? 0}</div><div class="text-[10px] text-slate-400">Assists</div></div>
                                                        <div><div class="font-bold text-base text-slate-600">{(seasonStats.goals ?? 0) + (seasonStats.assists ?? 0)}</div><div class="text-[10px] text-slate-400">Points</div></div>
                                                        <div><div class="font-bold text-base text-amber-600">{seasonPenaltyMins}</div><div class="text-[10px] text-slate-400">Pen.Min</div></div>
                                                    </div>
                                                </div>
                                            {/if}
                                        </button>
                                    {/each}
                                </div>
                            {:else}
                                <p class="text-slate-400 text-center py-6">No roster</p>
                            {/if}
                        </div>
                    </div>
                {/each}
            </div>

            <!-- Timeline Events with Period Separators -->
            <div class="bg-white rounded-xl border border-slate-300 overflow-hidden mb-4">
                <div class="bg-slate-50 px-4 py-3 border-b border-slate-300 flex items-center justify-between">
                    <h3 class="font-bold text-base sm:text-lg text-slate-800">Game Timeline</h3>
                    <span class="text-sm text-slate-400">{gameData.events?.length ?? 0}</span>
                </div>
                <div class="p-3 sm:p-6">
                    {#if gameData.events?.length}
                        {@const periodGroups = getEventsGroupedByPeriod()}
                        <div class="relative">
                            <div class="absolute left-1/2 top-0 bottom-0 w-0.5 sm:w-1 bg-slate-200 -translate-x-1/2 z-0"></div>
                            
                            <div class="space-y-0">
                                {#each periodGroups as group, groupIdx (groupIdx)}
                                    <!-- Period Separator - z-10 to be in front of the vertical line -->
                                    <div class="relative z-10 flex items-center justify-center py-3 sm:py-4">
                                        <div class="flex-1 h-0.5 bg-slate-300"></div>
                                        <span class="px-4 py-1 bg-slate-700 text-white font-bold text-sm rounded-full mx-4">
                                            {group.period === 4 ? 'OT' : `Period ${group.period}`}
                                        </span>
                                        <div class="flex-1 h-0.5 bg-slate-300"></div>
                                    </div>
                                    
                                    <div class="space-y-3 sm:space-y-4">
                                        {#each group.events as event, i (i)}
                                            {@const style = getEventStyle(event.event_type)}
                                            {@const isHome = event.event_team === 'home'}
                                            {@const roster = isHome ? gameData.players?.home : gameData.players?.guest}
                                            {@const teamColor = isHome ? homeTeamColors.primary : guestTeamColors.primary}
                                            {@const teamLogo = isHome ? homeTeamLogoUrl : guestTeamLogoUrl}
                                            {@const isGoal = event.event_type === 'goal'}
                                            
                                            <div class="relative flex items-stretch {isHome ? 'flex-row' : 'flex-row-reverse'}">
                                                <!-- Event card - spans from edge to center time -->
                                                <div class="w-[calc(50%-2rem)] sm:w-[calc(50%-3rem)] {style.bg} rounded-lg sm:rounded-xl p-2 sm:p-4 border border-slate-200">
                                                    <div class="flex items-center gap-2 sm:gap-3 mb-1 sm:mb-2 {isHome ? '' : 'flex-row-reverse'}">
                                                        <span class="text-lg sm:text-2xl">{style.icon}</span>
                                                        {#if teamLogo}
                                                            <img src={teamLogo} alt="" class="w-5 h-5 sm:w-6 sm:h-6 object-contain" />
                                                        {:else}
                                                            <span class="font-bold text-xs uppercase" style="color: {teamColor}">{isHome ? 'HOME' : 'GUEST'}</span>
                                                        {/if}
                                                        <span class="font-bold text-xs sm:text-base {style.text} capitalize">{event.event_type}</span>
                                                        <!-- Score only on goals, inline -->
                                                        {#if isGoal && event.home_goals !== undefined}
                                                            <span class="font-mono font-bold text-sm sm:text-lg text-slate-700 {isHome ? 'ml-auto' : 'mr-auto'}">{event.home_goals}:{event.guest_goals}</span>
                                                        {/if}
                                                    </div>
                                                    <div class="text-xs sm:text-base text-slate-700 {isHome ? '' : 'text-right'}">
                                                        {#if event.number}
                                                            <span class="font-mono bg-slate-200 px-1 sm:px-1.5 rounded text-xs sm:text-sm">#{event.number}</span>
                                                            <span class="font-semibold ml-1">{getPlayerName(roster, event.number)}</span>
                                                        {/if}
                                                        {#if event.assist}
                                                            <span class="text-slate-500 ml-1 sm:ml-2 text-xs sm:text-sm">(A: #{event.assist} {getPlayerName(roster, event.assist)})</span>
                                                        {/if}
                                                        {#if event.penalty_reason_string}
                                                            <span class="text-slate-500 block mt-1 text-xs sm:text-sm">{event.penalty_type_string} - {event.penalty_reason_string}</span>
                                                        {/if}
                                                    </div>
                                                </div>
                                                
                                                <!-- Time badge spanning the middle -->
                                                <div class="w-16 sm:w-24 flex items-center justify-center shrink-0 relative z-10">
                                                    <div class="bg-white border-2 border-slate-300 rounded-full px-2 sm:px-3 py-0.5 sm:py-1 font-mono text-[10px] sm:text-sm font-bold text-slate-600 whitespace-nowrap shadow-sm">
                                                        {event.time}
                                                    </div>
                                                </div>
                                                
                                                <!-- Empty space on opposite side -->
                                                <div class="w-[calc(50%-2rem)] sm:w-[calc(50%-3rem)]"></div>
                                            </div>
                                        {/each}
                                    </div>
                                {/each}
                            </div>
                        </div>
                    {:else}
                        <p class="text-slate-400 text-center py-10">No events</p>
                    {/if}
                </div>
            </div>

            <!-- League Table -->
            {#if leagueTable.length > 0 || (groupedTable && Object.keys(groupedTable).length > 0)}
                <div class="bg-white rounded-xl border border-slate-300 overflow-hidden mb-4">
                    <div class="bg-slate-50 px-4 py-3 border-b border-slate-300">
                        <h3 class="font-bold text-base sm:text-lg text-slate-800">League Table</h3>
                    </div>
                    <div class="p-2 sm:p-4 overflow-x-auto">
                        {#if isGroupedTable && groupedTable}
                            {#each Object.entries(groupedTable) as [groupName, teams]}
                                <h4 class="font-bold text-sm text-slate-700 mb-2 mt-4 first:mt-0">{groupName}</h4>
                                {@render tableRows(teams)}
                            {/each}
                        {:else}
                            {@render tableRows(leagueTable)}
                        {/if}
                    </div>
                </div>
            {/if}

            <!-- Debug -->
            <details class="bg-slate-800 rounded-xl overflow-hidden border border-slate-300">
                <summary class="px-4 py-2 text-slate-300 text-sm cursor-pointer hover:bg-slate-700">🔍 Raw JSON</summary>
                <pre class="p-4 text-xs text-slate-400 overflow-x-auto max-h-64">{JSON.stringify(gameData, null, 2)}</pre>
            </details>
        {/if}
    </div>
</div>

{#snippet tableRows(teams)}
    <table class="w-full text-xs sm:text-sm">
        <thead>
            <tr class="text-left text-slate-500 text-[10px] sm:text-xs uppercase">
                <th class="p-1 sm:p-2">#</th>
                <th class="p-1 sm:p-2">Team</th>
                <th class="p-1 sm:p-2 text-center hidden sm:table-cell">GP</th>
                <th class="p-1 sm:p-2 text-center">W</th>
                <th class="p-1 sm:p-2 text-center hidden sm:table-cell">D</th>
                <th class="p-1 sm:p-2 text-center">L</th>
                <th class="p-1 sm:p-2 text-center hidden md:table-cell">GF</th>
                <th class="p-1 sm:p-2 text-center hidden md:table-cell">GA</th>
                <th class="p-1 sm:p-2 text-center">GD</th>
                <th class="p-1 sm:p-2 text-center font-bold">Pts</th>
            </tr>
        </thead>
        <tbody>
            {#each teams as team, i (i)}
                {@const isMatchTeam = team.team_id === gameData.home_team_id || team.team_id === gameData.guest_team_id}
                {@const isHome = team.team_id === gameData.home_team_id}
                <tr class="border-t {isMatchTeam ? (isHome ? 'bg-blue-50' : 'bg-emerald-50') : ''} hover:bg-slate-50">
                    <td class="p-1 sm:p-2 font-bold text-slate-600">{team.position ?? i + 1}</td>
                    <td class="p-1 sm:p-2">
                        <div class="flex items-center gap-1 sm:gap-2">
                            {#if team.team_logo}<img src="https://saisonmanager.de{team.team_logo}" alt="" class="w-4 h-4 sm:w-5 sm:h-5 object-contain shrink-0" />{/if}
                            <span class="font-medium truncate max-w-[100px] sm:max-w-none {isMatchTeam ? 'font-bold' : ''}" style="color: {isHome ? homeTeamColors.primary : (team.team_id === gameData.guest_team_id ? guestTeamColors.primary : 'inherit')}">{team.team_name}</span>
                        </div>
                    </td>
                    <td class="p-1 sm:p-2 text-center hidden sm:table-cell">{team.games}</td>
                    <td class="p-1 sm:p-2 text-center">{team.won}</td>
                    <td class="p-1 sm:p-2 text-center hidden sm:table-cell">{team.draw}</td>
                    <td class="p-1 sm:p-2 text-center">{team.lost}</td>
                    <td class="p-1 sm:p-2 text-center hidden md:table-cell">{team.goals_scored}</td>
                    <td class="p-1 sm:p-2 text-center hidden md:table-cell">{team.goals_received}</td>
                    <td class="p-1 sm:p-2 text-center {team.goals_diff > 0 ? 'text-emerald-600' : team.goals_diff < 0 ? 'text-red-500' : ''}">{team.goals_diff > 0 ? '+' : ''}{team.goals_diff}</td>
                    <td class="p-1 sm:p-2 text-center font-bold text-base sm:text-lg">{team.points}</td>
                </tr>
            {/each}
        </tbody>
    </table>
{/snippet}

<style>
    summary::-webkit-details-marker { display: none; }
    summary { list-style: none; }
    
    /* Player card hover styles */
    .player-card {
        border: 1px solid transparent;
    }
    .player-card:hover {
        border-color: #94a3b8; /* slate-400 */
        background-color: #f8fafc; /* slate-50 */
        box-shadow: 0 1px 2px 0 rgb(0 0 0 / 0.05);
    }
    /* No special border for starters anymore */
    .player-card.selected {
        border-color: #3b82f6; /* blue-500 */
    }
</style>
