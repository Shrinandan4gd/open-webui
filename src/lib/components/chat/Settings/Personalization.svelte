<script lang="ts">
	import Switch from '$lib/components/common/Switch.svelte';
	import { config, models, settings, user, WEBUI_NAME, updateBranding } from '$lib/stores';
	import { createEventDispatcher, onMount, getContext, tick } from 'svelte';
	import { toast } from 'svelte-sonner';
	import ManageModal from './Personalization/ManageModal.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	const dispatch = createEventDispatcher();

	const i18n = getContext('i18n');

	export let saveSettings: Function;

	let showManageModal = false;

	// Addons
	let enableMemory = false;

	// Assistant Customization
	let assistantName = '';
	let customIcon = null;
	let customIconUrl = '';
	let fileInputElement;

	const handleIconUpload = async (event) => {
		const file = event.target.files?.[0];
		if (!file) return;

		// Validate file type
		const validTypes = ['image/png', 'image/svg+xml', 'image/x-icon', 'image/vnd.microsoft.icon'];
		if (!validTypes.includes(file.type)) {
			toast.error($i18n.t('Please upload a valid icon file (.ico, .png, .svg)'));
			return;
		}

		// Validate file size (max 1MB)
		if (file.size > 1024 * 1024) {
			toast.error($i18n.t('Icon file size must be less than 1MB'));
			return;
		}

		try {
			// Convert to base64
			const reader = new FileReader();
			reader.onload = (e) => {
				customIconUrl = e.target?.result as string;
				customIcon = {
					name: file.name,
					type: file.type,
					data: customIconUrl
				};
				// Update branding immediately and force favicon update
				updateBranding(undefined, customIcon);
				
				// Additional favicon forcing
				setTimeout(() => {
					const head = document.head;
					const links = head.querySelectorAll('link[rel*="icon"]');
					links.forEach(link => {
						const newLink = link.cloneNode() as HTMLLinkElement;
						newLink.href = customIconUrl + '?v=' + Date.now();
						head.removeChild(link);
						head.appendChild(newLink);
					});
				}, 100);
				
				toast.success($i18n.t('Icon uploaded successfully'));
			};
			reader.readAsDataURL(file);
		} catch (error) {
			console.error('Error uploading icon:', error);
			toast.error($i18n.t('Failed to upload icon'));
		}
	};

	const resetIcon = () => {
		customIcon = null;
		customIconUrl = '';
		if (fileInputElement) {
			fileInputElement.value = '';
		}
		// Reset to Jarvis logo
		updateBranding(undefined, null);
		toast.success($i18n.t('Icon reset to default'));
	};

	const updateAssistantName = () => {
		updateBranding(assistantName, undefined);
	};

	onMount(async () => {
		enableMemory = $settings?.memory ?? false;
		assistantName = $settings?.assistantName || '';
		customIcon = $settings?.customIcon || null;
		customIconUrl = customIcon?.data || '';

		// Apply settings on mount using the utility function
		updateBranding(assistantName, customIcon);
	});
</script>

<ManageModal bind:show={showManageModal} />

<form
	class="flex flex-col h-full justify-between space-y-3 text-sm"
	on:submit|preventDefault={async () => {
		await saveSettings({ 
			memory: enableMemory,
			assistantName: assistantName.trim(),
			customIcon: customIcon
		});
		
		// Cache settings to localStorage for login page
		localStorage.setItem('webui_settings', JSON.stringify({
			assistantName: assistantName.trim(),
			customIcon: customIcon
		}));
		
		dispatch('save');
	}}
>
	<div class="py-1 overflow-y-scroll max-h-[28rem] lg:max-h-full">
		<div>
			<div class="flex items-center justify-between mb-1">
				<Tooltip
					content={$i18n.t(
						'This is an experimental feature, it may not function as expected and is subject to change at any time.'
					)}
				>
					<div class="text-sm font-medium">
						{$i18n.t('Memory')}

						<span class=" text-xs text-gray-500">({$i18n.t('Experimental')})</span>
					</div>
				</Tooltip>

				<div class="">
					<Switch
						bind:state={enableMemory}
						on:change={async () => {
							saveSettings({ memory: enableMemory });
						}}
					/>
				</div>
			</div>
		</div>

		<div class="text-xs text-gray-600 dark:text-gray-400">
			<div>
				{$i18n.t(
					"You can personalize your interactions with LLMs by adding memories through the 'Manage' button below, making them more helpful and tailored to you."
				)}
			</div>

			<!-- <div class="mt-3">
				To understand what LLM remembers or teach it something new, just chat with it:

				<div>- “Remember that I like concise responses.”</div>
				<div>- “I just got a puppy!”</div>
				<div>- “What do you remember about me?”</div>
				<div>- “Where did we leave off on my last project?”</div>
			</div> -->
		</div>

		<div class="mt-3 mb-1 ml-1">
			<button
				type="button"
				class=" px-3.5 py-1.5 font-medium hover:bg-black/5 dark:hover:bg-white/5 outline outline-1 outline-gray-300 dark:outline-gray-800 rounded-3xl"
				on:click={() => {
					showManageModal = true;
				}}
			>
				{$i18n.t('Manage')}
			</button>
		</div>

		<!-- Assistant Name Field -->
		<div class="mt-6">
			<div class="text-sm font-medium mb-2">
				{$i18n.t('Assistant Name')}
			</div>
			<div class="text-xs text-gray-600 dark:text-gray-400 mb-3">
				{$i18n.t('Customize the name of your assistant. Leave empty to use "Jarvis" as default.')}
			</div>
			<input
				class="w-full rounded-lg p-3 text-sm bg-gray-50 dark:bg-gray-800 border border-gray-300 dark:border-gray-600 focus:outline-none focus:ring-2 focus:ring-blue-500"
				type="text"
				bind:value={assistantName}
				placeholder={$i18n.t('Enter assistant name (e.g., Jarvis, Assistant)')}
				on:input={() => {
					updateAssistantName();
				}}
			/>
		</div>

		<!-- Custom Icon Upload -->
		<div class="mt-6">
			<div class="text-sm font-medium mb-2">
				{$i18n.t('Upload Custom Icon')}
			</div>
			<div class="text-xs text-gray-600 dark:text-gray-400 mb-3">
				{$i18n.t('Upload a custom icon to replace the default branding. Supports .ico, .png, and .svg files (max 1MB).')}
			</div>
			
			{#if customIconUrl}
				<div class="mb-3 p-3 bg-gray-50 dark:bg-gray-800 rounded-lg border border-gray-300 dark:border-gray-600">
					<div class="flex items-center gap-3">
						<img src={customIconUrl} alt="Custom Icon" class="w-8 h-8 object-contain" />
						<div class="flex-1">
							<div class="text-sm font-medium">{customIcon?.name}</div>
							<div class="text-xs text-gray-500">{$i18n.t('Custom icon active')}</div>
						</div>
						<button
							type="button"
							class="px-2 py-1 text-xs bg-red-100 hover:bg-red-200 dark:bg-red-800 dark:hover:bg-red-700 text-red-700 dark:text-red-200 rounded"
							on:click={resetIcon}
						>
							{$i18n.t('Remove')}
						</button>
					</div>
				</div>
			{/if}

			<div class="flex gap-2">
				<input
					bind:this={fileInputElement}
					type="file"
					accept=".ico,.png,.svg,image/png,image/svg+xml,image/x-icon,image/vnd.microsoft.icon"
					on:change={handleIconUpload}
					class="hidden"
				/>
				<button
					type="button"
					class="px-3.5 py-1.5 font-medium bg-blue-100 hover:bg-blue-200 dark:bg-blue-800 dark:hover:bg-blue-700 text-blue-700 dark:text-blue-200 rounded-3xl"
					on:click={() => fileInputElement?.click()}
				>
					{$i18n.t(customIconUrl ? 'Change Icon' : 'Upload Icon')}
				</button>
			</div>
		</div>
	</div>

	<div class="flex justify-end text-sm font-medium">
		<button
			class="px-3.5 py-1.5 text-sm font-medium bg-black hover:bg-gray-900 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 transition rounded-full"
			type="submit"
		>
			{$i18n.t('Save')}
		</button>
	</div>
</form>
