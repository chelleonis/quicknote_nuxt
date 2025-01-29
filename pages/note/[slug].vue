<template>
  <div class="container mx-auto py-8">
    <!-- Loading and Error States -->
    <div v-if="loading" class="text-center py-4">
      Loading document...
    </div>
    
    <div v-if="error" class="bg-red-100 border-l-4 border-red-500 text-red-700 p-4 mb-4">
      {{ error }}
    </div>

    <!-- Connection Status -->
    <div class="mb-4">
      <span :class="{
        'text-green-600': connectionStatus === 'connected',
        'text-yellow-600': connectionStatus === 'connecting',
        'text-red-600': connectionStatus === 'disconnected'
      }">
        Status: {{ connectionStatus }}
      </span>
    </div>

    <!-- Document Title -->
    <h1 class="text-2xl font-bold mb-4">{{ docTitle }}</h1>

    <!-- Editor Component -->
    <NoteEditor
      v-if="!loading"
      v-model:content="docContent"
      @change="handleDocumentChange"
    />
  </div>
</template>

<script lang="ts" setup>
//pages/note/[slug].vue
const route = useRoute()

// State Management
const loading = ref(true)
const error = ref<string | null>(null)
const docTitle = ref('')
const docContent = ref('')
const connectionStatus = ref('disconnected')
const ws = ref<WebSocket | null>(null)
const accessKey = ref('')

// Document Initialization
async function initializeDocument() {
  loading.value = true
  error.value = null
  
  // Try to get access key from localStorage
  const storedKey = localStorage.getItem(`doc_${route.params.slug}`)
  if (storedKey) {
    accessKey.value = storedKey
  }

  try {
    // Load initial document state
    const response = await $fetch(`http://localhost:8802/document/get?id=${route.params.slug}`, {
      headers: {
        'X-Access-Key': accessKey.value
      }
    })

    // Update document state
    docTitle.value = response.title
    docContent.value = response.content

    // Store access key if provided
    if (response.accessKey) {
      accessKey.value = response.accessKey
      localStorage.setItem(`doc_${route.params.slug}`, response.accessKey)
    }

    // Initialize WebSocket connection
    initializeWebSocket()
  } catch (e) {
    error.value = e instanceof Error ? e.message : 'Failed to load document'
  } finally {
    loading.value = false
  }
}

// WebSocket Management
function initializeWebSocket() {
  closeWebSocket() // Close any existing connection

  const wsUrl = `ws://localhost:8802/ws?doc=${route.params.slug}`
  const socket = new WebSocket(wsUrl)
  
  socket.onopen = () => {
    connectionStatus.value = 'connected'
    // Request initial sync
    socket.send(JSON.stringify({
      type: 'sync',
      documentId: route.params.slug
    }))
  }

  socket.onclose = () => {
    connectionStatus.value = 'disconnected'
    handleReconnect()
  }

  socket.onerror = () => {
    error.value = 'Connection error occurred'
    connectionStatus.value = 'disconnected'
  }

  socket.onmessage = handleWebSocketMessage

  ws.value = socket
}

// WebSocket Message Handling
function handleWebSocketMessage(event: MessageEvent) {
  try {
    const message = JSON.parse(event.data)
    
    switch (message.type) {
      case 'sync':
        docContent.value = message.content
        break
      
      case 'insert':
        const before = docContent.value.slice(0, message.position)
        const after = docContent.value.slice(message.position)
        docContent.value = before + message.content + after
        break
      
      case 'delete':
        const before = docContent.value.slice(0, message.position)
        const after = docContent.value.slice(message.position + message.length)
        docContent.value = before + after
        break
    }
  } catch (e) {
    console.error('Failed to process message:', e)
  }
}

// Document Change Handling
function handleDocumentChange(change: { type: string; position: number; content?: string; length?: number }) {
  if (!ws.value || ws.value.readyState !== WebSocket.OPEN) {
    error.value = 'Connection lost. Changes may not be saved.'
    return
  }

  // Just send to server and wait for broadcast back
  ws.value.send(JSON.stringify({
    type: change.type,
    position: change.position,
    content: change.content,
    length: change.length,
    documentId: route.params.slug
  }))
}

// Connection Management
function closeWebSocket() {
  if (ws.value) {
    ws.value.close()
    ws.value = null
  }
}

let reconnectAttempts = 0
const maxReconnectDelay = 30000 // 30 seconds

function handleReconnect() {
  if (reconnectAttempts > 5) {
    error.value = 'Failed to reconnect. Please refresh the page.'
    return
  }

  const delay = Math.min(1000 * Math.pow(2, reconnectAttempts), maxReconnectDelay)
  setTimeout(() => {
    if (connectionStatus.value !== 'connected') {
      initializeWebSocket()
      reconnectAttempts++
    }
  }, delay)
}

onMounted(() => {
  initializeDocument()
})

onBeforeUnmount(() => {
  closeWebSocket()
})

</script>

<style>

</style>