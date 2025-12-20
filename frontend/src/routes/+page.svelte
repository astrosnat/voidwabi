<script lang="ts">
	import { onDestroy, onMount } from 'svelte';
	import { initSocket, disconnect, dmPanelSignal, type User } from '$lib/socket';
	import { requestNotificationPermission } from '$lib/notifications';
	import Chat from '$lib/components/Chat.svelte';
	import Login from '$lib/components/Login.svelte';
	import ChannelSidebar from '$lib/components/ChannelSidebar.svelte';
	import UserPanel from '$lib/components/UserPanel.svelte';
	import ScreenShareViewer from '$lib/components/ScreenShareViewer.svelte';
	import CallModal from '$lib/components/CallModal.svelte';
	import DMPanel from '$lib/components/DMPanel.svelte';
	import type { PageData } from './$types';

	export let data: PageData;

	let username = '';
	let loggedIn = false;
	let activeView: 'chat' | 'screen' = 'chat';

	// --- Unified Panel State ---
	type RightPanelView = 'none' | 'users' | 'dm';
	let rightPanelView: RightPanelView = 'none';
	
	let dmChannelId: string | null = null;
	let dmOtherUser: User | null = null;
	
	// --- UI & Layout State ---
	type SidebarMode = 'normal' | 'compact' | 'closed';
	let sidebarMode: SidebarMode = 'normal';
	let channelSidebarWidth = 240;
	let userPanelWidth = 250;
	let dmPanelWidth = 350;
	let isResizingChannel = false;
	let isResizingUser = false;
	let isResizingDM = false;
	let isMobile = false;
	let showMobileChannels = false;
	let mql: MediaQueryList;
	let resizeStartX = 0;
	let resizeStartWidth = 0;
	
	// --- Reactive Calculations ---
	// These are now purely for desktop visibility, mobile uses different classes
	$: showUserPanel = rightPanelView === 'users' && !isMobile;
	$: showDMPanel = rightPanelView === 'dm' && !isMobile;
	$: toggleButtonRight = (showUserPanel ? userPanelWidth : 0) + (showDMPanel ? dmPanelWidth : 0);

	// --- Lifecycle ---
	onMount(async () => {
		const notificationsEnabled = localStorage.getItem('notificationsEnabled') !== 'false';
		if (notificationsEnabled) await requestNotificationPermission();
		
		mql = window.matchMedia('(max-width: 768px)');
		isMobile = mql.matches;
		mql.addEventListener('change', (e) => {
			isMobile = e.matches;
			if (!isMobile) rightPanelView = 'none'; // Reset panels on resize to desktop
		});
	});
	
	onDestroy(() => {
		if (mql) mql.removeEventListener('change', (e) => (isMobile = e.matches));
		disconnect();
	});

	// --- Event Handlers & Logic ---

	// Open DM panel when signaled from another component (e.g., notification click)
	$: if ($dmPanelSignal) {
		dmChannelId = $dmPanelSignal.channelId;
		dmOtherUser = $dmPanelSignal.otherUser;
		rightPanelView = 'dm';
		dmPanelSignal.set(null);
	}

	function handleLogin(event: CustomEvent<string>) {
		username = event.detail;
		initSocket(username);
		loggedIn = true;
	}
	
	// Desktop resizing with snap points
	function startResizeChannel(e: MouseEvent) {
		if (sidebarMode === 'closed') return;
		isResizingChannel = true;
		resizeStartX = e.clientX;
		resizeStartWidth = channelSidebarWidth;
		e.preventDefault();
	}

	function startResizeUser(e: MouseEvent) {
		isResizingUser = true;
		resizeStartX = e.clientX;
		resizeStartWidth = userPanelWidth;
		e.preventDefault();
	}

	function startResizeDM(e: MouseEvent) {
		isResizingDM = true;
		resizeStartX = e.clientX;
		resizeStartWidth = dmPanelWidth;
		e.preventDefault();
	}

	function handleMouseMove(e: MouseEvent) {
		if (isResizingChannel) {
			const delta = e.clientX - resizeStartX;
			const newWidth = resizeStartWidth + delta;
			channelSidebarWidth = Math.max(0, Math.min(newWidth, 400));
		} else if (isResizingUser) {
			const delta = resizeStartX - e.clientX;
			const newWidth = resizeStartWidth + delta;
			userPanelWidth = Math.max(200, Math.min(newWidth, 500));
		} else if (isResizingDM) {
			const delta = resizeStartX - e.clientX;
			const newWidth = resizeStartWidth + delta;
			dmPanelWidth = Math.max(300, Math.min(newWidth, 600));
		}
	}

	function stopResize() {
		isResizingChannel = false;
		isResizingUser = false;
		isResizingDM = false;
	}

	function toggleSidebarMode() {
		if (sidebarMode === 'normal') {
			sidebarMode = 'compact';
			channelSidebarWidth = 70;
		} else if (sidebarMode === 'compact') {
			sidebarMode = 'closed';
			channelSidebarWidth = 0;
		} else {
			sidebarMode = 'normal';
			channelSidebarWidth = 240;
		}
	}

	function toggleSidebarClosed() {
		if (sidebarMode === 'closed') {
			sidebarMode = 'normal';
			channelSidebarWidth = 240;
		} else {
			sidebarMode = 'closed';
			channelSidebarWidth = 0;
		}
	}
	
	// Panel State Transitions
	function handleOpenDM(event: CustomEvent<{ channelId: string; otherUser: User }>) {
		dmChannelId = event.detail.channelId;
		dmOtherUser = event.detail.otherUser;
		rightPanelView = 'dm';
	}

	function handleCloseDM() {
		dmChannelId = null;
		dmOtherUser = null;
		// On mobile, go back to the user list. On desktop, just close.
		rightPanelView = isMobile ? 'users' : 'none';
	}

	function handleSelectDM(channelId: string, user: User) {
		dmChannelId = channelId;
		dmOtherUser = user;
	}

	function toggleDesktopUserPanel() {
		if (rightPanelView === 'users') {
			rightPanelView = 'none';
		} else {
			// If a DM is open, this button should still open the user list, replacing the DM
			rightPanelView = 'users';
		}
	}
	
	// Mobile Navigation
	function toggleMobileChannels() {
		showMobileChannels = !showMobileChannels;
		if (showMobileChannels) rightPanelView = 'none';
	}
	
	function toggleMobileUsers() {
		// This button now acts as a master toggle for the right-side panels
		if (rightPanelView === 'users' || rightPanelView === 'dm') {
			rightPanelView = 'none'; // Close whatever is open
		} else {
			rightPanelView = 'users'; // Open the user list
			showMobileChannels = false;
		}
	}
</script>

<svelte:window on:mousemove={handleMouseMove} on:mouseup={stopResize} />

{#if !loggedIn}
	<Login on:login={handleLogin} />
{:else}
	{#if isMobile}
		<!-- Mobile Bottom Navigation Bar -->
		<nav class="mobile-bottom-nav">
			<button class:active={!showMobileChannels && rightPanelView === 'none'} on:click={() => { showMobileChannels = false; rightPanelView = 'none'; }}>
				<svg width="24" height="24" viewBox="0 0 24 24"><path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"/></svg>
				<span>Chat</span>
			</button>
			<button class:active={showMobileChannels} on:click={toggleMobileChannels}>
				<svg width="24" height="24" viewBox="0 0 24 24"><line x1="4" y1="6" x2="20" y2="6"/><line x1="4" y1="12" x2="20" y2="12"/><line x1="4" y1="18" x2="20" y2="18"/></svg>
				<span>Channels</span>
			</button>
			<button class:active={rightPanelView === 'users' || rightPanelView === 'dm'} on:click={toggleMobileUsers}>
				<svg width="24" height="24" viewBox="0 0 24 24"><path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/></svg>
				<span>Users</span>
			</button>
			<a href="/business" class="nav-link">
				<svg width="24" height="24" viewBox="0 0 24 24"><rect x="3" y="3" width="7" height="7"/><rect x="14" y="3" width="7" height="7"/><rect x="14" y="14" width="7" height="7"/><rect x="3" y="14" width="7" height="7"/></svg>
				<span>Hub</span>
			</a>
		</nav>
	{/if}

	<div class="app-container" class:resizing={isResizingChannel || isResizingUser || isResizingDM}>
		<!-- Channel Sidebar (Left) -->
		{#if sidebarMode !== 'closed'}
			<div
				class="channel-sidebar-container"
				style:width="{channelSidebarWidth}px"
				class:mobile-visible={isMobile && showMobileChannels}
				class:compact={sidebarMode === 'compact'}
			>
				<ChannelSidebar on:close={() => showMobileChannels = false} on:toggleMode={toggleSidebarMode} on:toggleClosed={toggleSidebarClosed} bind:activeView sidebarWidth={channelSidebarWidth} />
				{#if !isMobile && sidebarMode === 'normal'}
					<div class="resize-handle resize-handle-channel" on:mousedown={startResizeChannel}></div>
				{/if}
			</div>
		{:else}
			<!-- Sidebar Toggle Button (when closed) -->
			{#if !isMobile}
				<button class="sidebar-toggle-btn" on:click={toggleSidebarMode} title="Open sidebar">☰</button>
			{/if}
		{/if}

		<!-- Main Content -->
		<div class="main-content">
			<div class:hidden={activeView !== 'chat'}><Chat /></div>
			<div class:hidden={activeView !== 'screen'}><ScreenShareViewer bind:activeView /></div>
		</div>
		
		<!-- User Panel (Right) -->
		<div
			class="user-panel-container"
			class:visible={showUserPanel}
			style:width="{showUserPanel ? userPanelWidth : 0}px"
			class:mobile-visible={isMobile && rightPanelView === 'users'}
		>
			<UserPanel on:openDM={handleOpenDM} on:close={() => rightPanelView = 'none'} />
			{#if !isMobile}
				<div class="resize-handle resize-handle-user" on:mousedown={startResizeUser}></div>
			{/if}
		</div>

		<!-- DM Panel (Far Right) -->
		<div
			class="dm-panel-container"
			class:visible={showDMPanel}
			style:width="{showDMPanel ? dmPanelWidth : 0}px"
			class:mobile-visible={isMobile && rightPanelView === 'dm'}
		>
			<DMPanel {dmChannelId} otherUser={dmOtherUser} onClose={handleCloseDM} onSelectDM={handleSelectDM} />
			{#if !isMobile}
				<div class="resize-handle resize-handle-dm" on:mousedown={startResizeDM}></div>
			{/if}
		</div>

		<!-- Desktop-Only Buttons -->
		{#if !isMobile}
			<button
				class="user-panel-toggle"
				class:open={showUserPanel || showDMPanel}
				on:click={toggleDesktopUserPanel}
				title={rightPanelView === 'users' ? 'Hide user panel' : 'Show user panel'}
				style:right="{toggleButtonRight}px"
			>
				{showUserPanel || showDMPanel ? '→' : '←'}
			</button>
		{/if}
	</div>
	<CallModal />
{/if}

<style>
	:global(body) {
		overflow: hidden;
	}
	.app-container {
		display: flex;
		height: 100vh;
		overflow: hidden;
		position: relative;
	}

	.app-container.resizing {
		cursor: col-resize;
		user-select: none;
	}
	
	.main-content {
		flex: 1;
		min-width: 0; /* Prevents flexbox overflow */
		position: relative;
	}

	.main-content > div { height: 100%; width: 100%; }
	.hidden { display: none !important; }

	.channel-sidebar-container {
		flex-shrink: 0;
		position: relative;
		overflow: hidden;
		transition: width 0.3s ease-out;
	}

	.sidebar-toggle-btn {
		flex-shrink: 0;
		width: 30px;
		height: 100%;
		background: var(--bg-secondary);
		border: 1px solid var(--border);
		border-radius: 0;
		cursor: pointer;
		font-size: 0.9rem;
		color: var(--text-secondary);
		transition: all 0.2s ease;
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 100;
		opacity: 0.7;
	}

	.sidebar-toggle-btn:hover {
		background: var(--bg-tertiary);
		color: var(--text-primary);
		opacity: 1;
		box-shadow: inset 0 0 4px rgba(123, 104, 238, 0.1);
	}

	/* Desktop Panel Styles */
	.user-panel-container,
	.dm-panel-container {
		flex-shrink: 0;
		position: relative;
		overflow: hidden;
		transition: width 0.2s ease-in-out;
		will-change: width;
	}

	.resize-handle {
		position: absolute;
		top: 0;
		bottom: 0;
		width: 12px;
		cursor: col-resize;
		z-index: 100;
		transition: background 0.2s;
		right: -6px;
		background: transparent;
	}
	.resize-handle:hover {
		background: var(--accent);
		opacity: 0.6;
	}
	.resize-handle-channel { }
	.resize-handle-user { left: -3px; }
	.resize-handle-dm { left: -3px; }
	
	.user-panel-toggle {
		position: fixed;
		right: 0;
		top: 50%;
		transform: translateY(-50%);
		width: 20px;
		height: 60px;
		background: var(--bg-secondary);
		border: 1px solid var(--border);
		border-radius: 8px 0 0 8px;
		cursor: pointer;
		font-size: 0.75rem;
		color: var(--text-secondary);
		transition: all 0.2s ease;
		z-index: 99;
		opacity: 0.5;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 0;
	}
	.user-panel-toggle:hover {
		opacity: 1;
		background: var(--bg-tertiary);
		box-shadow: inset 0 0 4px rgba(123, 104, 238, 0.15);
	}
	.user-panel-toggle.open {
		opacity: 0.7;
		background: var(--bg-secondary);
	}

	/* --- Mobile Styles --- */
	.mobile-bottom-nav { display: none; }
	
	@media (max-width: 768px) {
		.app-container { height: calc(100dvh - 56px); }
		.user-panel-toggle, .resize-handle { display: none; }

		.channel-sidebar-container,
		.user-panel-container,
		.dm-panel-container {
			display: none; /* Hidden by default */
			position: fixed;
			top: 0;
			left: 0;
			width: 100% !important; /* Override inline style */
			height: calc(100dvh - 56px);
			z-index: 1500;
			background: var(--bg-primary);
		}
		
		.channel-sidebar-container.mobile-visible,
		.user-panel-container.mobile-visible,
		.dm-panel-container.mobile-visible {
			display: block; /* Shown when active */
		}

		.mobile-bottom-nav {
			display: flex;
			justify-content: space-around;
			align-items: center;
			position: fixed;
			bottom: 0;
			left: 0;
			right: 0;
			height: 56px;
			background: var(--bg-tertiary);
			border-top: 1px solid var(--border);
			z-index: 2000;
			padding: 0;
			padding-bottom: env(safe-area-inset-bottom, 0);
		}
		.mobile-bottom-nav button, .mobile-bottom-nav .nav-link {
			display: flex;
			flex-direction: column;
			align-items: center;
			justify-content: center;
			gap: 0.125rem;
			background: transparent;
			border: none;
			color: var(--text-secondary);
			font-size: 0.6rem;
			padding: 0.375rem 0.5rem;
			text-decoration: none;
			transition: color 0.15s;
		}
		.mobile-bottom-nav button:hover, .mobile-bottom-nav .nav-link:hover { color: var(--text-primary); }
		.mobile-bottom-nav button.active { color: var(--accent); }
		.mobile-bottom-nav svg { width: 20px; height: 20px; stroke: currentColor; fill: none; stroke-width: 2; }
	}
</style>
