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
        <button @click="showOnlyFaulty" class="control-btn">
          🔍 Sadece Arızalı Parçalar
        </button>
        <button @click="showAllParts" class="control-btn">
          👁️ Tüm Parçaları Göster
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
            'highlighted': highlightedComponent === component.id,
            'hidden': !component.visible
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
          <div class="component-controls">
            <button 
              @click.stop="toggleComponentVisibility(component.id)"
              class="visibility-btn"
              :title="component.visible ? 'Parçayı Gizle' : 'Parçayı Göster'"
            >
              {{ component.visible ? '👁️' : '🙈' }}
            </button>
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
const allMeshes = new Map() // Tüm mesh'leri saklamak için

// Three.js nesneleri
let scene = null
let camera = null
let renderer = null
let controls = null
let modelGroup = null
let faultMeshes = new Map() // Arızalı mesh'leri saklamak için

// Arıza verisi (gerçek uygulamada API'den gelecek)
const faultData = ref({
  componentName: "11 petrol turbine",
  faultType: "Metal Fatigue",
  detectionDate: "2024-01-15",
  remainingLife: "72 saat",
  severity: "high",
  severityText: "Yüksek Öncelik",
  recommendedAction: "Acil Değişim Gerekli"
})

// Bileşen listesi - Gerçek motor parçaları
const components = ref([
  { id: 1, name: "11 petrol turbine", icon: "🔄", isFaulty: true, status: "faulty", statusText: "Arızalı", visible: true },
  { id: 2, name: "4 central mounting shaft", icon: "🔧", isFaulty: false, status: "good", statusText: "Sağlam", visible: true },
  { id: 3, name: "2 prime turbine bunshing", icon: "🌀", isFaulty: false, status: "good", statusText: "Sağlam", visible: true },
  { id: 4, name: "1 nose cone", icon: "🔺", isFaulty: false, status: "warning", statusText: "İzle", visible: true },
  { id: 5, name: "5 Rear mounting shaft", icon: "🔧", isFaulty: false, status: "good", statusText: "Sağlam", visible: true },
  { id: 6, name: "Jet fan1", icon: "💨", isFaulty: false, status: "good", statusText: "Sağlam", visible: true },
  { id: 7, name: "10 air turbine 1", icon: "🌀", isFaulty: false, status: "good", statusText: "Sağlam", visible: true },
  { id: 8, name: "10 air turbine 2", icon: "🌀", isFaulty: false, status: "good", statusText: "Sağlam", visible: true },
  { id: 9, name: "10 air turbine 3", icon: "🌀", isFaulty: false, status: "good", statusText: "Sağlam", visible: true },
  { id: 10, name: "10 air turbine 4", icon: "🌀", isFaulty: false, status: "good", statusText: "Sağlam", visible: true },
  { id: 11, name: "10 air turbine 5", icon: "🌀", isFaulty: false, status: "good", statusText: "Sağlam", visible: true },
  { id: 12, name: "12 petrol turbine 2", icon: "🔄", isFaulty: false, status: "good", statusText: "Sağlam", visible: true },
  { id: 13, name: "13. petrol turbine 3", icon: "🔄", isFaulty: false, status: "good", statusText: "Sağlam", visible: true },
  { id: 14, name: "14 petrol turbine 4", icon: "🔄", isFaulty: false, status: "good", statusText: "Sağlam", visible: true },
  { id: 15, name: "21  turbine 5", icon: "🔄", isFaulty: false, status: "good", statusText: "Sağlam", visible: true }
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
  controls.maxDistance = 500

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
    const isFaultyComponent = meshData.name && meshData.name.toLowerCase().includes('11 petrol turbine')
    const material = new THREE.MeshStandardMaterial({
      color: isFaultyComponent ? 0xff4444 : 0xcccccc, // Arızalı parçalar kırmızı
      side: THREE.DoubleSide,
      wireframe: wireframeMode.value
    })

    const mesh = new THREE.Mesh(geometry, material)
    
    // Gerçek parça isimlerine göre component ID atama
    let componentId = 1 // Varsayılan
    if (meshData.name) {
      const component = components.value.find(c => c.name === meshData.name)
      componentId = component ? component.id : 1
    } else {
      // Mesh ismi boşsa, mesh index'ine göre component ID atama
      // Jet fan1 mesh'leri: 9-27 (19 adet)
      if (meshIndex >= 9 && meshIndex <= 27) {
        componentId = 6 // Jet fan1 component ID
      }
    }
    
    mesh.userData = { 
      originalName: meshData.name,
      isFaulty: isFaultyComponent,
      meshIndex: meshIndex,
      componentId: componentId
    }
    
    modelGroup.add(mesh)

    // Tüm mesh'leri kaydet
    allMeshes.set(meshIndex, mesh)
    
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
    // Increase distance multiplier significantly to fit model completely in screen
    //cameraZ *= 5.0

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
  allMeshes.clear()
}

// Görünümü sıfırla
function resetView() {
  if (!modelGroup || modelGroup.children.length === 0) {
    if (camera && controls) {
      camera.position.set(10, 10, 10)
      camera.lookAt(0, 0, 0)
      controls.target.set(0, 0, 0)
      controls.update()
    }
    return
  }

  // Modelin bounding box'ını hesapla
  const bbox = new THREE.Box3()
  modelGroup.traverse((child) => {
    if (child.isMesh) {
      bbox.union(new THREE.Box3().setFromObject(child))
    }
  })

  if (!bbox.isEmpty()) {
    const center = new THREE.Vector3()
    const size = new THREE.Vector3()
    bbox.getCenter(center)
    bbox.getSize(size)

    const maxDim = Math.max(size.x, size.y, size.z)
    const fov = camera.fov * (Math.PI / 180)
    let cameraZ = Math.abs(maxDim / 2 / Math.tan(fov / 2))
    cameraZ *= (camera.aspect > 1 ? camera.aspect : 1)
    // Modeli ekrana sığdırmak için yeterli mesafe
    cameraZ *= 2.0

    camera.position.copy(center)
    camera.position.z += cameraZ
    camera.position.y += cameraZ * 0.5
    camera.lookAt(center)

    if (controls) {
      controls.target.copy(center)
      controls.update()
    }
  } else {
    if (camera && controls) {
      camera.position.set(10, 10, 10)
      camera.lookAt(0, 0, 0)
      controls.target.set(0, 0, 0)
      controls.update()
    }
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
  
  // İlgili mesh'leri highlight et
  if (modelGroup) {
    modelGroup.traverse((child) => {
      if (child.isMesh) {
        if (highlightedComponent.value === componentId && child.userData.componentId === componentId) {
          // Seçili bileşeni vurgula (sadece arızalı değilse)
          if (!child.userData.isFaulty) {
            child.material.color.setHex(0x00ff00) // Yeşil
          }
        } else if (child.userData.isFaulty) {
          // Arızalı bileşeni her zaman kırmızı göster
          child.material.color.setHex(0xff0000) // Kırmızı
        } else {
          // Diğer bileşenleri normal renkte göster
          child.material.color.setHex(0xcccccc) // Gri
        }
      }
    })
  }
}

// Bileşen görünürlüğünü değiştir
function toggleComponentVisibility(componentId) {
  const component = components.value.find(c => c.id === componentId)
  if (component) {
    component.visible = !component.visible
    
    // İlgili mesh'lerin görünürlüğünü değiştir
    allMeshes.forEach((mesh, index) => {
      if (mesh.userData.componentId === componentId) {
        mesh.visible = component.visible
      }
    })
  }
}

// Sadece arızalı parçaları göster
function showOnlyFaulty() {
  // Tüm bileşenleri gizle
  components.value.forEach(component => {
    component.visible = component.isFaulty
  })
  
  // Tüm mesh'leri gizle, sadece arızalı component'lere ait olanları göster
  allMeshes.forEach((mesh, index) => {
    const component = components.value.find(c => c.id === mesh.userData.componentId)
    mesh.visible = component ? component.isFaulty : false
  })
}

// Tüm parçaları göster
function showAllParts() {
  // Tüm bileşenleri göster
  components.value.forEach(component => {
    component.visible = true
  })
  
  // Tüm mesh'leri göster
  allMeshes.forEach((mesh, index) => {
    mesh.visible = true
  })
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

.component-item.hidden {
  opacity: 0.5;
  background: #f8f9fa;
  border-color: #dee2e6;
}

.component-icon {
  font-size: 2em;
  margin-right: 15px;
}

.component-info {
  flex: 1;
}

.component-controls {
  display: flex;
  align-items: center;
  margin-left: 10px;
}

.visibility-btn {
  background: none;
  border: none;
  font-size: 1.2em;
  cursor: pointer;
  padding: 5px;
  border-radius: 4px;
  transition: all 0.3s ease;
}

.visibility-btn:hover {
  background: rgba(0,0,0,0.1);
  transform: scale(1.1);
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
