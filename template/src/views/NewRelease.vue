<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { useCatalog } from '../composables/useCatalog'
import { useAuth } from '../composables/useAuth'
import GenreSelector from '../components/GenreSelector.vue'
import { getGenreByCode } from '../dictionaries/genres'

const router = useRouter()
const route = useRoute()
const { user } = useAuth()
const { 
  createRelease, 
  updateRelease, 
  saveDraft, 
  loadRelease,
  uploadCoverImage,
  uploadTrackAudio,
  addTrack,
  updateTrack,
  removeTrack,
  currentRelease,
  isLoading,
  error 
} = useCatalog()

// Check if editing existing release
const releaseId = ref(route.params.id || null)
const isEditMode = computed(() => !!releaseId.value)

// Wizard state
const currentStep = ref(1)
const totalSteps = 6
const isSaving = ref(false)
const uploadProgress = ref({})
const validationErrors = ref([])

// Track active Object URLs for proper memory cleanup
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

// Form data
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

// Computed
const currentStepTitle = computed(() => stepTitles[currentStep.value - 1])

const displayGenreName = computed(() => {
  if (releaseData.value.metadata.subgenreName) {
    return releaseData.value.metadata.subgenreName
  } else if (releaseData.value.metadata.subgenre) {
    return releaseData.value.metadata.subgenre
  }
  
  if (releaseData.value.metadata.genreName) {
    return releaseData.value.metadata.genreName
  } else if (releaseData.value.metadata.genre) {
    return releaseData.value.metadata.genre
  }
  
  if (releaseData.value.metadata.subgenreCode) {
    const subgenre = getGenreByCode(releaseData.value.metadata.subgenreCode, 'genre-truth')
    return subgenre?.name || 'Not set'
  } else if (releaseData.value.metadata.genreCode) {
    const genre = getGenreByCode(releaseData.value.metadata.genreCode, 'genre-truth')
    return genre?.name || 'Not set'
  }
  
  return 'Not set'
})

// Watch genre code changes to sync human-readable names
watch(() => releaseData.value.metadata.genreCode, (newCode) => {
  if (newCode) {
    const genre = getGenreByCode(newCode, 'genre-truth')
    if (genre) {
      releaseData.value.metadata.genreName = genre.name
      releaseData.value.metadata.genre = genre.name
    }
  } else {
    releaseData.value.metadata.genreName = ''
    releaseData.value.metadata.genre = ''
  }
}, { immediate: true })

watch(() => releaseData.value.metadata.subgenreCode, (newCode) => {
  if (newCode) {
    const subgenre = getGenreByCode(newCode, 'genre-truth')
    if (subgenre) {
      releaseData.value.metadata.subgenreName = subgenre.name
      releaseData.value.metadata.subgenre = subgenre.name
    }
  } else {
    releaseData.value.metadata.subgenreName = ''
    releaseData.value.metadata.subgenre = ''
  }
}, { immediate: true })

// Validation methods
const validateBasicInfo = () => {
  const errors = []
  const basic = releaseData.value.basic
  
  if (!basic.title || basic.title.trim() === '') {
    errors.push('Release title is required')
  } else if (basic.title.length > 200) {
    errors.push('Release title must be less than 200 characters')
  }
  
  if (!basic.displayArtist || basic.displayArtist.trim() === '') {
    errors.push('Display artist is required')
  } else if (basic.displayArtist.length > 200) {
    errors.push('Display artist must be less than 200 characters')
  }
  
  if (!basic.type) {
    errors.push('Release type is required')
  } else if (!['Single', 'EP', 'Album', 'Compilation'].includes(basic.type)) {
    errors.push('Invalid release type selected')
  }
  
  if (basic.label && basic.label.length > 100) {
    errors.push('Label name must be less than 100 characters')
  }
  
  if (!basic.releaseDate) {
    errors.push('Release date is required')
  } else {
    const releaseDate = new Date(basic.releaseDate)
    const today = new Date()
    today.setHours(0, 0, 0, 0)
    
    if (isNaN(releaseDate.getTime())) {
      errors.push('Invalid release date')
    } else if (releaseDate < new Date('1900-01-01')) {
      errors.push('Release date cannot be before 1900')
    } else if (releaseDate > new Date('2100-01-01')) {
      errors.push('Release date cannot be after 2100')
    }
  }
  
  if (basic.originalReleaseDate) {
    const originalDate = new Date(basic.originalReleaseDate)
    const releaseDate = new Date(basic.releaseDate)
    
    if (isNaN(originalDate.getTime())) {
      errors.push('Invalid original release date')
    } else if (!isNaN(releaseDate.getTime()) && originalDate > releaseDate) {
      errors.push('Original release date cannot be after the release date')
    }
  }
  
  if (!basic.barcode || basic.barcode.trim() === '') {
    errors.push('UPC/EAN barcode is required for DDEX delivery')
  } else {
    const cleanBarcode = basic.barcode.trim().replace(/[\s-]/g, '')
    
    if (!/^\d+$/.test(cleanBarcode)) {
      errors.push('Barcode must contain only numbers')
    } else if (cleanBarcode.length !== 12 && cleanBarcode.length !== 13 && cleanBarcode.length !== 14) {
      errors.push('Barcode must be 12 digits (UPC-A), 13 digits (EAN-13), or 14 digits (EAN-14)')
    } else if (cleanBarcode.length === 12 && !validateUPCChecksum(cleanBarcode)) {
      errors.push('Invalid UPC barcode - checksum validation failed')
    } else if (cleanBarcode.length === 13 && !validateEANChecksum(cleanBarcode)) {
      errors.push('Invalid EAN-13 barcode - checksum validation failed')
    }
  }
  
  if (basic.catalogNumber && basic.catalogNumber.trim() !== '') {
    const catalogNumber = basic.catalogNumber.trim()
    if (catalogNumber.length > 50) {
      errors.push('Catalog number must be less than 50 characters')
    }
    if (!/^[A-Za-z0-9\-_]+$/.test(catalogNumber)) {
      errors.push('Catalog number can only contain letters, numbers, hyphens, and underscores')
    }
  }
  
  return errors
}

const validateUPCChecksum = (upc) => {
  if (upc.length !== 12) return false
  let sum = 0
  for (let i = 0; i < 11; i++) {
    const digit = parseInt(upc[i])
    sum += (i % 2 === 0) ? digit * 3 : digit
  }
  const checkDigit = (10 - (sum % 10)) % 10
  return checkDigit === parseInt(upc[11])
}

const validateEANChecksum = (ean) => {
  if (ean.length !== 13) return false
  let sum = 0
  for (let i = 0; i < 12; i++) {
    const digit = parseInt(ean[i])
    sum += (i % 2 === 0) ? digit : digit * 3
  }
  const checkDigit = (10 - (sum % 10)) % 10
  return checkDigit === parseInt(ean[12])
}

const cleanDataForFirestore = (data) => {
  const deepClean = (obj) => {
    if (obj === null || obj === undefined) return null
    if (obj instanceof Date) return obj
    if (obj instanceof File) return null
    if (Array.isArray(obj)) return obj.map(item => deepClean(item)).filter(item => item !== undefined)
    if (typeof obj === 'object') {
      const cleaned = {}
      for (const [key, value] of Object.entries(obj)) {
        if (value !== undefined) {
          const cleanedValue = deepClean(value)
          if (cleanedValue !== undefined) {
            cleaned[key] = cleanedValue
          }
        }
      }
      return cleaned
    }
    return obj
  }
  
  const cleanedData = deepClean(data)
  
  return {
    basic: cleanedData.basic || {},
    tracks: cleanedData.tracks || [],
    assets: {
      coverImage: cleanedData.assets?.coverImage?.url ? {
        url: cleanedData.assets.coverImage.url,
        name: cleanedData.assets.coverImage.name || '',
        size: cleanedData.assets.coverImage.size || 0,
        dimensions: cleanedData.assets.coverImage.dimensions || null
      } : null,
      additionalImages: []
    },
    metadata: {
      genre: cleanedData.metadata?.genre || '',
      genreCode: cleanedData.metadata?.genreCode || '',
      genreName: cleanedData.metadata?.genreName || '',
      subgenre: cleanedData.metadata?.subgenre || '',
      subgenreCode: cleanedData.metadata?.subgenreCode || '',
      subgenreName: cleanedData.metadata?.subgenreName || '',
      language: cleanedData.metadata?.language || 'en',
      copyright: cleanedData.metadata?.copyright || '',
      copyrightYear: cleanedData.metadata?.copyrightYear || new Date().getFullYear(),
      productionYear: cleanedData.metadata?.productionYear || new Date().getFullYear()
    },
    territories: {
      mode: cleanedData.territories?.mode || 'worldwide',
      included: cleanedData.territories?.included || [],
      excluded: cleanedData.territories?.excluded || []
    },
    preview: {
      ernVersion: cleanedData.preview?.ernVersion || '4.3',
      profile: cleanedData.preview?.profile || 'AudioAlbum',
      validated: cleanedData.preview?.validated || false
    }
  }
}

const uploadPendingFiles = async () => {
  if (!releaseId.value) return
  
  if (releaseData.value.assets.coverImage?.file) {
    try {
      const coverResult = await uploadCoverImage(
        releaseData.value.assets.coverImage.file, 
        releaseId.value
      )
      releaseData.value.assets.coverImage = coverResult
    } catch (err) {
      console.error('Error uploading cover:', err)
    }
  }
  
  for (let i = 0; i < releaseData.value.tracks.length; i++) {
    const track = releaseData.value.tracks[i]
    if (track.audio?.file) {
      try {
        if (!track.id) {
          const newTrack = await addTrack(releaseId.value, {
            title: track.title,
            artist: track.artist,
            duration: track.duration || 0,
            isrc: track.isrc || '',
            sequenceNumber: i + 1
          })
          releaseData.value.tracks[i].id = newTrack.id
        }
        
        const audioResult = await uploadTrackAudio(
          track.audio.file,
          releaseId.value,
          releaseData.value.tracks[i].id,
          (progress) => {
            uploadProgress.value[`track_${releaseData.value.tracks[i].id}`] = progress
          }
        )
        releaseData.value.tracks[i].audio = audioResult
      } catch (err) {
        console.error(`Error uploading audio for track ${i + 1}:`, err)
      }
    }
  }
}

const canProceed = computed(() => {
  switch (currentStep.value) {
    case 1:
      return validateBasicInfo().length === 0
    case 2:
      return releaseData.value.tracks.length > 0
    case 3:
      return releaseData.value.assets.coverImage !== null
    case 4:
      const hasGenre = !!(
        releaseData.value.metadata.genreCode || 
        releaseData.value.metadata.subgenreCode ||
        releaseData.value.metadata.genre ||
        releaseData.value.metadata.subgenre
      )
      const hasCopyright = !!releaseData.value.metadata.copyright
      return hasGenre && hasCopyright
    case 5:
      return true
    case 6:
      return releaseData.value.preview.validated
    default:
      return false
  }
})

onMounted(async () => {
  if (isEditMode.value) {
    try {
      await loadRelease(releaseId.value)
      if (currentRelease.value) {
        releaseData.value = {
          basic: currentRelease.value.basic || releaseData.value.basic,
          tracks: currentRelease.value.tracks || [],
          assets: currentRelease.value.assets || releaseData.value.assets,
          metadata: currentRelease.value.metadata || releaseData.value.metadata,
          territories: currentRelease.value.territories || releaseData.value.territories,
          preview: currentRelease.value.preview || releaseData.value.preview
        }
      }
    } catch (err) {
      console.error('Error loading release:', err)
      await showErrorToast('Failed to load release')
      router.push('/catalog')
    }
  }
})

// Clean up memory on unmount
onUnmounted(() => {
  if (autoSaveTimer.value) clearTimeout(autoSaveTimer.value)
  createdObjectUrls.forEach(url => URL.revokeObjectURL(url))
  createdObjectUrls.clear()
})

watch(releaseData, () => {
  hasUnsavedChanges.value = true
  scheduleAutoSave()
}, { deep: true })

const nextStep = () => {
  if (currentStep.value === 1) {
    const errors = validateBasicInfo()
    if (errors.length > 0) {
      validationErrors.value = errors
      showErrorToast('Please fix the errors before proceeding')
      return
    }
  }
  
  if (currentStep.value < totalSteps) {
    currentStep.value++
    validationErrors.value = []
  }
}

const previousStep = () => {
  if (currentStep.value > 1) {
    currentStep.value--
  }
}

const goToStep = (step) => {
  currentStep.value = step
}

const scheduleAutoSave = () => {
  if (autoSaveTimer.value) clearTimeout(autoSaveTimer.value)
  
  autoSaveTimer.value = setTimeout(() => {
    if (hasUnsavedChanges.value && releaseId.value) {
      autoSave()
    }
  }, 3000)
}

const autoSave = async () => {
  if (!hasUnsavedChanges.value) return
  
  try {
    const cleanedData = cleanDataForFirestore(releaseData.value)
    await saveDraft(cleanedData, releaseId.value)
    lastSavedAt.value = new Date()
    hasUnsavedChanges.value = false
  } catch (err) {
    console.error('Auto-save failed:', err)
  }
}

const saveAsDraft = async () => {
  if (isSaving.value) return
  
  isSaving.value = true
  try {
    const cleanedData = cleanDataForFirestore(releaseData.value)
    const draft = await saveDraft(cleanedData, releaseId.value)
    
    if (!releaseId.value) {
      releaseId.value = draft.id
      await uploadPendingFiles()
    }
    
    lastSavedAt.value = new Date()
    hasUnsavedChanges.value = false
    
    await showSuccessToast('Draft saved successfully')
    
    setTimeout(() => {
      router.push('/catalog')
    }, 1500)
  } catch (err) {
    console.error('Error saving draft:', err)
    await showErrorToast('Failed to save draft')
  } finally {
    isSaving.value = false
  }
}

const generateERN = async () => {
  if (isSaving.value) return
  
  isSaving.value = true
  try {
    let release
    
    if (!releaseId.value) {
      const cleanedData = cleanDataForFirestore(releaseData.value)
      release = await createRelease({
        ...cleanedData,
        status: 'draft'
      })
      
      releaseId.value = release.id
      await uploadPendingFiles()
      
      const finalData = cleanDataForFirestore(releaseData.value)
      release = await updateRelease(releaseId.value, {
        ...finalData,
        status: 'ready'
      })
    } else {
      const cleanedData = cleanDataForFirestore(releaseData.value)
      release = await updateRelease(releaseId.value, {
        ...cleanedData,
        status: 'ready'
      })
    }
    
    await showSuccessToast('Release created successfully!')
    
    setTimeout(() => {
      router.push(`/catalog/${release.id}`)
    }, 1500)
  } catch (err) {
    console.error('Error generating ERN:', err)
    await showErrorToast(err.message || 'Failed to create release')
  } finally {
    isSaving.value = false
  }
}

const cancelCreation = async () => {
  if (hasUnsavedChanges.value) {
    const confirmed = confirm('You have unsaved changes. Are you sure you want to leave?')
    if (!confirmed) return
  }
  router.push('/catalog')
}

const handleAddTrack = async () => {
  const newTrack = {
    title: `Track ${releaseData.value.tracks.length + 1}`,
    artist: releaseData.value.basic.displayArtist,
    duration: 0,
    isrc: '',
    audio: null
  }
  
  if (releaseId.value) {
    try {
      const track = await addTrack(releaseId.value, newTrack)
      releaseData.value.tracks.push(track)
    } catch (err) {
      console.error('Error adding track:', err)
      await showErrorToast('Failed to add track')
    }
  } else {
    releaseData.value.tracks.push({
      ...newTrack,
      id: Date.now().toString(),
      sequenceNumber: releaseData.value.tracks.length + 1
    })
  }
}

const handleUpdateTrack = async (index, updates) => {
  const track = releaseData.value.tracks[index]
  
  if (releaseId.value && track.id) {
    try {
      await updateTrack(releaseId.value, track.id, updates)
      releaseData.value.tracks[index] = { ...track, ...updates }
    } catch (err) {
      console.error('Error updating track:', err)
      await showErrorToast('Failed to update track')
    }
  } else {
    releaseData.value.tracks[index] = { ...track, ...updates }
  }
}

const handleRemoveTrack = async (index) => {
  const track = releaseData.value.tracks[index]
  if (!confirm(`Remove "${track.title}"?`)) return
  
  if (releaseId.value && track.id) {
    try {
      await removeTrack(releaseId.value, track.id)
      releaseData.value.tracks.splice(index, 1)
    } catch (err) {
      console.error('Error removing track:', err)
      await showErrorToast('Failed to remove track')
    }
  } else {
    releaseData.value.tracks.splice(index, 1)
  }
}

const handleCoverImageUpload = async (event) => {
  const file = event.target.files[0]
  if (!file) return
  
  if (!file.type.startsWith('image/')) {
    await showErrorToast('Please select an image file')
    return
  }
  
  if (releaseData.value.assets.coverImage?.preview) {
    revokePreviewUrl(releaseData.value.assets.coverImage.preview)
  }

  const img = new Image()
  const tempUrl = createPreviewUrl(file)
  img.src = tempUrl

  await new Promise(resolve => img.onload = resolve)
  
  if (img.width < 3000 || img.height < 3000) {
    const proceed = confirm('Cover image should be at least 3000x3000px. Continue anyway?')
    if (!proceed) {
      revokePreviewUrl(tempUrl)
      return
    }
  }
  
  if (releaseId.value) {
    try {
      uploadProgress.value.cover = 0
      const result = await uploadCoverImage(file, releaseId.value)
      releaseData.value.assets.coverImage = result
      await showSuccessToast('Cover image uploaded successfully')
    } catch (err) {
      console.error('Error uploading cover:', err)
      await showErrorToast('Failed to upload cover image')
    } finally {
      delete uploadProgress.value.cover
      revokePreviewUrl(tempUrl)
    }
  } else {
    releaseData.value.assets.coverImage = {
      file,
      preview: tempUrl,
      name: file.name,
      size: file.size
    }
  }
}

const handleAudioUpload = async (event, trackIndex) => {
  const file = event.target.files[0]
  if (!file) return
  
  const track = releaseData.value.tracks[trackIndex]
  const validFormats = ['audio/wav', 'audio/x-wav', 'audio/flac', 'audio/mpeg', 'audio/mp3']
  
  if (!validFormats.includes(file.type)) {
    await showErrorToast('Please upload WAV, FLAC, or MP3 files')
    return
  }
  
  if (releaseId.value && track.id) {
    try {
      uploadProgress.value[`track_${track.id}`] = 0
      const result = await uploadTrackAudio(
        file, 
        releaseId.value, 
        track.id,
        (progress) => {
          uploadProgress.value[`track_${track.id}`] = progress
        }
      )
      track.audio = result
      await showSuccessToast(`Audio uploaded for "${track.title}"`)
    } catch (err) {
      console.error('Error uploading audio:', err)
      await showErrorToast('Failed to upload audio file')
    } finally {
      delete uploadProgress.value[`track_${track.id}`]
    }
  } else {
    if (track.audio?.preview) {
      revokePreviewUrl(track.audio.preview)
    }
    track.audio = {
      file,
      name: file.name,
      size: file.size,
      preview: createPreviewUrl(file)
    }
  }
}

const validateERN = async () => {
  isSaving.value = true
  try {
    await new Promise(resolve => setTimeout(resolve, 1500))
    const errors = []
    
    const basicErrors = validateBasicInfo()
    errors.push(...basicErrors)
    
    if (releaseData.value.tracks.length === 0) {
      errors.push('At least one track is required')
    } else {
      releaseData.value.tracks.forEach((track, index) => {
        if (!track.title || track.title.trim() === '') {
          errors.push(`Track ${index + 1}: Title is required`)
        }
        if (!track.artist || track.artist.trim() === '') {
          errors.push(`Track ${index + 1}: Artist is required`)
        }
        if (!track.audio) {
          errors.push(`Track ${index + 1}: Audio file is required`)
        }
      })
    }
    
    if (!releaseData.value.assets.coverImage) {
      errors.push('Cover image is required')
    }
    
    const hasGenre = !!(
      releaseData.value.metadata.genreCode || 
      releaseData.value.metadata.subgenreCode ||
      releaseData.value.metadata.genre ||
      releaseData.value.metadata.subgenre
    )
    
    if (!hasGenre) {
      errors.push('Genre is required')
    }
    if (!releaseData.value.metadata.copyright) {
      errors.push('Copyright information is required')
    }
    
    if (errors.length > 0) {
      validationErrors.value = errors
      await showErrorToast('Validation failed. Please check all required fields.')
      return
    }
    
    releaseData.value.preview.validated = true
    validationErrors.value = []
    await showSuccessToast('Validation successful!')
  } catch (err) {
    console.error('Validation error:', err)
    await showErrorToast('Validation failed')
  } finally {
    isSaving.value = false
  }
}

const showSuccessToast = (message) => {
  console.log('✅', message)
  return Promise.resolve()
}

const showErrorToast = (message) => {
  console.error('❌', message)
  return Promise.resolve()
}

const formatFileSize = (bytes) => {
  if (!bytes) return '0 B'
  const sizes = ['B', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(1024))
  return Math.round(bytes / Math.pow(1024, i) * 100) / 100 + ' ' + sizes[i]
}

const formatDuration = (seconds) => {
  if (!seconds) return '0:00'
  const mins = Math.floor(seconds / 60)
  const secs = seconds % 60
  return `${mins}:${secs.toString().padStart(2, '0')}`
}
</script>

<template>
  <div class="release-wizard max-w-5xl mx-auto p-6 bg-slate-50 min-h-screen text-slate-800">
    <!-- Header -->
    <header class="mb-8 flex justify-between items-center bg-white p-6 rounded-xl shadow-sm border border-slate-200">
      <div>
        <h1 class="text-2xl font-bold text-slate-900">{{ isEditMode ? 'Edit Release' : 'New Release Wizard' }}</h1>
        <p class="text-slate-500 text-sm mt-1">DDEX ERN 4.3 Compliant Release Builder</p>
      </div>
      <div class="flex items-center space-x-3 text-xs">
        <span v-if="lastSavedAt" class="text-slate-400">Auto-saved {{ new Date(lastSavedAt).toLocaleTimeString() }}</span>
        <button @click="saveAsDraft" :disabled="isSaving" class="px-3 py-2 bg-slate-100 hover:bg-slate-200 font-medium rounded-lg transition">
          Save Draft
        </button>
      </div>
    </header>

    <!-- Stepper Navigation -->
    <nav class="mb-8 bg-white p-4 rounded-xl shadow-sm border border-slate-200">
      <ol class="flex flex-wrap md:flex-nowrap justify-between gap-2">
        <li 
          v-for="(title, idx) in stepTitles" 
          :key="idx"
          @click="goToStep(idx + 1)"
          class="flex-1 cursor-pointer flex flex-col items-center p-2 rounded-lg transition"
          :class="currentStep === idx + 1 ? 'bg-blue-50 text-blue-600 font-semibold' : 'text-slate-400 hover:text-slate-600'"
        >
          <span class="text-xs font-mono mb-1">STEP 0{{ idx + 1 }}</span>
          <span class="text-xs text-center line-clamp-1">{{ title }}</span>
        </li>
      </ol>
    </nav>

    <!-- Validation Errors Top Bar -->
    <div v-if="validationErrors.length > 0" class="mb-6 bg-red-50 border border-red-200 p-4 rounded-xl">
      <p class="text-red-700 font-medium text-sm mb-2">Please fix the following validation errors:</p>
      <ul class="list-disc list-inside text-xs text-red-600 space-y-1">
        <li v-for="(err, idx) in validationErrors" :key="idx">{{ err }}</li>
      </ul>
    </div>

    <!-- Step Content Container -->
    <main class="bg-white p-8 rounded-xl shadow-sm border border-slate-200 min-h-[400px]">
      
      <!-- STEP 1: Basic Information -->
      <section v-if="currentStep === 1" class="space-y-6">
        <h2 class="text-lg font-bold border-b border-slate-100 pb-3">1. Basic Information</h2>
        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <div>
            <label class="block text-xs font-bold text-slate-700 uppercase mb-2">Release Title *</label>
            <input v-model="releaseData.basic.title" type="text" class="w-full border border-slate-300 rounded-lg p-3 text-sm focus:ring-2 focus:ring-blue-500 outline-none" placeholder="Album or Single Title" />
          </div>

          <div>
            <label class="block text-xs font-bold text-slate-700 uppercase mb-2">Display Artist *</label>
            <input v-model="releaseData.basic.displayArtist" type="text" class="w-full border border-slate-300 rounded-lg p-3 text-sm focus:ring-2 focus:ring-blue-500 outline-none" placeholder="Primary Performing Artist" />
          </div>

          <div>
            <label class="block text-xs font-bold text-slate-700 uppercase mb-2">Release Type *</label>
            <select v-model="releaseData.basic.type" class="w-full border border-slate-300 rounded-lg p-3 text-sm focus:ring-2 focus:ring-blue-500 outline-none bg-white">
              <option value="Single">Single</option>
              <option value="EP">EP</option>
              <option value="Album">Album</option>
              <option value="Compilation">Compilation</option>
            </select>
          </div>

          <div>
            <label class="block text-xs font-bold text-slate-700 uppercase mb-2">Record Label</label>
            <input v-model="releaseData.basic.label" type="text" class="w-full border border-slate-300 rounded-lg p-3 text-sm focus:ring-2 focus:ring-blue-500 outline-none" placeholder="Label Name" />
          </div>

          <div>
            <label class="block text-xs font-bold text-slate-700 uppercase mb-2">UPC / EAN Barcode *</label>
            <input v-model="releaseData.basic.barcode" type="text" class="w-full border border-slate-300 rounded-lg p-3 text-sm focus:ring-2 focus:ring-blue-500 outline-none" placeholder="12, 13, or 14-digit GTIN barcode" />
          </div>

          <div>
            <label class="block text-xs font-bold text-slate-700 uppercase mb-2">Catalog Number</label>
            <input v-model="releaseData.basic.catalogNumber" type="text" class="w-full border border-slate-300 rounded-lg p-3 text-sm focus:ring-2 focus:ring-blue-500 outline-none" placeholder="e.g. CAT-104" />
          </div>

          <div>
            <label class="block text-xs font-bold text-slate-700 uppercase mb-2">Release Date *</label>
            <input v-model="releaseData.basic.releaseDate" type="date" class="w-full border border-slate-300 rounded-lg p-3 text-sm focus:ring-2 focus:ring-blue-500 outline-none" />
          </div>

          <div>
            <label class="block text-xs font-bold text-slate-700 uppercase mb-2">Original Release Date</label>
            <input v-model="releaseData.basic.originalReleaseDate" type="date" class="w-full border border-slate-300 rounded-lg p-3 text-sm focus:ring-2 focus:ring-blue-500 outline-none" />
          </div>
        </div>
      </section>

      <!-- STEP 2: Track Management -->
      <section v-if="currentStep === 2" class="space-y-6">
        <div class="flex justify-between items-center border-b border-slate-100 pb-3">
          <h2 class="text-lg font-bold">2. Track Management</h2>
          <button @click="handleAddTrack" class="px-4 py-2 bg-blue-600 hover:bg-blue-700 text-white font-medium text-sm rounded-lg transition">
            + Add Track
          </button>
        </div>

        <div v-if="releaseData.tracks.length === 0" class="text-center py-12 text-slate-400 bg-slate-50 rounded-xl border border-dashed border-slate-200">
          No tracks added yet. Click "+ Add Track" to begin building your release sequence.
        </div>

        <div v-else class="space-y-4">
          <div v-for="(track, idx) in releaseData.tracks" :key="track.id || idx" class="p-4 border border-slate-200 rounded-xl bg-slate-50 space-y-4">
            <div class="flex justify-between items-center">
              <span class="text-xs font-mono font-bold text-slate-400">TRACK {{ idx + 1 }}</span>
              <button @click="handleRemoveTrack(idx)" class="text-xs text-red-500 hover:underline">Remove</button>
            </div>
            
            <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
              <div>
                <label class="block text-xs font-bold text-slate-600 mb-1">Track Title *</label>
                <input v-model="track.title" @change="handleUpdateTrack(idx, { title: track.title })" type="text" class="w-full border rounded-lg p-2 text-sm bg-white" />
              </div>
              <div>
                <label class="block text-xs font-bold text-slate-600 mb-1">Artist *</label>
                <input v-model="track.artist" @change="handleUpdateTrack(idx, { artist: track.artist })" type="text" class="w-full border rounded-lg p-2 text-sm bg-white" />
              </div>
              <div>
                <label class="block text-xs font-bold text-slate-600 mb-1">ISRC Code</label>
                <input v-model="track.isrc" @change="handleUpdateTrack(idx, { isrc: track.isrc })" type="text" placeholder="CC-XXX-YY-NNNNN" class="w-full border rounded-lg p-2 text-sm bg-white" />
              </div>
            </div>

            <div>
              <label class="block text-xs font-bold text-slate-600 mb-1">Audio Master File (WAV, FLAC, MP3) *</label>
              <div class="flex items-center space-x-3">
                <input type="file" accept="audio/*" @change="handleAudioUpload($event, idx)" class="text-xs text-slate-500 file:mr-4 file:py-2 file:px-4 file:rounded-lg file:border-0 file:text-xs file:font-semibold file:bg-blue-50 file:text-blue-700 hover:file:bg-blue-100" />
                <span v-if="track.audio" class="text-xs text-green-600 font-medium">✓ Uploaded: {{ track.audio.name }}</span>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- STEP 3: Asset Upload -->
      <section v-if="currentStep === 3" class="space-y-6">
        <h2 class="text-lg font-bold border-b border-slate-100 pb-3">3. Asset Upload</h2>
        
        <div>
          <label class="block text-xs font-bold text-slate-700 uppercase mb-2">Cover Artwork (Min 3000x3000px JPG/PNG) *</label>
          <div class="border-2 border-dashed border-slate-200 rounded-xl p-8 text-center bg-slate-50 flex flex-col items-center justify-center">
            
            <div v-if="releaseData.assets.coverImage?.preview || releaseData.assets.coverImage?.url" class="mb-4">
              <img :src="releaseData.assets.coverImage.preview || releaseData.assets.coverImage.url" class="w-48 h-48 object-cover rounded-lg shadow-md border" />
            </div>

            <input type="file" accept="image/*" @change="handleCoverImageUpload" class="text-sm text-slate-500 file:mr-4 file:py-2 file:px-4 file:rounded-lg file:border-0 file:text-xs file:font-semibold file:bg-blue-600 file:text-white hover:file:bg-blue-700 cursor-pointer" />
            <p class="text-xs text-slate-400 mt-2">Square aspect ratio 1:1 required for DDEX delivery.</p>
          </div>
        </div>
      </section>

      <!-- STEP 4: Metadata -->
      <section v-if="currentStep === 4" class="space-y-6">
        <h2 class="text-lg font-bold border-b border-slate-100 pb-3">4. Metadata & Classification</h2>
        
        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <div>
            <label class="block text-xs font-bold text-slate-700 uppercase mb-2">Primary Genre *</label>
            <GenreSelector 
              v-model="releaseData.metadata.genreCode" 
              @update:genreName="val => releaseData.metadata.genreName = val"
            />
            <p class="text-xs text-slate-500 mt-1">Current: {{ displayGenreName }}</p>
          </div>

          <div>
            <label class="block text-xs font-bold text-slate-700 uppercase mb-2">Primary Language *</label>
            <select v-model="releaseData.metadata.language" class="w-full border border-slate-300 rounded-lg p-3 text-sm focus:ring-2 focus:ring-blue-500 outline-none bg-white">
              <option value="en">English</option>
              <option value="es">Spanish</option>
              <option value="fr">French</option>
              <option value="de">German</option>
              <option value="ja">Japanese</option>
            </select>
          </div>

          <div>
            <label class="block text-xs font-bold text-slate-700 uppercase mb-2">Copyright (C) Line *</label>
            <input v-model="releaseData.metadata.copyright" type="text" class="w-full border border-slate-300 rounded-lg p-3 text-sm focus:ring-2 focus:ring-blue-500 outline-none" placeholder="e.g. 2026 Record Label Name" />
          </div>

          <div>
            <label class="block text-xs font-bold text-slate-700 uppercase mb-2">Copyright Year *</label>
            <input v-model="releaseData.metadata.copyrightYear" type="number" class="w-full border border-slate-300 rounded-lg p-3 text-sm focus:ring-2 focus:ring-blue-500 outline-none" />
          </div>
        </div>
      </section>

      <!-- STEP 5: Territories & Rights -->
      <section v-if="currentStep === 5" class="space-y-6">
        <h2 class="text-lg font-bold border-b border-slate-100 pb-3">5. Territories & Rights</h2>
        
        <div class="space-y-4">
          <label class="block text-xs font-bold text-slate-700 uppercase">Distribution Mode</label>
          <div class="flex space-x-6">
            <label class="flex items-center space-x-2 cursor-pointer">
              <input type="radio" v-model="releaseData.territories.mode" value="worldwide" class="text-blue-600" />
              <span class="text-sm font-medium">Worldwide Rights</span>
            </label>
            <label class="flex items-center space-x-2 cursor-pointer">
              <input type="radio" v-model="releaseData.territories.mode" value="restricted" class="text-blue-600" />
              <span class="text-sm font-medium">Specific Territories Only</span>
            </label>
          </div>
        </div>
      </section>

      <!-- STEP 6: Review & Generate -->
      <section v-if="currentStep === 6" class="space-y-6">
        <h2 class="text-lg font-bold border-b border-slate-100 pb-3">6. Review & DDEX ERN Generation</h2>
        
        <div class="p-4 bg-slate-50 border border-slate-200 rounded-xl space-y-3">
          <p class="text-sm"><strong>Release Title:</strong> {{ releaseData.basic.title || 'Not set' }}</p>
          <p class="text-sm"><strong>Display Artist:</strong> {{ releaseData.basic.displayArtist || 'Not set' }}</p>
          <p class="text-sm"><strong>Tracks Configured:</strong> {{ releaseData.tracks.length }}</p>
          <p class="text-sm"><strong>Artwork Status:</strong> {{ releaseData.assets.coverImage ? 'Attached' : 'Missing' }}</p>
          <p class="text-sm"><strong>DDEX Profile:</strong> ERN {{ releaseData.preview.ernVersion }} ({{ releaseData.preview.profile }})</p>
        </div>

        <div class="flex space-x-4">
          <button @click="validateERN" :disabled="isSaving" class="px-6 py-3 bg-slate-800 hover:bg-slate-900 text-white font-medium rounded-lg text-sm transition">
            Validate ERN Structure
          </button>
          <button @click="generateERN" :disabled="!releaseData.preview.validated || isSaving" class="px-6 py-3 bg-blue-600 hover:bg-blue-700 disabled:opacity-50 text-white font-bold rounded-lg text-sm transition">
            Generate ERN Package
          </button>
        </div>
      </section>

    </main>

    <!-- Navigation Footer -->
    <footer class="mt-8 flex justify-between items-center">
      <button 
        @click="previousStep" 
        :disabled="currentStep === 1"
        class="px-6 py-3 bg-white border border-slate-300 font-medium rounded-xl text-slate-700 hover:bg-slate-50 disabled:opacity-40 transition"
      >
        Back
      </button>

      <button 
        @click="nextStep" 
        :disabled="currentStep === totalSteps"
        class="px-6 py-3 bg-blue-600 hover:bg-blue-700 text-white font-bold rounded-xl disabled:opacity-40 transition"
      >
        Next Step
      </button>
    </footer>
  </div>
</template>
