<script setup>
import { ref, onMounted, watch, nextTick } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DecalGeometry } from 'three/examples/jsm/geometries/DecalGeometry.js'
import {
  Paintbrush,
  Type,
  Image as ImageIcon,
  Download,
  RotateCcw,
  Upload,
  Move,
  Check,
  Maximize2,
} from 'lucide-vue-next'

// --- State Variables ---
const activeTab = ref('color') // color, design, text
const isLoading = ref(true)
const loadingProgress = ref(0)
const errorMessage = ref('')

// Configurable parameters
const shirtColor = ref('#3b82f6') // default blue
const presetColors = [
  { name: 'Noir Black', hex: '#121214' },
  { name: 'Cotton White', hex: '#f3f4f6' },
  { name: 'Crimson Red', hex: '#ef4444' },
  { name: 'Ocean Blue', hex: '#3b82f6' },
  { name: 'Forest Green', hex: '#10b981' },
  { name: 'Sage Green', hex: '#86efac' },
  { name: 'Mustard Yellow', hex: '#f59e0b' },
  { name: 'Lavender Purple', hex: '#a855f7' },
]

// Decal Design State
const currentDesign = ref('/designs/retro_sunset.svg')
const presetDesigns = [
  { name: 'Retro Sunset', url: '/designs/retro_sunset.svg' },
  { name: 'Cyber Grid', url: '/designs/cyber_grid.svg' },
  { name: 'Minimalist Leaf', url: '/designs/minimalist_leaf.svg' },
  { name: 'Streetwear Badge', url: '/designs/streetwear_badge.svg' },
]
const designScale = ref(0.15) // standard scale in canvas space
const designX = ref(0.5) // centered
const designY = ref(0.37) // chest area on the enlarged decal canvas
const designRotation = ref(0) // in degrees

// Custom Text State
const customText = ref('CREATIVE STUDIO')
const textFont = ref('Inter')
const textFonts = ['Inter', 'Bungee', 'Pacifico', 'Playfair Display', 'JetBrains Mono']
const textSize = ref(20)
const textColor = ref('#ffffff')
const textX = ref(0.5) // center alignment
const textY = ref(0.45) // chest area on the enlarged decal canvas, below design
const textLetterSpacing = ref(4)
const textShape = ref('straight') // straight, circle, wave, heart, square

// 3D Scene Refs
const threeContainer = ref(null)
let scene, camera, renderer, controls, shirtMesh, shirtModel
let canvasTexture, offscreenCanvas, offscreenCtx
let decalGeometry, decalMesh, decalMaterial
let presetImages = {} // Cache loaded images

// --- Texture Drawing System (2D Offscreen Canvas) ---
const initOffscreenCanvas = () => {
  offscreenCanvas = document.createElement('canvas')
  offscreenCanvas.width = 1024
  offscreenCanvas.height = 1024
  offscreenCtx = offscreenCanvas.getContext('2d')
}

// Draw the texture based on state (Transparent background for projection Decal)
const drawTextureCanvas = () => {
  if (!offscreenCtx) return

  const ctx = offscreenCtx
  const w = offscreenCanvas.width
  const h = offscreenCanvas.height

  // Clear canvas (transparency) so only logo & text project
  ctx.clearRect(0, 0, w, h)

  // Render decal graphic design
  const renderDesign = () => {
    if (!currentDesign.value) return drawText()

    let img = presetImages[currentDesign.value]
    if (!img) {
      img = new Image()
      img.src = currentDesign.value
      img.onload = () => {
        presetImages[currentDesign.value] = img
        drawTextureCanvas() // Redraw once loaded
      }
      return
    }

    ctx.save()

    // Position parameters (0 to 1 mapped to canvas coordinates)
    const posX = designX.value * w
    const posY = designY.value * h

    // Calculate size maintaining aspect ratio
    const maxDimension = w * designScale.value
    let destW = maxDimension
    let destH = maxDimension
    if (img.width && img.height) {
      if (img.width > img.height) {
        destH = maxDimension * (img.height / img.width)
      } else {
        destW = maxDimension * (img.width / img.height)
      }
    }

    // Transformations
    ctx.translate(posX, posY)
    ctx.rotate((designRotation.value * Math.PI) / 180)
    ctx.drawImage(img, -destW / 2, -destH / 2, destW, destH)

    ctx.restore()
    drawText()
  }

  // Render customized text
  const drawText = () => {
    if (!customText.value.trim()) {
      updateTexture()
      return
    }

    ctx.save()

    const posX = textX.value * w
    const posY = textY.value * h

    ctx.translate(posX, posY)

    // Setup font styling
    ctx.fillStyle = textColor.value
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.font = `bold ${textSize.value}px "${textFont.value}"`

    const textString = customText.value
    const chars = textString.split('')
    const spacing = textLetterSpacing.value

    // Measure character widths
    const charWidths = chars.map((c) => ctx.measureText(c).width)
    const totalSpacing = (chars.length - 1) * spacing
    const sumWidths = charWidths.reduce((a, b) => a + b, 0)
    const totalLength = sumWidths + totalSpacing

    const shape = textShape.value

    if (shape === 'straight') {
      let currentX = -totalLength / 2
      for (let i = 0; i < chars.length; i++) {
        ctx.fillText(chars[i], currentX + charWidths[i] / 2, 0)
        currentX += charWidths[i] + spacing
      }
    } else {
      // Draw along shape path
      for (let i = 0; i < chars.length; i++) {
        // Find normalized position along path
        let t = 0.5
        if (chars.length > 1) {
          if (shape === 'heart' || shape === 'square') {
            // For closed shapes, distribute along [0, 1) to form the full shape
            t = i / chars.length
          } else {
            // For open shapes (circle, wave), map based on cumulative length to keep letter spacing consistent
            let accWidth = 0
            for (let j = 0; j < i; j++) {
              accWidth += charWidths[j] + spacing
            }
            accWidth += charWidths[i] / 2
            t = accWidth / totalLength
          }
        }

        let px = 0
        let py = 0
        let angle = 0

        if (shape === 'circle') {
          // Circular arch
          const R = Math.max(120, totalLength / Math.PI)
          const thetaSweep = totalLength / R
          const theta = -Math.PI / 2 - thetaSweep / 2 + t * thetaSweep
          px = R * Math.cos(theta)
          py = R * Math.sin(theta) + R
          angle = theta + Math.PI / 2
        } else if (shape === 'wave') {
          // Sine wave
          px = (t - 0.5) * totalLength
          const waveHeight = 25
          const waveFreq = 1.5
          py = waveHeight * Math.sin(t * Math.PI * 2 * waveFreq)
          
          const dt = 0.01
          const nextX = (t + dt - 0.5) * totalLength
          const nextY = waveHeight * Math.sin((t + dt) * Math.PI * 2 * waveFreq)
          angle = Math.atan2(nextY - py, nextX - px)
        } else if (shape === 'heart') {
          // Heart shape
          const getHeartPoint = (tVal) => {
            const theta = -Math.PI + tVal * 2 * Math.PI
            const x_raw = 16 * Math.pow(Math.sin(theta), 3)
            const y_raw = 13 * Math.cos(theta) - 5 * Math.cos(2 * theta) - 2 * Math.cos(3 * theta) - Math.cos(4 * theta)
            const scale = Math.max(6, textSize.value * 0.4)
            return {
              x: x_raw * scale,
              y: -y_raw * scale
            }
          }
          const p1 = getHeartPoint(t)
          const p2 = getHeartPoint(t + 0.005)
          px = p1.x
          py = p1.y
          angle = Math.atan2(p2.y - p1.y, p2.x - p1.x) + Math.PI / 2
        } else if (shape === 'square') {
          // Square shape
          const side = Math.max(120, totalLength / 4)
          const getSquarePoint = (tVal) => {
            let x = 0
            let y = 0
            let ang = 0
            if (tVal <= 0.25) {
              const u = tVal / 0.25
              x = -side / 2 + u * side
              y = -side / 2
              ang = 0
            } else if (tVal <= 0.5) {
              const u = (tVal - 0.25) / 0.25
              x = side / 2
              y = -side / 2 + u * side
              ang = Math.PI / 2
            } else if (tVal <= 0.75) {
              const u = (tVal - 0.5) / 0.25
              x = side / 2 - u * side
              y = side / 2
              ang = Math.PI
            } else {
              const u = (tVal - 0.75) / 0.25
              x = -side / 2
              y = side / 2 - u * side
              ang = -Math.PI / 2
            }
            return { x, y, angle: ang }
          }
          const p = getSquarePoint(t)
          px = p.x
          py = p.y
          angle = p.angle
        }

        ctx.save()
        ctx.translate(px, py)
        ctx.rotate(angle)
        ctx.fillText(chars[i], 0, 0)
        ctx.restore()
      }
    }

    ctx.restore()
    updateTexture()
  }

  const updateTexture = () => {
    if (canvasTexture) {
      canvasTexture.needsUpdate = true
    }
  }

  renderDesign()
}

// --- Create/Rebuild Decal Mesh ---
const createDecalMesh = () => {
  if (!shirtMesh || !shirtModel) return

  // Remove old decal if it exists
  if (decalMesh) {
    shirtModel.remove(decalMesh)
    decalMesh.geometry.dispose()
  }

  // Create Decal Geometry projected onto the front chest of the shirt model
  // Position coordinates matching front-torso center in local model coordinates
  const position = new THREE.Vector3(0, -0.05, 0.18)
  const orientation = new THREE.Euler(0, 0, 0)
  // Square projection box mapping square canvas (no stretching)
  const size = new THREE.Vector3(1.0, 1.0, 1.0)

  decalGeometry = new DecalGeometry(shirtMesh, position, orientation, size)

  decalMaterial = new THREE.MeshPhysicalMaterial({
    map: canvasTexture,
    transparent: true,
    depthWrite: false, // Prevents z-fighting artifacts
    polygonOffset: true,
    polygonOffsetFactor: -4,
    roughness: 0.7,
    metalness: 0.1,
    sheen: 0.8,
    sheenRoughness: 0.6,
    sheenColor: new THREE.Color('#ffffff'),
  })

  decalMesh = new THREE.Mesh(decalGeometry, decalMaterial)
  decalMesh.renderOrder = 2 // Render on top of the base shirt mesh

  shirtModel.add(decalMesh)
}

// --- 3D Scene Initialization ---
const initThreeScene = () => {
  const container = threeContainer.value
  if (!container) return

  // Scene Setup
  scene = new THREE.Scene()
  scene.background = new THREE.Color('#0a0a0c')
  scene.fog = new THREE.FogExp2('#0a0a0c', 0.12)

  // Camera Setup
  camera = new THREE.PerspectiveCamera(40, container.clientWidth / container.clientHeight, 0.1, 100)
  camera.position.set(0, 0, 1.5)

  // Renderer Setup
  renderer = new THREE.WebGLRenderer({ antialias: true, preserveDrawingBuffer: true })
  renderer.setSize(container.clientWidth, container.clientHeight)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.05
  container.appendChild(renderer.domElement)

  // OrbitControls
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05
  controls.maxDistance = 18
  controls.minDistance = 1
  controls.enablePan = true

  // Prevent right-click context menu on the canvas to allow right-click panning
  renderer.domElement.addEventListener('contextmenu', (e) => e.preventDefault())

  // Lights Setup
  const ambientLight = new THREE.AmbientLight('#ffffff', 0.6)
  scene.add(ambientLight)

  const keyLight = new THREE.DirectionalLight('#ffffff', 1.2)
  keyLight.position.set(5, 5, 8)
  keyLight.castShadow = true
  keyLight.shadow.mapSize.width = 2048
  keyLight.shadow.mapSize.height = 2048
  keyLight.shadow.bias = -0.0001
  scene.add(keyLight)

  const fillLight = new THREE.DirectionalLight('#ffffff', 0.5)
  fillLight.position.set(-5, 3, 5)
  scene.add(fillLight)

  const rimLight = new THREE.DirectionalLight('#ffffff', 0.8)
  rimLight.position.set(0, 8, -6)
  scene.add(rimLight)

  // Canvas Texture creation (standard UV coordinate mappings for decal geometry)
  canvasTexture = new THREE.CanvasTexture(offscreenCanvas)
  canvasTexture.colorSpace = THREE.SRGBColorSpace
  canvasTexture.anisotropy = renderer.capabilities.getMaxAnisotropy()

  // Load 3D Shirt Model
  const loader = new GLTFLoader()
  loader.load(
    '/models/shirt_baked.glb',
    (gltf) => {
      shirtModel = gltf.scene

      const box = new THREE.Box3().setFromObject(shirtModel)
      const center = box.getCenter(new THREE.Vector3())
      shirtModel.position.sub(center)
      shirtModel.position.y = 0.2 // Shift model slightly higher up in viewport

      shirtModel.traverse((child) => {
        if (child.isMesh) {
          shirtMesh = child
          child.castShadow = true
          child.receiveShadow = true

          // Re-create high quality physical material for shirt body (solid color)
          const material = new THREE.MeshPhysicalMaterial({
            color: new THREE.Color(shirtColor.value),
            roughness: 0.75,
            metalness: 0.1,
            clearcoat: 0.0,
            sheen: 0.7,
            sheenRoughness: 0.6,
            sheenColor: new THREE.Color('#ffffff'),
          })

          // Maintain original normals/folds maps
          if (child.material) {
            if (child.material.normalMap) {
              material.normalMap = child.material.normalMap
              material.normalScale = new THREE.Vector2(0.5, 0.5)
            }
            if (child.material.aoMap) {
              material.aoMap = child.material.aoMap
              material.aoMapIntensity = 0.55
            }
          }

          child.material = material
        }
      })

      scene.add(shirtModel)
      createDecalMesh()

      isLoading.value = false
      drawTextureCanvas()
    },
    (xhr) => {
      if (xhr.total > 0) {
        loadingProgress.value = Math.round((xhr.loaded / xhr.total) * 100)
      }
    },
    (err) => {
      console.error(err)
      errorMessage.value = 'Failed to load T-Shirt 3D model. Please reload.'
      isLoading.value = false
    },
  )

  // Animation Loop
  const animate = () => {
    requestAnimationFrame(animate)
    if (controls) controls.update()
    if (renderer && scene && camera) renderer.render(scene, camera)
  }
  animate()

  const handleResize = () => {
    if (!container || !camera || !renderer) return
    camera.aspect = container.clientWidth / container.clientHeight
    camera.updateProjectionMatrix()
    renderer.setSize(container.clientWidth, container.clientHeight)
  }
  window.addEventListener('resize', handleResize)
}

// Watchers
watch(shirtColor, (newVal) => {
  if (shirtMesh && shirtMesh.material) {
    shirtMesh.material.color.set(newVal)
  }
})

watch(
  [
    currentDesign,
    designScale,
    designX,
    designY,
    designRotation,
    customText,
    textFont,
    textSize,
    textColor,
    textX,
    textY,
    textLetterSpacing,
    textShape,
  ],
  () => {
    drawTextureCanvas()
  },
)

const resetView = () => {
  if (controls && camera) {
    controls.reset()
    camera.position.set(0, 0, 2.5)
  }
}

const handleImageUpload = (event) => {
  const file = event.target.files[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = (e) => {
    currentDesign.value = e.target.result
  }
  reader.readAsDataURL(file)
}

// Export 3D Mockup Image
const exportMockup = () => {
  if (!renderer || !scene || !camera) return
  renderer.render(scene, camera)
  const dataURL = renderer.domElement.toDataURL('image/png')
  const link = document.createElement('a')
  link.download = `3d-shirt-mockup-${Date.now()}.png`
  link.href = dataURL
  link.click()
}

// Export Flat Print Design Image
const exportPrintDesign = () => {
  if (!offscreenCanvas) return
  const dataURL = offscreenCanvas.toDataURL('image/png')
  const link = document.createElement('a')
  link.download = `flat-print-template-${Date.now()}.png`
  link.href = dataURL
  link.click()
}

onMounted(() => {
  initOffscreenCanvas()
  initThreeScene()
})
</script>

<template>
  <div class="flex flex-col h-screen w-screen bg-bg-primary overflow-hidden font-sans relative">
    <!-- Wavy background accents -->
    <div
      class="absolute inset-0 bg-radial-[at_0%_0%] from-indigo-500/8 via-transparent to-transparent pointer-events-none z-0"
    ></div>
    <div
      class="absolute inset-0 bg-radial-[at_100%_100%] from-pink-500/6 via-transparent to-transparent pointer-events-none z-0"
    ></div>

    <!-- Header -->
    <header class="glass-panel h-[60px] md:h-[70px] flex items-center justify-between px-4 md:px-8 my-2 md:my-4 mx-3 md:mx-5 z-10">
      <div class="flex items-center gap-3">
        <span
          class="w-2 h-2 md:w-2.5 md:h-2.5 rounded-full bg-emerald-500 shadow-[0_0_10px_#10b981] animate-pulse"
        ></span>
        <h1 class="text-lg md:text-xl font-black tracking-widest text-white">
          STUDIO <span class="text-accent-indigo">3D</span>
        </h1>
      </div>
      <p class="hidden md:block text-xs text-text-muted tracking-[1.5px] uppercase">
        Interactive T-Shirt Configurator
      </p>

      <div>
        <button
          class="flex items-center gap-2 bg-white/6 hover:bg-white/10 border border-border-subtle hover:border-white/20 text-white font-semibold py-1.5 md:py-2.5 px-3 md:px-4 rounded-lg text-xs md:text-sm cursor-pointer transition-all duration-300"
          @click="resetView"
          title="Reset View"
        >
          <RotateCcw :size="14" class="md:hidden" />
          <RotateCcw :size="16" class="hidden md:inline" />
          <span class="hidden sm:inline">Reset Camera</span>
        </button>
      </div>
    </header>

    <div class="flex flex-col md:flex-row flex-1 p-3 md:p-4 pb-3 md:pb-5 px-3 md:px-5 gap-3 md:gap-5 overflow-hidden z-5">
      <!-- 3D Viewport (Shown on top on mobile) -->
      <main
        class="w-full h-[45vh] md:h-auto md:flex-1 relative rounded-2xl overflow-hidden border border-border-subtle bg-black/10 order-1 md:order-2"
      >
        <div ref="threeContainer" class="w-full h-full">
          <!-- Floating Guide Overlay -->
          <div
            class="absolute bottom-4 md:bottom-6 left-1/2 -translate-x-1/2 flex gap-3 md:gap-4 p-2 px-4 md:p-2.5 md:px-5 rounded-full text-[10px] md:text-xs text-text-muted pointer-events-none glass-panel whitespace-nowrap"
            v-if="!isLoading"
          >
            <div class="flex items-center gap-1.5 md:gap-2">
              <span
                class="w-2.5 h-4 md:w-3 md:h-5 border-2 border-text-muted rounded-md relative after:content-[''] after:absolute after:top-0.5 after:left-1/2 after:-translate-x-1/2 after:w-0.5 after:h-0.5 md:after:h-1 after:bg-text-muted after:rounded-full"
              ></span>
              <span>Left-click drag to rotate</span>
            </div>
            <div class="flex items-center gap-1.5 md:gap-2">
              <span
                class="w-2.5 h-4 md:w-3 md:h-5 border-2 border-text-muted rounded-md relative after:content-[''] after:absolute after:top-0.5 after:right-0.5 after:w-0.5 after:h-1 md:after:h-1.5 after:bg-accent-indigo after:rounded-tr-sm"
              ></span>
              <span>Right-click drag to move</span>
            </div>
            <div class="flex items-center gap-1.5 md:gap-2">
              <span
                class="w-3.5 h-3.5 md:w-4 md:h-4 border-2 border-text-muted rounded-full relative after:content-['↕'] after:absolute after:top-1/2 after:left-1/2 after:-translate-x-1/2 after:-translate-y-1/2 after:text-[8px] md:after:text-[10px] after:leading-none"
              ></span>
              <span>Scroll to zoom</span>
            </div>
          </div>
        </div>

        <!-- Loading Overlay -->
        <div
          class="absolute inset-0 flex items-center justify-center bg-bg-primary/85 backdrop-blur-md z-[100]"
          v-if="isLoading"
        >
          <div
            class="p-6 md:p-10 max-w-[340px] md:max-w-[380px] w-11/12 text-center flex flex-col items-center gap-3 md:gap-4 glass-panel"
          >
            <div
              class="w-10 h-10 md:w-12 md:h-12 border-3 border-white/5 border-t-accent-indigo rounded-full animate-spin"
            ></div>
            <h3 class="text-white font-bold text-base md:text-lg">Loading Studio Asset</h3>
            <p class="text-xs md:text-sm text-text-muted">Initializing high-resolution 3D materials...</p>
            <div class="w-full h-1 bg-white/8 rounded-full overflow-hidden mt-1 md:mt-2">
              <div
                class="h-full bg-accent-indigo transition-all duration-100 shadow-[0_0_10px_rgba(99,102,241,0.5)]"
                :style="{ width: loadingProgress + '%' }"
              ></div>
            </div>
            <span class="text-[10px] md:text-xs text-text-muted font-mono">{{ loadingProgress }}%</span>
          </div>
        </div>

        <!-- Error Message Overlay -->
        <div
          class="absolute inset-0 flex items-center justify-center bg-bg-primary/85 backdrop-blur-md z-[100]"
          v-if="errorMessage"
        >
          <div
            class="p-6 md:p-10 max-w-[340px] md:max-w-[380px] w-11/12 text-center flex flex-col items-center gap-3 md:gap-4 glass-panel"
          >
            <h3 class="text-red-500 font-bold text-base md:text-lg">Something went wrong</h3>
            <p class="text-xs md:text-sm text-text-muted">{{ errorMessage }}</p>
          </div>
        </div>
      </main>

      <!-- Sidebar Controls -->
      <aside class="glass-panel w-full md:w-[420px] flex flex-col overflow-hidden flex-1 md:flex-initial min-h-0 md:h-full order-2 md:order-1">
        <!-- Navigation Tabs -->
        <nav class="flex border-b border-border-subtle bg-white/[0.01] shrink-0">
          <button
            :class="[
              'flex-1 flex flex-col items-center gap-1 bg-none border-b-2 border-transparent text-text-muted py-2.5 md:py-3.5 px-1 text-[10px] md:text-[11px] font-semibold cursor-pointer uppercase tracking-[0.5px] transition-all duration-300 hover:text-white hover:bg-white/[0.02]',
              {
                'text-accent-indigo border-accent-indigo bg-accent-indigo/4': activeTab === 'color',
              },
            ]"
            @click="activeTab = 'color'"
          >
            <Paintbrush :size="16" class="md:hidden" />
            <Paintbrush :size="18" class="hidden md:block" />
            Base Color
          </button>
          <button
            :class="[
              'flex-1 flex flex-col items-center gap-1 bg-none border-b-2 border-transparent text-text-muted py-2.5 md:py-3.5 px-1 text-[10px] md:text-[11px] font-semibold cursor-pointer uppercase tracking-[0.5px] transition-all duration-300 hover:text-white hover:bg-white/[0.02]',
              {
                'text-accent-indigo border-accent-indigo bg-accent-indigo/4':
                  activeTab === 'design',
              },
            ]"
            @click="activeTab = 'design'"
          >
            <ImageIcon :size="16" class="md:hidden" />
            <ImageIcon :size="18" class="hidden md:block" />
            Decal Design
          </button>
          <button
            :class="[
              'flex-1 flex flex-col items-center gap-1 bg-none border-b-2 border-transparent text-text-muted py-2.5 md:py-3.5 px-1 text-[10px] md:text-[11px] font-semibold cursor-pointer uppercase tracking-[0.5px] transition-all duration-300 hover:text-white hover:bg-white/[0.02]',
              {
                'text-accent-indigo border-accent-indigo bg-accent-indigo/4': activeTab === 'text',
              },
            ]"
            @click="activeTab = 'text'"
          >
            <Type :size="16" class="md:hidden" />
            <Type :size="18" class="hidden md:block" />
            Custom Text
          </button>
        </nav>

        <!-- Tab Content -->
        <div class="flex-1 overflow-y-auto p-4 md:p-6">
          <!-- T-Shirt Color Tab -->
          <div v-if="activeTab === 'color'" class="animate-fade-in">
            <h2 class="text-base md:text-lg font-bold mb-1">Shirt Base Color</h2>
            <p class="text-[11px] md:text-xs text-text-muted mb-4 md:mb-6 leading-relaxed">
              Select a preset canvas tone or choose a custom aesthetic hue.
            </p>

            <div class="grid grid-cols-4 gap-2 md:gap-3 mb-5 md:mb-7">
              <button
                v-for="color in presetColors"
                :key="color.hex"
                class="h-10 md:h-13 rounded-xl border border-border-subtle cursor-pointer relative transition-all duration-300 hover:scale-[1.06] hover:shadow-[0_4px_12px_rgba(0,0,0,0.4)]"
                :style="{ backgroundColor: color.hex }"
                :title="color.name"
                @click="shirtColor = color.hex"
              >
                <span
                  class="absolute inset-0 flex items-center justify-center"
                  v-if="shirtColor.toLowerCase() === color.hex.toLowerCase()"
                >
                  <Check
                    :size="12"
                    :stroke-width="3"
                    :class="color.hex === '#f3f4f6' ? 'text-[#121214]' : 'text-white'"
                    class="md:hidden"
                  />
                  <Check
                    :size="14"
                    :stroke-width="3"
                    :class="color.hex === '#f3f4f6' ? 'text-[#121214]' : 'text-white'"
                    class="hidden md:inline-block"
                  />
                </span>
              </button>
            </div>

            <div class="pt-4 border-t border-border-subtle">
              <label
                class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider mb-2.5 block"
                >Custom Hue</label
              >
              <div class="flex gap-3 items-center">
                <div
                  class="w-9 h-9 md:w-11 md:h-11 rounded-lg border border-border-subtle relative overflow-hidden cursor-pointer"
                  :style="{ backgroundColor: shirtColor }"
                >
                  <input
                    type="color"
                    v-model="shirtColor"
                    class="absolute -top-1 -left-1 w-12 h-12 md:w-15 md:h-15 border-0 p-0 cursor-pointer opacity-0"
                  />
                </div>
                <input
                  type="text"
                  v-model="shirtColor"
                  class="flex-1 p-2 md:p-2.5 px-3 md:px-3.5 bg-white/4 border border-border-subtle rounded-lg text-white font-sans text-xs md:text-sm outline-none transition-all duration-300 focus:border-accent-indigo focus:shadow-[0_0_0_3px_rgba(99,102,241,0.15)]"
                />
              </div>
            </div>
          </div>

          <!-- Graphic Decal Tab -->
          <div v-if="activeTab === 'design'" class="animate-fade-in">
            <h2 class="text-base md:text-lg font-bold mb-1">Select Graphic Decal</h2>
            <p class="text-[11px] md:text-xs text-text-muted mb-4 md:mb-6 leading-relaxed">
              Choose from high-fidelity vector decals or upload your own transparent graphic.
            </p>

            <div class="grid grid-cols-4 md:grid-cols-2 gap-2 md:gap-3 mb-4 md:mb-5">
              <button
                v-for="design in presetDesigns"
                :key="design.url"
                :class="[
                  'bg-white/[0.02] border border-border-subtle rounded-xl p-1.5 md:p-2.5 cursor-pointer transition-all duration-300 flex flex-col items-center gap-1.5 md:gap-2 hover:bg-white/[0.04] hover:border-white/15 hover:-translate-y-0.5',
                  { 'border-accent-indigo bg-accent-indigo/5': currentDesign === design.url },
                ]"
                @click="currentDesign = design.url"
              >
                <div
                  class="w-full h-12 md:h-20 flex items-center justify-center bg-black/20 rounded-lg p-1.5 md:p-2"
                >
                  <img
                    :src="design.url"
                    :alt="design.name"
                    class="max-w-full max-h-full object-contain"
                  />
                </div>
                <span
                  :class="[
                    'text-[10px] md:text-[11px] font-semibold text-text-muted text-center line-clamp-1',
                    { 'text-white': currentDesign === design.url },
                  ]"
                  >{{ design.name }}</span
                >
              </button>
            </div>

            <!-- Custom File Upload -->
            <div class="mb-4 md:mb-6">
              <label
                class="group block border-2 border-dashed border-border-subtle rounded-xl p-4 md:p-5 text-center cursor-pointer transition-all duration-300 hover:border-accent-indigo hover:bg-accent-indigo/2"
              >
                <input
                  type="file"
                  @change="handleImageUpload"
                  accept="image/png, image/jpeg, image/svg+xml"
                  class="hidden"
                />
                <div class="flex flex-col items-center gap-1.5 md:gap-2">
                  <Upload
                    :size="20"
                    class="text-text-muted group-hover:text-accent-indigo group-hover:-translate-y-0.5 transition-all duration-300 md:hidden"
                  />
                  <Upload
                    :size="24"
                    class="text-text-muted group-hover:text-accent-indigo group-hover:-translate-y-0.5 transition-all duration-300 hidden md:block"
                  />
                  <span class="text-[11px] md:text-xs font-semibold">Upload Custom PNG/SVG</span>
                  <span class="text-[9px] md:text-[11px] text-text-muted"
                    >Transparent background recommended</span
                  >
                </div>
              </label>
            </div>

            <!-- Position Controls -->
            <div class="pt-4 md:pt-5 border-t border-border-subtle" v-if="currentDesign">
              <h3 class="text-xs md:text-sm font-bold mb-3 md:mb-4 flex items-center gap-1.5">
                <Move :size="12" class="md:hidden" />
                <Move :size="14" class="hidden md:inline" /> Position & Scale
              </h3>

              <div class="mb-3 md:mb-4">
                <div class="flex justify-between items-center mb-1">
                  <span class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider"
                    >Scale</span
                  >
                  <span class="text-[11px] md:text-xs text-accent-indigo font-semibold"
                    >{{ Math.round(designScale * 100) }}%</span
                  >
                </div>
                <input type="range" min="0.1" max="0.8" step="0.01" v-model.number="designScale" />
              </div>

              <div class="mb-3 md:mb-4">
                <div class="flex justify-between items-center mb-1">
                  <span class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider"
                    >Vertical Shift</span
                  >
                  <span class="text-[11px] md:text-xs text-accent-indigo font-semibold">{{
                    Math.round((designY - 0.4) * -100)
                  }}</span>
                </div>
                <input type="range" min="0.0" max="1.0" step="0.01" v-model.number="designY" />
              </div>

              <div class="mb-3 md:mb-4">
                <div class="flex justify-between items-center mb-1">
                  <span class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider"
                    >Horizontal Shift</span
                  >
                  <span class="text-[11px] md:text-xs text-accent-indigo font-semibold">{{
                    Math.round((designX - 0.5) * 100)
                  }}</span>
                </div>
                <input type="range" min="0.1" max="0.9" step="0.01" v-model.number="designX" />
              </div>

              <div class="mb-3 md:mb-4">
                <div class="flex justify-between items-center mb-1">
                  <span class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider"
                    >Rotation</span
                  >
                  <span class="text-[11px] md:text-xs text-accent-indigo font-semibold"
                    >{{ designRotation }}°</span
                  >
                </div>
                <input type="range" min="-180" max="180" step="1" v-model.number="designRotation" />
              </div>
            </div>
          </div>

          <!-- Custom Text Tab -->
          <div v-if="activeTab === 'text'" class="animate-fade-in">
            <h2 class="text-base md:text-lg font-bold mb-1">Add Text Overlay</h2>
            <p class="text-[11px] md:text-xs text-text-muted mb-4 md:mb-6 leading-relaxed">
              Type your custom slogan and refine typography parameters.
            </p>

            <div class="mb-4 md:mb-5">
              <label
                class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider mb-2 block"
                >Slogan / Custom Text</label
              >
              <input
                type="text"
                v-model="customText"
                placeholder="TYPE YOUR TEXT HERE"
                maxlength="32"
                class="w-full p-2 md:p-2.5 px-3 md:px-3.5 bg-white/4 border border-border-subtle rounded-lg text-white font-sans text-sm md:text-base font-semibold outline-none transition-all duration-300 focus:border-accent-indigo focus:shadow-[0_0_0_3px_rgba(99,102,241,0.15)]"
              />
            </div>

            <div class="pt-4 md:pt-5 border-t border-border-subtle" v-if="customText">
              <div class="mb-3 md:mb-4">
                <label
                  class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider mb-2 block"
                  >Font Family</label
                >
                <select
                  v-model="textFont"
                  class="w-full p-2 md:p-2.5 px-3 md:px-3.5 bg-white/4 border border-border-subtle rounded-lg text-white font-sans text-xs md:text-sm outline-none transition-all duration-300 focus:border-accent-indigo focus:shadow-[0_0_0_3px_rgba(99,102,241,0.15)]"
                >
                  <option v-for="font in textFonts" :key="font" :value="font" class="bg-[#121218]">
                    {{ font }}
                  </option>
                </select>
              </div>

              <div class="mb-3 md:mb-4">
                <label
                  class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider mb-2 block"
                  >Text Shape</label
                >
                <div class="grid grid-cols-5 gap-1 bg-white/4 p-1 rounded-xl border border-border-subtle">
                  <button
                    type="button"
                    v-for="shapeOpt in [
                      { value: 'straight', label: 'Flat' },
                      { value: 'circle', label: 'Arc' },
                      { value: 'wave', label: 'Wave' },
                      { value: 'heart', label: 'Heart' },
                      { value: 'square', label: 'Square' }
                    ]"
                    :key="shapeOpt.value"
                    :class="[
                      'py-1.5 rounded-lg text-[10px] md:text-xs font-bold transition-all duration-300 cursor-pointer text-center',
                      textShape === shapeOpt.value
                        ? 'bg-accent-indigo text-white shadow-[0_2px_8px_rgba(99,102,241,0.3)]'
                        : 'text-text-muted hover:text-white hover:bg-white/4'
                    ]"
                    @click="textShape = shapeOpt.value"
                  >
                    {{ shapeOpt.label }}
                  </button>
                </div>
              </div>

              <div class="mb-3 md:mb-4">
                <div class="flex justify-between items-center mb-1">
                  <span class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider"
                    >Text Color</span
                  >
                </div>
                <div class="flex gap-3 items-center">
                  <div
                    class="w-9 h-9 md:w-11 md:h-11 rounded-lg border border-border-subtle relative overflow-hidden cursor-pointer"
                    :style="{ backgroundColor: textColor }"
                  >
                    <input
                      type="color"
                      v-model="textColor"
                      class="absolute -top-1 -left-1 w-12 h-12 md:w-15 md:h-15 border-0 p-0 cursor-pointer opacity-0"
                    />
                  </div>
                  <input
                    type="text"
                    v-model="textColor"
                    class="flex-1 p-2 md:p-2.5 px-3 md:px-3.5 bg-white/4 border border-border-subtle rounded-lg text-white font-sans text-xs md:text-sm outline-none transition-all duration-300 focus:border-accent-indigo focus:shadow-[0_0_0_3px_rgba(99,102,241,0.15)]"
                  />
                </div>
              </div>

              <div class="mb-3 md:mb-4">
                <div class="flex justify-between items-center mb-1">
                  <span class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider"
                    >Text Size</span
                  >
                  <span class="text-[11px] md:text-xs text-accent-indigo font-semibold">{{ textSize }}px</span>
                </div>
                <input type="range" min="4" max="100" step="1" v-model.number="textSize" />
              </div>

              <div class="mb-3 md:mb-4">
                <div class="flex justify-between items-center mb-1">
                  <span class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider"
                    >Letter Spacing</span
                  >
                  <span class="text-[11px] md:text-xs text-accent-indigo font-semibold"
                    >{{ textLetterSpacing }}px</span
                  >
                </div>
                <input type="range" min="0" max="25" step="1" v-model.number="textLetterSpacing" />
              </div>

              <div class="mb-3 md:mb-4">
                <div class="flex justify-between items-center mb-1">
                  <span class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider"
                    >Vertical Position</span
                  >
                  <span class="text-[11px] md:text-xs text-accent-indigo font-semibold">{{
                    Math.round((textY - 0.68) * 100)
                  }}</span>
                </div>
                <input type="range" min="0.0" max="1.0" step="0.01" v-model.number="textY" />
              </div>

              <div class="mb-3 md:mb-4">
                <div class="flex justify-between items-center mb-1">
                  <span class="text-[10px] md:text-xs font-semibold text-text-muted uppercase tracking-wider"
                    >Horizontal Position</span
                  >
                  <span class="text-[11px] md:text-xs text-accent-indigo font-semibold">{{
                    Math.round((textX - 0.5) * 100)
                  }}</span>
                </div>
                <input type="range" min="0.1" max="0.9" step="0.01" v-model.number="textX" />
              </div>
            </div>
          </div>
        </div>

        <!-- Export Actions Panel -->
        <div class="p-3 md:p-5 px-4 md:px-6 border-t border-border-subtle grid grid-cols-2 gap-2 md:gap-3 bg-white/[0.01] shrink-0">
          <button
            class="flex items-center justify-center gap-1.5 md:gap-2 bg-accent-indigo hover:bg-indigo-600 text-white font-semibold py-2 md:py-3 px-3 md:px-5 rounded-lg cursor-pointer transition-all duration-300 shadow-[0_4px_14px_rgba(99,102,241,0.2)] hover:shadow-[0_6px_20px_rgba(99,102,241,0.3)] hover:-translate-y-0.5 text-xs md:text-sm"
            @click="exportMockup"
          >
            <Download :size="14" class="md:hidden" />
            <Download :size="16" class="hidden md:inline" />
            Export 3D
          </button>
          <button
            class="flex items-center justify-center gap-1.5 md:gap-2 bg-white/6 hover:bg-white/10 border border-border-subtle hover:border-white/20 text-white font-semibold py-2 md:py-3 px-3 md:px-5 rounded-lg cursor-pointer transition-all duration-300 text-xs md:text-sm"
            @click="exportPrintDesign"
          >
            <Maximize2 :size="14" class="md:hidden" />
            <Maximize2 :size="16" class="hidden md:inline" />
            Print Template
          </button>
        </div>
      </aside>
    </div>
  </div>
</template>
