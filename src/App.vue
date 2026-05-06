<template>
  <div class="container">
    <!-- Header -->
    <div class="header">
      <h1>🗺️ Cartographie des Employés</h1>
      <div class="header-info">EHPAD Agir Castres - Visualisation des distances</div>
    </div>

    <!-- Main content -->
    <div style="display: flex; width: 100%; flex: 1; overflow: hidden;">
      <!-- Map -->
      <div class="map-container">
        <div id="map"></div>
      </div>

      <!-- Sidebar -->
      <div class="sidebar">
        <div class="sidebar-header">
          <h2>Employés</h2>
          <input 
            v-model="searchQuery" 
            type="text" 
            class="search-box" 
            placeholder="Rechercher un employé..."
          >
        </div>

        <div class="employees-list">
          <div 
            v-if="loadingStatus && Object.keys(markers).length < employees.length" 
            class="loading"
          >
            {{ loadingStatus }}
          </div>
          
          <div 
            v-if="filteredEmployees.length === 0 && employees.length > 0" 
            class="loading"
          >
            Aucun employé trouvé
          </div>
          
          <div
            v-for="employee in filteredEmployees"
            :key="employee.id"
            class="employee-item"
            :class="{ active: selectedEmployee?.id === employee.id }"
            @click="selectEmployee(employee)"
          >
            <div class="employee-name">{{ employee.name }}</div>
            <div class="employee-distance">📍 {{ employee.distance }} km</div>
            <div class="employee-address">{{ employee.address }}, {{ employee.postalCode }} {{ employee.city }}</div>
          </div>
        </div>

        <div class="stats-container">
          <div class="stat-item">
            <span class="stat-label">Total employés:</span> {{ employees.length }}
          </div>
          <div class="stat-item">
            <span class="stat-label">Distance moyenne:</span> {{ averageDistance }} km
          </div>
          <div class="stat-item">
            <span class="stat-label">Plus proche:</span> {{ minDistance }} km
          </div>
          <div class="stat-item">
            <span class="stat-label">Plus loin:</span> {{ maxDistance }} km
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'
import L from 'leaflet'
import Papa from 'papaparse'

// Fix pour les icônes Leaflet
delete L.Icon.Default.prototype._getIconUrl
L.Icon.Default.mergeOptions({
  iconRetinaUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/images/marker-icon-2x.png',
  iconUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/images/marker-icon.png',
  shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/images/marker-shadow.png'
})

// Données
const employees = ref([])
const selectedEmployee = ref(null)
const searchQuery = ref('')
const map = ref(null)
const markers = ref({})
const lines = ref({})
const distanceLabels = ref({})
const loadingStatus = ref('Initialisation...')
const geocodeCache = ref({})

// EHPAD coordinates
const EHPAD = {
  name: 'EHPAD Agir Castres',
  lat: 43.6068,
  lng: 2.2386
}

// Computed properties
const filteredEmployees = computed(() => {
  if (!searchQuery.value) return employees.value
  const query = searchQuery.value.toLowerCase()
  return employees.value.filter(emp => 
    emp.name.toLowerCase().includes(query) ||
    emp.address.toLowerCase().includes(query) ||
    emp.city.toLowerCase().includes(query)
  )
})

const averageDistance = computed(() => {
  if (employees.value.length === 0) return 0
  const total = employees.value.reduce((sum, emp) => sum + emp.distance, 0)
  return (total / employees.value.length).toFixed(2)
})

const minDistance = computed(() => {
  if (employees.value.length === 0) return 0
  return Math.min(...employees.value.map(emp => emp.distance)).toFixed(2)
})

const maxDistance = computed(() => {
  if (employees.value.length === 0) return 0
  return Math.max(...employees.value.map(emp => emp.distance)).toFixed(2)
})

// Fonction pour obtenir la couleur selon la distance
const getColorByDistance = (distance) => {
  if (distance < 5) return '#00cc00' // Vert - très proche
  if (distance < 15) return '#66cc00' // Vert clair - proche
  if (distance < 30) return '#ffcc00' // Jaune - moyen
  if (distance < 50) return '#ff9900' // Orange - loin
  return '#ff3333' // Rouge - très loin
}

// Methods
const loadData = async () => {
  try {
    const response = await fetch('/liste salariés avec distances.csv')
    const text = await response.text()
    
    Papa.parse(text, {
      header: true,
      skipEmptyLines: true,
      complete: (results) => {
        employees.value = results.data
          .filter(row => row['Distance (km)'] && row.Salarié && !row.Salarié.includes('Total'))
          .map((row, index) => ({
            id: index,
            name: row.Salarié || 'Inconnu',
            address: row.Adresse || '',
            postalCode: row['Code\npostal'] || '',
            city: row.Ville || '',
            distance: parseFloat(row['Distance (km)']) || 0,
            verifyUrl: row['Vérifier sur Google Maps'] || ''
          }))
      },
      error: (error) => {
        console.error('Erreur lors du chargement du CSV:', error)
      }
    })
  } catch (error) {
    console.error('Erreur lors de la récupération du fichier:', error)
  }
}

const initMap = () => {
  // Créer la carte centrée sur l'EHPAD
  map.value = L.map('map').setView([EHPAD.lat, EHPAD.lng], 10)

  // Ajouter la couche de tuiles
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap contributors',
    maxZoom: 19
  }).addTo(map.value)

  // Ajouter marqueur EHPAD - GRAND ET VISIBLE
  const ehpadHtml = `
    <div style="
      width: 60px; 
      height: 60px; 
      background-color: #e74c3c; 
      border-radius: 50%; 
      display: flex; 
      align-items: center; 
      justify-content: center; 
      color: white; 
      font-size: 30px;
      font-weight: bold;
      box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
      border: 3px solid white;
    ">
      🏥
    </div>
  `
  
  const ehpadIcon = L.divIcon({
    html: ehpadHtml,
    className: '',
    iconSize: [60, 60],
    iconAnchor: [30, 60]
  })

  L.marker([EHPAD.lat, EHPAD.lng], { icon: ehpadIcon })
    .addTo(map.value)
    .bindPopup(`<div class="employee-popup">
      <div class="popup-header" style="font-size: 16px;">${EHPAD.name}</div>
      <div class="popup-info">Siège central</div>
      <div class="popup-info" style="font-weight: 600; margin-top: 8px;">34 Rue Camille Rabaud, 81100 Castres</div>
    </div>`)
}

const addEmployeeMarkers = async () => {
  let successCount = 0
  const failedEmployees = []
  
  for (let i = 0; i < employees.value.length; i++) {
    const employee = employees.value[i]
    const address = `${employee.address}, ${employee.postalCode} ${employee.city}`
    
    loadingStatus.value = `Chargement: ${i + 1}/${employees.value.length}`
    
    try {
      // Vérifier le cache
      let coords
      if (geocodeCache.value[address]) {
        coords = geocodeCache.value[address]
      } else {
        // Essayer d'abord avec l'adresse complète
        let response = await geocodeWithRetry(address)
        let data = await response.json()
        
        if (!data.features || data.features.length === 0) {
          // Fallback: essayer avec juste la ville
          response = await geocodeWithRetry(`${employee.city}, ${employee.postalCode}`)
          data = await response.json()
        }
        
        if (data.features && data.features.length > 0) {
          coords = data.features[0].geometry.coordinates
          geocodeCache.value[address] = coords
        } else {
          failedEmployees.push(employee.name)
          continue
        }
      }

      const lat = coords[1]
      const lng = coords[0]

      const color = getColorByDistance(employee.distance)
      const initials = employee.name.split(' ').map(n => n[0]).join('').substring(0, 2).toUpperCase()

      const markerHtml = `
        <div style="
          width: 50px; 
          height: 50px; 
          background-color: ${color}; 
          border-radius: 50%; 
          display: flex; 
          align-items: center; 
          justify-content: center; 
          color: white; 
          font-size: 18px;
          font-weight: bold;
          box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
          border: 3px solid white;
          cursor: pointer;
        ">
          ${initials}
        </div>
      `

      const icon = L.divIcon({
        html: markerHtml,
        className: '',
        iconSize: [50, 50],
        iconAnchor: [25, 50]
      })

      const marker = L.marker([lat, lng], { icon })
        .addTo(map.value)
        .bindPopup(`<div class="employee-popup">
          <div class="popup-header" style="color: ${color};">${employee.name}</div>
          <div class="popup-distance" style="color: ${color};">📍 ${employee.distance} km</div>
          <div class="popup-info">${address}</div>
          ${employee.verifyUrl ? `<div style="margin-top: 8px;"><a href="${employee.verifyUrl}" target="_blank" style="color: ${color};">Google Maps</a></div>` : ''}
        </div>`)

      markers.value[employee.id] = { marker, lat, lng }
      addDistanceLine(employee, lat, lng, color)
      successCount++

      // Petit délai pour éviter surcharge API
      await new Promise(r => setTimeout(r, 30))
    } catch (error) {
      failedEmployees.push(employee.name)
      console.warn(`Erreur pour ${employee.name}:`, error.message)
    }
  }
  
  const message = failedEmployees.length > 0 
    ? `✅ ${successCount}/${employees.value.length} (Non trouvés: ${failedEmployees.join(', ')})`
    : `✅ ${successCount}/${employees.value.length} employés`
  
  loadingStatus.value = message
  console.log('Employés non géocodés:', failedEmployees)
}

const geocodeWithRetry = async (query) => {
  const controller = new AbortController()
  const timeout = setTimeout(() => controller.abort(), 8000)
  
  try {
    const response = await fetch(
      `https://api-adresse.data.gouv.fr/search/?q=${encodeURIComponent(query)}&limit=3`,
      { signal: controller.signal }
    )
    clearTimeout(timeout)
    return response
  } catch (error) {
    clearTimeout(timeout)
    throw error
  }
}

const addDistanceLine = (employee, empLat, empLng, color) => {
  const latlngs = [
    [EHPAD.lat, EHPAD.lng],
    [empLat, empLng]
  ]

  const polyline = L.polyline(latlngs, {
    color: color,
    weight: 2,
    opacity: 0.5,
    dashArray: '5, 5'
  }).addTo(map.value)

  lines.value[employee.id] = polyline
}

const selectEmployee = (employee) => {
  selectedEmployee.value = employee
  
  // Si le marqueur existe, le montrer directement
  if (markers.value[employee.id]) {
    const { marker, lat, lng } = markers.value[employee.id]
    map.value.setView([lat, lng], 13)
    marker.openPopup()
  } else {
    // Sinon, refaire le geocodage pour trouver le marqueur
    const address = `${employee.address}, ${employee.postalCode} ${employee.city}`
    geocodeWithRetry(address)
      .then(res => res.json())
      .then(data => {
        if (data.features && data.features.length > 0) {
          const coords = data.features[0].geometry.coordinates
          const lat = coords[1]
          const lng = coords[0]
          map.value.setView([lat, lng], 13)
          
          // Ouvrir le popup du marqueur s'il existe
          if (markers.value[employee.id]) {
            markers.value[employee.id].marker.openPopup()
          }
        }
      })
      .catch(err => console.error('Erreur selectEmployee:', err))
  }
}

// Lifecycle
onMounted(async () => {
  initMap()
  loadData()
  
  // Attendre que les données soient chargées
  const waitForData = setInterval(async () => {
    if (employees.value.length > 0) {
      clearInterval(waitForData)
      loadingStatus.value = 'Géocodage en cours...'
      await addEmployeeMarkers()
    }
  }, 200)
})

// Cleanup
onBeforeUnmount(() => {
  if (map.value) {
    map.value.remove()
  }
})
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  width: 100%;
}

.header {
  flex-shrink: 0;
}
</style>
