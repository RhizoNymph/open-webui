<script lang="ts">
	import fileSaver from 'file-saver';
	const { saveAs } = fileSaver;

	import { toast } from 'svelte-sonner';

	import panzoom, { type PanZoom } from 'panzoom';
	import DOMPurify from 'dompurify';
	import { Canvg } from 'canvg';

	import { onMount, getContext } from 'svelte';
	const i18n = getContext('i18n');

	import { copyToClipboard } from '$lib/utils';

	import DocumentDuplicate from '../icons/DocumentDuplicate.svelte';
	import Tooltip from './Tooltip.svelte';
	import Clipboard from '../icons/Clipboard.svelte';
	import Reset from '../icons/Reset.svelte';
	import ArrowDownTray from '../icons/ArrowDownTray.svelte';

	export let className = '';
	export let svg = '';
	export let content = '';

	let instance: PanZoom;

	let sceneParentElement: HTMLElement;
	let sceneElement: HTMLElement;

	$: if (sceneElement) {
		instance = panzoom(sceneElement, {
			bounds: true,
			boundsPadding: 0.1,

			zoomSpeed: 0.065
		});
	}
	const resetPanZoomViewport = () => {
		instance.moveTo(0, 0);
		instance.zoomAbs(0, 0, 1);
		console.log(instance.getTransform());
	};

	const downloadAsSVG = () => {
		const svgBlob = new Blob([svg], { type: 'image/svg+xml' });
		saveAs(svgBlob, `diagram.svg`);
	};

	const downloadAsPNG = async () => {
		const canvas = document.createElement('canvas');
		const ctx = canvas.getContext('2d');
		// No longer need the rendered element if we modify the string
		// const renderedSvgElement = sceneElement?.firstElementChild as SVGElement | null;

		if (!ctx || !svg) {
			toast.error($i18n.t('Missing canvas context or SVG data'));
			return;
		}
		// if (!renderedSvgElement) { // No longer need this check
		// 	toast.error($i18n.t('Could not find rendered SVG element'));
		// 	return;
		// }

		try {
			// --- Dimension extraction ---
			const parser = new DOMParser();
			const svgDoc = parser.parseFromString(svg, 'image/svg+xml');
			const svgElement = svgDoc.documentElement as SVGGraphicsElement & {
				width?: SVGAnimatedLength;
				height?: SVGAnimatedLength;
			};

			let width, height;
			// Using viewBox is generally preferred for Mermaid diagrams if available
			if (svgElement.viewBox?.baseVal?.width && svgElement.viewBox?.baseVal?.height) {
				width = svgElement.viewBox.baseVal.width;
				height = svgElement.viewBox.baseVal.height;
			} else if (svgElement.width?.baseVal?.value && svgElement.height?.baseVal?.value) {
				width = svgElement.width.baseVal.value;
				height = svgElement.height.baseVal.value;
			}

			if (!width || width <= 0 || !height || height <= 0) {
				console.warn('SVG dimensions not found or invalid, attempting fallback calculation.');
				// Fallback: Calculate from getBBox if possible
                try {
                    const bbox = svgElement.getBBox();
                    width = bbox.width;
                    height = bbox.height;
                    console.log(`Using getBBox dimensions: ${width}x${height}`);
                     if (!width || width <= 0 || !height || height <= 0) throw new Error("Invalid BBox");
                } catch(e) {
				    console.warn('getBBox failed, using default 800x600.');
				    toast.info($i18n.t('SVG dimensions not found, using default size.'));
				    width = 800;
				    height = 600;
                }
			}

			canvas.width = width;
			canvas.height = height;
			// --- End of dimension extraction ---

			// --- Modify SVG String ---
			console.log("Original SVG for PNG:", svg);

			// Remove conflicting inline fill/stroke from actor rects
			let modifiedSvg = svg.replace(
				/<rect([^>]*?)class="actor[^"]*?"([^>]*?)fill="#eaeaea"([^>]*?)stroke="#666"([^>]*?)>/g,
				(match, g1, g2, g3, g4) => `<rect${g1}class="actor"${g2}${g3}${g4}>` // Remove fill and stroke
			);
            modifiedSvg = modifiedSvg.replace(
				/<rect([^>]*?)fill="#eaeaea"([^>]*?)stroke="#666"([^>]*?)class="actor[^"]*?"([^>]*?)>/g,
				(match, g1, g2, g3, g4) => `<rect${g1}${g2}${g3}class="actor"${g4}>` // Order might vary
			);

			console.log("Modified SVG for PNG:", modifiedSvg);
			// --- End Modify SVG String ---


			// *** Use the *modified* SVG string with Canvg ***
			const v = new Canvg(ctx, modifiedSvg); // Pass the modified SVG string
			await v.render();

			canvas.toBlob(
				(blob) => {
					if (blob) {
						saveAs(blob, 'diagram.png');
					} else {
						toast.error($i18n.t('Failed to create PNG image'));
					}
				},
				'image/png',
				1
			);
		} catch (error) {
			console.error('Error converting SVG to PNG:', error);
			toast.error($i18n.t('Error converting SVG to PNG'));
		}
	};
</script>

<div bind:this={sceneParentElement} class="relative {className}">
	<div bind:this={sceneElement} class="flex h-full max-h-full justify-center items-center">
		{@html svg}
	</div>

	{#if content}
		<div class=" absolute top-1 right-1">
			<div class="flex gap-1">
				<Tooltip content={$i18n.t('Download as SVG')}>
					<button
						class="p-1.5 rounded-lg border border-gray-100 dark:border-none dark:bg-gray-850 hover:bg-gray-50 dark:hover:bg-gray-800 transition"
						on:click={() => {
							downloadAsSVG();
						}}
					>
						<ArrowDownTray className=" size-4" />
					</button>
				</Tooltip>

				<Tooltip content={$i18n.t('Download as PNG')}>
					<button
						class="p-1.5 rounded-lg border border-gray-100 dark:border-none dark:bg-gray-850 hover:bg-gray-50 dark:hover:bg-gray-800 transition"
						on:click={() => {
							downloadAsPNG();
						}}
					>
						<ArrowDownTray className=" size-4" />
					</button>
				</Tooltip>

				<Tooltip content={$i18n.t('Reset view')}>
					<button
						class="p-1.5 rounded-lg border border-gray-100 dark:border-none dark:bg-gray-850 hover:bg-gray-50 dark:hover:bg-gray-800 transition"
						on:click={() => {
							resetPanZoomViewport();
						}}
					>
						<Reset className=" size-4" />
					</button>
				</Tooltip>

				<Tooltip content={$i18n.t('Copy to clipboard')}>
					<button
						class="p-1.5 rounded-lg border border-gray-100 dark:border-none dark:bg-gray-850 hover:bg-gray-50 dark:hover:bg-gray-800 transition"
						on:click={() => {
							copyToClipboard(content);
							toast.success($i18n.t('Copied to clipboard'));
						}}
					>
						<Clipboard className=" size-4" strokeWidth="1.5" />
					</button>
				</Tooltip>
			</div>
		</div>
	{/if}
</div>
