<template>
  <div class="h-screen flex flex-col bg-gray-50 dark:bg-gray-950">
    <UContainer class="flex-1 w-full max-w-none p-4 flex gap-4 overflow-hidden">
      
      <!-- Sidebar / Tree -->
      <div class="w-1/3 min-w-[300px] flex flex-col bg-white dark:bg-gray-900 rounded-lg shadow-sm border border-gray-200 dark:border-gray-800 overflow-hidden">
        <div class="p-4 border-b border-gray-200 dark:border-gray-800">
           <h2 class="text-lg font-semibold text-gray-900 dark:text-white mb-2">History</h2>
           <UInput
              v-model="search"
              icon="i-heroicons-magnifying-glass-20-solid"
              size="sm"
              color="neutral"
              variant="outline"
              placeholder="Filter..."
              class="w-full"
           />
        </div>
        
        <div class="flex-1 overflow-y-auto p-2">
            <div v-if="status === 'pending'" class="flex justify-center py-4">
                <UIcon name="i-heroicons-arrow-path" class="w-6 h-6 animate-spin text-primary-500" />
            </div>
             <div v-else-if="status === 'error'" class="text-center py-4 text-red-500">
                <p>Failed to load.</p>
            </div>
            <UTree 
              v-else
              :items="filteredTreeItems" 
              :ui="{
                 label: 'truncate',
                 icon: { base: 'shrink-0' }
              }"   
              class="w-full"
            />
        </div>
      </div>

      <!-- Main Content -->
      <div class="flex-1 flex flex-col bg-white dark:bg-gray-900 rounded-lg shadow-sm border border-gray-200 dark:border-gray-800 overflow-hidden relative">
        <div v-if="selectedLog" class="flex-1 overflow-y-auto p-8 flex flex-col gap-8">
            <!-- Header Metadata -->
            <div class="flex flex-wrap items-center justify-between gap-4 border-b border-gray-100 dark:border-gray-800 pb-4">
               <div class="flex items-center gap-3">
                  <UBadge color="primary" variant="subtle" size="sm">{{ selectedLog.header }}</UBadge>
                  <span class="text-sm text-gray-500 font-mono">{{ selectedLog.formattedTime }}</span>
               </div>
               <div class="flex gap-2">
                   <UBadge v-for="prod in selectedLog.products" :key="prod" color="neutral" variant="outline" size="xs">{{ prod }}</UBadge>
               </div>
            </div>

            <!-- User Prompt -->
            <div class="flex justify-end pl-12 group">
                <div class="flex flex-col items-end gap-1 max-w-[85%]">
                     <div class="flex items-center gap-2 opacity-60 mr-1">
                         <span class="text-xs font-medium">You</span>
                         <UIcon name="i-heroicons-user" class="w-3 h-3" />
                     </div>
                     <div class="bg-gray-100 dark:bg-gray-800 text-gray-900 dark:text-white rounded-2xl rounded-tr-md px-5 py-3 shadow-sm text-base leading-relaxed whitespace-pre-wrap">
                        {{ selectedLog.cleanTitle }}
                     </div>
                </div>
            </div>

            <!-- Assistant Response -->
            <div class="flex justify-start pr-12">
                <div class="flex flex-col items-start gap-1 w-full max-w-[90%]">
                     <div class="flex items-center gap-2 opacity-60 ml-1">
                         <UIcon name="i-heroicons-sparkles" class="w-3 h-3 text-primary-500" />
                         <span class="text-xs font-medium">Gemini</span>
                     </div>
                     
                     <div v-if="selectedLog.responseHtml" class="w-full bg-white dark:bg-gray-900 border border-gray-200 dark:border-gray-800 rounded-2xl rounded-tl-md p-6 shadow-sm overflow-hidden">
                         <div class="prose dark:prose-invert max-w-none prose-neutral prose-sm sm:prose-base">
                             <div v-html="selectedLog.responseHtml" class="w-full overflow-hidden break-words" />
                         </div>
                     </div>
                     <div v-else class="text-gray-500 italic px-4">
                        No content to display for this item.
                     </div>
                </div>
            </div>
        </div>

        <div v-else class="flex flex-col items-center justify-center h-full text-gray-400">
           <UIcon name="i-heroicons-cursor-arrow-rays" class="w-16 h-16 mb-4 opacity-50" />
           <p class="text-lg">Select a conversation to view details</p>
        </div>
      </div>

    </UContainer>
  </div>
</template>

<script setup lang="ts">
import type { TreeItem } from '@nuxt/ui'

interface HtmlItem {
  html: string
}

interface LogItem {
  header: string
  title: string
  time: string
  products: string[]
  activityControls: string[]
  safeHtmlItem?: HtmlItem[]
  cleanTitle?: string
  formattedTime?: string
  responseHtml?: string
}

// Fetch the data
const { data: rawLogs, status, error } = await useFetch<LogItem[]>('/MyActivity.json', {
  server: false,
  lazy: true
})

// Processed logs
const processedLogs = computed(() => {
  if (!rawLogs.value) return []
  return rawLogs.value.map(item => ({
    ...item,
    cleanTitle: item.title ? item.title.replace(/^Prompted\s*/, '') : 'No Title',
    formattedTime: item.time ? new Date(item.time).toLocaleDateString(undefined, {
      year: 'numeric',
      month: 'short',
      day: 'numeric',
      hour: '2-digit',
      minute: '2-digit'
    }) : 'Unknown Date',
    responseHtml: item.safeHtmlItem?.[0]?.html || ''
  }))
})

// Tree Logic
const search = ref('')
const selectedLog = ref<LogItem | null>(null)

// Helper to filter tree
const filterTree = (items: any[], query: string): any[] => {
  if (!query) return items
  return items.map(item => {
    if (item.children) {
      const filteredChildren = filterTree(item.children, query)
      if (filteredChildren.length > 0) {
        return { ...item, children: filteredChildren, defaultExpanded: true }
      }
    }
    // If leaf (Conceptually, or generic match on label)
    const matches = item.label?.toLowerCase().includes(query.toLowerCase())
    if (matches) return item
    return null
  }).filter(Boolean) as any[]
}

const treeItems = computed(() => {
  if (!processedLogs.value.length) return []

  // Grouping
  const groups: Record<string, Record<string, Record<string, LogItem[]>>> = {}
  
  processedLogs.value.forEach(log => {
      if (!log.time) return
      const date = new Date(log.time)
      const year = date.getFullYear().toString()
      const month = date.toLocaleString('default', { month: 'long' })
      const day = date.getDate().toString().padStart(2, '0')
      
      if (!groups[year]) groups[year] = {}
      if (!groups[year][month]) groups[year][month] = {}
      if (!groups[year][month][day]) groups[year][month][day] = []
      
      groups[year][month][day].push(log)
  })

  // Build Tree Structure
  const rawTree = Object.keys(groups).sort((a, b) => b.localeCompare(a)).map(year => ({
      label: year,
      icon: 'i-heroicons-calendar',
      children: Object.keys(groups[year]).map(month => ({
          label: month,
          icon: 'i-heroicons-calendar-days',
          children: Object.keys(groups[year][month]).sort((a,b) => parseInt(b) - parseInt(a)).map(day => ({
              label: `Day ${day}`,
              icon: 'i-heroicons-calendar-days',
              children: groups[year][month][day].map(log => ({
                  label: log.cleanTitle || 'No Title',
                  icon: 'i-heroicons-chat-bubble-left',
                  onSelect: () => { selectedLog.value = log }
              }))
          }))
      }))
  }))

  return rawTree
})

const filteredTreeItems = computed(() => {
    return filterTree(treeItems.value, search.value)
})
</script>
