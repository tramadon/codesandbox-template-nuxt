<template>
  <div class="studio-shell" :style="shellStyle">
    <div class="texture-layer" />
    <canvas ref="canvas" class="orbital-canvas" />

    <section class="control-panel">
      <h1>Orbital Illustrations Lab</h1>
      <p class="subtitle">Generative circles inspired by geometric motion and painterly color rhythms.</p>

      <div class="style-card">
        <p class="label">Current style</p>
        <h2>{{ styleProfile.name }}</h2>
        <p>{{ styleProfile.description }}</p>
      </div>

      <div class="formula-grid">
        <article>
          <h3>Motion formula</h3>
          <p>{{ styleProfile.formula }}</p>
        </article>
        <article>
          <h3>Core principle</h3>
          <p>{{ styleProfile.principle }}</p>
        </article>
      </div>

      <button class="regen" @click="regenerate">
        Regenerate style
      </button>
    </section>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'

type OrbitMode = 'complex' | 'golden' | 'fibonacci' | 'lissajous'

interface StyleProfile {
  name: string
  description: string
  formula: string
  principle: string
  mode: OrbitMode
  palette: string[]
  background: string
  glow: string
  speed: number
}

interface Dot {
  baseAngle: number
  radialScale: number
  radius: number
  sizePulse: number
  phase: number
  lane: number
  color: string
  opacity: number
}

const STYLE_LIBRARY: StyleProfile[] = [
  {
    name: 'Aegean Pulse',
    description:
      'Layered circles sweep like marine currents: transparent cyan wakes, dense cobalt anchors, and bright pearl highlights.',
    formula: 'Z(θ) = e^{iθ} + 0.65·e^{iπθ}',
    principle: 'Complex plane superposition with phase drift.',
    mode: 'complex',
    palette: ['#ffffff', '#16e1ff', '#26c6ff', '#2557ff', '#2f0a98'],
    background: '#0f6ac8',
    glow: 'rgba(104, 221, 255, 0.24)',
    speed: 1,
  },
  {
    name: 'Golden Vortex',
    description:
      'The swarm expands and folds around golden-angle spokes, creating botanical spiral echoes with smooth oceanic hues.',
    formula: 'θₙ = n·φ·2π,  rₙ = k·√n',
    principle: 'Golden ratio spacing for non-overlapping radial growth.',
    mode: 'golden',
    palette: ['#f6f8ff', '#35deff', '#49b9ff', '#4d57ff', '#3818a8'],
    background: '#126fcb',
    glow: 'rgba(120, 245, 255, 0.22)',
    speed: 0.82,
  },
  {
    name: 'Fibonacci Tide',
    description:
      'Circle clusters breathe through Fibonacci intervals, giving wave trains that feel both mathematical and organic.',
    formula: 'r = F(n) mod R,  θ = n·0.42 + sin(t)',
    principle: 'Recursive integer rhythm mapped into radial lanes.',
    mode: 'fibonacci',
    palette: ['#ffffff', '#23d7ff', '#1fb8ff', '#1f64ff', '#2a0a9f'],
    background: '#0d67c7',
    glow: 'rgba(114, 233, 255, 0.2)',
    speed: 0.92,
  },
  {
    name: 'Lissajous Bloom',
    description:
      'Cross-wave frequencies sketch luminous loops, then stack translucent discs into soft illustrative ribbons.',
    formula: 'x = sin(a·t + δ), y = sin(b·t)',
    principle: 'Frequency interference and harmonic modulation.',
    mode: 'lissajous',
    palette: ['#f9fbff', '#1de7ff', '#31c7ff', '#345fff', '#3b149c'],
    background: '#106dca',
    glow: 'rgba(84, 227, 255, 0.25)',
    speed: 1.08,
  },
]

export default Vue.extend({
  name: 'OrbitalArtStudio',
  data() {
    return {
      dots: [] as Dot[],
      styleProfile: STYLE_LIBRARY[0],
      frameId: 0,
      startedAt: 0,
      resizeHandler: null as null | (() => void),
    }
  },
  computed: {
    shellStyle(): Record<string, string> {
      return {
        '--bg-color': this.styleProfile.background,
        '--glow-color': this.styleProfile.glow,
      }
    },
  },
  mounted() {
    this.regenerate()
    this.resizeHandler = () => {
      this.prepareCanvas()
      this.spawnDots()
    }
    window.addEventListener('resize', this.resizeHandler)
  },
  beforeDestroy() {
    if (this.frameId) {
      cancelAnimationFrame(this.frameId)
    }
    if (this.resizeHandler) {
      window.removeEventListener('resize', this.resizeHandler)
    }
  },
  methods: {
    regenerate() {
      const style = STYLE_LIBRARY[Math.floor(Math.random() * STYLE_LIBRARY.length)]
      this.styleProfile = style
      this.prepareCanvas()
      this.spawnDots()
      this.startedAt = performance.now()
      if (this.frameId) {
        cancelAnimationFrame(this.frameId)
      }
      this.animate()
    },
    prepareCanvas() {
      const canvas = this.$refs.canvas as HTMLCanvasElement | undefined
      if (!canvas) {
        return
      }
      const dpr = window.devicePixelRatio || 1
      const width = canvas.clientWidth
      const height = canvas.clientHeight
      canvas.width = Math.floor(width * dpr)
      canvas.height = Math.floor(height * dpr)
      const context = canvas.getContext('2d')
      if (context) {
        context.setTransform(dpr, 0, 0, dpr, 0, 0)
      }
    },
    spawnDots() {
      const canvas = this.$refs.canvas as HTMLCanvasElement | undefined
      if (!canvas) {
        return
      }
      const count = 170
      this.dots = Array.from({ length: count }, (_, index) => {
        const lane = index / count
        return {
          baseAngle: lane * Math.PI * 2,
          radialScale: 0.1 + Math.random() * 0.95,
          radius: 8 + Math.random() * 28,
          sizePulse: 0.3 + Math.random() * 1.4,
          phase: Math.random() * Math.PI * 2,
          lane,
          color: this.styleProfile.palette[index % this.styleProfile.palette.length],
          opacity: 0.16 + Math.random() * 0.84,
        }
      })
    },
    pointFor(dot: Dot, t: number, width: number, height: number) {
      const cx = width * 0.5
      const cy = height * 0.5
      const unit = Math.min(width, height) * 0.42

      if (this.styleProfile.mode === 'complex') {
        const theta = dot.baseAngle + t * 0.45
        const x = Math.cos(theta) + 0.65 * Math.cos(Math.PI * theta)
        const y = Math.sin(theta) + 0.65 * Math.sin(Math.PI * theta)
        return {
          x: cx + x * unit * dot.radialScale,
          y: cy + y * unit * dot.radialScale,
        }
      }

      if (this.styleProfile.mode === 'golden') {
        const phi = (1 + Math.sqrt(5)) / 2
        const theta = dot.lane * phi * Math.PI * 8 + t * 0.2
        const radius = Math.sqrt(dot.lane * 420 + 2) * 11
        return {
          x: cx + Math.cos(theta) * radius * 0.65,
          y: cy + Math.sin(theta) * radius * 0.65,
        }
      }

      if (this.styleProfile.mode === 'fibonacci') {
        const fibA = 55
        const fibB = 89
        const pulse = (fibA * dot.lane + fibB * 0.5) % 37
        const theta = dot.baseAngle * 1.2 + t * 0.33 + pulse * 0.04
        const radius = ((fibA + fibB * dot.lane) % 180) + dot.radialScale * unit
        return {
          x: cx + Math.cos(theta) * radius,
          y: cy + Math.sin(theta * 1.06) * radius * 0.82,
        }
      }

      const a = 3
      const b = 4
      const delta = Math.PI / 2.4
      const theta = t * 0.35 + dot.phase
      const x = Math.sin(a * theta + delta)
      const y = Math.sin(b * theta)
      return {
        x: cx + x * unit * dot.radialScale * 1.18,
        y: cy + y * unit * dot.radialScale * 1.18,
      }
    },
    animate() {
      const canvas = this.$refs.canvas as HTMLCanvasElement | undefined
      if (!canvas) {
        return
      }
      const context = canvas.getContext('2d')
      if (!context) {
        return
      }

      const width = canvas.clientWidth
      const height = canvas.clientHeight
      const elapsed = ((performance.now() - this.startedAt) / 1000) * this.styleProfile.speed

      context.clearRect(0, 0, width, height)

      for (const dot of this.dots) {
        const position = this.pointFor(dot, elapsed + dot.phase, width, height)
        const pulse = 0.7 + 0.3 * Math.sin(elapsed * 1.4 + dot.phase * dot.sizePulse)
        const radius = dot.radius * pulse

        context.globalAlpha = dot.opacity
        context.fillStyle = dot.color
        context.beginPath()
        context.arc(position.x, position.y, radius, 0, Math.PI * 2)
        context.fill()
      }

      context.globalAlpha = 1
      this.frameId = requestAnimationFrame(this.animate)
    },
  },
})
</script>

<style scoped>
.studio-shell {
  --bg-color: #0f6ac8;
  --glow-color: rgba(104, 221, 255, 0.24);
  position: relative;
  min-height: 100vh;
  background: var(--bg-color);
  overflow: hidden;
  color: #eff8ff;
  font-family: 'Inter', 'Segoe UI', sans-serif;
}

.texture-layer {
  position: absolute;
  inset: 0;
  background:
    radial-gradient(circle at 20% 20%, var(--glow-color), transparent 50%),
    radial-gradient(circle at 80% 0%, rgba(255, 255, 255, 0.08), transparent 45%),
    linear-gradient(145deg, rgba(6, 32, 121, 0.2), transparent 60%);
  pointer-events: none;
}

.orbital-canvas {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
}

.control-panel {
  position: relative;
  z-index: 2;
  max-width: 560px;
  margin: 2rem;
  padding: 1.2rem;
  border-radius: 18px;
  background: rgba(3, 20, 74, 0.45);
  backdrop-filter: blur(8px);
  box-shadow: 0 14px 38px rgba(0, 10, 36, 0.4);
}

h1 {
  margin: 0;
  font-size: clamp(1.5rem, 2vw, 2rem);
}

.subtitle {
  margin: 0.65rem 0 1rem;
  color: rgba(235, 247, 255, 0.86);
}

.style-card {
  padding: 0.85rem 1rem;
  border-radius: 14px;
  background: rgba(153, 220, 255, 0.16);
  margin-bottom: 0.9rem;
}

.label {
  margin: 0;
  font-size: 0.72rem;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  opacity: 0.74;
}

.style-card h2 {
  margin: 0.32rem 0;
  font-size: 1.2rem;
}

.style-card p {
  margin: 0;
  line-height: 1.4;
}

.formula-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 0.75rem;
  margin-bottom: 1rem;
}

.formula-grid article {
  padding: 0.75rem;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.08);
}

.formula-grid h3 {
  margin: 0 0 0.3rem;
  font-size: 0.86rem;
}

.formula-grid p {
  margin: 0;
  font-size: 0.91rem;
  line-height: 1.34;
}

.regen {
  border: none;
  border-radius: 999px;
  padding: 0.72rem 1.2rem;
  color: #003b87;
  font-weight: 700;
  cursor: pointer;
  background: linear-gradient(110deg, #f2fbff, #92f5ff);
}

.regen:hover {
  filter: brightness(1.05);
}

@media (max-width: 700px) {
  .control-panel {
    margin: 1rem;
    padding: 1rem;
  }

  .formula-grid {
    grid-template-columns: 1fr;
  }
}
</style>
