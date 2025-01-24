<template>
  <div>
    <div class="container mx-auto py-8">
    <NoteEditor
    :document-id="route.params.slug"
    :access-key="accessKey"
    />
  </div>
  </div>
</template>

<script lang="ts" setup>
//pages/note/[slug].vue
const route = useRoute()
const loading = ref(true)
const error = ref(null)
const accessKey = ref('')

onMounted(() => {
  // Try to get access key from localStorage
  const storedKey = localStorage.getItem(`doc_${route.params.slug}`)
  if (storedKey) {
    accessKey.value = storedKey
  }
  loadDocument()
})

async function loadDocument() {
  loading.value = true
  error.value = null

  try {
    const response = await fetch(`http://localhost:8801/document/get?id=${route.params.slug}`, {
      headers: {
        'X-Access-Key': accessKey.value
      }
    })

    if (!response.ok) {
      throw new Error(
        response.status === 403 ? 'Access denied' : 'Failed to load document'
      )
    }

    const data = await response.json()
    loading.value = false

    // If this is a successful access with a new key, store it
    if (accessKey.value) {
      localStorage.setItem(`doc_${route.params.slug}`, accessKey.value)
    }

  } catch (e) {
    error.value = e.message
    loading.value = false
  }
}
</script>

<style>

</style>