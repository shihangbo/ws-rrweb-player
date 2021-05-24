<script lang="ts">
  import { onMount, onDestroy, afterUpdate } from 'svelte';
  import { Replayer, unpack, mirror } from 'rrweb';
  import type { eventWithTime, playerConfig } from 'rrweb/typings/types';
  import {
    inlineCss,
    openFullscreen,
    exitFullscreen,
    isFullscreen,
    onFullscreenChange,
    typeOf,
    parseTime,
  } from './utils';
  import Controller from './Controller.svelte';

  export let width: number = 600;
  export let height: number = 466;
  export let events: eventWithTime[] = [];
  // export let skipInactive: boolean = true;
  export let autoPlay: boolean = true;
  export let speedOption: number[] = [1, 2, 4, 8];
  export let speed: number = 1;
  export let showController: boolean = true;
  export let tags: Record<string, string> = {};
  export let showTimeStamp: boolean = false;
  export let timestampRgihtWidth: number = 20

  export const getMirror = () => mirror;

  let firstChild = null
  const controllerHeight = 80;
  let player: HTMLElement;
  let frame: HTMLElement;
  let replayer: Replayer;
  let fullscreenListener: undefined | (() => void);
  let _width: number = width;
  let _height: number = height;
  let controller: {
    toggle: () => void;
    setSpeed: (speed: number) => void;
    toggleSkipInactive: () => void;
  } & Controller;

  let style: string;
  $: style = inlineCss({
    width: `${width}px`,
    height: `${height}px`,
  });
  let playerStyle: string;
  $: playerStyle = inlineCss({
    width: `${width}px`,
    height: `${height + (showController ? controllerHeight : 0)}px`,
  });
  let timestampStyle: string;
  $: timestampStyle = inlineCss({
    right: `${timestampRgihtWidth}px`
  })


  const updateScale = (
    el: HTMLElement,
    frameDimension: { width: number; height: number },
  ) => {
    const widthScale = width / frameDimension.width;
    const heightScale = height / frameDimension.height;
    el.style.transform =
      `scale(${Math.min(widthScale, heightScale, 1)})` +
      'translate(-50%, -50%)';
  };

  export const triggerResize = () => {
    updateScale(replayer.wrapper, {
      width: replayer.iframe.offsetWidth,
      height: replayer.iframe.offsetHeight,
    });
  };

  export const toggleFullscreen = () => {
    if (player) {
      // isFullscreen() ? exitFullscreen() : openFullscreen(player);
      if (isFullscreen()) {
        exitFullscreen()
        getTimestampRgihtWidth(width)
      } else {
        openFullscreen(player);
        getTimestampRgihtWidth(window.innerWidth)
      }
    }
  };

  export const addEventListener = (
    event: string,
    handler: (detail: unknown) => unknown,
  ) => {
    replayer.on(event, handler);
    switch (event) {
      case 'ui-update-current-time':
      case 'ui-update-progress':
      case 'ui-update-player-state':
        controller.$on(event, ({ detail }) => handler(detail));
      default:
        break;
    }
  };

  export const addEvent = (event: eventWithTime) => {
    replayer.addEvent(event);
  };
  export const getMetaData = () => replayer.getMetaData();
  export const getReplayer = () => replayer;

  // by pass controller methods as public API
  export const toggle = () => {
    controller.toggle();
  };
  export const setSpeed = (speed: number) => {
    controller.setSpeed(speed);
  };
  export const toggleSkipInactive = () => {
    controller.toggleSkipInactive();
  };
  export const play = () => {
    controller.play();
  };
  export const pause = () => {
    controller.pause();
  };
  export const goto = (timeOffset: number) => {
    controller.goto(timeOffset);
  };

  let timestamp = 0
  function changeTimestamp (e:CustomEvent):void {
    timestamp = e.detail.t
  }

  function getTimestampRgihtWidth(wd:number):void {
    if (!firstChild && frame.children && frame.children.length) {
      setTimeout(() => {
        let frameWidth = frame.clientWidth
        firstChild = frame.children[0]
        let replayerWrapperWidth = frameWidth
        try {
          replayerWrapperWidth = Math.min(wd / replayer.iframe.offsetWidth, height / replayer.iframe.offsetHeight, 1) * firstChild.clientWidth
        } catch (error) {}
        if (replayerWrapperWidth < frameWidth) {
          timestampRgihtWidth = (frameWidth - replayerWrapperWidth) / 2 + 20
        }
      }, 300)
    }
  }
  
  onMount(() => {
    // runtime type check
    if (speedOption !== undefined && typeOf(speedOption) !== 'array') {
      throw new Error('speedOption must be array');
    }
    speedOption.forEach((item) => {
      if (typeOf(item) !== 'number') {
        throw new Error('item of speedOption must be number');
      }
    });
    if (speedOption.indexOf(speed) < 0) {
      throw new Error(`speed must be one of speedOption,
        current config:
        {
          ...
          speed: ${speed},
          speedOption: [${speedOption.toString()}]
          ...
        }
        `);
    }

    replayer = new Replayer(events, {
      speed,
      root: frame,
      unpackFn: unpack,
      ...$$props,
    });

    replayer.on('resize', (dimension) => {
      updateScale(
        replayer.wrapper,
        dimension as { width: number; height: number },
      );
    });

    fullscreenListener = onFullscreenChange(() => {
      if (isFullscreen()) {
        setTimeout(() => {
          _width = width;
          _height = height;
          width = player.offsetWidth;
          height = player.offsetHeight;
          updateScale(replayer.wrapper, {
            width: replayer.iframe.offsetWidth,
            height: replayer.iframe.offsetHeight,
          });
        }, 0);
      } else {
        width = _width;
        height = _height;
        updateScale(replayer.wrapper, {
          width: replayer.iframe.offsetWidth,
          height: replayer.iframe.offsetHeight,
        });
      }
    });
  });
  
  afterUpdate(() => {
    getTimestampRgihtWidth(width)
  })

  onDestroy(() => {
    fullscreenListener && fullscreenListener();
  });
</script>

<div class="rr-player" bind:this={player} style={playerStyle}>
  {#if showTimeStamp}
    {#if timestamp}
      <div class="rr-timestamp" style={timestampStyle}>
        {parseTime(timestamp, '{y}-{m}-{d} {h}:{i}:{s}')}
      </div>
    {/if}
  {/if}
  <div class="rr-player__frame" bind:this={frame} {style} />
  {#if replayer}
    <Controller
      bind:this={controller}
      {replayer}
      {showController}
      {autoPlay}
      {speedOption}
      {tags}
      on:fullscreen={() => toggleFullscreen()}
      on:changeTimestamp={(e) => changeTimestamp(e)}/>
  {/if}
</div>

<style global>
  @import 'rrweb/dist/rrweb.min.css';

  .rr-player {
    position: relative;
    background: white;
    float: left;
    border-radius: 5px;
    box-shadow: 0 24px 48px rgba(17, 16, 62, 0.12);
  }

  .rr-player__frame {
    overflow: hidden;
  }

  .replayer-wrapper {
    float: left;
    clear: both;
    transform-origin: top left;
    left: 50%;
    top: 50%;
  }

  .replayer-wrapper > iframe {
    border: none;
  }
  .rr-timestamp {
    position: absolute;
    top: 20px;
    color: #DD8C16;
    font-size: 13px;
    z-index: 1;
  }
</style>

