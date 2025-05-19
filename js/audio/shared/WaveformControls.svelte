<script lang="ts">
	import { Play, Pause, Forward, Backward, Undo, Trim } from "@gradio/icons";
	import { get_skip_rewind_amount } from "../shared/utils";
	import type { I18nFormatter } from "@gradio/utils";
	import WaveSurfer from "wavesurfer.js";
	import RegionsPlugin, {
		type Region
	} from "wavesurfer.js/dist/plugins/regions.js";
	import type { WaveformOptions } from "./types";
	import VolumeLevels from "./VolumeLevels.svelte";
	import VolumeControl from "./VolumeControl.svelte";
	import { createEventDispatcher, tick } from "svelte";

	export let waveform: WaveSurfer | undefined;
	export let audio_duration: number;
	export let i18n: I18nFormatter;
	export let playing: boolean;
	export let show_redo = false;
	export let interactive = false;
	export let handle_trim_audio: (start: number, end: number) => void;
	export let mode = "";
	export let container: HTMLDivElement;
	export let handle_reset_value: () => void;
	export let waveform_options: WaveformOptions = {};
	export let trim_region_settings: WaveformOptions = {};
	export let show_volume_slider = false;
	export let editable = true;
	export let segment_tags: Record<
		string,
		Array<Array<string | number>>
	> | null = null;
	export let trimDuration = 0;
	export let newRegionDuration = 0;
	export let update_region_comment: {
		id: string;
		comment: string | undefined;
	} | null = null;

	let playbackSpeeds = [0.5, 1, 1.5, 2];
	let playbackSpeed = playbackSpeeds[1];

	let trimRegion: RegionsPlugin | null = null;
	let newRegion: RegionsPlugin | null = null;
	let tagRegion: RegionsPlugin | null = null;
	let activeRegion: Region | null = null;

	let leftRegionHandle: HTMLDivElement | null;
	let rightRegionHandle: HTMLDivElement | null;
	let activeHandle = "";

	let currentVolume = 1;

	let waveformInitialized = false;

	let current_segment_tags: typeof segment_tags | null = null;
	let original_segment_tags: typeof segment_tags | null = null;
	let colorDict: Record<string, string> = {};
	let tagContent: Record<string, string> = {};
	let uniqueCounter = 0;

	$: if (waveform && container) {
		trimRegion = waveform.registerPlugin(RegionsPlugin.create());
		newRegion = waveform.registerPlugin(RegionsPlugin.create());
		tagRegion = waveform.registerPlugin(RegionsPlugin.create());
	}

	$: if (segment_tags && !original_segment_tags) {
		original_segment_tags = JSON.parse(JSON.stringify(segment_tags));
		current_segment_tags = JSON.parse(JSON.stringify(segment_tags));
	}

	$: if (waveform && !waveformInitialized) {
		waveformInitialized = true;
		waveform.on("ready", () => {
			renderSegmentTags(current_segment_tags ?? segment_tags);
		});
	}

	$: if (tagRegion) {
		tagRegion.on("region-updated", (region) => {
			const regionId = String(region.id);
			let updatedTags = JSON.parse(JSON.stringify(current_segment_tags));
			for (const tag_type in updatedTags) {
				const segments = updatedTags[tag_type];
				const index = (
					updatedTags[tag_type] as Array<
						[string, number, number, number, string]
					>
				).findIndex((seg) => seg[0] === regionId);
				if (index !== -1) {
					const { id, conf, start, duration, symbol } = segments[index];
					const newStart = region.start;
					const newEnd = region.end;

					const adjustedTags = adjustEditedSegmentToAvoidCollision(
						updatedTags,
						tag_type,
						index,
						newStart,
						newEnd
					);
					current_segment_tags = JSON.parse(JSON.stringify(adjustedTags));
					renderSegmentTags(current_segment_tags ?? segment_tags);
					break;
				}
			}
		});
	}

	function adjustEditedSegmentToAvoidCollision(
		tags: Record<string, Array<Array<string | number>>>,
		tagType: string,
		editedIndex: number,
		newStart: number,
		newEnd: number
	) {
		const adjustedTags: typeof tags = {};
		const segment = tags[tagType][editedIndex];
		const [id, conf, oldStart, oldDuration, symbol] = segment;
		const originalDuration = Number(oldDuration);
		let adjustedStart = newStart;
		let adjustedEnd = newEnd;

		const allSegments = Object.entries(tags).flatMap(([type, segments]) =>
			segments.map((seg, index) => ({ type, index, segment: seg }))
		);

		const updatedSegments = allSegments.map((item) =>
			item.segment[0] === id
				? { ...item, segment: [id, conf, newStart, newEnd - newStart, symbol] }
				: item
		);

		const sorted = updatedSegments.sort(
			(a, b) => Number(a.segment[2]) - Number(b.segment[2])
		);
		const newIndex = sorted.findIndex((item) => item.segment[0] === id);
		const previous = sorted[newIndex - 1];
		const next = sorted[newIndex + 1];

		if (previous) {
			const prevEnd = Number(previous.segment[2]) + Number(previous.segment[3]);
			if (adjustedStart < prevEnd) adjustedStart = prevEnd;
		}

		if (next) {
			const nextStart = Number(next.segment[2]);
			if (adjustedEnd > nextStart) adjustedEnd = nextStart;
		}

		for (const [type, segments] of Object.entries(tags)) {
			adjustedTags[type] = [...segments];
			if (type === tagType) {
				adjustedTags[type][editedIndex] = [
					id,
					conf,
					adjustedStart,
					adjustedEnd - adjustedStart,
					symbol
				];
			}
		}

		return adjustedTags;
	}

	async function renderSegmentTags(tags: typeof segment_tags) {
		if (!tags || !tagRegion) return;
		clearRegions();

		for (const [tag_type, segments] of Object.entries(tags)) {
			segments.forEach(
				([utt_id, conf, start_time, duration, symbol], index) => {
					const color = calcRegionColor(String(utt_id));
					tagContent[String(utt_id)] = String(symbol);

					const region = tagRegion?.addRegion({
						start: Number(start_time),
						end: Number(start_time) + Number(duration),
						color,
						drag: editable,
						resize: editable,
						id: String(utt_id),
						content: String(symbol)
					});

					const regionEl = region?.element;
					if (regionEl) {
						regionEl.classList.add("segment-region");
						regionEl.style.borderColor = color;
						regionEl.style.setProperty("--region-color", color);
					}

					regionEl?.addEventListener("mouseover", () => {
						dispatch("region_hover", {
							start: region?.start ?? 0,
							end: region?.end ?? 0,
							content: tagContent[String(utt_id)],
							id: String(utt_id)
						});
					});
				}
			);
		}
	}

	const random = (min: number, max: number): number =>
		Math.random() * (max - min) + min;
	const randomColor = (): string =>
		`rgba(${random(0, 255)}, ${random(0, 255)}, ${random(0, 255)}, 1)`;

	function calcRegionColor(id: string) {
		if (!colorDict[id]) {
			colorDict[id] = randomColor();
		}

		return colorDict[id];
	}

	function updateSegmentComment(id: string, comment: string | undefined) {
		if (!current_segment_tags) return;

		for (const tag_type in current_segment_tags) {
			current_segment_tags[tag_type] = current_segment_tags[tag_type].map(
				(segment) => {
					const [utt_id, conf, st, dur, symbol] = segment;
					if (utt_id === id) {
						return [utt_id, conf, st, dur, comment ?? symbol];
					}
					return segment;
				}
			);
		}

		current_segment_tags = JSON.parse(JSON.stringify(current_segment_tags));
		renderSegmentTags(current_segment_tags ?? segment_tags);
	}

	$: if (update_region_comment) {
		updateSegmentComment(
			update_region_comment.id,
			update_region_comment.comment
		);
	}

	$: trimRegion?.on("region-out", (region) => {
		region.play();
	});

	$: trimRegion?.on("region-updated", (region) => {
		trimDuration = region.end - region.start;
	});

	$: trimRegion?.on("region-clicked", (region, e) => {
		e.stopPropagation(); // prevent triggering a click on the waveform
		activeRegion = region;
		waveform?.setTime(region.start);
		waveform?.play();
	});

	$: newRegion?.on("region-out", (region) => {
		region.play();
	});

	$: newRegion?.on("region-updated", (region) => {
		newRegionDuration = region.end - region.start;
	});

	$: newRegion?.on("region-clicked", (region, e) => {
		e.stopPropagation(); // prevent triggering a click on the waveform
		activeRegion = region;
		waveform?.setTime(region.start);
		waveform?.play();
	});

	const addTrimRegion = (): void => {
		if (!trimRegion) return;
		activeRegion = trimRegion?.addRegion({
			start: audio_duration / 4,
			end: audio_duration / 2,
			...trim_region_settings
		});

		trimDuration = activeRegion.end - activeRegion.start;
	};

	const addNewRegion = (): void => {
		if (!newRegion) return;
		activeRegion = newRegion?.addRegion({
			start: audio_duration / 4,
			end: audio_duration / 2,
			...trim_region_settings
		});
		newRegionDuration = activeRegion.end - activeRegion.start;
	};

	$: if (activeRegion) {
		const shadowRoot = container.children[0]!.shadowRoot!;

		rightRegionHandle = shadowRoot.querySelector('[data-resize="right"]');
		leftRegionHandle = shadowRoot.querySelector('[data-resize="left"]');

		if (leftRegionHandle && rightRegionHandle) {
			leftRegionHandle.setAttribute("role", "button");
			rightRegionHandle.setAttribute("role", "button");
			leftRegionHandle?.setAttribute("aria-label", "Drag to adjust start time");
			rightRegionHandle?.setAttribute("aria-label", "Drag to adjust end time");
			leftRegionHandle?.setAttribute("tabindex", "0");
			rightRegionHandle?.setAttribute("tabindex", "0");

			leftRegionHandle.addEventListener("focus", () => {
				if (trimRegion) activeHandle = "left";
				else if (newRegion) activeHandle = "left";
			});

			rightRegionHandle.addEventListener("focus", () => {
				if (trimRegion) activeHandle = "right";
				else if (newRegion) activeHandle = "right";
			});
		}
	}

	const trimAudio = async (): Promise<void> => {
		if (waveform && trimRegion && activeRegion) {
			const start = activeRegion.start;
			const end = activeRegion.end;
			const newTags = recalculateSegmentTags(start, end, current_segment_tags);
			current_segment_tags = JSON.parse(JSON.stringify(newTags));
			handle_trim_audio(start, end);
			waveformInitialized = false;
			mode = "";
			activeRegion = null;
		}
	};

	function generateUniqueId() {
		const now = Date.now();
		return `seg_${now}_${uniqueCounter++}`;
	}

	const saveNewRegion = async (): Promise<void> => {
		if (waveform && newRegion && activeRegion) {
			const tagType = "default";
			if (!current_segment_tags) current_segment_tags = {};
			if (!current_segment_tags[tagType]) current_segment_tags[tagType] = [];

			const id = generateUniqueId();
			const start = activeRegion.start;
			const end = activeRegion.end;

			current_segment_tags[tagType].push([
				String(id),
				"1",
				start,
				end - start,
				""
			]);

			const editedIndex = current_segment_tags[tagType].length - 1;
			const adjustedTags = adjustEditedSegmentToAvoidCollision(
				current_segment_tags,
				tagType,
				editedIndex,
				start,
				end
			);
			current_segment_tags = JSON.parse(JSON.stringify(adjustedTags));
			const segment = current_segment_tags?.[tagType]?.[editedIndex];
			if (segment) {
				const [_, __, adjustedStart, adjustedDuration] = segment;

				tagRegion?.addRegion({
					start: Number(adjustedStart),
					end: Number(adjustedStart) + Number(adjustedDuration),
					color: calcRegionColor(String(id)),
					drag: true,
					resize: true,
					id: String(id),
					content: ""
				});
				renderSegmentTags(current_segment_tags);
				mode = "";
				activeRegion = null;
			}
		}
	};

	function recalculateSegmentTags(
		start: number,
		end: number,
		tags: typeof segment_tags
	): typeof segment_tags {
		if (!tags) return {};
		const result: typeof segment_tags = {};

		for (const [tag_type, segments] of Object.entries(tags)) {
			const updated = segments
				.filter(([_, __, st, dur]) => {
					const et = Number(st) + Number(dur);
					return et > start && Number(st) < end;
				})
				.map(([utt_id, conf, st, dur, symbol]) => {
					const newStart = Math.max(0, Number(st) - start);
					const originalEnd = Number(st) + Number(dur);
					const newEnd = Math.min(end, originalEnd) - start;
					const newDur = newEnd - newStart;
					return [utt_id, conf, newStart, newDur, symbol];
				});

			if (updated.length) {
				result[tag_type] = updated;
			}
		}

		return result;
	}

	const dispatch = createEventDispatcher<{
		region_hover: {
			start: number;
			end: number;
			content?: string;
			id: string;
		};
	}>();

	const clearRegions = (): void => {
		tagRegion?.getRegions().forEach((region) => {
			region.remove();
		});
		tagRegion?.clearRegions();
		trimRegion?.getRegions().forEach((region) => {
			region.remove();
		});
		trimRegion?.clearRegions();
		newRegion?.getRegions().forEach((region) => {
			region.remove();
		});
		newRegion?.clearRegions();
	};

	const toggleTrimmingMode = (): void => {
		clearRegions();
		if (mode === "edit") {
			mode = "";
		} else {
			mode = "edit";
			addTrimRegion();
		}
	};

	const toggleAddRegionMode = (): void => {
		clearRegions();
		if (mode === "add") {
			mode = "";
		} else {
			mode = "add";
			addNewRegion();
		}
	};

	const adjustRegionHandles = (handle: string, key: string): void => {
		let newStart;
		let newEnd;

		if (!activeRegion) return;
		if (handle === "left") {
			if (key === "ArrowLeft") {
				newStart = activeRegion.start - 0.05;
				newEnd = activeRegion.end;
			} else {
				newStart = activeRegion.start + 0.05;
				newEnd = activeRegion.end;
			}
		} else {
			if (key === "ArrowLeft") {
				newStart = activeRegion.start;
				newEnd = activeRegion.end - 0.05;
			} else {
				newStart = activeRegion.start;
				newEnd = activeRegion.end + 0.05;
			}
		}

		activeRegion.setOptions({
			start: newStart,
			end: newEnd
		});

		trimDuration = activeRegion.end - activeRegion.start;
	};

	$: trimRegion &&
		window.addEventListener("keydown", (e) => {
			if (e.key === "ArrowLeft") {
				adjustRegionHandles(activeHandle, "ArrowLeft");
			} else if (e.key === "ArrowRight") {
				adjustRegionHandles(activeHandle, "ArrowRight");
			}
		});
</script>

<div class="controls" data-testid="waveform-controls">
	<div class="control-wrapper">
		<button
			class="action icon volume"
			style:color={show_volume_slider
				? "var(--color-accent)"
				: "var(--neutral-400)"}
			aria-label="Adjust volume"
			on:click={() => (show_volume_slider = !show_volume_slider)}
		>
			<VolumeLevels {currentVolume} />
		</button>

		{#if show_volume_slider}
			<VolumeControl bind:currentVolume bind:show_volume_slider {waveform} />
		{/if}

		<button
			class:hidden={show_volume_slider}
			class="playback icon"
			aria-label={`Adjust playback speed to ${
				playbackSpeeds[
					(playbackSpeeds.indexOf(playbackSpeed) + 1) % playbackSpeeds.length
				]
			}x`}
			on:click={() => {
				playbackSpeed =
					playbackSpeeds[
						(playbackSpeeds.indexOf(playbackSpeed) + 1) % playbackSpeeds.length
					];

				waveform?.setPlaybackRate(playbackSpeed);
			}}
		>
			<span>{playbackSpeed}x</span>
		</button>
	</div>

	<div class="play-pause-wrapper">
		<button
			class="rewind icon"
			aria-label={`Skip backwards by ${get_skip_rewind_amount(
				audio_duration,
				waveform_options.skip_length
			)} seconds`}
			on:click={() =>
				waveform?.skip(
					get_skip_rewind_amount(audio_duration, waveform_options.skip_length) *
						-1
				)}
		>
			<Backward />
		</button>
		<button
			class="play-pause-button icon"
			on:click={() => waveform?.playPause()}
			aria-label={playing ? i18n("audio.pause") : i18n("audio.play")}
		>
			{#if playing}
				<Pause />
			{:else}
				<Play />
			{/if}
		</button>
		<button
			class="skip icon"
			aria-label="Skip forward by {get_skip_rewind_amount(
				audio_duration,
				waveform_options.skip_length
			)} seconds"
			on:click={() =>
				waveform?.skip(
					get_skip_rewind_amount(audio_duration, waveform_options.skip_length)
				)}
		>
			<Forward />
		</button>
	</div>

	<div class="settings-wrapper">
		{#if editable && interactive}
			{#if show_redo && mode === ""}
				<button
					class="action icon"
					aria-label="Reset audio"
					on:click={() => {
						handle_reset_value();
						current_segment_tags = JSON.parse(
							JSON.stringify(original_segment_tags)
						);
						renderSegmentTags(current_segment_tags);
						waveformInitialized = false;
						mode = "";
					}}
				>
					<Undo />
				</button>
			{/if}

			{#if mode === ""}
				<button
					class="action icon"
					aria-label="Trim audio to selection"
					on:click={toggleTrimmingMode}
				>
					<Trim />
				</button>
				<div class="add-row-container">
					<button class="add-row-button" on:click={() => toggleAddRegionMode()}>
						<span>+</span>
					</button>
				</div>
			{:else if mode === "edit"}
				<button class="text-button" on:click={trimAudio}>Trim</button>
				<button
					class="text-button"
					on:click={() => {
						toggleTrimmingMode();
						renderSegmentTags(current_segment_tags ?? segment_tags);
					}}
				>
					Cancel
				</button>
			{:else if mode === "add"}
				<button class="text-button" on:click={saveNewRegion}>Add</button>
				<button
					class="text-button"
					on:click={() => {
						toggleAddRegionMode();
						renderSegmentTags(current_segment_tags ?? segment_tags);
					}}
				>
					Cancel
				</button>
			{/if}
		{/if}
	</div>
</div>

<style>
	.settings-wrapper {
		display: flex;
		justify-self: self-end;
		align-items: center;
		grid-area: editing;
	}
	.text-button {
		border: 1px solid var(--neutral-400);
		border-radius: var(--radius-sm);
		font-weight: 300;
		font-size: var(--size-3);
		text-align: center;
		color: var(--neutral-400);
		height: var(--size-5);
		font-weight: bold;
		padding: 0 5px;
		margin-left: 5px;
	}

	.text-button:hover,
	.text-button:focus {
		color: var(--color-accent);
		border-color: var(--color-accent);
	}

	.controls {
		display: grid;
		grid-template-columns: 1fr 1fr 1fr;
		grid-template-areas: "controls playback editing";
		margin-top: 5px;
		align-items: center;
		position: relative;
		flex-wrap: wrap;
		justify-content: space-between;
	}
	.controls div {
		margin: var(--size-1) 0;
	}

	@media (max-width: 600px) {
		.controls {
			grid-template-columns: 1fr 1fr;
			grid-template-rows: auto auto;
			grid-template-areas:
				"playback playback"
				"controls editing";
		}
	}

	@media (max-width: 319px) {
		.controls {
			overflow-x: scroll;
		}
	}

	.hidden {
		display: none;
	}

	.control-wrapper {
		display: flex;
		justify-self: self-start;
		align-items: center;
		justify-content: space-between;
		grid-area: controls;
	}

	.action {
		width: var(--size-5);
		color: var(--neutral-400);
		margin-left: var(--spacing-md);
	}
	.icon:hover,
	.icon:focus {
		color: var(--color-accent);
	}
	.play-pause-wrapper {
		display: flex;
		justify-self: center;
		grid-area: playback;
	}

	@media (max-width: 600px) {
		.play-pause-wrapper {
			margin: var(--spacing-md);
		}
	}
	.playback {
		border: 1px solid var(--neutral-400);
		border-radius: var(--radius-sm);
		width: 5.5ch;
		font-weight: 300;
		font-size: var(--size-3);
		text-align: center;
		color: var(--neutral-400);
		height: var(--size-5);
		font-weight: bold;
	}

	.playback:hover,
	.playback:focus {
		color: var(--color-accent);
		border-color: var(--color-accent);
	}

	.rewind,
	.skip {
		margin: 0 10px;
		color: var(--neutral-400);
	}

	.play-pause-button {
		width: var(--size-8);
		display: flex;
		align-items: center;
		justify-content: center;
		color: var(--neutral-400);
		fill: var(--neutral-400);
	}

	.volume {
		position: relative;
		display: flex;
		justify-content: center;
		margin-right: var(--spacing-xl);
		width: var(--size-5);
	}

	.add-row-container {
		margin-top: var(--size-2);
	}

	.add-row-button {
		width: 100%;
		padding: var(--size-1);
		background: transparent;
		border: 1px dashed var(--border-color-primary);
		border-radius: var(--radius-sm);
		color: var(--body-text-color);
		cursor: pointer;
		transition: all 150ms;
	}

	.add-row-button:hover {
		background: var(--background-fill-secondary);
		border-style: solid;
	}
</style>
