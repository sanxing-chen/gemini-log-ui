<template>
  <div class="h-screen flex flex-col bg-gray-50 dark:bg-gray-950">
    <UContainer class="flex-1 w-full p-4 flex gap-4 overflow-hidden">
      
      <!-- Sidebar / TOC -->
      <div class="w-1/3 min-w-[300px] flex flex-col bg-white dark:bg-gray-900 rounded-lg shadow-sm border border-gray-200 dark:border-gray-800 overflow-hidden">
        <div class="p-4 border-b border-gray-200 dark:border-gray-800">
           <UInput
              v-model="search"
              icon="i-heroicons-magnifying-glass-20-solid"
              size="sm"
              color="neutral"
              variant="outline"
              placeholder="Filter..."
             class="w-full"
          />
          <div class="mt-2">
            <UTabs :items="tabItems" v-model="viewMode" class="w-full" />
          </div>
       </div>
        
        <div class="flex-1 overflow-y-auto p-2 scrollbar-thin scrollbar-thumb-gray-200 dark:scrollbar-thumb-gray-700">
            <div v-if="status === 'pending'" class="flex justify-center py-4">
                <UIcon name="i-heroicons-arrow-path" class="w-6 h-6 animate-spin text-primary-500" />
            </div>
             <div v-else-if="status === 'error' && !uploadedLogs" class="flex flex-col items-center justify-center p-6 space-y-4">
                <div class="text-center">
                    <UIcon name="i-heroicons-exclamation-circle" class="w-8 h-8 text-red-500 mb-2 mx-auto" />
                    <p class="text-sm text-gray-600 dark:text-gray-300 font-medium">Data source missing</p>
                    <p class="text-xs text-gray-400 dark:text-gray-500 mb-4">Upload MyActivity.json to view your history</p>
                </div>
                <div class="w-full">
                     <UFileUpload 
                        v-model="uploadFile" 
                        @change="handleFileUpload" 
                        accept=".json"
                        icon="i-heroicons-arrow-up-tray"
                     />
                </div>
            </div>
            
            <nav v-else class="space-y-1">
                <!-- Calendar -->
                <div class="px-2 py-2 mb-2 border-b border-gray-100 dark:border-gray-800 flex justify-center">
                    <UCalendar 
                        v-model="calendarDate" 
                        :is-date-unavailable="isDateDisabled" 
                        variant="subtle"
                        class="p-2"
                    />
                </div>

                <!-- Data Empty State -->
                <div v-if="!tocStructure.length" class="text-center text-gray-400 text-sm py-8">
                    No results found
                </div>

                <!-- Custom TOC -->
                <div v-for="year in tocStructure" :key="year.label" class="space-y-1">
                    <!-- Year Header -->
                     <button 
                        @click="toggleYear(year)"
                        class="w-full flex items-center gap-2 px-2 py-1.5 text-sm font-semibold rounded-md transition-colors hover:bg-gray-100 dark:hover:bg-gray-800"
                        :class="year.expanded ? 'text-gray-900 dark:text-gray-100' : 'text-gray-500 dark:text-gray-400'"
                     >
                        <UIcon 
                            :name="year.expanded ? 'i-heroicons-chevron-down' : 'i-heroicons-chevron-right'" 
                            class="w-4 h-4 transition-transform duration-200" 
                        />
                        <span>{{ year.label }}</span>
                     </button>

                     <!-- Months -->
                     <div v-show="year.expanded" class="pl-4 space-y-1 border-l border-gray-100 dark:border-gray-800 ml-3">
                        <div v-for="month in year.children" :key="month.label" class="space-y-1">
                            <button 
                                @click="toggleMonth(month)"
                                class="w-full flex items-center gap-2 px-2 py-1 text-sm font-medium rounded-md transition-colors hover:bg-gray-100 dark:hover:bg-gray-800"
                                :class="month.expanded ? 'text-gray-900 dark:text-gray-100' : 'text-gray-500 dark:text-gray-500'"
                            >
                                <span class="capitalize">{{ month.label }}</span>
                            </button>

                            <!-- Days -->
                            <div v-show="month.expanded" class="pl-4 space-y-1 border-l border-gray-100 dark:border-gray-800 ml-1">
                                <div v-for="day in month.children" :key="day.label" class="space-y-1">
                                     <!-- Day Header (Clickable to select date) -->
                                    <button
                                        @click="selectDate(day.dateKey)"
                                        class="w-full text-left px-2 py-0.5 text-xs font-bold uppercase tracking-wider mt-2 mb-1 hover:text-primary-500 transition-colors"
                                        :class="selectedDateKey === day.dateKey ? 'text-primary-500' : 'text-gray-400'"
                                    >
                                        {{ day.label }}
                                    </button>
                                    
                                    <!-- Log Items -->
                                    <div v-show="day.dateKey === selectedDateKey || search.length > 0" class="space-y-0.5">
                                        <button 
                                            v-for="log in day.children" 
                                            :key="log.id"
                                            @click="selectLog(log)"
                                            class="w-full text-left px-2 py-1.5 text-sm rounded-md transition-all duration-200 border-l-2 flex items-center justify-between gap-2 group"
                                            :class="activeLogId === log.id 
                                                ? 'bg-primary-50 dark:bg-primary-900/10 text-primary-600 dark:text-primary-400 border-primary-500 font-medium' 
                                                : 'text-gray-600 dark:text-gray-400 border-transparent hover:bg-gray-50 dark:hover:bg-gray-800 hover:text-gray-900 dark:hover:text-gray-200'"
                                        >
                                            <span class="truncate flex-1 min-w-0">{{ log.label }}</span>
                                            <span class="text-xs font-mono opacity-50 group-hover:opacity-100 transition-opacity whitespace-nowrap">{{ log.time }}</span>
                                        </button>
                                    </div>
                                </div>
                            </div>
                        </div>
                     </div>
                </div>
            </nav>
        </div>
      </div>

      <!-- Main Content -->
      <div class="flex-1 flex flex-col bg-white dark:bg-gray-900 rounded-lg shadow-sm border border-gray-200 dark:border-gray-800 overflow-hidden relative">
        <div class="flex-1 overflow-y-auto p-0 scroll-smooth" ref="mainScroll">
             <div v-if="activeDayLogs.length === 0" class="h-full flex flex-col items-center justify-center text-gray-400">
                 <UIcon name="i-heroicons-inbox" class="w-16 h-16 mb-4 opacity-50" />
                 <p>No messages to display</p>
             </div>
             <div v-else class="pb-20">
                
                <!-- Date Header -->
                <div class="sticky top-0 z-10 bg-white/80 dark:bg-gray-900/80 backdrop-blur-md border-b border-gray-100 dark:border-gray-800 p-4 flex items-center justify-between">
                    <h1 class="text-xl font-bold text-gray-900 dark:text-white">
                        {{ activeDateLabel }}
                    </h1>
                    <UColorModeSwitch />
                </div>

                 <div v-for="log in activeDayLogs" :key="log.id" :id="log.id" class="px-4 py-6 flex flex-col gap-6" data-log-item>
                    <!-- User Prompt -->
                    <div class="flex flex-col items-end pl-12 group">
                        <!-- Time and Metadata -->
                        <div class="flex items-center gap-2 mb-1 opacity-60 mr-1">
                             <div class="flex gap-1" v-if="log.products && log.products.length">
                                <UBadge v-for="prod in log.products" :key="prod" color="neutral" variant="outline" size="xs">{{ prod }}</UBadge>
                             </div>
                             <span class="text-xs font-mono" v-if="log.shortTime">{{ log.shortTime }}</span>
                        </div>

                        <div class="flex flex-col items-end gap-1 max-w-[85%]">
                             <div class="bg-gray-100 dark:bg-gray-800 text-gray-900 dark:text-white rounded-2xl rounded-tr-md px-5 py-3 shadow-sm text-base leading-relaxed whitespace-pre-wrap">
                                {{ log.cleanTitle }}
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
                             
                             <div v-if="log.responseHtml" class="w-full bg-white dark:bg-gray-900 border border-gray-200 dark:border-gray-800 rounded-2xl rounded-tl-md p-6 shadow-sm overflow-hidden group/response transition-all hover:shadow-md">
                                 <div class="rich-content text-sm sm:text-base">
                                     <div v-html="log.responseHtml" class="w-full overflow-hidden break-words" />
                                 </div>
                             </div>
                             <div v-else class="text-gray-500 italic px-4">
                                No content to display for this item.
                             </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
      </div>

    </UContainer>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, nextTick, onMounted } from 'vue'
import { CalendarDate } from '@internationalized/date'
import type { DateValue } from '@internationalized/date'

interface HtmlItem {
  html: string
}

interface LogItem {
  id: string
  header: string
  title: string
  time: string
  products: string[]
  activityControls: string[]
  safeHtmlItem?: HtmlItem[]
  cleanTitle?: string
  formattedTime?: string
  responseHtml?: string
  dateKey?: string
  timestamp?: number
  isWork?: boolean
}

// Fetch the data
const { data: rawLogs, status, error } = await useFetch<LogItem[]>('/MyActivity.json', {
  server: false,
  lazy: true
})

// State
const search = ref('')
const activeLogId = ref<string | null>(null)
const selectedDateKey = ref<string | null>(null)
const mainScroll = ref<HTMLElement | null>(null)
const uploadFile = ref<File | null>(null)
const uploadedLogs = ref<LogItem[] | null>(null)
const viewMode = ref<'all' | 'work' | 'life'>('all')

const tabItems = [
    { label: 'All', value: 'all' },
    { label: 'Work', value: 'work' },
    { label: 'Life', value: 'life' }
]

// Removed selectedTabIndex computed property as we can now v-model directly to viewMode
let observer: IntersectionObserver | null = null

// Expansion State (managed manually to allow independent toggling)
const expandedYears = ref<Set<string>>(new Set())
const expandedMonths = ref<Set<string>>(new Set())

const handleFileUpload = (e: any) => {
    // Determine the file to read
    let file: File | null = null
    if (uploadFile.value) {
        file = uploadFile.value
    } else if (e?.target?.files?.[0]) {
        file = e.target.files[0]
    }
    
    if (!file) return

    const reader = new FileReader()
    reader.onload = (event) => {
        try {
            const content = event.target?.result as string
            const parsed = JSON.parse(content)
            if (Array.isArray(parsed)) {
                uploadedLogs.value = parsed
            } else {
                alert('Invalid JSON format: Expected an array')
            }
        } catch (err) {
            console.error('Error parsing JSON:', err)
            alert('Error parsing JSON file')
        }
    }
    reader.readAsText(file)
}

// Processed logs
const processedLogs = computed(() => {
  const source = uploadedLogs.value || rawLogs.value
  if (!source) return []
  return source
    .map((item, index) => {
        const dateObj = item.time ? new Date(item.time) : new Date(0);
        const yyyy = dateObj.getFullYear();
        const mm = String(dateObj.getMonth() + 1).padStart(2, '0');
        const dd = String(dateObj.getDate()).padStart(2, '0');
        
        const responseHtml = item.safeHtmlItem?.[0]?.html || '';

        return {
            ...item,
            id: `log-${index}`,
            cleanTitle: item.title ? item.title.replace(/^Prompted\s*/, '') : 'No Title',
            formattedTime: item.time ? dateObj.toLocaleDateString(undefined, {
                year: 'numeric',
                month: 'short',
                day: 'numeric',
                hour: '2-digit',
                minute: '2-digit'
            }) : 'Unknown Date',
            shortTime: item.time ? dateObj.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', hour12: false }) : '',
            responseHtml: responseHtml,
            dateKey: `${yyyy}-${mm}-${dd}`,
            timestamp: dateObj.getTime(),
            isWork: responseHtml.includes('<code') || 
                   responseHtml.includes('<pre') || 
                   responseHtml.includes('class="code') || 
                   ((responseHtml.match(/\$/g) || []).length + (item.title?.match(/\$/g) || []).length) > 2
        }
    })
    .sort((a, b) => (b.timestamp || 0) - (a.timestamp || 0))
})

const visibleLogs = computed(() => {
    return processedLogs.value.filter(log => {
        if (viewMode.value === 'work') return log.isWork
        if (viewMode.value === 'life') return !log.isWork
        return true
    })
})

// Watch for data load to initialize
watch(visibleLogs, (logs) => {
    if (logs.length > 0 && !selectedDateKey.value) {
        selectedDateKey.value = logs[0]!.dateKey || null
        if (selectedDateKey.value) {
            // Auto expand the first date
            const date = new Date(logs[0]!.time)
            expandedYears.value.add(date.getFullYear().toString())
            expandedMonths.value.add(`${date.getFullYear()}-${date.toLocaleString('default', { month: 'long' })}`)
        }
    }
}, { immediate: true })

const selectDate = (dateKey?: string) => {
    if (!dateKey) return
    selectedDateKey.value = dateKey
    // Scroll to top of main view
    if (mainScroll.value) mainScroll.value.scrollTop = 0
}

const selectLog = (log: LogItem) => {
    if (log.dateKey && log.dateKey !== selectedDateKey.value) {
        selectedDateKey.value = log.dateKey
    }
    
    // Wait for render then scroll
    nextTick(() => {
        scrollToLogId(log.id)
    })
}

const scrollToLogId = (id: string) => {
  const el = document.getElementById(id)
  if (el) {
    el.scrollIntoView({ behavior: 'smooth', block: 'start' })
  }
}

const toggleYear = (yearItem: any) => {
    if (expandedYears.value.has(yearItem.label)) {
        expandedYears.value.delete(yearItem.label)
    } else {
        expandedYears.value.add(yearItem.label)
    }
}

const toggleMonth = (monthItem: any) => {
    if (expandedMonths.value.has(monthItem.uniqueId)) { // Use uniqueId for month to avoid name collisions across years
        expandedMonths.value.delete(monthItem.uniqueId)
    } else {
        // Enforce single active month
        expandedMonths.value.clear()
        expandedMonths.value.add(monthItem.uniqueId)
    }
}

// Calendar Logic
const calendarDate = computed({
  get: () => {
    if (!selectedDateKey.value) {
        return new CalendarDate(new Date().getFullYear(), new Date().getMonth() + 1, new Date().getDate())
    }
    const parts = selectedDateKey.value.split('-').map(Number)
    return new CalendarDate(parts[0] ?? 2024, parts[1] ?? 1, parts[2] ?? 1)
  },
  set: (val: DateValue) => {
    if (!val) return
    const y = val.year
    const m = String(val.month).padStart(2, '0')
    const d = String(val.day).padStart(2, '0')
    const dateKey = `${y}-${m}-${d}`
    
    // Select date if distinct
    if (selectedDateKey.value !== dateKey) {
        selectDate(dateKey)
    }
  }
})

const availableDates = computed(() => {
    const set = new Set<string>()
    if (visibleLogs.value) {
        visibleLogs.value.forEach(l => {
            if (l.dateKey) set.add(l.dateKey)
        })
    }
    return set
})

const isDateDisabled = (date: DateValue) => {
    const y = date.year
    const m = String(date.month).padStart(2, '0')
    const d = String(date.day).padStart(2, '0')
    const k = `${y}-${m}-${d}`
    return !availableDates.value.has(k)
}

watch(selectedDateKey, (newKey) => {
    if (!newKey) return
    const parts = newKey.split('-').map(Number)
    const y = parts[0]
    const m = parts[1]
    const d = parts[2]
    
    if (y === undefined || m === undefined || d === undefined) return
    
    // Construct date to get month name (be careful with timezone, use local parts)
    const date = new Date(y, m - 1, d)
    const year = y.toString()
    const month = date.toLocaleString('default', { month: 'long' })
    
    expandedYears.value.add(year)
    // Clear other months to ensure only one is active
    expandedMonths.value.clear()
    expandedMonths.value.add(`${year}-${month}`)
}, { immediate: true })

const activeDayLogs = computed(() => {
    if (!selectedDateKey.value) return []
    return visibleLogs.value.filter(l => l.dateKey === selectedDateKey.value).reverse()
})

const activeDateLabel = computed(() => {
    if (!selectedDateKey.value) return ''
    const log = activeDayLogs.value[0]
    if (log && log.time) {
        return new Date(log.time).toLocaleDateString(undefined, { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' })
    }
    return selectedDateKey.value
})




// Intersection Observer
onMounted(() => {
  watch(() => activeDayLogs.value, () => {
       if (activeDayLogs.value.length > 0) {
           nextTick(initObserver)
       }
  })
})

const initObserver = () => {
    if (observer) observer.disconnect()
    
    observer = new IntersectionObserver((entries) => {
        // Find visible entries
        const visible = entries.filter(e => e.isIntersecting)
        if (visible.length > 0) {
             activeLogId.value = visible[0]!.target.id
        }
    }, {
        root: mainScroll.value,
        threshold: 0.1, 
        rootMargin: '-10% 0px -70% 0px' 
    })
    
    const items = document.querySelectorAll('[data-log-item]')
    items.forEach(el => observer?.observe(el))
}

// TOC Construction
const tocStructure = computed(() => {
  if (!visibleLogs.value.length) return []

  const query = search.value.toLowerCase()
  const groups: Record<string, Record<string, Record<string, any[]>>> = {}
  
  // Group logs
  visibleLogs.value.forEach(log => {
      // Filter if search is active
      if (query && !log.cleanTitle?.toLowerCase()?.includes(query)) {
          return
      }

      if (!log.time) return
      const date = new Date(log.time)
      const year = date.getFullYear().toString()
      const month = date.toLocaleString('default', { month: 'long' })
      const day = date.getDate().toString().padStart(2, '0')
      const time = date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', hour12: false })
      
      if (!groups[year]) groups[year] = {}
      if (!groups[year][month]) groups[year][month] = {}
      if (!groups[year][month][day]) groups[year][month][day] = []
      
      groups[year][month][day].push({
          id: log.id,
          label: log.cleanTitle || 'No Title',
          dateKey: log.dateKey,
          time
      })
  })

  // Build Hierarchy
  return Object.keys(groups).sort((a, b) => b.localeCompare(a)).map(year => {
      const yearHasMatch = query.length > 0 // If filtered, expand
      const isExpandedYear = expandedYears.value.has(year) || yearHasMatch

      const childrenMonths = Object.keys(groups[year] || {}).map(month => {
          const uniqueMonthId = `${year}-${month}`
          const isExpandedMonth = expandedMonths.value.has(uniqueMonthId) || yearHasMatch

          const childrenDays = Object.keys(groups[year]?.[month] || {})
            .sort((a,b) => parseInt(b) - parseInt(a))
            .map(day => {
                const dayLogs = groups[year]![month][day]!
                return {
                    label: `Day ${day}`,
                    dateKey: dayLogs[0]?.dateKey,
                    children: dayLogs.reverse()
                }
            })

          return {
              label: month,
              uniqueId: uniqueMonthId,
              children: childrenDays,
              expanded: isExpandedMonth
          }
      })

      return {
          label: year,
          children: childrenMonths,
          expanded: isExpandedYear
      }
  })
})
</script>
