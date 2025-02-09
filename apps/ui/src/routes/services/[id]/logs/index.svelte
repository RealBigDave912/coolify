<script lang="ts">
	import { page } from '$app/stores';
	import LoadingLogs from './_Loading.svelte';
	import { get } from '$lib/api';
	import { t } from '$lib/translations';
	import { errorNotification } from '$lib/common';
	import { onDestroy, onMount } from 'svelte';
	import Tooltip from '$lib/components/Tooltip.svelte';

	let service: any = {};
	let logsLoading = false;
	let loadLogsInterval: any = null;
	let logs: any = [];
	let lastLog: any = null;
	let followingInterval: any;
	let followingLogs: any;
	let logsEl: any;
	let position = 0;

	const { id } = $page.params;

	onMount(async () => {
		const response = await get(`/services/${id}`);
		service = response.service;
		loadAllLogs();
		loadLogsInterval = setInterval(() => {
			loadLogs();
		}, 1000);
	});

	onDestroy(() => {
		clearInterval(loadLogsInterval);
		clearInterval(followingInterval);
	});
	async function loadAllLogs() {
		try {
			logsLoading = true;
			const data: any = await get(`/services/${id}/logs`);
			if (data?.logs) {
				lastLog = data.logs[data.logs.length - 1];
				logs = data.logs;
			}
		} catch (error) {
			return errorNotification(error);
		} finally {
			logsLoading = false;
		}
	}
	async function loadLogs() {
		if (logsLoading) return;
		try {
			const newLogs: any = await get(`/services/${id}/logs?since=${lastLog?.split(' ')[0] || 0}`);

			if (newLogs?.logs && newLogs.logs[newLogs.logs.length - 1] !== logs[logs.length - 1]) {
				logs = logs.concat(newLogs.logs);
				lastLog = newLogs.logs[newLogs.logs.length - 1];
			}
		} catch (error) {
			return errorNotification(error);
		}
	}
	function detect() {
		if (position < logsEl.scrollTop) {
			position = logsEl.scrollTop;
		} else {
			if (followingLogs) {
				clearInterval(followingInterval);
				followingLogs = false;
			}
			position = logsEl.scrollTop;
		}
	}

	function followBuild() {
		followingLogs = !followingLogs;
		if (followingLogs) {
			followingInterval = setInterval(() => {
				logsEl.scrollTop = logsEl.scrollHeight;
				window.scrollTo(0, document.body.scrollHeight);
			}, 1000);
		} else {
			clearInterval(followingInterval);
		}
	}
</script>

<div class="flex flex-row justify-center space-x-2 px-10 pt-6">
	{#if logs.length === 0}
		<div class="text-xl font-bold tracking-tighter">{$t('application.build.waiting_logs')}</div>
	{:else}
		<div class="relative w-full">
			<div class="text-right " />
			{#if loadLogsInterval}
				<LoadingLogs />
			{/if}
			<div class="flex justify-end sticky top-0 p-1 mx-1">
				<button
					id="follow"
					on:click={followBuild}
					class="bg-transparent btn btn-sm btn-link"
					class:text-green-500={followingLogs}
				>
					<svg
						xmlns="http://www.w3.org/2000/svg"
						class="w-6 h-6"
						viewBox="0 0 24 24"
						stroke-width="1.5"
						stroke="currentColor"
						fill="none"
						stroke-linecap="round"
						stroke-linejoin="round"
					>
						<path stroke="none" d="M0 0h24v24H0z" fill="none" />
						<circle cx="12" cy="12" r="9" />
						<line x1="8" y1="12" x2="12" y2="16" />
						<line x1="12" y1="8" x2="12" y2="16" />
						<line x1="16" y1="12" x2="12" y2="16" />
					</svg>
				</button>
				<Tooltip triggeredBy="#follow">Follow Logs</Tooltip>
			</div>
			<div class="font-mono w-full bg-coolgray-200 p-5 overflow-x-auto overflox-y-auto rounded mb-20 flex flex-col -mt-12 scrollbar-thumb-coollabs scrollbar-track-coolgray-200 scrollbar-w-1">
				{#each logs as log}
					<p>{log + '\n'}</p>
				{/each}
			</div>
		</div>
	{/if}
</div>
