<script lang="ts">
	import { WEBUI_BASE_URL } from '$lib/constants';
	import { marked } from 'marked';

	import { config, user, temporaryChatEnabled, WEBUI_NAME, settings } from '$lib/stores';
	import { onMount, getContext } from 'svelte';

	import { fade } from 'svelte/transition';

	import Suggestions from './Suggestions.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import EyeSlash from '$lib/components/icons/EyeSlash.svelte';

	const i18n = getContext('i18n');

	export let submitPrompt;

	let mounted = false;

	// Custom branding
	$: customLogo = $settings?.customIcon?.data || '/jarvis-logo.svg';
	$: assistantName = $settings?.assistantName || $WEBUI_NAME || 'Jarvis';

	onMount(() => {
		mounted = true;
	});
</script>

{#key mounted}
	<div class="m-auto w-full max-w-6xl px-8 lg:px-20">
		<div class="flex justify-start">
			<div class="flex -space-x-4 mb-0.5" in:fade={{ duration: 200 }}>
				<!-- Single custom assistant logo instead of model avatars -->
				<div>
					<img
						crossorigin="anonymous"
						src={customLogo}
						class=" size-[2.7rem] rounded-full border-[1px] border-gray-100 dark:border-none"
						alt="{assistantName} logo"
						draggable="false"
					/>
				</div>
			</div>
		</div>

		{#if $temporaryChatEnabled}
			<Tooltip
				content={$i18n.t('This chat won’t appear in history and your messages will not be saved.')}
				className="w-full flex justify-center mb-0.5"
				placement="top"
			>
				<div class="flex items-center gap-2 text-gray-500 font-medium text-lg my-2 w-fit">
					<EyeSlash strokeWidth="2.5" className="size-5" />{$i18n.t('Temporary Chat')}
				</div>
			</Tooltip>
		{/if}

		<div
			class=" mt-2 mb-4 text-3xl text-gray-800 dark:text-gray-100 font-medium text-left flex items-center gap-4 font-primary"
		>
			<div>
				<div class=" capitalize line-clamp-1" in:fade={{ duration: 200 }}>
					{$i18n.t('Welcome to {{name}}', { name: assistantName })}
				</div>

				<div in:fade={{ duration: 200, delay: 200 }}>
					<div class=" font-medium text-gray-400 dark:text-gray-500 line-clamp-1 font-p">
						{$i18n.t('How can I help you today?')}
					</div>
				</div>
			</div>
		</div>

		<div class=" w-full font-primary" in:fade={{ duration: 200, delay: 300 }}>
			<Suggestions
				className="grid grid-cols-2"
				suggestionPrompts={$config?.default_prompt_suggestions ?? []}
				on:select={(e) => {
					submitPrompt(e.detail);
				}}
			/>
		</div>
	</div>
{/key}
