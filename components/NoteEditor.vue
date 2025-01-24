<template>
  <div class="editor-container">
    <div class="status-bar">
      Status: {{ connectionStatus }}
      <span v-if="error" class="error-message"> {{ error }}</span>
    </div>
    <textarea
    v-model="content"
    class="editor-textarea"
    placeholder="Start Typing..."
    @input="handleChange" />
  </div>
</template>

<script lang="ts" setup>
// components/QuickNoteEditor.vue
const props = defineProps({
  documentId: {
    type: String,
    required: true
  },
  accessKey: {
    type: String,
    default: ''
  }
})

const ws = ref(null)
const content = ref('')
const connectionStatus = ref('disconnected')
const error = ref(null)

//websocket management
const connectWebSocket = () => {
  const wsUrl = `ws://localhost:8801/ws?doc=${props.documentId}`
  const socket = new WebSocket(wsUrl)

  socket.onopen = () => {
    connectionStatus.value = 'connected'
    //initial sync request
    socket.send(JSON.stringify({
      type: 'sync',
      documentId: props.documentId
    }))
  }

  socket.onclose = () => {
    connectionStatus.value = 'disconnected'

    //TODO: RECONNECTION LOGIC
    setTimeout(connectWebSocket, 3000)
  }

  socket.onerror = () => {
    error.value = 'Connection error. Retrying...'
  }

  ws.value = socket
}

const handleMessage = (message) => {
  switch (message.type) {
    case 'sync': 
    content.value = message.content
    break
    case 'insert':
      const beforeInsert = content.value.slice(0, message.position)
      const afterInsert = content.value.slice(message.position)
      content.value = beforeInsert + message.content + afterInsert
    case 'delete':
      const beforeDelete = content.value.slice(0, message.position)
      const afterDelete = content.value.slice(message.position + message.length)
      content.value = beforeDelete + afterDelete
      break
  }
}

const findDiff = (oldStr, newStr) => {
  if (newStr.length > oldStr.length) {
    let i = 0 
    while (i < oldStr.length && oldStr[i] === newStr[i]) i++
    return {
      type: 'insert',
      position: i,
      content: newStr.slice(i, i + (newStr.length - oldStr.length))
    }
  } else {
    let i = 0
    while (i < newStr.length && oldStr[i] === newStr[i]) i++
    return {
      type: 'delete',
      position: i,
      length: oldStr.length - newStr.length
    }
  }
}

const handleChange = (event) => {
 const newContent = event.target.value
 const diff = findDiff(content.value, newContent)
 
 if (diff && ws.value?.readyState === WebSocket.OPEN) {
  ws.value.send(JSON.stringify({
    type: diff.type,
    position: diff.position,
    content: diff.length,
    documentId: props.documentId
  }))
 }
}

onMounted(() => {
  connectWebSocket()
})

onBeforeUnmount(() => {
  if (ws.value) {
    ws.value.close()
  }
})
//TODO: transfer styling to tailwind.
</script>

<style>

</style>