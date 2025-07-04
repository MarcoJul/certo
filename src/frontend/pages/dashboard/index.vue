<script setup>
import { computed, onMounted, ref } from 'vue'
import { useRouter } from 'vue-router'
import { apiClient } from '~/api/api-client'
import CertificateCard from '~/components/CertificateCard.vue'
import ImportCertificate from '~/components/ImportCertificate.vue'
import { useAuthStore } from '~/stores/auth'

defineOptions({
  name: 'DashboardPage'
})

definePageMeta({
  middleware: 'auth'
})

const authStore = useAuthStore()
const isAuthenticated = computed(() => authStore.isAuthenticated)
const activeTab = ref('received')
const showImportModal = ref(false)

const receivedCertificates = ref([])
const issuedCertificates = ref([])
const loading = ref(false)
const error = ref(null)

const router = useRouter()

onMounted(async () => {
  if (isAuthenticated.value) {
    await loadCertificates()
  }
})

async function loadCertificates() {
  loading.value = true
  error.value = null

  try {
    // Load both received and issued certificates
    const [receivedResponse, issuedResponse] = await Promise.all([
      apiClient.getReceivedCertificates(),
      apiClient.getIssuedCertificates(),
    ])

    // The API client already formats the credentials
    receivedCertificates.value = receivedResponse.data || []
    issuedCertificates.value = issuedResponse.data || []
  }
  catch (err) {
    console.error('Error loading certificates:', err)
    error.value = 'Failed to load certificates. Please try again later.'
  }
  finally {
    loading.value = false
  }
}

function handleImportSuccess() {
  showImportModal.value = false
  loadCertificates() // Reload certificates to include the new one
}

function handleExport() {
  // Just reload the certificates after export (not strictly necessary)
  loadCertificates()
}

function handleRevoke() {
  // Reload to refresh the certificate status
  loadCertificates()
}

function toggleImportModal() {
  showImportModal.value = !showImportModal.value
}

function handleView(id) {
  router.push(`/credentials/${encodeURIComponent(id)}`)
}

useHead({
  title: 'Dashboard | Certo',
  meta: [
    { name: 'description', content: 'Manage your credentials and certificates' }
  ]
})
</script>

<template>
  <div class="container mx-auto px-4 py-8">
    <h1 class="text-3xl font-bold mb-8">
      Credential Dashboard
    </h1>

    <!-- Auth check -->
    <template v-if="!isAuthenticated">
      <div class="bg-yellow-50 border border-yellow-200 p-6 rounded-lg text-center">
        <div class="i-lucide-alert-triangle w-12 h-12 mx-auto text-yellow-500 mb-3" />
        <h2 class="text-xl font-semibold mb-2">
          Authentication Required
        </h2>
        <p class="mb-4 text-gray-600">
          You need to be logged in to view your credentials.
        </p>
        <div class="flex justify-center gap-4">
          <NuxtLink to="/login" class="inline-flex items-center px-4 py-2 bg-primary-600 hover:bg-primary-700 text-white rounded-md">
            Log In
          </NuxtLink>
          <NuxtLink to="/register" class="inline-flex items-center px-4 py-2 bg-gray-100 hover:bg-gray-200 text-gray-800 rounded-md">
            Register
          </NuxtLink>
        </div>
      </div>
    </template>

    <template v-else>
      <div class="flex justify-between items-center mb-6">
        <div>
          <NTab
            v-model="activeTab"
            :items="[
              { value: 'received', label: 'Received Credentials' },
              { value: 'issued', label: 'Issued Credentials' },
            ]"
          />
        </div>

        <div class="flex space-x-4">
          <NButton
            v-if="activeTab === 'received'"
            class="flex items-center"
            @click="toggleImportModal"
          >
            <div class="i-lucide-upload mr-2" />
            Import Certificate
          </NButton>

          <NuxtLink to="/issue">
            <NButton v-if="activeTab === 'issued'" class="flex items-center">
              <div class="i-lucide-plus mr-2" />
              Issue New Badge
            </NButton>
          </NuxtLink>
        </div>
      </div>

      <!-- Loading state -->
      <div v-if="loading" class="flex justify-center py-12">
        <div class="i-lucide-loader animate-spin w-8 h-8" />
      </div>

      <!-- Error state -->
      <NAlert v-else-if="error" variant="error" class="mb-6">
        {{ error }}
        <template #action>
          <NButton size="sm" @click="loadCertificates">
            Retry
          </NButton>
        </template>
      </NAlert>

      <!-- Received credentials -->
      <div v-else-if="activeTab === 'received'">
        <div v-if="receivedCertificates.length === 0" class="text-center py-12 bg-gray-50 rounded-lg">
          <div class="i-lucide-inbox w-12 h-12 mx-auto text-gray-400 mb-3" />
          <h3 class="text-lg font-medium mb-2">
            No Credentials Yet
          </h3>
          <p class="text-gray-600 mb-4">
            You haven't received any credentials yet.
          </p>
          <NButton variant="outline" @click="toggleImportModal">
            Import a Certificate
          </NButton>
        </div>

        <div v-else class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <CertificateCard
            v-for="certificate in receivedCertificates"
            :key="certificate.id"
            :certificate="certificate"
            @export="handleExport"
            @view="handleView"
          />
        </div>
      </div>

      <!-- Issued credentials -->
      <div v-else-if="activeTab === 'issued'">
        <div v-if="issuedCertificates.length === 0" class="text-center py-12 bg-gray-50 rounded-lg">
          <div class="i-lucide-award w-12 h-12 mx-auto text-gray-400 mb-3" />
          <h3 class="text-lg font-medium mb-2">
            No Issued Credentials
          </h3>
          <p class="text-gray-600 mb-4">
            You haven't issued any credentials yet.
          </p>
          <NuxtLink to="/issue">
            <NButton variant="outline">
              Issue a Badge
            </NButton>
          </NuxtLink>
        </div>

        <div v-else class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <CertificateCard
            v-for="certificate in issuedCertificates"
            :key="certificate.id"
            :certificate="certificate"
            @export="handleExport"
            @revoke="handleRevoke"
            @view="handleView"
          />
        </div>
      </div>
    </template>

    <!-- Import Modal -->
    <NModal v-model="showImportModal" size="lg">
      <ImportCertificate @import-success="handleImportSuccess" />
    </NModal>
  </div>
</template>
