<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import { useRouter, useRoute } from 'vue-router'

// Fallback mocks in case composables aren't resolving cleanly
const router = useRouter()
const route = useRoute()

// Wizard state
const currentStep = ref(1)
const totalSteps = 6
const isSaving = ref(false)
const uploadProgress = ref({})
const validationErrors = ref([])

// Track active Object URLs for memory management
const createdObjectUrls = new Set()

const createPreviewUrl = (file) => {
  const url = URL.createObjectURL(file)
  createdObjectUrls.add(url)
  return url
}

const revokePreviewUrl = (url) => {
  if (url && createdObjectUrls.has(url)) {
    URL.revokeObjectURL(url)
    createdObjectUrls.delete(url)
  }
}

// Form data structure
const releaseData = ref({
  basic: {
    title: '',
    displayArtist: '',
    type: 'Album',
    label: '',
    catalogNumber: '',
    barcode: '',
    releaseDate: '',
    originalReleaseDate: ''
  },
  tracks: [],
  assets: {
    coverImage: null,
    additionalImages: []
  },
  metadata: {
    genre: '',
    genreCode: '',  
    genreName: '', 
    subgenre: '',
    subgenreCode: '', 
    subgenreName: '', 
    language: 'en',
    copyright: '',
    copyrightYear: new Date().getFullYear(),
    productionYear: new Date().getFullYear()
  },
  territories: {
    mode: 'worldwide',
    included: [],
    excluded: []
  },
  preview: {
    ernVersion: '4.3',
    profile: 'AudioAlbum',
    validated: false
  }
})

// Auto-save draft functionality
const autoSaveTimer = ref(null)
const lastSavedAt = ref(null)
const hasUnsavedChanges = ref(false)

// Step titles
const stepTitles = [
  'Basic Information',
  'Track Management',
  'Asset Upload',
  'Metadata',
  'Territories & Rights',
  'Review & Generate'
]

const currentStepTitle = computed(() => stepTitles[currentStep.value - 1])

onUnmounted(() => {
  if (autoSaveTimer.value) clearTimeout(autoSaveTimer.value)
  createdObjectUrls.forEach(url => URL.revokeObjectURL(url))
  createdObjectUrls.clear()
})
</script>

<template>
  <div class="release-wizard container mx-auto p-6">
    <header class="mb-6">
      <h1 class="text-2xl font-bold">Release Builder</h1>
      <p class="text-gray-600">Step {{ currentStep }} of {{ totalSteps }}: {{ currentStepTitle }}</p>
    </header>

    <div class="wizard-content bg-white p-6 rounded shadow">
      <!-- Step 1: Basic Info -->
      <section v-if="currentStep === 1" class="space-y-4">
        <div>
          <label class="block text-sm font-medium mb-1">Release Title</label>
          <input 
            v-model="releaseData.basic.title" 
            type="text" 
            class="w-full border rounded p-2" 
            placeholder="Enter title"
          />
        </div>
        <div>
          <label class="block text-sm font-medium mb-1">Display Artist</label>
          <input 
            v-model="releaseData.basic.displayArtist" 
            type="text" 
            class="w-full border rounded p-2" 
            placeholder="Artist name"
          />
        </div>
      </section>

      <!-- Placeholder fallback for other steps -->
      <section v-else class="py-8 text-center text-gray-500">
        <p>Step {{ currentStep }} content interface goes here.</p>
      </section>
    </div>

    <!-- Navigation Footer -->
    <footer class="mt-6 flex justify-between">
      <button 
        @click="currentStep--" 
        :disabled="currentStep === 1"
        class="px-4 py-2 bg-gray-200 rounded disabled:opacity-50"
      >
        Previous
      </button>
      <button 
        @click="currentStep++" 
        :disabled="currentStep === totalSteps"
        class="px-4 py-2 bg-blue-600 text-white rounded disabled:opacity-50"
      >
        Next
      </button>
    </footer>
  </div>
</template>
