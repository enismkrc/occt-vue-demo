<template>
  <div id="app">
    <div class="app-container">
      <header class="app-header">
        <div class="header-content">
          <h1 class="app-title">✈️ Uçak Bakım Sistemi</h1>
          <p class="app-subtitle">Predictive Maintenance & 3D Model Görüntüleme</p>
        </div>
      </header>

      <nav class="module-navigation">
        <div class="nav-container">
          <button 
            @click="currentModule = 'maintenance'" 
            :class="['nav-button', { active: currentModule === 'maintenance' }]"
          >
            <span class="nav-icon">🔧</span>
            <span class="nav-text">Bakım</span>
            <span class="nav-desc">Arıza Tespiti</span>
          </button>
          
          <button 
            @click="currentModule = 'convert'" 
            :class="['nav-button', { active: currentModule === 'convert' }]"
          >
            <span class="nav-icon">🔄</span>
            <span class="nav-text">Convert</span>
            <span class="nav-desc">STEP → JSON</span>
          </button>
          
          <button 
            @click="currentModule = 'viewer'" 
            :class="['nav-button', { active: currentModule === 'viewer' }]"
          >
            <span class="nav-icon">👁️</span>
            <span class="nav-text">Viewer</span>
            <span class="nav-desc">JSON → 3D</span>
          </button>
        </div>
      </nav>

      <main class="app-main">
        <div class="module-container">
          <StepViewer v-if="currentModule === 'maintenance'" />
          <ConvertModule v-if="currentModule === 'convert'" />
          <ViewerModule v-if="currentModule === 'viewer'" />
        </div>
      </main>

      <footer class="app-footer">
        <div class="footer-content">
          <p>Built with Vue 3, Three.js, and OCCT Import JS</p>
          <div class="footer-links">
            <a href="#" class="footer-link">Documentation</a>
            <a href="#" class="footer-link">GitHub</a>
            <a href="#" class="footer-link">Support</a>
          </div>
        </div>
      </footer>
    </div>
  </div>
</template>

<script>
import { ref } from 'vue'
import StepViewer from './components/StepViewer.vue'
import ConvertModule from './components/ConvertModule.vue'
import ViewerModule from './components/ViewerModule.vue'

export default {
  name: 'App',
  components: {
    StepViewer,
    ConvertModule,
    ViewerModule
  },
  setup() {
    const currentModule = ref('maintenance')
    
    return {
      currentModule
    }
  }
}
</script>

<style>
#app {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  margin: 0;
  padding: 0;
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

body {
  margin: 0;
  padding: 0;
}

.app-container {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.app-header {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
  padding: 20px 0;
  box-shadow: 0 2px 20px rgba(0, 0, 0, 0.1);
}

.header-content {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 20px;
  text-align: center;
}

.app-title {
  font-size: 2.5em;
  margin: 0 0 10px 0;
  background: linear-gradient(135deg, #667eea, #764ba2);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  font-weight: 700;
}

.app-subtitle {
  font-size: 1.2em;
  color: #7f8c8d;
  margin: 0;
  font-weight: 300;
}

.module-navigation {
  background: rgba(255, 255, 255, 0.9);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
  padding: 15px 0;
}

.nav-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 20px;
  display: flex;
  justify-content: center;
  gap: 20px;
}

.nav-button {
  background: white;
  border: 2px solid #e0e0e0;
  border-radius: 12px;
  padding: 20px 30px;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  min-width: 150px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
}

.nav-button:hover {
  transform: translateY(-3px);
  box-shadow: 0 5px 20px rgba(0, 0, 0, 0.15);
  border-color: #3498db;
}

.nav-button.active {
  background: linear-gradient(135deg, #3498db, #2980b9);
  color: white;
  border-color: #2980b9;
  transform: translateY(-3px);
  box-shadow: 0 5px 20px rgba(52, 152, 219, 0.3);
}

.nav-icon {
  font-size: 2em;
}

.nav-text {
  font-size: 1.2em;
  font-weight: bold;
}

.nav-desc {
  font-size: 0.9em;
  opacity: 0.8;
}

.app-main {
  flex: 1;
  padding: 30px 0;
}

.module-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 20px;
}

.app-footer {
  background: rgba(255, 255, 255, 0.9);
  backdrop-filter: blur(10px);
  border-top: 1px solid rgba(255, 255, 255, 0.2);
  padding: 20px 0;
  margin-top: auto;
}

.footer-content {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 20px;
  text-align: center;
  color: #7f8c8d;
}

.footer-links {
  margin-top: 10px;
  display: flex;
  justify-content: center;
  gap: 20px;
}

.footer-link {
  color: #3498db;
  text-decoration: none;
  transition: color 0.3s ease;
}

.footer-link:hover {
  color: #2980b9;
}

@media (max-width: 768px) {
  .app-title {
    font-size: 2em;
  }
  
  .nav-container {
    flex-direction: column;
    align-items: center;
  }
  
  .nav-button {
    min-width: 200px;
  }
  
  .footer-links {
    flex-direction: column;
    gap: 10px;
  }
}
</style>