<script>
	import { onDestroy, onMount, tick, getContext, createEventDispatcher } from 'svelte';
	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	import Markdown from './Markdown.svelte';
	import { chatId, mobile, showArtifacts, showControls, showOverview } from '$lib/stores';
	import FloatingButtons from '../ContentRenderer/FloatingButtons.svelte';
	import { createMessagesList } from '$lib/utils';

	export let id;
	export let content;
	export let history;
	export let model = null;
	export let sources = null; 

	export let save = false;
	export let floatingButtons = true;

	export let onSourceClick = () => {};
	export let onAddMessages = () => {};

	let contentContainerElement;

	let floatingButtonsElement;

	const updateButtonPosition = (event) => {
		const buttonsContainerElement = document.getElementById(`floating-buttons-${id}`);
		if (
			!contentContainerElement?.contains(event.target) &&
			!buttonsContainerElement?.contains(event.target)
		) {
			closeFloatingButtons();
			return;
		}

		setTimeout(async () => {
			await tick();

			if (!contentContainerElement?.contains(event.target)) return;

			let selection = window.getSelection();

			if (selection.toString().trim().length > 0) {
				const range = selection.getRangeAt(0);
				const rect = range.getBoundingClientRect();

				const parentRect = contentContainerElement.getBoundingClientRect();

				// Adjust based on parent rect
				const top = rect.bottom - parentRect.top;
				const left = rect.left - parentRect.left;

				if (buttonsContainerElement) {
					buttonsContainerElement.style.display = 'block';

					// Calculate space available on the right
					const spaceOnRight = parentRect.width - left;
					let halfScreenWidth = $mobile ? window.innerWidth / 2 : window.innerWidth / 3;

					if (spaceOnRight < halfScreenWidth) {
						const right = parentRect.right - rect.right;
						buttonsContainerElement.style.right = `${right}px`;
						buttonsContainerElement.style.left = 'auto'; // Reset left
					} else {
						// Enough space, position using 'left'
						buttonsContainerElement.style.left = `${left}px`;
						buttonsContainerElement.style.right = 'auto'; // Reset right
					}
					buttonsContainerElement.style.top = `${top + 5}px`; // +5 to add some spacing
				}
			} else {
				closeFloatingButtons();
			}
		}, 0);
	};

	const closeFloatingButtons = () => {
		const buttonsContainerElement = document.getElementById(`floating-buttons-${id}`);
		if (buttonsContainerElement) {
			buttonsContainerElement.style.display = 'none';
		}

		if (floatingButtonsElement) {
			floatingButtonsElement.closeHandler();
		}
	};

	const keydownHandler = (e) => {
		if (e.key === 'Escape') {
			closeFloatingButtons();
		}
	};

	onMount(() => {
		if (floatingButtons) {
			contentContainerElement?.addEventListener('mouseup', updateButtonPosition);
			document.addEventListener('mouseup', updateButtonPosition);
			document.addEventListener('keydown', keydownHandler);
		}
	});

	onDestroy(() => {
		if (floatingButtons) {
			contentContainerElement?.removeEventListener('mouseup', updateButtonPosition);
			document.removeEventListener('mouseup', updateButtonPosition);
			document.removeEventListener('keydown', keydownHandler);
		}
	});

	export let message;

	const splitMessageBlocks = (content) => {
		if (!content) return [];
		const blocks = [];
		let currentBlock = {
			type: 'default',
			content: ''
		};

		const lines = content.split('\n');
		let isInToolSection = false;
		let isInResultSection = false;
		let isInObservationSection = false;

		for (let line of lines) {
			console.log('[contentRenderer] currentBlock.content', currentBlock.content);
			console.log('[contentRenderer] line', line);

			if (!line.includes('🤔') && 
				!line.includes('🔧') && 
				!line.includes('👀') && 
				!line.includes('📊') && 
				!line.includes('✅')) {
				
				// 현재 블록이 비어있지 않으면 새로운 줄 추가
				if (currentBlock.content) {
					currentBlock.content += '\n';
				}
				currentBlock.content += line;
				
				// 현재 블록이 아직 blocks 배열에 추가되지 않았다면 추가
				if (!blocks.includes(currentBlock)) {
					blocks.push(currentBlock);
				}
        	} else {
			
				if (line.includes('🤔 분석을 시작합니다')) {
					if (currentBlock.content) blocks.push(currentBlock);
					currentBlock = { type: 'start', content: line };
					blocks.push(currentBlock);
					currentBlock = { type: 'default', content: '' };
				}
				else if (line.includes('🔧 도구를 실행합니다')) {
					if (currentBlock.content) blocks.push(currentBlock);
					currentBlock = { type: 'tool-header', content: line };
					blocks.push(currentBlock);
					isInToolSection = true;
					isInObservationSection = false;
					currentBlock = { type: 'tool-content', content: '' };
				}
				else if (line.includes('👀 관찰 결과')) {
					if (currentBlock.content) blocks.push(currentBlock);
					currentBlock = { type: 'observation-header', content: line };
					blocks.push(currentBlock);
					isInToolSection = false;
					isInObservationSection = true;
					currentBlock = { type: 'observation-content', content: '' };
				}
				else if (line.includes('📊 분석 결과')) {
					if (currentBlock.content) blocks.push(currentBlock);
					currentBlock = { type: 'result-header', content: line };
					blocks.push(currentBlock);
					isInToolSection = false;
					isInObservationSection = false;
					isInResultSection = true;
					currentBlock = { type: 'result-content', content: '' };
				}
				else if (line.includes('✅ 분석이 완료되었습니다')) {
					if (currentBlock.content) blocks.push(currentBlock);
					currentBlock = { type: 'complete', content: line };
					blocks.push(currentBlock);
					isInToolSection = false;
					isInObservationSection = false;
					isInResultSection = false;
					currentBlock = { type: 'default', content: '' };
				}
				else {
					if (currentBlock.content) currentBlock.content += '\n';
					currentBlock.content += line;
				}
			}
		}
		if (currentBlock.content && !blocks.includes(currentBlock)) {
       	 	blocks.push(currentBlock);
   	 	}		
		console.log('[contentRenderer] blocks', blocks);
		return blocks;
	};

	const getStyleClass = (type) => {
		const baseClass = 'p-4 rounded-lg mb-2 transition-all duration-300';
		switch(type) {
			case 'start':
				return `${baseClass} bg-gray-100 border-l-4 border-gray-500`;
			case 'tool-header':
				return `${baseClass} bg-blue-100 border-l-4 border-blue-500`;
			case 'tool-content':
				return `${baseClass} bg-blue-50`;
			case 'observation-header':
				return `${baseClass} bg-purple-100 border-l-4 border-purple-500`;
			case 'observation-content':
				return `${baseClass} bg-purple-50`;
			case 'result-header':
				return `${baseClass} bg-yellow-100 border-l-4 border-yellow-500`;
			case 'result-content':
				return `${baseClass} bg-yellow-50`;
			case 'complete':
				return `${baseClass} bg-green-100 border-l-4 border-green-500`;
			default:
				return '';
		}
	};

	let dots = '';
	let dotsInterval;

	onMount(() => {
		dotsInterval = setInterval(() => {
			dots = dots.length >= 6 ? '' : dots + '.';
		}, 500);
	});

	onDestroy(() => {
		if (dotsInterval) clearInterval(dotsInterval);
	});

	$: messageBlocks = splitMessageBlocks(content);
	$: lastMessageType = messageBlocks.length > 0 ? messageBlocks[messageBlocks.length - 1].type : null;
	$: isWaitingForNext = lastMessageType && ['start', 'tool-header', 'result-header'].includes(lastMessageType);
</script>

<style>
	@keyframes fadeIn {
		from {
			opacity: 0;
		}
		to {
			opacity: 1;
		}
	}

	@keyframes slideIn {
		from {
			opacity: 0;
			transform: translateY(-10px);
		}
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	:global(.animate-fade-in) {
		animation: fadeIn 0.5s ease-in-out forwards;
	}

	:global(.animate-slide-in) {
		animation: slideIn 0.5s ease-in-out forwards;
	}

	:global(.delay-100) {
		animation-delay: 100ms;
	}
</style>

<div bind:this={contentContainerElement}>
	{#each messageBlocks as block}
		{#if block.type === 'default'}
			<div class="markdown-body">
				<Markdown
					{id}
					content={block.content}
					{model}
					{save}
					sourceIds={[]}
					{onSourceClick}
				/>
			</div>
		{:else if ['start', 'tool-header', 'observation-header', 'result-header', 'complete'].includes(block.type)}
			<div class={getStyleClass(block.type)}>
				<div class="flex items-center">
					<span>{block.content}{block === messageBlocks[messageBlocks.length - 1] && isWaitingForNext ? dots : ''}</span>
				</div>
			</div>
		{:else if ['tool-content', 'observation-content', 'result-content'].includes(block.type)}
			<div class={getStyleClass(block.type)}>
				<div class="markdown-body">
					<Markdown
						{id}
						content={block.content}
						{model}
						{save}
						sourceIds={(sources ?? []).reduce((acc, s) => {
							let ids = [];
							s.document.forEach((document, index) => {
								const metadata = s.metadata?.[index];
								const id = metadata?.source ?? 'N/A';

								if (metadata?.name) {
									ids.push(metadata.name);
									return ids;
								}

								if (id.startsWith('http://') || id.startsWith('https://')) {
									ids.push(id);
								} else {
									ids.push(s?.source?.name ?? id);
								}

								return ids;
							});

							acc = [...acc, ...ids];

							// remove duplicates
							return acc.filter((item, index) => acc.indexOf(item) === index);
						}, [])}
						{onSourceClick}
						on:update={(e) => {
							dispatch('update', e.detail);
						}}
						on:code={(e) => {
							const { lang, code } = e.detail;

							if (
								(['html', 'svg'].includes(lang) || (lang === 'xml' && code.includes('svg'))) &&
								!$mobile &&
								$chatId
							) {
								showArtifacts.set(true);
								showControls.set(true);
							}
						}}
					/>
				</div>
			</div>
		{/if}
	{/each}
</div>

{#if floatingButtons && model}	
	<FloatingButtons
		bind:this={floatingButtonsElement}
		{id}
		model={model?.id}
		messages={createMessagesList(history, id)}
		onAdd={({ modelId, parentId, messages }) => {
			console.log(modelId, parentId, messages);
			onAddMessages({ modelId, parentId, messages });
			closeFloatingButtons();
		}}
	/>
{/if}
