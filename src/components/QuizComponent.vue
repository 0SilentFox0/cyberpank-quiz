<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import gsap from 'gsap'
import { createNoise3D } from 'simplex-noise'

const currentQuestion = ref({
  id: 1,
  text: 'Upon arriving at the docks your stomach growls. Fortunately, there are plenty of fish in the sea! But how will you catch one?',
  options: [
    { id: 1, text: 'USE THE SHARP STICK' },
    { id: 2, text: 'STEAL THE NET' },
    { id: 3, text: 'USE YOUR HANDS' },
    { id: 4, text: 'FORAGE FOR KELP INSTEAD' },
  ],
  totalQuestions: 8,
  currentQuestionNumber: 2,
})

const canvasRef = ref(null)
const bgMusic = ref(null)
const isTransitioning = ref(false)
const isMusicPlaying = ref(false)
let scene,
  camera,
  renderer,
  inkBlobs = []

const noise = createNoise3D()

// Previous Three.js related functions remain the same...
// Sound effect creation
const createSelectSound = () => {
  const audioContext = new (window.AudioContext || window.webkitAudioContext)()
  const oscillator = audioContext.createOscillator()
  const gainNode = audioContext.createGain()

  oscillator.type = 'sine'
  oscillator.frequency.setValueAtTime(880, audioContext.currentTime)
  oscillator.frequency.exponentialRampToValueAtTime(
    440,
    audioContext.currentTime + 0.2,
  )

  gainNode.gain.setValueAtTime(0.3, audioContext.currentTime)
  gainNode.gain.exponentialRampToValueAtTime(
    0.01,
    audioContext.currentTime + 0.2,
  )

  oscillator.connect(gainNode)
  gainNode.connect(audioContext.destination)

  oscillator.start()
  oscillator.stop(audioContext.currentTime + 0.2)
}

// Create ink material
const createInkMaterial = () => {
  return new THREE.ShaderMaterial({
    uniforms: {
      time: { value: 0 },
      color: { value: new THREE.Color('#000033') },
      noiseScale: { value: 1.0 },
    },
    vertexShader: `
      varying vec2 vUv;
      varying vec3 vPosition;

      void main() {
        vUv = uv;
        vPosition = position;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: `
      uniform float time;
      uniform vec3 color;
      uniform float noiseScale;
      varying vec2 vUv;
      varying vec3 vPosition;

      // Simplex noise function
      float noise(vec3 p) {
        return fract(sin(dot(p, vec3(12.9898, 78.233, 45.5432))) * 43758.5453);
      }

      void main() {
        float n = noise(vec3(vUv * noiseScale, time));
        float opacity = smoothstep(0.3, 0.7, n);
        gl_FragColor = vec4(color, opacity * 0.6);
      }
    `,
    transparent: true,
    blending: THREE.AdditiveBlending,
  })
}

// Create and add ink blob to scene
const createInkBlob = (delay = 0) => {
  if (!scene) return

  const geometry = new THREE.SphereGeometry(1, 32, 32)
  const material = createInkMaterial()
  const mesh = new THREE.Mesh(geometry, material)

  mesh.position.set(
    (Math.random() - 0.5) * 10,
    (Math.random() - 0.5) * 10,
    Math.random() * -5,
  )

  mesh.scale.set(0.001, 0.001, 0.001)
  scene.add(mesh)
  inkBlobs.push(mesh)

  // Initial animation
  gsap.to(mesh.scale, {
    x: 1 + Math.random(),
    y: 1 + Math.random(),
    z: 1 + Math.random(),
    duration: 2,
    delay,
    ease: 'power2.out',
  })

  // Continuous rotation
  gsap.to(mesh.rotation, {
    x: Math.PI * 2 * Math.random(),
    y: Math.PI * 2 * Math.random(),
    z: Math.PI * 2 * Math.random(),
    duration: 10,
    repeat: -1,
    ease: 'none',
  })

  // Noise animation
  gsap.to(material.uniforms.noiseScale, {
    value: 2.0,
    duration: 3,
    yoyo: true,
    repeat: -1,
    ease: 'sine.inOut',
  })

  return mesh
}

// Initialize Three.js scene
const initThree = () => {
  scene = new THREE.Scene()
  camera = new THREE.PerspectiveCamera(
    75,
    window.innerWidth / window.innerHeight,
    0.1,
    1000,
  )
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    alpha: true,
  })

  renderer.setSize(window.innerWidth, window.innerHeight)
  camera.position.z = 5

  // Add initial ink blobs with delays
  for (let i = 0; i < 5; i++) {
    createInkBlob(i * 0.5)
  }

  // Handle window resize
  const handleResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
  }

  window.addEventListener('resize', handleResize)
}

// Animation loop
const animate = () => {
  if (!renderer) return

  requestAnimationFrame(animate)

  inkBlobs.forEach(blob => {
    if (blob.material.uniforms.time) {
      blob.material.uniforms.time.value += 0.01
    }
  })

  renderer.render(scene, camera)
}

// Handle music play
const startBackgroundMusic = () => {
  if (bgMusic.value && !isMusicPlaying.value) {
    bgMusic.value.volume = 0.3
    bgMusic.value.loop = true
    bgMusic.value
      .play()
      .then(() => {
        isMusicPlaying.value = true
      })
      .catch(error => {
        console.log('Audio playback failed:', error)
      })
  }
}

// Handle answer selection
const selectAnswer = async option => {
  if (isTransitioning.value) return

  // Start music if not playing (user interaction)
  startBackgroundMusic()

  isTransitioning.value = true

  // Play selection sound
  createSelectSound()

  // Create new ink blob effect
  createInkBlob()

  // Animate answer selection
  await gsap.to('.quiz-content', {
    opacity: 0,
    duration: 0.5,
    ease: 'power2.in',
  })

  // Simulate loading next question
  await new Promise(resolve => setTimeout(resolve, 1000))

  // Update question
  currentQuestion.value.currentQuestionNumber++

  // Fade in new question
  await gsap.to('.quiz-content', {
    opacity: 1,
    duration: 0.5,
    ease: 'power2.out',
  })

  isTransitioning.value = false
}

onMounted(() => {
  initThree()
  animate()
})

onBeforeUnmount(() => {
  // Cleanup Three.js resources
  inkBlobs.forEach(blob => {
    blob.geometry.dispose()
    blob.material.dispose()
  })
  scene?.dispose()
  renderer?.dispose()
})
</script>

<template>
  <div class="quiz-container">
    <!-- Background music -->
    <audio ref="bgMusic" src="/bg-music.mp3" preload="auto"></audio>

    <!-- Three.js canvas -->
    <canvas ref="canvasRef" class="absolute inset-0 -z-10"></canvas>

    <!-- Music start button (shown only if music isn't playing) -->
    <button
      v-if="!isMusicPlaying"
      @click="startBackgroundMusic"
      class="music-button"
    >
      🎵 Start Music
    </button>

    <!-- Quiz content -->
    <div class="relative z-10 quiz-content">
      <!-- Question counter -->
      <div class="question-counter">
        <span class="cyber-text"
          >{{ currentQuestion.currentQuestionNumber }} /
          {{ currentQuestion.totalQuestions }}</span
        >
      </div>

      <!-- Question text -->
      <div class="question-text">
        {{ currentQuestion.text }}
      </div>

      <!-- Options -->
      <div class="options-container">
        <button
          v-for="option in currentQuestion.options"
          :key="option.id"
          class="option-button"
          :disabled="isTransitioning"
          @click="selectAnswer(option)"
        >
          {{ option.text }}
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.quiz-container {
  @apply relative min-h-screen bg-gradient-to-b from-slate-900 to-indigo-900 text-cyan-100 p-8 overflow-hidden;
}

.music-button {
  @apply fixed top-4 right-4 px-4 py-2 bg-cyan-400/20
         border border-cyan-400/30 rounded-lg
         hover:bg-cyan-400/30 transition-all duration-300
         text-cyan-100 font-mono z-50;
}

.quiz-content {
  @apply fixed inset-0 flex flex-col items-center justify-center
         max-w-2xl mx-auto p-8;
  text-shadow: 0 0 10px rgba(103, 232, 249, 0.5);
}

.question-counter {
  @apply text-lg font-mono tracking-wider mb-8;
  background: linear-gradient(45deg, #ff66b3, #66ffff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.question-text {
  @apply text-2xl text-center font-light leading-relaxed mb-12
         bg-slate-900/80 p-6 rounded-lg backdrop-blur-sm
         border border-cyan-400/20;
  text-shadow: 0 0 20px rgba(103, 232, 249, 0.3);
}

.options-container {
  @apply flex flex-col gap-4 w-full max-w-lg;
}

.option-button {
  @apply w-full py-4 px-6 text-lg font-mono tracking-wider
         border-2 border-cyan-400/30 bg-slate-900/80
         hover:bg-cyan-400/10 hover:border-cyan-400
         disabled:opacity-50 disabled:cursor-not-allowed
         transition-all duration-300 ease-out
         rounded-lg backdrop-blur-sm;
  box-shadow: 0 0 20px rgba(103, 232, 249, 0.1);
}

.option-button:hover:not(:disabled) {
  @apply shadow-lg;
  text-shadow: 0 0 10px rgba(103, 232, 249, 0.8);
  box-shadow: 0 0 30px rgba(103, 232, 249, 0.2);
}

.cyber-text {
  @apply relative inline-block px-4 py-1;
}

.cyber-text::before {
  content: '';
  @apply absolute inset-0 border-2 border-cyan-400/30
         transform skew-x-12 -z-10;
}
</style>
