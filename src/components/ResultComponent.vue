<script setup>
import { ref, computed, onMounted } from 'vue'
import gsap from 'gsap'

const props = defineProps(['resultCharacter'])

const showDescription = ref(false)
const elementAppeared = ref(false)

const characterImages = {
  Tanjiro:
    'https://preview.redd.it/ke6gfqvldzh91.jpg?auto=webp&s=14769f3e4e290c74a0af26ca1c50a830255b056e',
  Zenitsu: 'https://motionbgs.com/media/5028/zenitsu-demon-slayer.jpg',
  Inosuke:
    'https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcS-FtMwJpwcgQAjLz558SQfx5j-K-M0fnw8OA&s',
  Giyu: 'https://images4.alphacoders.com/122/1225703.jpg',
}

const characterColors = {
  Tanjiro: '#3CAEA3',
  Zenitsu: '#F6D55C',
  Inosuke: '#ED553B',
  Giyu: '#20639B',
}

const characterDescription = computed(() => {
  switch (props.resultCharacter) {
    case 'Tanjiro':
      return "You're compassionate and determined, always putting others first. Your unwavering spirit burns as bright as the sun!"
    case 'Zenitsu':
      return "Though you may seem afraid, you possess hidden strength that emerges when needed most. You're the lightning in the storm!"
    case 'Inosuke':
      return 'Fierce and competitive, you always strive to be the strongest. Your wild spirit is as untameable as the mountain winds!'
    case 'Giyu':
      return 'Calm and disciplined, you have a strong sense of duty. Your resolve flows as steadily as a mighty river!'
    default:
      return ''
  }
})

onMounted(() => {
  gsap.to('.result-character', {
    opacity: 1,
    y: 0,
    duration: 1,
    ease: 'power3.out',
  })

  gsap.to('.character-image', {
    scale: 1,
    opacity: 1,
    duration: 1,
    delay: 0.5,
    ease: 'back.out(1.7)',
  })

  gsap.to('.katana-left, .katana-right', {
    opacity: 1,
    x: 0,
    duration: 0.8,
    delay: 1,
    ease: 'power2.out',
  })

  setTimeout(() => {
    showDescription.value = true
    gsap.from('.result-description', {
      opacity: 0,
      y: 20,
      duration: 0.8,
      ease: 'power2.out',
    })
  }, 2000)
})

const handleIntersection = entries => {
  const [entry] = entries
  if (entry.isIntersecting) {
    elementAppeared.value = true
  }
}
</script>

<template>
  <div
    class="result-container"
    :style="{ '--character-color': characterColors[resultCharacter] }"
  >
    <h2 class="result-title">Your Demon Slayer Spirit</h2>
    <div class="result-content">
      <div class="katana katana-left"></div>
      <div class="result-character-container">
        <h3 class="result-character">{{ resultCharacter }}</h3>
        <img
          :src="characterImages[resultCharacter]"
          :alt="resultCharacter"
          class="character-image"
        />
      </div>
      <div class="katana katana-right"></div>
    </div>
    <p v-if="showDescription" class="result-description">
      {{ characterDescription }}
    </p>
  </div>
</template>

<style scoped>
.result-container {
  @apply relative text-center max-w-2xl mx-auto p-8 rounded-lg overflow-hidden;
  background: radial-gradient(
    circle,
    rgba(0, 0, 0, 0.7) 0%,
    rgba(0, 0, 0, 0.9) 100%
  );
  box-shadow: 0 0 30px var(--character-color);
}

.result-title {
  @apply text-4xl font-bold mb-8 text-white;
  text-shadow:
    0 0 10px var(--character-color),
    0 0 20px var(--character-color);
}

.result-content {
  @apply flex items-center justify-center mb-8;
}

.result-character-container {
  @apply relative;
}

.result-character {
  @apply text-6xl font-bold text-white mb-4 opacity-0 transform translate-y-10;
  text-shadow:
    2px 2px 4px rgba(0, 0, 0, 0.5),
    0 0 20px var(--character-color);
}

.character-image {
  @apply w-64 h-64 object-cover rounded-full opacity-0 scale-50;
  box-shadow: 0 0 30px var(--character-color);
}

.katana {
  @apply w-24 h-2 bg-gray-300 opacity-0 mx-4;
  box-shadow: 0 0 10px var(--character-color);
}

.katana-left {
  transform: translateX(-100%);
}

.katana-right {
  transform: translateX(100%);
}

.result-description {
  @apply text-xl text-white mt-8 p-4 rounded-lg;
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(5px);
}

.ninja-stars {
  @apply absolute top-0 left-0 w-full h-full pointer-events-none;
}

.ninja-star {
  @apply absolute w-6 h-6 opacity-0;
  background: url('/ninja-star.svg') no-repeat center center / contain;
  animation:
    spin 2s linear infinite,
    fall 3s ease-in infinite;
  animation-delay: var(--delay);
}

@keyframes spin {
  100% {
    transform: rotate(360deg);
  }
}

@keyframes fall {
  0% {
    top: -10%;
    left: random(100%);
    opacity: 1;
  }
  100% {
    top: 110%;
    left: random(100%);
    opacity: 0;
  }
}
</style>
