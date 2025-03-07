<script>
	import '../app.css';

    import { getPreferences } from '$lib/preferences.js';

    import { selectedServer, selectedVersion, selectedWallet } from '$lib/stores.js';
	import {getWallets, checkPassword, hasWalletsFile} from '$lib/wallets'

    import { openServer } from '$lib/shell.js';

    import { onMount } from 'svelte';
    import { writable } from 'svelte/store';

    import ProfileSelector from '$lib/components/ProfileSelector.svelte';

    import TextBox from '$lib/components/element/form/TextBox.svelte';
    import Button from '$lib/components/element/form/Button.svelte';
    import ModalContainer from '$lib/components/modal/ModalContainer.svelte';
    import Cog from '$lib/icon/Cog.svelte';

    let versions = [];
    let loading = true;
    let installing = false;
    let preferences;

    let firstStart = false;
    let passwordAdded = false;

    let tempPass, tempConfirmPass;
	let wallets;

    onMount(async() => {
        preferences = await getPreferences();
        versions = await getVersions();
		
        $selectedVersion = versions[0];

        selectedServer.subscribe(server => {
            if (server.hasOwnProperty('proto_max')) {
                let recommendedVersion = versions.filter(i => i.branch.protocolVersion === server.proto_max && !i.branch.isDev);
                if (recommendedVersion.length) {
                    console.log(recommendedVersion);
                    $selectedVersion = recommendedVersion[0];
				}
            }
        });

        firstStart = !(await hasWalletsFile());

        loading = false;
	})

    import { getVersions, getInstalledVersions, openVersionFolder } from '$lib/api/versions';
    import { downloadVersion, unzipVersion } from '$lib/api/download';

    let username = writable('');
    let password = writable('');

    import Navbar from '$lib/components/nav/Navbar.svelte';
    import NavbarItem from '$lib/components/nav/NavbarItem.svelte';

    import VersionDropdown from './VersionDropdown.svelte';

    import FullLoader from '$lib/components/FullLoader.svelte';

    import Folder from '$lib/icon/Folder.svelte';

    async function install(version) {
        installing = true;
		await downloadVersion(version.name)
			.then(async () => {
                versions = await getVersions(true)
				let selVer = $selectedVersion;
                selVer.installed = true;
                $selectedVersion = selVer;
            }) // force list to update
			.catch(err => console.log(err));
        installing = false;
			//.finally(installing = false);
	}

    async function initFirstTime() {
        if (tempPass !== tempConfirmPass) {
            //TODO dialog api
            alert('no');
            return;
		}

        loading = true;
        // load userdata for the first time, which will init and encrypt a blank file
        await getWallets(tempPass);
		(await getWallets()).subscribe(val => wallets = val)
        // not needed anymore, userdata is in memory (unencrypted)
        tempPass = null;
        tempConfirmPass = null;

        loading = false;

        passwordAdded = true;
	}

    async function login() {
        if (await checkPassword(tempPass)) {
            loading = true;
            await getWallets(tempPass);
			(await getWallets()).subscribe(val => wallets = val)
            tempPass = null;
            loading = false;
            passwordAdded = true;
        }
        else {
            alert('This password is invalid. Please try again.');
            tempPass = '';
		}
	}

	async function checkKeyInput(event){
		if (event.keyCode == 13 && tempPass) login();
	}

    async function doOpenServer(server, version) {
		let Solar = await import("@solar-network/crypto/dist/index")
		const keys = Solar.Identities.Keys.fromPassphrase(wallets.selected.mnemonic);
		const message = wallets.selected.mnemonic;
		const hash = Solar.Crypto.HashAlgorithms.sha256(message);
		const signature = Solar.Crypto.Hash.signSchnorr(hash, keys);
        await openServer(server, wallets.selected.address, signature, version.name);
	}
</script>
{#if loading}
	<FullLoader />
{:else}
	{#if passwordAdded}
		<Navbar>
			<svelte:fragment slot="left">
				<NavbarItem href="/" label="Profile" />
				<!--<NavbarItem href="/news" label="News" />-->
				<NavbarItem href="/games" label="Play Offline" />        		
				<NavbarItem href="/servers" label="Play Online" />
				<!--<NavbarItem href="#" label="CTF Rankings" />-->
				<!--<NavbarItem href="#" label="Virtual Land" />-->
				<!--<NavbarItem href="#" label="NFT" />-->
				<!--<NavbarItem href="#" label="Auctions" />-->
				<NavbarItem href="/screenshots" label="Screenshots" />
			</svelte:fragment>
			<svelte:fragment slot="right">
				<NavbarItem href="/settings">
					<Cog />
				</NavbarItem>
			</svelte:fragment>
		</Navbar>
		<div class="w-full h-full">
			<div class="pt-12 pb-24">
				<slot />
			</div>
		</div>
		<div class="fixed w-full z-20 bottom-0 h-26 bg-darkest">
			<div class="w-full grid grid-cols-3 gap-4 px-8 py-4">
				<div class="flex flex-col">
					<div class="text-lg font-bold text-white pb-1">
						<span class="mr-2">Current Version</span>
						{#if $selectedVersion.installed}
							<button class="hover:text-dark-green hover:cursor-pointer" on:click={() => openVersionFolder($selectedVersion.name)}>
								<Folder />
							</button>

						{/if}
					</div>
					<VersionDropdown />
				</div>
				<div class="flex justify-center">
						{#if wallets.selected.address == ""}
						<button class="p-4 font-bold text-white flex flex-col items-center">
							<div>Add a wallet before playing!</div>
						</button>
						{:else if $selectedVersion.hasOwnProperty('installed') && !$selectedVersion.installed}
							<button on:click={async () => await install($selectedVersion)} class="bg-solar-orange bg-solar-orange-hover p-4 font-bold text-white flex flex-col items-center">
								{#if installing}
									<div>Installing...</div>
									<div class="font-medium text-sm">please be patient :)</div>
								{:else}
									<div>Install</div>
									<div class="font-medium text-sm">version {$selectedVersion.name}</div>
								{/if}
							</button>
						{:else if $selectedServer.hasOwnProperty('address')}
							<button on:click={() => doOpenServer($selectedServer,
							$selectedVersion)} class="bg-solar-orange bg-solar-orange-hover p-4 font-bold text-white flex flex-col items-center">
								<div>Play</div>
								<div class="font-medium text-sm">{$selectedServer.address}:{$selectedServer.port || 30000}</div>
							</button>
						{/if}
				</div>
				<div class="flex flex-col">
						<ProfileSelector />
				</div>
			</div>
		</div>
	{:else if firstStart}
		<div class="w-full h-full flex items-center justify-center flex-col text-center">
			<span class="text-2xl font-bold">First time?</span>
			<span class="w-96 py-4">
					Welcome to the District 53 Launcher. <br>
					Before we get started, you need to <strong>set a password</strong> to locally encrypt some sensitive settings (passwords for servers, mnemonics, etc.)
			</span>
			<div class="w-64">
				<div class="pb-4">
					<TextBox bind:value={tempPass} isPassword placeholder="Password" />
				</div>
				<div class="pb-4">
					<TextBox bind:value={tempConfirmPass} isPassword placeholder="Confirm Password" />
				</div>
				<div class="flex w-full justify-center">
					{#if (tempConfirmPass && tempPass) && tempConfirmPass === tempPass}
						<Button on:click={() => initFirstTime()} class="bg-solar-orange bg-solar-orange-hover py-2 rounded-md">
							Set
						</Button>
					{/if}
				</div>
			</div>
		</div>
	{:else}
		<div class="w-full h-full flex items-center justify-center flex-col text-center">
			<span class="text-2xl font-bold">Login</span>
			<span class="w-96 py-4">
				Welcome to the District 53 Launcher. <br>
				Please enter your password to decrypt your user data.
			</span>
			<div class="w-64">
				<div class="pb-4">
					<TextBox bind:value={tempPass} isPassword placeholder="Password" on:keypress={checkKeyInput}  />
				</div>
				<div class="flex w-full justify-center">
					{#if tempPass}
						<Button on:click={() => login()} class="bg-solar-orange bg-solar-orange-hover py-2 rounded-md">
							Login
						</Button>
					{/if}
				</div>
			</div>
		</div>
	{/if}
{/if}
<ModalContainer />