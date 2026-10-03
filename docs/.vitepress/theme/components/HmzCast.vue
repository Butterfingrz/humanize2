<!--
  A terminal demo, played in the reader's browser from the text the terminal was sent:

    <HmzCast name="tui" alt="hmz opens, / lists its commands, and /flow opens a flow's menu" />

  `name` is a recording under `public/demo/`, `<name>.cast`, made from `tapes/<name>.tape` by
  `tapes/render.sh`. The words on it are real text -- they stay sharp at any zoom and can be
  selected -- and `alt` says what happens on it for anyone who cannot watch.

  It powers on like a tube when it is first scrolled into view, leans toward the pointer, and
  plays only while it is on screen. Under `prefers-reduced-motion` none of that moves: it shows
  the recording's last frame, and the play button is there for anyone who wants the motion.
-->
<script setup lang="ts">
import 'asciinema-player/dist/bundle/asciinema-player.css'

import { withBase } from 'vitepress'
import { onMounted, onUnmounted, ref } from 'vue'

import { motion } from '../motion/gsap'

const props = defineProps<{ name: string; alt: string; title?: string }>()

const root = ref<HTMLElement>()
const screen = ref<HTMLElement>()
const progress = ref(0)
const state = ref<'idle' | 'playing' | 'paused' | 'ended'>('idle')

type Player = import('asciinema-player').Player
let player: Player | undefined
let seen: IntersectionObserver | undefined
let frame = 0
let visible = false
let still = false
const undo: (() => void)[] = []

function tick() {
  const duration = player?.getDuration()
  if (player && duration) progress.value = Math.min(1, player.getCurrentTime() / duration)
  frame = requestAnimationFrame(tick)
}

async function load() {
  const { create } = await import('asciinema-player')
  player = create(withBase(`/demo/${props.name}.cast`), screen.value!, {
    fit: 'width',
    controls: false,
    autoPlay: false,
    preload: true,
    idleTimeLimit: 2,
    theme: 'hmz',
    poster: 'npt:9999',
    terminalFontFamily: 'var(--vp-font-family-mono)',
    terminalLineHeight: 1.25,
  })
  player.addEventListener('playing', () => (state.value = 'playing'))
  player.addEventListener('pause', () => (state.value = state.value === 'ended' ? 'ended' : 'paused'))
  player.addEventListener('ended', () => {
    state.value = 'ended'
    progress.value = 1
    if (!still) setTimeout(() => visible && state.value === 'ended' && replay(), 2600)
  })
}

function powerOn() {
  const gsap = motion()
  const el = root.value!
  return gsap
    .timeline()
    .from(el, { y: 48, rotateX: 14, scale: 0.94, opacity: 0, filter: 'blur(10px)', duration: 1.1 })
    .fromTo(
      el.querySelector('.hmz-cast-glass'),
      { scaleY: 0.004, scaleX: 0.6, filter: 'brightness(6)' },
      { scaleY: 1, scaleX: 1, filter: 'brightness(1)', duration: 0.7, ease: 'expo.out' },
      '-=0.55',
    )
    .fromTo(el.querySelector('.hmz-cast-flash'), { opacity: 0.9 }, { opacity: 0, duration: 0.8 }, '<0.1')
}

async function play() {
  if (!player) return
  if (state.value === 'ended') return replay()
  await player.play()
}

async function replay() {
  if (!player) return
  const gsap = motion()
  const glass = root.value!.querySelector('.hmz-cast-glass')
  await gsap.to(glass, { scaleY: 0.004, filter: 'brightness(5)', duration: 0.28, ease: 'cine.in' })
  await player.seek(0)
  gsap.to(glass, { scaleY: 1, filter: 'brightness(1)', duration: 0.5, ease: 'expo.out' })
  await player.play()
}

function toggle() {
  if (state.value === 'playing') player?.pause()
  else play()
}

// The window leans toward the pointer, and a light follows it across the glass.
function lean(el: HTMLElement) {
  const gsap = motion()
  const rx = gsap.quickTo(el, 'rotateX', { duration: 0.6 })
  const ry = gsap.quickTo(el, 'rotateY', { duration: 0.6 })
  const move = (e: PointerEvent) => {
    const box = el.getBoundingClientRect()
    const x = (e.clientX - box.left) / box.width
    const y = (e.clientY - box.top) / box.height
    ry((x - 0.5) * 6)
    rx((0.5 - y) * 5)
    el.style.setProperty('--hmz-cast-x', `${x * 100}%`)
    el.style.setProperty('--hmz-cast-y', `${y * 100}%`)
  }
  const leave = () => {
    rx(0)
    ry(0)
  }
  el.addEventListener('pointermove', move)
  el.addEventListener('pointerleave', leave)
  undo.push(() => {
    el.removeEventListener('pointermove', move)
    el.removeEventListener('pointerleave', leave)
  })
}

onMounted(() => {
  still = matchMedia('(prefers-reduced-motion: reduce)').matches
  const el = root.value!
  let started = false
  seen = new IntersectionObserver(
    async ([entry]) => {
      visible = entry.isIntersecting
      if (!visible) {
        if (state.value === 'playing') player?.pause()
        return
      }
      if (!started) {
        started = true
        await load()
        if (still) return
        lean(el)
        powerOn()
        setTimeout(() => visible && play(), 700)
      } else if (!still && state.value === 'paused') {
        play()
      }
    },
    { threshold: 0.35 },
  )
  seen.observe(el)
  const hidden = () => document.hidden && state.value === 'playing' && player?.pause()
  document.addEventListener('visibilitychange', hidden)
  undo.push(() => document.removeEventListener('visibilitychange', hidden))
  frame = requestAnimationFrame(tick)
})

onUnmounted(() => {
  cancelAnimationFrame(frame)
  seen?.disconnect()
  undo.forEach((f) => f())
  player?.dispose()
})
</script>

<template>
  <figure ref="root" class="hmz-cast" :class="`is-${state}`" :aria-label="alt">
    <div class="hmz-cast-halo" aria-hidden="true" />
    <div class="hmz-cast-window">
      <div class="hmz-cast-bar">
        <span class="hmz-cast-dots" aria-hidden="true"><i /><i /><i /></span>
        <span class="hmz-cast-title">{{ title ?? 'hmz' }}</span>
        <span class="hmz-cast-live" aria-hidden="true">{{ state === 'playing' ? 'LIVE' : '' }}</span>
        <button
          class="hmz-cast-button"
          type="button"
          :aria-label="state === 'playing' ? 'Pause the demo' : 'Play the demo'"
          @click="toggle"
        >
          <svg v-if="state === 'playing'" viewBox="0 0 16 16"><path d="M4 3h3v10H4zM9 3h3v10H9z" /></svg>
          <svg v-else-if="state === 'ended'" viewBox="0 0 16 16">
            <path d="M8 3a5 5 0 1 1-4.6 3H1.5L4.5 2.5 7.5 6H5.6A3.2 3.2 0 1 0 8 4.8z" />
          </svg>
          <svg v-else viewBox="0 0 16 16"><path d="M4 2.5v11l9.5-5.5z" /></svg>
        </button>
      </div>
      <div class="hmz-cast-glass" @click="toggle">
        <div ref="screen" class="hmz-cast-screen" />
        <div class="hmz-cast-scan" aria-hidden="true" />
        <div class="hmz-cast-flash" aria-hidden="true" />
      </div>
      <div class="hmz-cast-progress" aria-hidden="true">
        <span :style="{ transform: `scaleX(${progress})` }" />
      </div>
    </div>
    <figcaption class="hmz-cast-alt">{{ alt }}</figcaption>
  </figure>
</template>

<style>
@property --hmz-cast-angle {
  syntax: '<angle>';
  initial-value: 0deg;
  inherits: false;
}

.vp-doc .hmz-cast,
.hmz-cast {
  --hmz-cast-x: 50%;
  --hmz-cast-y: 0%;
  position: relative;
  max-width: 760px;
  margin: 28px 0 32px;
  perspective: 1200px;
  transform-style: preserve-3d;
  will-change: transform;
}

/* A ring of light turning around the window, blurred into a halo under it. */
.hmz-cast-halo {
  position: absolute;
  inset: -2px;
  border-radius: 14px;
  background: conic-gradient(
    from var(--hmz-cast-angle),
    #14b8a6,
    #4a90d9,
    #a855f7,
    #e0803a,
    #14b8a6
  );
  filter: blur(18px);
  opacity: 0.35;
  animation: hmz-cast-turn 8s linear infinite;
  transition: opacity 0.6s;
}

.hmz-cast.is-playing .hmz-cast-halo {
  opacity: 0.6;
}

.hmz-cast-window {
  position: relative;
  border-radius: 12px;
  padding: 1px;
  background: conic-gradient(
    from var(--hmz-cast-angle),
    rgba(20, 184, 166, 0.9),
    rgba(74, 144, 217, 0.4),
    rgba(168, 85, 247, 0.9),
    rgba(224, 128, 58, 0.4),
    rgba(20, 184, 166, 0.9)
  );
  animation: hmz-cast-turn 8s linear infinite;
  overflow: hidden;
}

.hmz-cast-bar,
.hmz-cast-glass,
.hmz-cast-progress {
  background: #16171d;
}

.hmz-cast-bar {
  display: flex;
  align-items: center;
  gap: 12px;
  height: 34px;
  padding: 0 10px 0 14px;
  border-radius: 11px 11px 0 0;
  background: #202129;
  color: #8b8e9c;
  font-family: var(--vp-font-family-mono);
  font-size: 12px;
}

.hmz-cast-dots {
  display: flex;
  gap: 8px;
}

.hmz-cast-dots i {
  width: 11px;
  height: 11px;
  border-radius: 50%;
  background: #ff5f57;
}

.hmz-cast-dots i:nth-child(2) {
  background: #febc2e;
}

.hmz-cast-dots i:nth-child(3) {
  background: #28c840;
}

.hmz-cast-title {
  flex: 1;
}

.hmz-cast-live {
  color: #f87171;
  font-size: 10px;
  letter-spacing: 0.12em;
}

.hmz-cast-live:not(:empty)::before {
  content: '';
  display: inline-block;
  width: 7px;
  height: 7px;
  margin-right: 6px;
  border-radius: 50%;
  background: #f87171;
  box-shadow: 0 0 8px #f87171;
  vertical-align: 1px;
  animation: hmz-cast-pulse 1.2s ease-in-out infinite;
}

.hmz-cast-button {
  display: grid;
  place-items: center;
  width: 26px;
  height: 26px;
  border-radius: 6px;
  color: #e2e4ea;
  transition: background 0.2s;
}

.hmz-cast-button:hover {
  background: rgba(255, 255, 255, 0.08);
}

.hmz-cast-button svg {
  width: 14px;
  height: 14px;
  fill: currentColor;
}

.hmz-cast-glass {
  position: relative;
  padding: 10px 12px;
  overflow: hidden;
  cursor: pointer;
  transform-origin: 50% 50%;
}

/* A light under the pointer, and a slow band sweeping down the glass like a tube's refresh. */
.hmz-cast-glass::after {
  content: '';
  position: absolute;
  inset: 0;
  pointer-events: none;
  background: radial-gradient(
    420px circle at var(--hmz-cast-x) var(--hmz-cast-y),
    rgba(45, 212, 191, 0.09),
    transparent 60%
  );
}

.hmz-cast-scan {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background:
    linear-gradient(to bottom, transparent 0, rgba(45, 212, 191, 0.06) 50%, transparent 100%) 0
      -30% / 100% 30% no-repeat,
    repeating-linear-gradient(to bottom, rgba(255, 255, 255, 0.025) 0 1px, transparent 1px 3px);
  animation: hmz-cast-sweep 6s linear infinite;
  mix-blend-mode: screen;
}

.hmz-cast-flash {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background: radial-gradient(ellipse at center, #fff 0, rgba(45, 212, 191, 0.6) 30%, transparent 70%);
  opacity: 0;
}

.hmz-cast-progress {
  height: 3px;
  border-radius: 0 0 11px 11px;
}

.hmz-cast-progress span {
  display: block;
  height: 100%;
  background: linear-gradient(90deg, #14b8a6, #4a90d9, #a855f7);
  box-shadow: 0 0 10px rgba(45, 212, 191, 0.8);
  transform-origin: left;
}

/* What happens on it, for a screen reader: the picture says it to everyone else. */
.hmz-cast-alt {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip-path: inset(50%);
  white-space: nowrap;
}

/* The player itself: no frame of its own, and the colours of the site's terminal. */
.hmz-cast .ap-wrapper {
  justify-content: flex-start;
}

.hmz-cast .ap-player,
.asciinema-player-theme-hmz {
  --term-color-foreground: #e2e4ea;
  --term-color-background: #16171d;
  --term-color-0: #282a36;
  --term-color-1: #ff5c57;
  --term-color-2: #5af78e;
  --term-color-3: #f3f99d;
  --term-color-4: #57c7ff;
  --term-color-5: #ff6ac1;
  --term-color-6: #9aedfe;
  --term-color-7: #f1f1f0;
  --term-color-8: #686868;
  --term-color-9: #ff5c57;
  --term-color-10: #5af78e;
  --term-color-11: #f3f99d;
  --term-color-12: #57c7ff;
  --term-color-13: #ff6ac1;
  --term-color-14: #9aedfe;
  --term-color-15: #f1f1f0;
  border-radius: 0;
  background: transparent;
}

@keyframes hmz-cast-turn {
  to {
    --hmz-cast-angle: 360deg;
  }
}

@keyframes hmz-cast-sweep {
  to {
    background-position:
      0 130%,
      0 0;
  }
}

@keyframes hmz-cast-pulse {
  50% {
    opacity: 0.3;
  }
}

@media (prefers-reduced-motion: reduce) {
  .hmz-cast-halo,
  .hmz-cast-window,
  .hmz-cast-scan,
  .hmz-cast-live::before {
    animation: none;
  }
}
</style>
