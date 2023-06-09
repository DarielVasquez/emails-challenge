<template>
  <div class="container mx-auto pt-24 pb-20">
    <template v-if="loading">
      <div class="flex items-center justify-center h-[80vh]">
        <v-icon name="pr-spinner" animation="spin" scale="2" fill="gray"></v-icon>
      </div>
    </template>
    <template v-else>
      <div class="lg:grid lg:grid-cols-3 gap-4 bg-white p-1 lg:p-0 space-y-4 lg:space-y-0">
        <div class="flex flex-col lg:col-span-2 border rounded-md h-fit max-h-[80vh] p-5">
          <div class="sm:flex justify-between p-2 space-y-4 sm:space-y-0">
            <div class="text-center self-center">
              <span>Show </span>
              <select
                class="border border-gray-300 rounded-md"
                name="limit"
                id="limit"
                v-model="emailsPerPage"
                @change="handleLimitChange"
              >
                <option :value="limit" v-for="(limit, index) in limits" :key="index">
                  {{ limit }}
                </option>
              </select>
              <span> entries</span>
            </div>
            <div class="relative">
              <div class="absolute inset-y-0 left-0 flex items-center pl-3 pointer-events-none">
                <v-icon name="md-search" fill="gray"></v-icon>
              </div>
              <input
                type="search"
                class="search block w-full p-2 pl-10 border border-gray-300 rounded-md focus:ring-gray-500 focus:border-gray-700 focus:outline-none"
                placeholder="Search..."
                v-model.trim="searchQuery"
                @keyup.enter="changeQuery, getEmails()"
              />
            </div>
          </div>
          <div v-if="errorMsg" class="text-center text-red-500">{{ errorMsg }}</div>
          <div class="overflow-x-auto border border-gray-300 rounded-md">
            <table class="table-auto w-full">
              <thead class="sticky top-0 drop-shadow">
                <tr class="bg-gray-200 cursor-pointer">
                  <th class="px-4 py-2 min-w-[120px]" @click="sortTable('subject')">
                    <span> Subject </span>
                    <sort-icons-vue
                      :sortDirection="sortDirection"
                      header="subject"
                    ></sort-icons-vue>
                  </th>
                  <th class="px-4 py-2 min-w-[105px]" @click="sortTable('x_origin')">
                    Origin
                    <sort-icons-vue
                      :sortDirection="sortDirection"
                      header="x_origin"
                    ></sort-icons-vue>
                  </th>
                  <th class="px-4 py-2 min-w-[100px]" @click="sortTable('from')">
                    From
                    <sort-icons-vue :sortDirection="sortDirection" header="from"></sort-icons-vue>
                  </th>
                  <th class="px-4 py-2 min-w-[100px]" @click="sortTable('to_0_')">
                    To
                    <sort-icons-vue :sortDirection="sortDirection" header="to_0_"></sort-icons-vue>
                  </th>
                  <th class="px-4 py-2 min-w-[105px]" @click="sortTable('x_folder')">
                    Folder
                    <sort-icons-vue
                      :sortDirection="sortDirection"
                      header="x_folder"
                    ></sort-icons-vue>
                  </th>
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="email in sortedTableEmails"
                  :key="email._timestamp"
                  class="border-t mx-auto cursor-pointer"
                  @click="selectFeaturedEmail(email)"
                >
                  <td class="px-4 py-2" v-html="highlight(email.subject)"></td>
                  <td class="px-4 py-2" v-html="highlight(email.x_origin)"></td>
                  <td class="px-4 py-2" v-html="highlight(email.from)"></td>
                  <td
                    class="px-4 py-2"
                    v-html="highlight(email.to_0_)?.split(',').splice(0, 5)"
                  ></td>
                  <td class="px-4 py-2" v-html="highlight(email.x_folder)"></td>
                </tr>
              </tbody>
            </table>
          </div>
          <pagination-vue
            :currentPage="currentPage"
            :totalPages="totalPages"
            @update:currentPage="updateCurrentPage"
          ></pagination-vue>
          <div class="flex justify-center text-center">
            Showing {{ indexOfFirstEmail + 1 }} to
            {{ indexOfLastEmail > emails.length ? emails.length : indexOfLastEmail }} of
            {{ emails.length }} entries ({{ elapsedTime.toFixed(2) }} seconds)
          </div>
        </div>
        <div class="px-5 py-10 overflow-x-auto border rounded-md h-fit max-h-[80vh]">
          <div class="font-bold" v-html="highlight(featuredEmail.subject)"></div>
          <div v-html="highlight(featuredEmail.content)"></div>
        </div>
      </div>
    </template>
  </div>
</template>

<script>
import { ref, computed, watch } from 'vue'
import axios from 'axios'
import he from 'he'
import SortIconsVue from './SortIcons.vue'
import PaginationVue from './Pagination.vue'
export default {
  name: 'DataTable',
  components: {
    SortIconsVue,
    PaginationVue
  },
  setup() {
    const emails = ref([])
    const elapsedTime = ref(0)
    const errorMsg = ref('')
    const loading = ref(true)
    const payload = ref({
      query: {
        sql: 'SELECT * FROM emails',
        size: 1000
      }
    })

    const getEmails = () => {
      const startTime = performance.now()
      axios
        .post(import.meta.env.VITE_URI, payload.value, {
          auth: {
            username: import.meta.env.VITE_AUTH_USERNAME,
            password: import.meta.env.VITE_AUTH_PASSWORD
          }
        })
        .then((response) => {
          if (response.data.hits.length === 0) {
            errorMsg.value = 'No data was found, please try a different query.'
          } else {
            emails.value = response.data.hits.map((email) => {
              const endTime = performance.now()
              elapsedTime.value = (endTime - startTime) / 1000
              email.content = he.decode(email.content).replace(/\n/g, '<br>')
              email.subject = he.decode(email.subject)
              email.x_folder = email.x_folder.split('\\').pop()
              currentPage.value = 1 // Set current page back to 1
              sortDirection.value = {} // Set all sort headers to default
              loading.value = false
              return email
            })
            featuredEmail.value = response.data.hits[0]
            errorMsg.value = ''
          }
        })
        .catch((error) => {
          errorMsg.value = 'Error retrieving data.'
          loading.value = false
          console.log(error)
        })
    }

    const getHelloWorld = () => {
      axios
        .get(import.meta.env.VITE_TEST_API)
        .then((response) => {
          console.log(response.data)
        })
        .catch((error) => {
          console.log(error)
        })
    }

    // Featured Email

    const featuredEmail = ref({})

    const selectFeaturedEmail = (email) => {
      featuredEmail.value = email
    }

    // Search Query

    const searchQuery = ref('')
    const searchQueryASCII = ref('')

    const highlight = (fileName) => {
      if (!fileName) {
        return '(untitled)'
      }
      if (searchQuery.value === '') {
        return fileName
      } else {
        const searchPattern = new RegExp(`(?<!<[^>]*)(${searchQuery.value})`, 'gi') // Excludes <br> tags from getting highlighted
        return fileName?.replaceAll(searchPattern, (match) => {
          return `<span class="bg-yellow-500">${match}</span>`
        })
      }
    }

    watch(searchQuery, (newValue) => {
      // Update search query to replace single quotes to ASCII
      searchQueryASCII.value = newValue.replace(/'/g, '&#39;')
    })

    const changeQuery = computed(() => {
      // Update sql query with the search input
      payload.value.query.sql = `SELECT * FROM emails WHERE match_all('${searchQueryASCII.value}')`
    })

    // Sort Table

    const sortBy = ref('')
    const sortDirection = ref({})

    const sortTable = (column) => {
      if (sortBy.value === column) {
        sortDirection.value[column] = sortDirection.value[column] === 'asc' ? 'desc' : 'asc'
      } else {
        sortBy.value = column
        sortDirection.value[column] = 'asc'
      }
      for (const key in sortDirection.value) {
        if (sortDirection.value.hasOwnProperty(key)) {
          if (column !== key) {
            sortDirection.value[key] = ''
          }
        }
      }
    }

    const sortedTableEmails = computed(() => {
      // Sort the array of emails according to the table header
      const column = sortBy.value
      const direction = sortDirection.value[column]
      return filteredEmails.value.sort((a, b) => {
        const aValue = a[column]?.toUpperCase()
        const bValue = b[column]?.toUpperCase()
        if (aValue < bValue) return direction === 'asc' ? -1 : 1
        if (aValue > bValue) return direction === 'asc' ? 1 : -1
        return 0
      })
    })

    // Pagination

    const currentPage = ref(1)
    const emailsPerPage = ref(10)
    const limits = ref([5, 10, 15, 20, 25, 50, 100, 500, 1000])

    const handleLimitChange = (event) => {
      currentPage.value = 1
      emailsPerPage.value = event.target.value
    }

    const updateCurrentPage = (value) => {
      currentPage.value = value
    }

    const indexOfLastEmail = computed(() => {
      return currentPage.value * Math.round(emailsPerPage.value / 5) * 5
    })

    const indexOfFirstEmail = computed(() => {
      return indexOfLastEmail.value - Math.round(emailsPerPage.value / 5) * 5
    })

    const filteredEmails = computed(() => {
      return emails.value.slice(indexOfFirstEmail.value, indexOfLastEmail.value)
    })

    const totalPages = computed(() => {
      return Math.ceil(emails.value.length / emailsPerPage.value)
    })

    // API Requests

    getEmails()
    getHelloWorld()

    return {
      emails,
      featuredEmail,
      sortBy,
      sortDirection,
      currentPage,
      emailsPerPage,
      limits,
      elapsedTime,
      searchQuery,
      searchQueryASCII,
      errorMsg,
      loading,
      payload,
      getEmails,
      highlight,
      selectFeaturedEmail,
      sortTable,
      handleLimitChange,
      updateCurrentPage,
      changeQuery,
      sortedTableEmails,
      indexOfLastEmail,
      indexOfFirstEmail,
      totalPages
    }
  }
}
</script>

<style scoped>
/* Styling for Webkit-based browsers (e.g., Chrome, Safari) */
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
  background-color: #f3f3f3;
  border-radius: 10px;
}

/* Track */
::-webkit-scrollbar-track {
  border-radius: 10px;
}

::-webkit-scrollbar-track-piece {
  border-radius: 10px;
}

::-webkit-scrollbar-corner {
  border-radius: 10px;
}

::-webkit-scrollbar-thumb {
  background-color: #999999;
  border-radius: 10px;
}

::-webkit-scrollbar-thumb:hover {
  background-color: #666666;
}

/* Styling for Firefox */
* {
  scrollbar-width: thin;
  scrollbar-color: #999999 #f3f3f3;
}

/* Styling the search input cancel button */
.search::-webkit-search-cancel-button {
  background-color: gray;
  cursor: pointer;
  -webkit-appearance: none;
  height: 16px;
  width: 16px;
  background-image: url("data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='%23fff'><path d='M19 6.41L17.59 5 12 10.59 6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 12 13.41 17.59 19 19 17.59 13.41 12z'/></svg>");
  border-radius: 50%;
}
</style>
