<script>
	import { toast } from 'svelte-sonner';

	import { onMount, getContext, tick } from 'svelte';
	import { goto } from '$app/navigation';
	import { page } from '$app/stores';

	import { getBackendConfig } from '$lib/apis';
	import { ldapUserSignIn, getSessionUser, userSignIn, userSignUp } from '$lib/apis/auths';

	import { WEBUI_API_BASE_URL, WEBUI_BASE_URL } from '$lib/constants';
	import { WEBUI_NAME, config, user, socket, settings } from '$lib/stores';
	import { getUserSettings } from '$lib/apis/users';

	import { generateInitialsImage, canvasPixelTest } from '$lib/utils';

	import Spinner from '$lib/components/common/Spinner.svelte';
	import OnBoarding from '$lib/components/OnBoarding.svelte';

	const i18n = getContext('i18n');

	let loaded = false;

	let mode = $config?.features.enable_ldap ? 'ldap' : 'signin';

	let name = '';
	let email = '';
	let password = '';

	let ldapUsername = '';

	// Dynamic logo source - default to Jarvis logo
	$: logoSrc = $settings?.customIcon?.data || '/jarvis-logo.svg';
	$: assistantName = $settings?.assistantName || 'Jarvis';

	// Load user settings for branding if available
	const loadGlobalSettings = async () => {
		// Set default branding first
		WEBUI_NAME.set('Jarvis');
		
		try {
			// Try to load from localStorage first (for cached settings)
			const cachedSettings = localStorage.getItem('webui_settings');
			if (cachedSettings) {
				const parsed = JSON.parse(cachedSettings);
				if (parsed.assistantName || parsed.customIcon) {
					settings.update(s => ({
						...s,
						...parsed
					}));
					if (parsed.assistantName) {
						WEBUI_NAME.set(parsed.assistantName.trim() || 'Jarvis');
					}
					return;
				}
			}
			
			// If token available, try to load from server
			if (localStorage.token) {
				const userSettings = await getUserSettings(localStorage.token);
				if (userSettings?.ui) {
					const { assistantName, customIcon } = userSettings.ui;
					settings.update(s => ({
						...s,
						assistantName,
						customIcon
					}));
					if (assistantName) {
						WEBUI_NAME.set(assistantName.trim() || 'Jarvis');
					}
					// Cache settings to localStorage
					localStorage.setItem('webui_settings', JSON.stringify(userSettings.ui));
				}
			}
		} catch (error) {
			console.log('No saved settings available, using defaults:', error);
		}
	};

	const querystringValue = (key) => {
		const querystring = window.location.search;
		const urlParams = new URLSearchParams(querystring);
		return urlParams.get(key);
	};

	const setSessionUser = async (sessionUser) => {
		if (sessionUser) {
			console.log(sessionUser);
			toast.success($i18n.t(`You're now logged in.`));
			if (sessionUser.token) {
				localStorage.token = sessionUser.token;
			}
			$socket.emit('user-join', { auth: { token: sessionUser.token } });
			await user.set(sessionUser);
			await config.set(await getBackendConfig());

			const redirectPath = querystringValue('redirect') || '/';
			goto(redirectPath);
		}
	};

	const signInHandler = async () => {
		const sessionUser = await userSignIn(email, password).catch((error) => {
			toast.error(`${error}`);
			return null;
		});

		await setSessionUser(sessionUser);
	};

	const signUpHandler = async () => {
		const sessionUser = await userSignUp(name, email, password, generateInitialsImage(name)).catch(
			(error) => {
				toast.error(`${error}`);
				return null;
			}
		);

		await setSessionUser(sessionUser);
	};

	const ldapSignInHandler = async () => {
		const sessionUser = await ldapUserSignIn(ldapUsername, password).catch((error) => {
			toast.error(`${error}`);
			return null;
		});
		await setSessionUser(sessionUser);
	};

	const submitHandler = async () => {
		if (mode === 'ldap') {
			await ldapSignInHandler();
		} else if (mode === 'signin') {
			await signInHandler();
		} else {
			await signUpHandler();
		}
	};

	const checkOauthCallback = async () => {
		if (!$page.url.hash) {
			return;
		}
		const hash = $page.url.hash.substring(1);
		if (!hash) {
			return;
		}
		const params = new URLSearchParams(hash);
		const token = params.get('token');
		if (!token) {
			return;
		}
		const sessionUser = await getSessionUser(token).catch((error) => {
			toast.error(`${error}`);
			return null;
		});
		if (!sessionUser) {
			return;
		}
		localStorage.token = token;
		await setSessionUser(sessionUser);
	};

	let onboarding = false;

	onMount(async () => {
		// Load global settings for branding
		await loadGlobalSettings();
		
		if ($user !== undefined) {
			const redirectPath = querystringValue('redirect') || '/';
			goto(redirectPath);
		}
		await checkOauthCallback();

		loaded = true;

		if (($config?.features.auth_trusted_header ?? false) || $config?.features.auth === false) {
			await signInHandler();
		} else {
			onboarding = $config?.onboarding ?? false;
		}
	});
</script>

<svelte:head>
	<title>
		{`${$WEBUI_NAME}`}
	</title>
</svelte:head>

<OnBoarding
	bind:show={onboarding}
	getStartedHandler={() => {
		onboarding = false;
		mode = $config?.features.enable_ldap ? 'ldap' : 'signup';
	}}
/>

<div class=" bg-white dark:bg-gray-950">
	<div class="w-full absolute top-0 left-0 right-0 h-8 drag-region" />

	{#if loaded}
		<div class="relative min-h-screen w-full flex">
			<div class="flex flex-1 flex-col justify-center space-y-8 px-4 sm:px-6 lg:px-8">
				<div>
					<div class=" mx-auto w-full max-w-sm lg:w-96">
						<div>
							<div class=" flex items-center justify-center">
							<div class=" -mb-3">
							<img
							crossorigin="anonymous"
							src={logoSrc}
							class=" size-12 rounded-full border border-gray-50 dark:border-gray-800"
							alt="logo"
							draggable="false"
							/>
							</div>
							</div>
							<h2 class=" mt-6 text-center text-3xl font-bold tracking-tight text-gray-900 dark:text-gray-100">
								{#if ($config?.features.auth_trusted_header ?? false) || $config?.features.auth === false}
									{$i18n.t('Signing in to {{WEBUI_NAME}}', { WEBUI_NAME: $WEBUI_NAME })}

									<div class="flex justify-center">
										<Spinner />
									</div>
								{:else if mode === 'ldap'}
									{$i18n.t('Sign in to {{WEBUI_NAME}} with LDAP', { WEBUI_NAME: $WEBUI_NAME })}
								{:else if mode === 'signin'}
									{$i18n.t('Sign in to {{WEBUI_NAME}}', { WEBUI_NAME: $WEBUI_NAME })}
								{:else}
									{#if $config?.onboarding ?? false}
										{$i18n.t(`Get started with {{WEBUI_NAME}}`, { WEBUI_NAME: $WEBUI_NAME })}
									{:else}
										{$i18n.t('Sign up to {{WEBUI_NAME}}', { WEBUI_NAME: $WEBUI_NAME })}
									{/if}
								{/if}
							</h2>

							{#if $config?.onboarding ?? false}
								<div class=" mt-1 text-xs font-medium text-gray-600 dark:text-gray-500 text-center">
									ⓘ {$WEBUI_NAME}
									{$i18n.t(
										'does not make any external connections, and your data stays securely on your locally hosted server.'
									)}
								</div>
							{/if}
						</div>

						<div class="mt-8">
							<div>
								{#if !(($config?.features.auth_trusted_header ?? false) || $config?.features.auth === false)}
									<form
										class=" space-y-6"
										on:submit={(e) => {
											e.preventDefault();
											submitHandler();
										}}
									>
										{#if $config?.features.enable_login_form || $config?.features.enable_ldap}
											<div>
												{#if mode === 'signup'}
													<div>
														<label for="name" class=" text-sm font-medium text-gray-900 dark:text-gray-300"
															>{$i18n.t('Name')}</label
														>
														<div class="mt-1">
															<input
																bind:value={name}
																id="name"
																name="name"
																type="text"
																autocomplete="name"
																placeholder={$i18n.t('Enter Your Full Name')}
																required
																class=" appearance-none rounded-md relative block w-full px-3 py-2 border dark:border-gray-600 border-gray-300 dark:bg-gray-600 placeholder-gray-500 dark:placeholder-gray-200 text-gray-900 dark:text-gray-100 focus:outline-none focus:ring-1 focus:ring-gray-500 focus:border-gray-500 focus:z-10 sm:text-sm"
															/>
														</div>
													</div>
												{/if}

												{#if mode === 'ldap'}
													<div>
														<label for="username" class=" text-sm font-medium text-gray-900 dark:text-gray-300"
															>{$i18n.t('Username')}</label
														>
														<div class="mt-1">
															<input
																bind:value={ldapUsername}
																id="username"
																name="username"
																type="text"
																autocomplete="username"
																placeholder={$i18n.t('Enter Your Username')}
																required
																class=" appearance-none rounded-md relative block w-full px-3 py-2 border dark:border-gray-600 border-gray-300 dark:bg-gray-600 placeholder-gray-500 dark:placeholder-gray-200 text-gray-900 dark:text-gray-100 focus:outline-none focus:ring-1 focus:ring-gray-500 focus:border-gray-500 focus:z-10 sm:text-sm"
															/>
														</div>
													</div>
												{:else}
													<div>
														<label for="email" class=" text-sm font-medium text-gray-900 dark:text-gray-300"
															>{$i18n.t('Email')}</label
														>
														<div class="mt-1">
															<input
																bind:value={email}
																id="email"
																name="email"
																type="email"
																autocomplete="email"
																placeholder={$i18n.t('Enter Your Email')}
																required
																class=" appearance-none rounded-md relative block w-full px-3 py-2 border dark:border-gray-600 border-gray-300 dark:bg-gray-600 placeholder-gray-500 dark:placeholder-gray-200 text-gray-900 dark:text-gray-100 focus:outline-none focus:ring-1 focus:ring-gray-500 focus:border-gray-500 focus:z-10 sm:text-sm"
															/>
														</div>
													</div>
												{/if}

												<div>
													<label for="password" class=" text-sm font-medium text-gray-900 dark:text-gray-300"
														>{$i18n.t('Password')}</label
													>
													<div class="mt-1">
														<input
															bind:value={password}
															id="password"
															name="password"
															type="password"
															autocomplete="current-password"
															placeholder={$i18n.t('Enter Your Password')}
															required
															class=" appearance-none rounded-md relative block w-full px-3 py-2 border dark:border-gray-600 border-gray-300 dark:bg-gray-600 placeholder-gray-500 dark:placeholder-gray-200 text-gray-900 dark:text-gray-100 focus:outline-none focus:ring-1 focus:ring-gray-500 focus:border-gray-500 focus:z-10 sm:text-sm"
														/>
													</div>
												</div>
											</div>
										{/if}

										<div>
											{#if $config?.features.enable_login_form || $config?.features.enable_ldap}
												{#if mode === 'ldap'}
													<button
														type="submit"
														class="group relative w-full flex justify-center py-2 px-4 border border-transparent text-sm font-medium rounded-md text-white bg-gray-700 hover:bg-gray-800 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500"
													>
														{$i18n.t('Authenticate')}
													</button>
												{:else}
													<button
														type="submit"
														class="group relative w-full flex justify-center py-2 px-4 border border-transparent text-sm font-medium rounded-md text-white bg-gray-700 hover:bg-gray-800 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500"
													>
														{mode === 'signin'
															? $i18n.t('Sign in')
															: ($config?.onboarding ?? false)
																? $i18n.t('Create Admin Account')
																: $i18n.t('Create Account')}
													</button>

													{#if $config?.features.enable_signup && !($config?.onboarding ?? false)}
														<div class=" mt-4 text-sm text-center">
															{mode === 'signin'
																? $i18n.t("Don't have an account?")
																: $i18n.t('Already have an account?')}

															<button
																class=" font-medium text-gray-700 dark:text-gray-100 hover:text-gray-500 dark:hover:text-gray-300"
																type="button"
																on:click={() => {
																	if (mode === 'signin') {
																		mode = 'signup';
																	} else {
																		mode = 'signin';
																	}
																}}
															>
																{mode === 'signin' ? $i18n.t('Sign up') : $i18n.t('Sign in')}
															</button>
														</div>
													{/if}
												{/if}
											{/if}
										</div>
									</form>

									{#if Object.keys($config?.oauth?.providers ?? {}).length > 0}
										<div class="relative flex items-center justify-center my-6">
											<div class="flex-grow border-t border-gray-300 dark:border-gray-600"></div>
											{#if $config?.features.enable_login_form || $config?.features.enable_ldap}
												<span class="flex-shrink-0 bg-white dark:bg-gray-950 px-4 text-sm text-gray-500 dark:text-gray-400">{$i18n.t('or')}</span>
											{/if}
											<div class="flex-grow border-t border-gray-300 dark:border-gray-600"></div>
										</div>

										<div class="space-y-3">
											{#if $config?.oauth?.providers?.google}
												<button
													type="button"
													class="w-full flex justify-center items-center px-4 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm text-sm font-medium text-gray-700 dark:text-gray-200 bg-white dark:bg-gray-800 hover:bg-gray-50 dark:hover:bg-gray-700"
													on:click={() => {
														window.location.href = `${WEBUI_BASE_URL}/oauth/google/login`;
													}}
												>
													<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 48 48" class="w-5 h-5 mr-2">
														<path
															fill="#EA4335"
															d="M24 9.5c3.54 0 6.71 1.22 9.21 3.6l6.85-6.85C35.9 2.38 30.47 0 24 0 14.62 0 6.51 5.38 2.56 13.22l7.98 6.19C12.43 13.72 17.74 9.5 24 9.5z"
														/><path
															fill="#4285F4"
															d="M46.98 24.55c0-1.57-.15-3.09-.38-4.55H24v9.02h12.94c-.58 2.96-2.26 5.48-4.78 7.18l7.73 6c4.51-4.18 7.09-10.36 7.09-17.65z"
														/><path
															fill="#FBBC05"
															d="M10.53 28.59c-.48-1.45-.76-2.99-.76-4.59s.27-3.14.76-4.59l-7.98-6.19C.92 16.46 0 20.12 0 24c0 3.88.92 7.54 2.56 10.78l7.97-6.19z"
														/><path
															fill="#34A853"
															d="M24 48c6.48 0 11.93-2.13 15.89-5.81l-7.73-6c-2.15 1.45-4.92 2.3-8.16 2.3-6.26 0-11.57-4.22-13.47-9.91l-7.98 6.19C6.51 42.62 14.62 48 24 48z"
														/><path fill="none" d="M0 0h48v48H0z" />
													</svg>
													{$i18n.t('Continue with {{provider}}', { provider: 'Google' })}
												</button>
											{/if}
											{#if $config?.oauth?.providers?.microsoft}
												<button
													type="button"
													class="w-full flex justify-center items-center px-4 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm text-sm font-medium text-gray-700 dark:text-gray-200 bg-white dark:bg-gray-800 hover:bg-gray-50 dark:hover:bg-gray-700"
													on:click={() => {
														window.location.href = `${WEBUI_BASE_URL}/oauth/microsoft/login`;
													}}
												>
													<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 21 21" class="w-5 h-5 mr-2">
														<rect x="1" y="1" width="9" height="9" fill="#f25022" /><rect
															x="1"
															y="11"
															width="9"
															height="9"
															fill="#00a4ef"
														/><rect x="11" y="1" width="9" height="9" fill="#7fba00" /><rect
															x="11"
															y="11"
															width="9"
															height="9"
															fill="#ffb900"
														/>
													</svg>
													{$i18n.t('Continue with {{provider}}', { provider: 'Microsoft' })}
												</button>
											{/if}
											{#if $config?.oauth?.providers?.github}
												<button
													type="button"
													class="w-full flex justify-center items-center px-4 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm text-sm font-medium text-gray-700 dark:text-gray-200 bg-white dark:bg-gray-800 hover:bg-gray-50 dark:hover:bg-gray-700"
													on:click={() => {
														window.location.href = `${WEBUI_BASE_URL}/oauth/github/login`;
													}}
												>
													<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" class="w-5 h-5 mr-2">
														<path
															fill="currentColor"
															d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.92 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57C20.565 21.795 24 17.31 24 12c0-6.63-5.37-12-12-12z"
														/>
													</svg>
													{$i18n.t('Continue with {{provider}}', { provider: 'GitHub' })}
												</button>
											{/if}
											{#if $config?.oauth?.providers?.oidc}
												<button
													type="button"
													class="w-full flex justify-center items-center px-4 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm text-sm font-medium text-gray-700 dark:text-gray-200 bg-white dark:bg-gray-800 hover:bg-gray-50 dark:hover:bg-gray-700"
													on:click={() => {
														window.location.href = `${WEBUI_BASE_URL}/oauth/oidc/login`;
													}}
												>
													<svg
														xmlns="http://www.w3.org/2000/svg"
														fill="none"
														viewBox="0 0 24 24"
														stroke-width="1.5"
														stroke="currentColor"
														class="w-5 h-5 mr-2"
													>
														<path
															stroke-linecap="round"
															stroke-linejoin="round"
															d="M15.75 5.25a3 3 0 0 1 3 3m3 0a6 6 0 0 1-7.029 5.912c-.563-.097-1.159.026-1.563.43L10.5 17.25H8.25v2.25H6v2.25H2.25v-2.818c0-.597.237-1.17.659-1.591l6.499-6.499c.404-.404.527-1 .43-1.563A6 6 0 1 1 21.75 8.25Z"
														/>
													</svg>

													<span
														>{$i18n.t('Continue with {{provider}}', {
															provider: $config?.oauth?.providers?.oidc ?? 'SSO'
														})}</span
													>
												</button>
											{/if}
										</div>
									{/if}

									{#if $config?.features.enable_ldap && $config?.features.enable_login_form}
										<div class="mt-6 text-center">
											<button
												class="text-sm text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-200 underline"
												type="button"
												on:click={() => {
													if (mode === 'ldap')
														mode = ($config?.onboarding ?? false) ? 'signup' : 'signin';
													else mode = 'ldap';
												}}
											>
												{mode === 'ldap'
													? $i18n.t('Continue with Email')
													: $i18n.t('Continue with LDAP')}
											</button>
										</div>
									{/if}
								{/if}
							</div>
						</div>
					</div>
				</div>
			</div>
		</div>
	{/if}
</div>
