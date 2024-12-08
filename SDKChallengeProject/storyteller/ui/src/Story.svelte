<script lang="ts">
  import type { eventWithTime } from "rrweb/typings/types";
  import Replayer from "rrweb-player";
  import { mirror } from "rrweb";
  import { onMount, tick } from "svelte";
  import videojs from "video.js";
  import type { Tooltip, Skip, Story, RtcVideo } from "./utils";
  import { env } from "./store";
  // @ts-ignore
  require("@videojs/http-streaming");

  let playerEl: HTMLDivElement;
  let player: Replayer;
  let chapterIdx = 0;
  $: chapter = story.chapters[chapterIdx];
  let sceneIdx = 0;
  $: scene = chapter?.sequence[sceneIdx];
  $: tooltips = scene.effects
    .filter((e) => e.type === "tooltip")
    .map((e) => e.payload) as Tooltip[];
  $: skips = scene.effects
    .filter((e) => e.type === "skip")
    .map((e) => e.payload) as Skip[];
  $: videos = scene.effects
    .filter((e) => e.type === "rtc-video")
    .map((e) => e.payload) as RtcVideo[];
  let tooltipIdx = 0;
  $: nextTooltip = tooltips[tooltipIdx];
  let tooltipEl: HTMLDivElement;
  let tooltipTargetEl: HTMLElement;
  let tooltipTargetMaskEl: HTMLDivElement;
  let skipIdx = 0;
  $: nextSkip = skips[skipIdx];
  let videoIdx = 0;
  $: nextVideo = videos[videoIdx];

  export let story: Story;

  let percent = 0;

  function playScene() {
    percent = 0;
    playerEl.innerHTML = "";
    const { width, height } = playerEl.getBoundingClientRect();
    player = new Replayer({
      target: playerEl,
      props: {
        events: scene.events,
        skipInactive: true,
        autoPlay: true,
        showController: false,
        width,
        height,
        showDebug: false,
        showWarning: false,
      },
    });
    const mirror = player.getMirror();
    const rrwebReplayer = player.getReplayer();
    let scale = 1;
    const base = {
      left: 0,
      top: 0,
    };
    player.addEventListener("resize", () => {
      const matchedArr = getComputedStyle(
        rrwebReplayer.wrapper
      ).transform.match(/matrix\(([\d\.]+)/);
      if (matchedArr) {
        scale = parseFloat(matchedArr[1]);
      }
      const { left, top } = rrwebReplayer.wrapper.getBoundingClientRect();
      const {
        left: _left,
        top: _top,
      } = rrwebReplayer.wrapper.parentElement.getBoundingClientRect();
      base.left = left - _left;
      base.top = top - _top;
    });
    player.addEventListener("ui-update-progress", (event) => {
      percent = event.payload;
    });
    player.addEventListener("finish", async () => {
      percent = 1;
      if (sceneIdx < chapter.sequence.length - 1) {
        sceneIdx++;
        tooltipIdx = 0;
        await tick();
        return playScene();
      }
      while (chapterIdx < story.chapters.length - 1) {
        chapterIdx++;
        sceneIdx = 0;
        tooltipIdx = 0;
        await tick();
        if (scene) {
          return playScene();
        }
      }
    });
    player.addEventListener("ui-update-current-time", (event) => {
      if (nextTooltip && nextTooltip.timeOffset <= event.payload) {
        player.pause();
        videoPlayer?.pause();
        tooltipTargetEl = (mirror.getNode(
          nextTooltip.id
        ) as unknown) as HTMLElement;
        if (tooltipTargetEl) {
          const {
            left,
            top,
            width,
            height,
          } = tooltipTargetEl.getBoundingClientRect();
          Object.assign(tooltipEl.style, {
            left: `${(left + width) * scale + base.left}px`,
            top: `${(top + height) * scale + base.top}px`,
            display: "inherit",
          });
          Object.assign(tooltipTargetMaskEl.style, {
            left: `${left * scale + base.left}px`,
            top: `${top * scale + base.top}px`,
            width: `${width * scale}px`,
            height: `${height * scale}px`,
            display: "inherit",
          });
        }
      }
      if (nextSkip && nextSkip.timeOffset <= event.payload) {
        const nextTimeOffset = nextSkip.timeOffset + nextSkip.duration;
        player.goto(nextTimeOffset);
        skipIdx++;
      }
      if (nextVideo && nextVideo.timeOffset <= event.payload) {
        playVideo(nextVideo.file);
        videoIdx++;
      }
    });
  }

  function next() {
    Object.assign(tooltipEl.style, {
      left: 0,
      top: 0,
      display: "none",
    });
    Object.assign(tooltipTargetMaskEl.style, {
      left: 0,
      top: 0,
      display: "none",
    });
    tooltipIdx++;
    player.play();
    videoPlayer?.play();
  }

  let videoPlaceholder: HTMLDivElement;
  let videoPlayer;
  function playVideo(file: string) {
    const videoEl = document.createElement("video");
    videoEl.id = "player";
    videoEl.width = 150;
    videoEl.height = 150;
    videoEl.classList.add("video-js", "vjs-default-skin");
    videoPlaceholder.appendChild(videoEl);
    videoPlaceholder.style.display = "block";
    videoPlayer = videojs("#player");
    player?.pause();
    videoPlayer.ready(() => {
      videoPlayer.play();
      videoPlayer.on("canplay", () => {
        player?.play();
      });
    });
    videoPlayer.on("ended", function () {
      this.dispose();
      videoPlaceholder.style.display = "none";
    });
    videoPlayer.src({
      type: "application/x-mpegURL",
      src: `${$env.STORAGE_BUCKET_URL}/${file}`,
    });
  }

  onMount(() => {
    if (!scene) {
      return;
    }
    playScene();
  });
</script>

<style>
  .story-wrapper {
    padding: 0.5em;
    height: calc(100vh - 78px);
  }

  .story-player {
    height: 100%;
  }

  .story-tooltip {
    position: absolute;
    z-index: 999999;
    background: white;
    color: #212529;
    padding: 0.5em 0.75em;
    display: none;
    border: 1px solid rgba(0, 0, 0, 0.2);
    border-radius: 0.3rem;
  }

  .story-tooltip-meta {
    border-top: 1px solid rgba(0, 0, 0, 0.2);
    margin-top: 4px;
    padding-top: 4px;
    text-align: right;
  }

  .story-tooltip-target-mask {
    position: absolute;
    z-index: 999999;
    display: none;
    outline: 2px solid #e74a3b;
    outline-offset: -1px;
    cursor: pointer;
    background-color: transparent;
  }
  .story-tooltip-target-mask:hover {
    background-color: rgba(255, 255, 255, 0.25);
  }

  .base-wrapper {
    height: 100%;
    position: relative;
  }

  .story-scene-progress {
    height: 2px;
    background: #eee;
    position: relative;
    border-radius: 1px;
    box-sizing: border-box;
  }

  .story-scene-progress-step {
    height: 100%;
    position: absolute;
    left: 0;
    top: 0;
    background: rgba(78, 115, 223, 0.5);
    border-radius: 1px;
  }

  .story-video-placeholder {
    position: absolute;
    right: 2em;
    bottom: 2em;
    width: 150px;
    height: 150px;
    border-radius: 75px;
    overflow: hidden;
    display: none;
  }
</style>

<div class="story-wrapper row">
  <div class="col-9">
    <div class="base-wrapper">
      <div class="story-player" bind:this={playerEl} />
      <div bind:this={tooltipEl} class="story-tooltip">
        {#if nextTooltip}
          <div>{nextTooltip.content}</div>
          <div class="story-tooltip-meta">
            {tooltipIdx + 1}/{tooltips.length}
          </div>
        {/if}
      </div>
      <div
        bind:this={tooltipTargetMaskEl}
        class="story-tooltip-target-mask"
        on:click={next} />
      <div class="story-video-placeholder" bind:this={videoPlaceholder} />
    </div>
  </div>
  <div class="col-3">
    <div class="story-chapters">
      {#each story.chapters as chapter, cIdx}
        <div
          class={`card ${cIdx === chapterIdx ? 'bg-primary text-white' : 'bg-light text-black'} mb-1 shadow`}>
          <div class="card-body">
            {chapter.name}
            {#if cIdx !== chapterIdx}
              <div
                class={`${cIdx === chapterIdx ? 'text-white-50' : 'text-black-50'} small`}>
                {chapter.sequence.map((s) => s.name).join(' -> ')}
              </div>
            {/if}
          </div>
        </div>
        {#if cIdx === chapterIdx}
          <div class="pl-4">
            {#each chapter.sequence as scene, sIdx}
              <div class="card bg-light text-black shadow card-sm mb-1">
                {scene.name}
                <div class="story-scene-progress mt2">
                  <div
                    class="story-scene-progress-step"
                    style="width: {sIdx < sceneIdx ? 100 : sIdx > sceneIdx ? 0 : 100 * percent}%" />
                </div>
              </div>
            {/each}
          </div>
        {/if}
      {/each}
    </div>
  </div>
</div>
