<script lang="ts">
	import { getContext, onDestroy, onMount } from 'svelte';
	import { models, user as sessionUser } from '$lib/stores';
	import {
		getChatById,
		getSubagentsByChatId,
		type SubagentSummary
	} from '$lib/apis/chats';
	import { createMessagesList } from '$lib/utils';

	import Messages from '../Messages.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';
	import ChevronLeft from '$lib/components/icons/ChevronLeft.svelte';

	const i18n = getContext('i18n');

	// The parent (orchestrator) chat. This component NEVER changes it:
	// no $chatId writes, no navigation, no model selection changes.
	export let chatId: string;

	const POLL_MS = 3000;

	let subagents: SubagentSummary[] = [];
	let loadingList = true;

	let selected: SubagentSummary | null = null;
	let history: any = { messages: {}, currentId: null };
	let messages = [];
	let autoScroll = true;
	let loadingDetail = false;
	let finalFetched = false;

	let timer: ReturnType<typeof setInterval> | null = null;
	let currentParent = '';

	$: messages = createMessagesList(history, history.currentId);

	const modelName = (id: string | null) =>
		(id && $models.find((m) => m.id === id)?.name) || id || '';

	const loadList = async () => {
		if (!chatId) return;
		const parent = chatId;
		const res = await getSubagentsByChatId(localStorage.token, parent);
		// Ignore stale responses if the user switched chats meanwhile.
		if (parent !== chatId) return;
		subagents = res;
		loadingList = false;
	};

	const loadDetail = async (showSpinner = false) => {
		if (!selected) return;
		const target = selected.id;
		if (showSpinner) loadingDetail = true;
		const chat = await getChatById(localStorage.token, target).catch(() => null);
		if (!selected || selected.id !== target) return;
		if (chat?.chat?.history) {
			history = chat.chat.history;
		}
		loadingDetail = false;
	};

	const open = async (agent: SubagentSummary) => {
		selected = agent;
		history = { messages: {}, currentId: null };
		autoScroll = true;
		finalFetched = false;
		await loadDetail(true);
	};

	const back = () => {
		selected = null;
		history = { messages: {}, currentId: null };
		loadList();
	};

	const poll = async () => {
		await loadList();
		if (selected) {
			// Keep status badge in sync, then refresh content while it is running.
			const fresh = subagents.find((s) => s.id === selected.id);
			if (fresh) selected = fresh;
			if (!fresh || fresh.status === 'running') {
				await loadDetail();
			} else if (!finalFetched) {
				// One last fetch after it finishes so the final output is complete.
				finalFetched = true;
				await loadDetail();
			}
		}
	};

	// Switching to a different orchestrator chat resets the panel.
	$: if (chatId !== currentParent) {
		currentParent = chatId;
		selected = null;
		subagents = [];
		loadingList = true;
		loadList();
	}

	onMount(() => {
		timer = setInterval(poll, POLL_MS);
	});
	onDestroy(() => {
		if (timer) clearInterval(timer);
	});

	const statusClass = (status: string) =>
		status === 'running'
			? 'bg-blue-500 animate-pulse'
			: status === 'error'
				? 'bg-red-500'
				: 'bg-green-500';

	const statusLabel = (status: string) =>
		status === 'running'
			? $i18n.t('Running')
			: status === 'error'
				? $i18n.t('Failed')
				: $i18n.t('Done');
</script>

<div class="flex flex-col h-full w-full min-h-0">
	{#if selected}
		<div
			class="flex items-center gap-2 px-3 py-2 border-b border-gray-100 dark:border-gray-850 shrink-0"
		>
			<button
				type="button"
				class="flex items-center gap-1 text-sm text-gray-600 dark:text-gray-300 hover:text-black dark:hover:text-white transition shrink-0"
				on:click={back}
			>
				<ChevronLeft className="size-4" />
				{$i18n.t('Back')}
			</button>
			<div class="flex-1 min-w-0">
				<div class="text-xs text-gray-700 dark:text-gray-200 truncate">{selected.task}</div>
				<div class="text-[0.6875rem] text-gray-400 flex items-center gap-1.5">
					<span class="size-1.5 rounded-full {statusClass(selected.status)}"></span>
					{statusLabel(selected.status)} · {modelName(selected.model)}
				</div>
			</div>
			<span
				class="text-[0.6875rem] px-1.5 py-0.5 rounded-md bg-gray-100 dark:bg-gray-800 text-gray-500 shrink-0"
			>
				{$i18n.t('Read-only')}
			</span>
		</div>

		<div class="flex-1 min-h-0 relative">
			{#if loadingDetail}
				<div class="absolute inset-0 flex items-center justify-center">
					<Spinner className="size-5" />
				</div>
			{:else}
				<!-- Read-only transcript. Props only: nothing here can modify the orchestrator chat. -->
				<Messages
					className="h-full flex pt-2 pb-4"
					user={$sessionUser}
					chatId={selected.id}
					readOnly={true}
					allowDelete={false}
					compactPreview={true}
					editCodeBlock={false}
					messagesCount={null}
					selectedModels={[selected.model ?? '']}
					bind:history
					bind:messages
					bind:autoScroll
					messagesContainerId="subagent-messages-container"
					sendMessage={() => {}}
					continueResponse={() => {}}
					regenerateResponse={() => {}}
					mergeResponses={() => {}}
					chatActionHandler={() => {}}
					showMessage={() => {}}
					submitMessage={() => {}}
					addMessages={() => {}}
				/>
			{/if}
		</div>
	{:else}
		<div class="flex-1 min-h-0 overflow-y-auto px-3 py-2">
			{#if loadingList}
				<div class="flex items-center justify-center py-10">
					<Spinner className="size-5" />
				</div>
			{:else if subagents.length === 0}
				<div class="text-center text-sm text-gray-400 dark:text-gray-500 py-10 px-4">
					{$i18n.t('No sub-agents have been started in this chat.')}
				</div>
			{:else}
				<div class="flex flex-col gap-1.5">
					{#each subagents as agent (agent.id)}
						<button
							type="button"
							class="w-full text-left rounded-xl border border-gray-100 dark:border-gray-850 px-3 py-2 hover:bg-gray-50 dark:hover:bg-gray-850/60 transition"
							on:click={() => open(agent)}
						>
							<div class="flex items-center gap-2">
								<span class="size-2 rounded-full shrink-0 {statusClass(agent.status)}"></span>
								<span class="text-sm text-gray-800 dark:text-gray-100 truncate flex-1">
									{agent.task || agent.title}
								</span>
							</div>
							<div class="mt-0.5 ml-4 text-[0.6875rem] text-gray-400 flex gap-2">
								<span>{statusLabel(agent.status)}</span>
								<span>·</span>
								<span class="truncate">{modelName(agent.model)}</span>
								{#if agent.mode === 'background'}
									<span>·</span>
									<span>{$i18n.t('Background')}</span>
								{/if}
							</div>
						</button>
					{/each}
				</div>
			{/if}
		</div>
	{/if}
</div>
