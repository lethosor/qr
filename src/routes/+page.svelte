<script lang="ts">
	import '@tabler/core/dist/css/tabler.min.css';
	import QRCode from 'qrcode';
	import { onMount } from 'svelte';

	const storageKey = 'qr.input';
	let value = $state('');
	let margin = $state(4);
	let scale = $state(8);
	let preview = $derived(value ? QRCode.toDataURL(value, { margin, scale }) : null);
	let downloadMenuOpen = $state(false);
	let downloadToggle = $state<HTMLButtonElement>();
	let downloadMenu = $state<HTMLDivElement>();
	let toast = $state<{ message: string; success: boolean } | null>(null);

	function readSetting(key: string, min: number, max: number, fallback: number) {
		const stored = localStorage.getItem(key);
		const number = stored?.trim() ? Number(stored) : NaN;
		return Number.isInteger(number) && number >= min && number <= max ? number : fallback;
	}

	onMount(() => {
		try {
			value = localStorage.getItem(storageKey) ?? '';
			margin = readSetting('qr.margin', 0, 10, margin);
			scale = readSetting('qr.scale', 1, 20, scale);
		} catch {
			// Browser storage may be disabled.
		}
	});

	function saveValue(key: string, input: string) {
		try {
			if (input) localStorage.setItem(key, input);
			else localStorage.removeItem(key);
		} catch {
			// Keep the generator usable when storage is unavailable.
		}
	}

	$effect(() => {
		if (toast) {
			const timer = setTimeout(() => (toast = null), 3000);
			return () => clearTimeout(timer);
		}
	});

	async function downloadSvg() {
		downloadMenuOpen = false;
		downloadToggle?.focus();
		try {
			const width = (QRCode.create(value).modules.size + 2 * margin) * scale;
			const svg = await QRCode.toString(value, { type: 'svg', margin, width });
			const link = document.createElement('a');
			link.href = 'data:image/svg+xml;charset=utf-8,' + encodeURIComponent(svg);
			link.download = 'qr-code.svg';
			link.click();
		} catch {
			toast = { message: 'Could not download the SVG.', success: false };
		}
	}

	function dismissDownloadMenu(event: Event) {
		const target = event.target as Node;
		if (!downloadToggle?.contains(target) && !downloadMenu?.contains(target)) {
			downloadMenuOpen = false;
		}
	}

	async function copyImage(png: string) {
		const copiedPreview = preview;
		try {
			const bytes = Uint8Array.from(atob(png.split(',')[1]), (byte) => byte.charCodeAt(0));
			const blob = new Blob([bytes], { type: 'image/png' });
			await navigator.clipboard.write([new ClipboardItem({ 'image/png': blob })]);
			if (preview === copiedPreview) toast = { message: 'QR code copied.', success: true };
		} catch {
			if (preview === copiedPreview)
				toast = { message: 'Could not copy the QR code.', success: false };
		}
	}
</script>

<svelte:window
	onpointerdown={dismissDownloadMenu}
	onfocusin={dismissDownloadMenu}
	onkeydown={(event) => {
		if (event.key === 'Escape' && downloadMenuOpen) {
			downloadMenuOpen = false;
			downloadToggle?.focus();
		}
	}}
/>

<svelte:head>
	<title>QR generator</title>
</svelte:head>

<main class="container-sm py-5">
	<input
		class="form-control form-control-lg"
		type="text"
		aria-label="Text to encode"
		placeholder="Enter text or a URL"
		bind:value
		oninput={(event) => saveValue(storageKey, event.currentTarget.value)}
	/>

	<div class="mt-3">
		<label class="form-label" for="margin">
			Margin <span class="form-label-description">{margin}</span>
		</label>
		<input
			class="form-range"
			id="margin"
			type="range"
			min="0"
			max="10"
			step="1"
			bind:value={margin}
			oninput={(event) => saveValue('qr.margin', event.currentTarget.value)}
		/>
	</div>
	<div class="mt-3">
		<label class="form-label" for="scale">
			Scale <span class="form-label-description">{scale}</span>
		</label>
		<input
			class="form-range"
			id="scale"
			type="range"
			min="1"
			max="20"
			step="1"
			bind:value={scale}
			oninput={(event) => saveValue('qr.scale', event.currentTarget.value)}
		/>
	</div>

	{#if preview}
		{#await preview then png}
			<img
				class="d-block mx-auto mt-4"
				src={png}
				alt="QR code for the entered text"
				width="288"
				height="288"
			/>
			<div class="d-flex justify-content-center mt-3">
				<div class="btn-group" role="group" aria-label="QR code actions">
					<button class="btn" type="button" onclick={() => copyImage(png)}>
						<svg
							class="icon"
							aria-hidden="true"
							viewBox="0 0 24 24"
							fill="none"
							stroke="currentColor"
							stroke-width="2"
							stroke-linecap="round"
							stroke-linejoin="round"
						>
							<rect x="8" y="8" width="12" height="12" rx="2" />
							<path d="M16 8V6a2 2 0 0 0-2-2H6a2 2 0 0 0-2 2v8a2 2 0 0 0 2 2h2" />
						</svg>
						Copy
					</button>
					<a class="btn" href={png} download="qr-code.png">
						<svg
							class="icon"
							aria-hidden="true"
							viewBox="0 0 24 24"
							fill="none"
							stroke="currentColor"
							stroke-width="2"
							stroke-linecap="round"
							stroke-linejoin="round"
						>
							<path d="M4 16v2a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2v-2M7 10l5 5 5-5M12 15V3" />
						</svg>
						Download
					</a>
					<button
						class="btn dropdown-toggle dropdown-toggle-split"
						type="button"
						aria-label="Download options"
						aria-expanded={downloadMenuOpen}
						aria-controls="download-options"
						bind:this={downloadToggle}
						onclick={() => (downloadMenuOpen = !downloadMenuOpen)}
					></button>
					<div
						bind:this={downloadMenu}
						id="download-options"
						class="dropdown-menu end-0 top-100"
						class:show={downloadMenuOpen}
					>
						<button class="dropdown-item" type="button" onclick={downloadSvg}>
							Download SVG
						</button>
					</div>
				</div>
			</div>
		{:catch}
			<p class="text-danger mt-2" role="alert">Text is too long for a QR code.</p>
		{/await}
	{/if}
</main>

<div class="toast-container position-fixed bottom-0 end-0 p-3" role="status" aria-atomic="true">
	{#if toast}
		<div class="toast show w-auto" class:toast-success={toast.success}>
			<div class="toast-body">{toast.message}</div>
		</div>
	{/if}
</div>

<style>
	main {
		max-width: 36rem;
	}

	img {
		max-width: 100%;
		height: auto;
		image-rendering: pixelated;
	}
</style>
