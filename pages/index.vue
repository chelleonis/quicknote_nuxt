<template>
<h1>
  hello world in dev
</h1>
<div>
  <h1> create new note </h1>
  <div>
    <label> Document Type </label>
    <div> 
      <button 
              @click="accessLevel = 'public'"
            >
              Public
            </button>
            <button 
              @click="accessLevel = 'private'"
            >
              Private
            </button>
    </div>
    <p>
      {{ accessLevel === 'public' 
              ? 'Anyone with the link can access this note.'
              : 'Only users with the access key can view and edit this note.' 
            }}
    </p>
    <input
            v-model="docTitle"
            id="docTitle"
            type="text"
            required
          />
  </div>
  <div>
    <button @click="createDocument"
    :disabled="loading">
    {{ loading ? 'Creating...' : 'Create Note' }}
  </button>
  </div>
  <!-- document info if submission successful -->
   <div v-if="documentInfo">
    <h3> Document created succesffully! </h3>

    <p> <span> Document ID:</span>
    {{ documentInfo.documentId }}
    </p>

    <div v-if="documentInfo.accessKey">
      <!-- eventually store in the store or account store or something -->
      <p>
        Save this access key! you'll need it to access the document!
      </p>
      <code>
        {{  documentInfo.accessKey }}
      </code>
      <button 
                  @click="copyAccessKey"
                >
                  Copy
                </button>
    </div>
    <NuxtLink 
                :to="`/note/${documentInfo.documentId}`"
              >
                Open Document →
              </NuxtLink>
   </div>
</div>
</template>

<script lang="ts" setup>
//pages/index.vue
const accessLevel = ref('public')
const loading = ref(false)
const error = ref(null)
const documentInfo = ref(null)
const docTitle = ref('')

async function createDocument() {
  loading.value = true
  error.value = null
  documentInfo.value = null

  try {
    const response = await $fetch('http://localhost:8801/document/new', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: {
        accessLevel: accessLevel.value,
        title: docTitle.value.trim()
      }
    })

    const data = await response.json()
    documentInfo.value = data

    if (data.accessKey) {
      localStorage.setItem(`doc_${data.documentId}`, data.accessKey)
    }
  } catch(e) {
    error.value = e.message
  } finally {
    loading.value = false
  }

function copyAccesskey() {
  if (documentInfo.value?.accessKey) {
    navigator.clipboard.writeText(documentInfo.value.accessKey)
  }
}

}

</script>

<style>

</style>