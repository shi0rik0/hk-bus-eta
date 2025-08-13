<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'

// Define bus company category enum
enum BusCompany {
  KMB = 'KMB',
  CTB = 'CTB',
}

// Define TypeScript interface for list items
interface ListItem {
  dest: string
  stopName: string
  category: string
  company: BusCompany
  stopId: string
  route: string
  serviceType: string // Only valid for KMB
  eta: number[]
}

const items = ref<ListItem[]>([
  {
    dest: '宝琳',
    stopName: '將軍澳隧道巴士轉乘站',
    category: '將軍澳隧道巴士轉乘站->坑口',
    stopId: '97D1981DE7ECBCE7',
    route: '297',
    serviceType: '1',
    company: BusCompany.KMB,
    eta: [],
  },
  {
    dest: '坑口',
    stopName: '將軍澳隧道巴士轉乘站',
    category: '將軍澳隧道巴士轉乘站->坑口',
    stopId: '003595',
    route: '798',
    serviceType: '',
    company: BusCompany.CTB,
    eta: [],
  },
  {
    dest: '坑口',
    stopName: '將軍澳隧道巴士轉乘站',
    category: '將軍澳隧道巴士轉乘站->坑口',
    stopId: '003595',
    route: 'A29',
    serviceType: '',
    company: BusCompany.CTB,
    eta: [],
  },
  {
    dest: '機場',
    stopName: '帝都酒店',
    category: '家->機場',
    stopId: '6ACFA599CC0EA4E0',
    route: 'A41',
    serviceType: '1',
    company: BusCompany.KMB,
    eta: [],
  },
  {
    dest: '機場',
    stopName: '帝都酒店',
    category: '家->機場',
    stopId: '6ACFA599CC0EA4E0',
    route: 'A46',
    serviceType: '1',
    company: BusCompany.KMB,
    eta: [],
  },
])

// Reactive object to store the collapsed state for each category
// Key: category name (string), Value: is collapsed (boolean, true for collapsed, false for expanded)
const collapsedCategories = ref<Record<string, boolean>>({})

// Computed property to extract all unique categories from items
const categories = computed(() => {
  const uniqueCategories = new Set(items.value.map((item) => item.category))
  return Array.from(uniqueCategories)
})

// When the component is mounted, initialize all categories to expanded state
onMounted(() => {
  categories.value.forEach((category) => {
    collapsedCategories.value[category] = false // Default to expanded
  })
})

// Function to toggle the collapsed state of a category
const toggleCategory = (category: string) => {
  collapsedCategories.value[category] = !collapsedCategories.value[category]
}

// Function to get list items by category
const getItemsByCategory = (category: string) => {
  return items.value.filter((item) => item.category === category)
}

const intervalId = ref<number | null>(null)

// Asynchronous function
const fetchDataPeriodically = async () => {
  for (const item of items.value) {
    const { company, stopId, route, serviceType } = item
    const tempEta = []
    if (company === BusCompany.KMB) {
      // Send request to https://data.etabus.gov.hk/v1/transport/kmb/eta/{stop_id}/{route}/{service_type}
      try {
        const response = await fetch(
          `https://data.etabus.gov.hk/v1/transport/kmb/eta/${stopId}/${route}/${serviceType}`,
        )
        const data = await response.json()
        if (data.data && Array.isArray(data.data)) {
          for (const i of data.data) {
            const eta = new Date(i.eta)
            const currentTime = new Date()
            if (eta < currentTime) {
              continue
            }
            const difference = (eta.getTime() - currentTime.getTime()) / 1000 / 60
            tempEta.push(Math.floor(difference))
          }
        }
      } catch (error) {
        console.error(`Error fetching KMB ETA for ${route} at ${stopId}:`, error)
      }
    } else if (company === BusCompany.CTB) {
      try {
        const response = await fetch(
          `https://rt.data.gov.hk/v2/transport/citybus/eta/CTB/${stopId}/${route}`,
        )
        const data = await response.json()
        if (data.data && Array.isArray(data.data)) {
          for (const i of data.data) {
            const eta = new Date(i.eta)
            const currentTime = new Date()
            if (eta < currentTime) {
              continue
            }
            const difference = (eta.getTime() - currentTime.getTime()) / 1000 / 60
            tempEta.push(Math.floor(difference))
          }
        }
      } catch (error) {
        console.error(`Error fetching CTB ETA for ${route} at ${stopId}:`, error)
      }
    } else {
      console.error(`Unknown bus company: ${company}`)
    }
    item.eta = tempEta
  }
}

onMounted(() => {
  // Execute the asynchronous function immediately after component is mounted
  fetchDataPeriodically()
  // Start the main timer
  intervalId.value = setInterval(() => {
    fetchDataPeriodically()
  }, 10000) // 10000 milliseconds = 10 seconds
})

onBeforeUnmount(() => {
  // Clear the timer before the component is unmounted
  if (intervalId.value) {
    clearInterval(intervalId.value)
    intervalId.value = null
  }
})
</script>

<template>
  <div class="min-h-screen bg-gray-100 p-4">
    <h1 class="text-3xl font-bold text-gray-800 text-center my-4">香港巴士預計到達時間</h1>

    <!-- List container, limited max width and centered, with shadow and rounded corners -->
    <div class="max-w-md mx-auto bg-white shadow-lg rounded-lg overflow-hidden">
      <!-- Iterate through all categories -->
      <div
        v-for="category in categories"
        :key="category"
        class="border-b border-gray-200 last:border-b-0"
      >
        <!-- Category title button, click to toggle collapsed state -->
        <button
          @click="toggleCategory(category)"
          class="w-full flex justify-between items-center p-4 text-left text-lg font-semibold text-gray-700 bg-gray-50 hover:bg-gray-100 transition-colors duration-200"
        >
          <span>{{ category }}</span>
          <!-- Collapse/expand icon, rotates based on state -->
          <svg
            :class="{ 'rotate-180': collapsedCategories[category] }"
            class="w-5 h-5 text-gray-500 transform transition-transform duration-200"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
            xmlns="http://www.w3.org/2000/svg"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M19 9l-7 7-7-7"
            ></path>
          </svg>
        </button>

        <!-- List content under the category, show/hide based on collapsed state -->
        <div
          class="overflow-hidden transition-all duration-300 ease-in-out"
          :class="{
            'max-h-0': collapsedCategories[category],
            'max-h-screen': !collapsedCategories[category],
          }"
        >
          <ul class="divide-y divide-gray-100">
            <!-- Iterate through all list items in the current category -->
            <li
              v-for="item in getItemsByCategory(category)"
              :key="item.company + '-' + item.route"
              class="px-4 py-3 hover:bg-gray-50 transition-colors duration-150"
            >
              <div class="flex justify-between items-center">
                <!-- Left Section: Route, Destination, and Stop Name -->
                <div class="flex-grow">
                  <h3 class="text-lg font-bold text-gray-800">
                    {{ item.route }}
                    <span class="text-base font-normal text-gray-600 ml-2">往 {{ item.dest }}</span>
                  </h3>
                  <p class="text-sm text-gray-500 mt-1">{{ item.stopName }}</p>
                </div>

                <!-- Right Section: ETA -->
                <div class="text-right flex-shrink-0 pl-4">
                  <!-- Conditional rendering for ETA display -->
                  <p v-if="item.eta.length > 0" class="font-semibold text-green-600">
                    <!-- First ETA with larger font -->
                    <span class="text-2xl">{{ item.eta[0] }}</span>
                    <!-- Subsequent ETAs with normal font, if they exist -->
                    <span v-if="item.eta.length > 1" class="text-lg ml-1">
                      / {{ item.eta.slice(1, 3).join(' / ') }}
                    </span>
                    <span class="text-sm font-normal ml-1">分鐘</span>
                  </p>
                  <!-- No data message -->
                  <p v-else class="text-sm text-gray-400">暫無資料</p>
                </div>
              </div>
            </li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped></style>
