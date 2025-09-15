<template>
  <div class="step-viewer">
    <div class="viewer-header">
      <h2>🔧 Uçak Motoru Bakım Görüntüleyicisi</h2>
      <p>Predictive Maintenance - Arıza Tespiti ve Görselleştirme</p>
    </div>

    <!-- Arıza Bilgileri Paneli -->
    <div class="fault-info-panel">
      <div class="fault-header">
        <h3>⚠️ Tespit Edilen Arıza</h3>
        <div class="fault-severity" :class="faultData.severity">
          {{ faultData.severityText }}
        </div>
      </div>
      
      <div class="fault-details">
        <div class="fault-item">
          <strong>Arızalı Bileşen:</strong>
          <span>{{ faultData.componentName }}</span>
        </div>
        <div class="fault-item">
          <strong>Arıza Türü:</strong>
          <span>{{ faultData.faultType }}</span>
        </div>
        <div class="fault-item">
          <strong>Tespit Tarihi:</strong>
          <span>{{ faultData.detectionDate }}</span>
        </div>
        <div class="fault-item">
          <strong>Öngörülen Kalan Ömür:</strong>
          <span>{{ faultData.remainingLife }}</span>
        </div>
        <div class="fault-item">
          <strong>Önerilen Aksiyon:</strong>
          <span>{{ faultData.recommendedAction }}</span>
        </div>
      </div>
    </div>

    <!-- 3D Görüntüleyici -->
    <div class="viewer-container">
      <div class="viewer-controls">
        <button @click="resetView" class="control-btn">🔄 Görünümü Sıfırla</button>
        <button @click="toggleHighlight" class="control-btn">
          {{ showHighlight ? '🔍 Highlight Kapat' : '🔍 Arızalı Parçayı Göster' }}
        </button>
        <button @click="toggleWireframe" class="control-btn">
          {{ wireframeMode ? '🔲 Katı' : '🔳 Tel Kafes' }}
        </button>
      </div>
      
      <div class="canvas-container">
        <canvas ref="threeCanvas" class="three-canvas"></canvas>
        <div v-if="loading" class="loading-overlay">
          <div class="loading-spinner">⏳</div>
          <p>Motor modeli yükleniyor...</p>
        </div>
      </div>
    </div>

    <!-- Bileşen Listesi -->
    <div class="component-list">
      <h3>📋 Motor Bileşenleri</h3>
      <div class="component-grid">
        <div 
          v-for="component in components" 
          :key="component.id"
          class="component-item"
          :class="{ 
            'faulty': component.isFaulty,
            'highlighted': highlightedComponent === component.id 
          }"
          @click="highlightComponent(component.id)"
        >
          <div class="component-icon">{{ component.icon }}</div>
          <div class="component-info">
            <div class="component-name">{{ component.name }}</div>
            <div class="component-status" :class="component.status">
              {{ component.statusText }}
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

// Reaktif değişkenler
const threeCanvas = ref(null)
const loading = ref(true)
const wireframeMode = ref(false)
const showHighlight = ref(true)
const highlightedComponent = ref(null)
const animationFrameId = ref(null)

// Three.js nesneleri
let scene = null
let camera = null
let renderer = null
let controls = null
let modelGroup = null
let faultMeshes = new Map() // Arızalı mesh'leri saklamak için

// Arıza verisi (gerçek uygulamada API'den gelecek)
const faultData = ref({
  componentName: "Turbine Blade Assembly",
  faultType: "Metal Fatigue",
  detectionDate: "2024-01-15",
  remainingLife: "72 saat",
  severity: "high",
  severityText: "Yüksek Öncelik",
  recommendedAction: "Acil Değişim Gerekli"
})

// Bileşen listesi
const components = ref([
  { id: 1, name: "Turbine Blade Assembly", icon: "🔄", isFaulty: true, status: "faulty", statusText: "Arızalı" },
  { id: 2, name: "Compressor Stage 1", icon: "🌀", isFaulty: false, status: "good", statusText: "Sağlam" },
  { id: 3, name: "Combustion Chamber", icon: "🔥", isFaulty: false, status: "good", statusText: "Sağlam" },
  { id: 4, name: "Fan Assembly", icon: "💨", isFaulty: false, status: "warning", statusText: "İzle" },
  { id: 5, name: "Bearing Assembly", icon: "⚙️", isFaulty: false, status: "good", statusText: "Sağlam" },
  { id: 6, name: "Nozzle Assembly", icon: "🚀", isFaulty: false, status: "good", statusText: "Sağlam" }
])

// Three.js sahnesini başlat
function initThreeDScene() {
  const canvas = threeCanvas.value
  if (!canvas) return

  // Sahne oluştur
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0xf0f0f0)

  // Kamera oluştur
  camera = new THREE.PerspectiveCamera(75, canvas.clientWidth / canvas.clientHeight, 0.1, 1000)
  camera.position.set(10, 10, 10)

  // Renderer oluştur
  renderer = new THREE.WebGLRenderer({ canvas, antialias: true })
  renderer.setPixelRatio(window.devicePixelRatio)
  renderer.setSize(canvas.clientWidth, canvas.clientHeight)

  // Işıklandırma
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6)
  scene.add(ambientLight)
  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
  directionalLight.position.set(1, 1, 1).normalize()
  scene.add(directionalLight)

  // Kontroller
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.25
  controls.screenSpacePanning = false
  controls.minDistance = 5
  controls.maxDistance = 100

  // Model grubu
  modelGroup = new THREE.Group()
  scene.add(modelGroup)

  // Animasyon döngüsünü başlat
  animate()
}

// Animasyon döngüsü
function animate() {
  animationFrameId.value = requestAnimationFrame(animate)
  if (controls) controls.update()
  if (renderer && scene && camera) {
    renderer.render(scene, camera)
  }
}

// JSON dosyasını yükle ve modeli oluştur
async function loadEngineModel() {
  try {
    loading.value = true
    
    // Engine JSON dosyasını yükle
    const response = await fetch('/engine_converted.json')
    const engineData = await response.json()
    
    if (!engineData.success || !engineData.meshes) {
      throw new Error('Motor verisi yüklenemedi')
    }

    // Modeli render et
    await nextTick()
    renderEngineModel(engineData)
    
  } catch (error) {
    console.error('Motor yükleme hatası:', error)
  } finally {
    loading.value = false
  }
}

// Motor modelini render et
function renderEngineModel(engineData) {
  if (!scene || !camera || !renderer) {
    initThreeDScene()
    if (!scene) return
  }

  // Mevcut modelleri temizle
  clearModel()

  if (!engineData.meshes || engineData.meshes.length === 0) {
    return
  }

  const bbox = new THREE.Box3()
  let firstMesh = true
  let meshIndex = 0

  engineData.meshes.forEach(meshData => {
    if (!meshData.attributes?.position?.array || !meshData.index?.array) {
      return
    }

    // Geometri oluştur
    const geometry = new THREE.BufferGeometry()
    const positions = new Float32Array(meshData.attributes.position.array)
    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3))

    if (meshData.attributes.normal?.array) {
      const normals = new Float32Array(meshData.attributes.normal.array)
      geometry.setAttribute('normal', new THREE.BufferAttribute(normals, 3))
    } else {
      geometry.computeVertexNormals()
    }

    const indices = new (positions.length / 3 > 65535 ? Uint32Array : Uint16Array)(meshData.index.array)
    geometry.setIndex(new THREE.BufferAttribute(indices, 1))

    // Materyal oluştur - arızalı parçaları kırmızı yap
    const isFaultyComponent = meshData.name && meshData.name.toLowerCase().includes('turbine')
    const material = new THREE.MeshStandardMaterial({
      color: isFaultyComponent ? 0xff4444 : 0xcccccc, // Arızalı parçalar kırmızı
      side: THREE.DoubleSide,
      wireframe: wireframeMode.value
    })

    const mesh = new THREE.Mesh(geometry, material)
    mesh.userData = { 
      originalName: meshData.name,
      isFaulty: isFaultyComponent,
      meshIndex: meshIndex 
    }
    
    modelGroup.add(mesh)

    // Arızalı mesh'leri kaydet
    if (isFaultyComponent) {
      faultMeshes.set(meshIndex, mesh)
    }

    // Bounding box güncelle
    if (firstMesh) {
      bbox.setFromObject(mesh)
      firstMesh = false
    } else {
      bbox.union(new THREE.Box3().setFromObject(mesh))
    }
    
    meshIndex++
  })

  // Kamerayı modele uyarla
  if (!bbox.isEmpty()) {
    const center = new THREE.Vector3()
    const size = new THREE.Vector3()
    bbox.getCenter(center)
    bbox.getSize(size)

    const maxDim = Math.max(size.x, size.y, size.z)
    const fov = camera.fov * (Math.PI / 180)
    let cameraZ = Math.abs(maxDim / 2 / Math.tan(fov / 2))
    cameraZ *= (camera.aspect > 1 ? camera.aspect : 1)

    camera.position.copy(center)
    camera.position.z += cameraZ
    camera.position.y += cameraZ * 0.5
    camera.lookAt(center)

    if (controls) {
      controls.target.copy(center)
      controls.update()
    }
  }
}

// Modeli temizle
function clearModel() {
  if (modelGroup) {
    while (modelGroup.children.length > 0) {
      const object = modelGroup.children[0]
      modelGroup.remove(object)
      if (object.geometry) object.geometry.dispose()
      if (object.material) {
        if (Array.isArray(object.material)) object.material.forEach(m => m.dispose())
        else object.material.dispose()
      }
    }
  }
  faultMeshes.clear()
}

// Görünümü sıfırla
function resetView() {
  if (camera && controls) {
    camera.position.set(10, 10, 10)
    camera.lookAt(0, 0, 0)
    controls.target.set(0, 0, 0)
    controls.update()
  }
}

// Highlight'ı aç/kapat
function toggleHighlight() {
  showHighlight.value = !showHighlight.value
  
  // Arızalı mesh'lerin rengini değiştir
  faultMeshes.forEach((mesh, index) => {
    if (mesh.material) {
      mesh.material.color.setHex(showHighlight.value ? 0xff4444 : 0xcccccc)
    }
  })
}

// Wireframe modunu aç/kapat
function toggleWireframe() {
  wireframeMode.value = !wireframeMode.value
  
  if (modelGroup) {
    modelGroup.traverse((child) => {
      if (child.isMesh && child.material) {
        child.material.wireframe = wireframeMode.value
      }
    })
  }
}

// Bileşeni highlight et
function highlightComponent(componentId) {
  highlightedComponent.value = highlightedComponent.value === componentId ? null : componentId
  
  // İlgili mesh'leri highlight et (basit implementasyon)
  if (modelGroup) {
    modelGroup.traverse((child) => {
      if (child.isMesh) {
        if (child.userData.isFaulty && componentId === 1) {
          child.material.color.setHex(0xff0000) // Kırmızı
        } else if (!child.userData.isFaulty) {
          child.material.color.setHex(0x00ff00) // Yeşil
        }
      }
    })
  }
}

// Pencere boyutu değişikliği
function onWindowResize() {
  const canvas = threeCanvas.value
  if (canvas && camera && renderer) {
    const parent = canvas.parentElement
    const width = parent.clientWidth
    const height = Math.min(width * 0.75, 600)

    camera.aspect = width / height
    camera.updateProjectionMatrix()
    renderer.setSize(width, height)
  }
}

// Bileşen yüklendiğinde
onMounted(() => {
  nextTick(() => {
    initThreeDScene()
    loadEngineModel()
  })
  window.addEventListener('resize', onWindowResize)
})

// Bileşen yok edilmeden önce
onBeforeUnmount(() => {
  if (animationFrameId.value) {
    cancelAnimationFrame(animationFrameId.value)
  }
  if (renderer) {
    renderer.dispose()
  }
  if (controls) {
    controls.dispose()
  }
  window.removeEventListener('resize', onWindowResize)
})
</script>

<style scoped>
.step-viewer {
  padding: 20px;
  max-width: 1400px;
  margin: 0 auto;
}

.viewer-header {
  text-align: center;
  margin-bottom: 30px;
}

.viewer-header h2 {
  color: #2c3e50;
  margin-bottom: 10px;
}

.fault-info-panel {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 25px;
  border-radius: 15px;
  margin-bottom: 30px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.1);
}

.fault-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.fault-severity {
  padding: 8px 16px;
  border-radius: 20px;
  font-weight: bold;
  text-transform: uppercase;
  font-size: 0.9em;
}

.fault-severity.high {
  background: #e74c3c;
  color: white;
}

.fault-details {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 15px;
}

.fault-item {
  display: flex;
  justify-content: space-between;
  padding: 10px;
  background: rgba(255,255,255,0.1);
  border-radius: 8px;
}

.viewer-container {
  background: white;
  border-radius: 15px;
  padding: 20px;
  margin-bottom: 30px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
}

.viewer-controls {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
  flex-wrap: wrap;
}

.control-btn {
  padding: 10px 20px;
  border: none;
  border-radius: 8px;
  background: #3498db;
  color: white;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.3s ease;
}

.control-btn:hover {
  background: #2980b9;
  transform: translateY(-2px);
}

.canvas-container {
  position: relative;
  width: 100%;
  height: 600px;
  border-radius: 10px;
  overflow: hidden;
  background: #f8f9fa;
}

.three-canvas {
  width: 100%;
  height: 100%;
  display: block;
}

.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(255,255,255,0.9);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 10;
}

.loading-spinner {
  font-size: 2em;
  margin-bottom: 10px;
  animation: spin 2s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.component-list {
  background: white;
  border-radius: 15px;
  padding: 25px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
}

.component-list h3 {
  color: #2c3e50;
  margin-bottom: 20px;
  text-align: center;
}

.component-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 15px;
}

.component-item {
  display: flex;
  align-items: center;
  padding: 15px;
  border-radius: 10px;
  cursor: pointer;
  transition: all 0.3s ease;
  border: 2px solid transparent;
}

.component-item:hover {
  transform: translateY(-3px);
  box-shadow: 0 6px 20px rgba(0,0,0,0.15);
}

.component-item.faulty {
  background: #ffeaea;
  border-color: #e74c3c;
}

.component-item.highlighted {
  background: #e8f4fd;
  border-color: #3498db;
}

.component-icon {
  font-size: 2em;
  margin-right: 15px;
}

.component-info {
  flex: 1;
}

.component-name {
  font-weight: 600;
  color: #2c3e50;
  margin-bottom: 5px;
}

.component-status {
  padding: 4px 8px;
  border-radius: 12px;
  font-size: 0.8em;
  font-weight: 500;
  text-transform: uppercase;
}

.component-status.good {
  background: #d4edda;
  color: #155724;
}

.component-status.warning {
  background: #fff3cd;
  color: #856404;
}

.component-status.faulty {
  background: #f8d7da;
  color: #721c24;
}

@media (max-width: 768px) {
  .step-viewer {
    padding: 15px;
  }
  
  .fault-details {
    grid-template-columns: 1fr;
  }
  
  .viewer-controls {
    justify-content: center;
  }
  
  .component-grid {
    grid-template-columns: 1fr;
  }
}
</style>
