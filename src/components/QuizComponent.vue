<script setup>
import { ref, onMounted, onBeforeUnmount, computed } from 'vue'
import * as THREE from 'three'
import gsap from 'gsap'
import ResultComponent from './ResultComponent.vue'

const questions = [
  {
    id: 1,
    text: 'You encounter a demon. What`s your first instinct?',
    options: [
      { id: 1, text: 'ATTACK IMMEDIATELY', character: 'Inosuke' },
      { id: 2, text: 'ANALYZE ITS MOVEMENTS', character: 'Giyu' },
      { id: 3, text: 'PROTECT NEARBY CIVILIANS', character: 'Tanjiro' },
      { id: 4, text: 'CALL FOR BACKUP', character: 'Zenitsu' },
    ],
  },
  {
    id: 2,
    text: 'Which quality do you value most in yourself?',
    options: [
      { id: 1, text: 'STRENGTH', character: 'Inosuke' },
      { id: 2, text: 'DISCIPLINE', character: 'Giyu' },
      { id: 3, text: 'COMPASSION', character: 'Tanjiro' },
      { id: 4, text: 'LOYALTY', character: 'Zenitsu' },
    ],
  },
  {
    id: 3,
    text: 'How do you approach training?',
    options: [
      { id: 1, text: 'PUSH MYSELF TO THE LIMIT', character: 'Inosuke' },
      { id: 2, text: 'FOCUS ON PERFECTING TECHNIQUE', character: 'Giyu' },
      { id: 3, text: 'BALANCE MIND AND BODY', character: 'Tanjiro' },
      {
        id: 4,
        text: 'PRACTICE UNTIL IT BECOMES INSTINCT',
        character: 'Zenitsu',
      },
    ],
  },
  {
    id: 4,
    text: 'What`s your biggest fear?',
    options: [
      { id: 1, text: 'BEING WEAK', character: 'Inosuke' },
      { id: 2, text: 'FAILING TO PROTECT OTHERS', character: 'Giyu' },
      { id: 3, text: 'LOSING LOVED ONES', character: 'Tanjiro' },
      { id: 4, text: 'FACING DANGERS ALONE', character: 'Zenitsu' },
    ],
  },
  {
    id: 5,
    text: 'How do you deal with setbacks?',
    options: [
      { id: 1, text: 'TRAIN HARDER', character: 'Inosuke' },
      { id: 2, text: 'REFLECT AND IMPROVE', character: 'Giyu' },
      { id: 3, text: 'STAY POSITIVE AND KEEP GOING', character: 'Tanjiro' },
      { id: 4, text: 'WORRY, THEN OVERCOME', character: 'Zenitsu' },
    ],
  },
]

const currentQuestionIndex = ref(0)
const currentQuestion = computed(() => questions[currentQuestionIndex.value])
const isQuestionVisible = ref(true)
const quizCompleted = ref(false)
const characterCounts = ref({
  Inosuke: 0,
  Giyu: 0,
  Tanjiro: 0,
  Zenitsu: 0,
})
const resultCharacter = ref('')

const canvasRef = ref(null)
const bgMusic = ref(null)
const isTransitioning = ref(false)
const isMusicPlaying = ref(false)
let scene,
  camera,
  renderer,
  inkBlobs = []

// Sound toggle function
const toggleMusic = () => {
  if (bgMusic.value) {
    if (isMusicPlaying.value) {
      bgMusic.value.pause()
    } else {
      bgMusic.value.play()
    }
    isMusicPlaying.value = !isMusicPlaying.value
  }
}

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

const selectAnswer = async option => {
  if (isTransitioning.value) return

  isTransitioning.value = true

  // Play selection sound
  createSelectSound()

  // Create new ink blob effect
  createInkBlob()

  // Track character selection
  characterCounts.value[option.character]++

  // Hide current question
  await gsap.to('.quiz-content', {
    opacity: 0,
    duration: 0.5,
    ease: 'power2.in',
  })

  // Tunnel animation
  await gsap.to(camera.position, {
    z: -10,
    duration: 1,
    ease: 'power2.in',
  })

  // Update to next question or complete quiz
  currentQuestionIndex.value++
  if (currentQuestionIndex.value >= questions.length) {
    completeQuiz()
  } else {
    // Reset camera position
    gsap.set(camera.position, { z: 10 })

    // Animate camera back to original position
    await gsap.to(camera.position, {
      z: 5,
      duration: 1,
      ease: 'power2.out',
    })

    // Show new question
    await gsap.to('.quiz-content', {
      opacity: 1,
      duration: 0.5,
      ease: 'power2.out',
    })
  }

  isTransitioning.value = false
}

const completeQuiz = () => {
  quizCompleted.value = true
  resultCharacter.value = Object.entries(characterCounts.value).reduce(
    (a, b) => (a[1] > b[1] ? a : b),
  )[0]

  // Show result
  gsap.to('.quiz-content', {
    opacity: 1,
    duration: 0.5,
    ease: 'power2.out',
  })
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
    <audio ref="bgMusic" src="/bg-music.mp3" preload="auto" loop></audio>

    <!-- Three.js canvas -->
    <canvas ref="canvasRef" class="absolute inset-0 -z-10"></canvas>

    <!-- Music toggle button -->
    <button @click="toggleMusic" class="music-button">
      🎵 {{ isMusicPlaying ? 'Turn Off' : 'Turn On' }} Music
    </button>

    <!-- Quiz content -->
    <div class="relative z-10 quiz-content">
      <template v-if="!quizCompleted">
        <!-- Question counter -->
        <div class="question-counter">
          <span class="cyber-text"
            >{{ currentQuestionIndex + 1 }} / {{ questions.length }}</span
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
      </template>

      <ResultComponent v-else :resultCharacter="resultCharacter" />
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
}

.cyber-text {
  @apply relative inline-block px-4 py-1 bg-gradient-to-r from-pink-500 to-cyan-500 text-transparent bg-clip-text;
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
</style>
