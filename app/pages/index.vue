<script setup lang="ts">
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
}

// Fetch the data
const { data: rawLogs, status, error } = await useFetch<LogItem[]>('/MyActivity.json', {
  server: false,
  lazy: true
})

// Processed logs with cleaner titles and dates
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

// Search functionality
const search = ref('')
const filteredLogs = computed(() => {
  if (!search.value) return processedLogs.value
  const q = search.value.toLowerCase()
  return processedLogs.value.filter(log => 
    log.cleanTitle.toLowerCase().includes(q) || 
    log.responseHtml.toLowerCase().includes(q)
  )
})

// Pagination
const page = ref(1)
const itemsPerPage = 10

const pageCount = computed(() => Math.ceil(filteredLogs.value.length / itemsPerPage))
const paginatedLogs = computed(() => {
  const start = (page.value - 1) * itemsPerPage
  return filteredLogs.value.slice(start, start + itemsPerPage)
})

// Watch search to reset page
watch(search, () => {
  page.value = 1
})
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-gray-50 to-gray-100 dark:from-gray-950 dark:to-gray-900 py-12 px-4 sm:px-6 lg:px-8">
    <UContainer>
      
      <!-- Header Section -->
      <div class="mb-12 text-center relative z-10">
        <h1 class="text-4xl md:text-6xl font-extrabold text-transparent bg-clip-text bg-gradient-to-r from-primary-500 to-indigo-600 dark:from-primary-400 dark:to-indigo-400 mb-4 animate-fade-in-up">
          Gemini Journey
        </h1>
        <p class="text-xl text-gray-600 dark:text-gray-300 max-w-2xl mx-auto mb-8 animate-fade-in-up delay-100">
          Explore your conversations with AI. A digital archive of your thoughts and discoveries.
        </p>

        <!-- Search Bar -->
        <div class="max-w-xl mx-auto animate-fade-in-up delay-200">
          <UInput
            v-model="search"
            icon="i-heroicons-magnifying-glass-20-solid"
            size="xl"
            color="neutral"
            placeholder="Search prompts and responses..."
            class="shadow-lg"
          >
            <template #trailing>
              <UBadge v-if="search" color="neutral" variant="soft" class="mr-1">
                {{ filteredLogs.length }} matches
              </UBadge>
            </template>
          </UInput>
        </div>
      </div>

      <!-- Content State -->
      <div v-if="status === 'pending'" class="flex justify-center py-20">
        <UIcon name="i-heroicons-arrow-path" class="w-10 h-10 animate-spin text-primary-500" />
      </div>

      <div v-else-if="status === 'error'" class="text-center py-20 text-red-500">
        <p>Failed to load activity logs.</p>
        <p class="text-sm mt-2">{{ error }}</p>
      </div>

      <div v-else class="space-y-8 relative z-10">
        
        <!-- Timeline Items -->
        <div v-for="(log, index) in paginatedLogs" :key="index" 
             class="group relative pl-8 border-l-2 border-gray-200 dark:border-gray-800 hover:border-primary-500 dark:hover:border-primary-500 transition-colors duration-300">
          
          <!-- Timeline Dot -->
          <div class="absolute -left-[9px] top-6 w-4 h-4 rounded-full bg-gray-200 dark:bg-gray-800 border-2 border-white dark:border-gray-950 group-hover:bg-primary-500 transition-colors duration-300"></div>

          <UCard class="ring-1 ring-gray-200 dark:ring-gray-800 shadow-sm hover:shadow-xl hover:ring-primary-500/50 dark:hover:ring-primary-400/50 transition-all duration-300 bg-white/50 dark:bg-gray-900/50 backdrop-blur-sm">
            
            <template #header>
              <div class="flex flex-col sm:flex-row sm:items-center justify-between gap-2">
                <div class="flex items-center gap-2">
                  <UBadge color="primary" variant="subtle" size="xs">{{ log.header }}</UBadge>
                  <span class="text-xs text-gray-500 font-mono">{{ log.formattedTime }}</span>
                </div>
                <div class="flex gap-2">
                   <UBadge v-for="prod in log.products" :key="prod" color="neutral" variant="outline" size="xs">{{ prod }}</UBadge>
                </div>
              </div>
            </template>

            <!-- User Prompt -->
            <div class="mb-6">
              <h3 class="text-lg font-semibold text-gray-900 dark:text-gray-100 mb-2">
                <UIcon name="i-heroicons-user" class="mr-2 text-primary-500 inline-block align-text-bottom" />
                {{ log.cleanTitle }}
              </h3>
            </div>

            <!-- AI Response -->
            <div v-if="log.responseHtml" class="prose dark:prose-invert max-w-none text-gray-700 dark:text-gray-300 bg-gray-50 dark:bg-gray-950/50 p-4 rounded-lg border border-gray-100 dark:border-gray-800">
              <div class="flex items-start gap-3">
                 <UIcon name="i-heroicons-sparkles" class="w-5 h-5 text-indigo-500 mt-1 shrink-0" />
                 <div v-html="log.responseHtml" class="w-full overflow-hidden break-words"></div>
              </div>
            </div>

          </UCard>
        </div>

        <!-- Empty State -->
        <div v-if="paginatedLogs.length === 0" class="text-center py-20 text-gray-500">
            <UIcon name="i-heroicons-document-magnifying-glass" class="w-12 h-12 mx-auto mb-4 opacity-50" />
            <p>No conversations found matching your search.</p>
        </div>

        <!-- Pagination -->
        <div class="flex justify-center mt-12 mb-8">
          <UPagination
            v-model:page="page"
            :items-per-page="itemsPerPage"
            :total="filteredLogs.length"
            :sibling-count="1"
          />
        </div>

      </div>
    </UContainer>
  </div>
</template>

<style scoped>
.animate-fade-in-up {
  animation: fadeInUp 0.8s ease-out forwards;
  opacity: 0;
  transform: translateY(20px);
}

.delay-100 { animation-delay: 0.1s; }
.delay-200 { animation-delay: 0.2s; }

@keyframes fadeInUp {
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>
