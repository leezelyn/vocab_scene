<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'

interface InteractableObject {
  id: string
  name: string
  translation: string
  type: 'box' | 'sphere'
  position: [number, number, number]
  color: number
  scale?: [number, number, number]
}

const container = ref<HTMLDivElement | null>(null)

let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let renderer: THREE.WebGLRenderer
let controls: OrbitControls
let animationId: number

const initScene = () => {
  if (!container.value) return

  // Scene
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0xf0f0f0)

  // Camera
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(0, 2, 5)

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.shadowMap.enabled = true
  container.value.appendChild(renderer.domElement)

  // Controls
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05

  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
  directionalLight.position.set(5, 10, 7)
  directionalLight.castShadow = true
  scene.add(directionalLight)

  createRoom()
  populateObjects()

  animate()
}

const raycaster = new THREE.Raycaster()
const mouse = new THREE.Vector2()
const selectedObject = ref<InteractableObject | null>(null)

const onPointerMove = (event: MouseEvent) => {
  if (!container.value) return
  const rect = container.value.getBoundingClientRect()
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1
  
  checkIntersection()
}

const onClick = () => {
  const intersects = getIntersects()
  if (intersects.length > 0) {
    const intersect = intersects[0]
    // Safe check and cast
    if (intersect.object && intersect.object.userData) {
        selectedObject.value = intersect.object.userData as InteractableObject
    }
  } else {
    selectedObject.value = null
  }
}

const getIntersects = () => {
  raycaster.setFromCamera(mouse, camera)
  // Recursively check children if we use groups later, but for now simple objects
  return raycaster.intersectObjects(scene.children).filter((intersect) => intersect.object.userData?.isInteractable)
}

const checkIntersection = () => {
  const intersects = getIntersects()
  if (intersects.length > 0) {
    document.body.style.cursor = 'pointer'
  } else {
    document.body.style.cursor = 'default'
  }
}

const createRoom = () => {
  // Floor
  const floorGeo = new THREE.PlaneGeometry(10, 10)
  const floorMat = new THREE.MeshStandardMaterial({ color: 0xeeeeee, side: THREE.DoubleSide })
  const floor = new THREE.Mesh(floorGeo, floorMat)
  floor.rotation.x = -Math.PI / 2
  floor.receiveShadow = true
  scene.add(floor)

  // Walls
  const wallMat = new THREE.MeshStandardMaterial({ color: 0xffffff, side: THREE.DoubleSide })
  
  // Back Wall
  const backWall = new THREE.Mesh(new THREE.PlaneGeometry(10, 5), wallMat)
  backWall.position.set(0, 2.5, -5)
  backWall.receiveShadow = true
  scene.add(backWall)

  // Left Wall
  const leftWall = new THREE.Mesh(new THREE.PlaneGeometry(10, 5), wallMat)
  leftWall.rotation.y = Math.PI / 2
  leftWall.position.set(-5, 2.5, 0)
  leftWall.receiveShadow = true
  scene.add(leftWall)
}

const objectsData: InteractableObject[] = [
  { id: 'table', name: 'Table', translation: '机 (Tsukue)', type: 'box', position: [0, 0.75, 0], color: 0x8B4513, scale: [2, 1.5, 1.5] },
  { id: 'chair', name: 'Chair', translation: '椅子 (Isu)', type: 'box', position: [0, 0.5, 1.5], color: 0xA0522D, scale: [0.8, 1, 0.8] },
  { id: 'lamp', name: 'Lamp', translation: 'ランプ (Ranpu)', type: 'sphere', position: [1.2, 1.8, -0.2], color: 0xFFFFE0, scale: [0.3, 0.3, 0.3] },
  { id: 'plant', name: 'Plant', translation: '植物 (Shokubutsu)', type: 'box', position: [-2, 0.5, -2], color: 0x228B22, scale: [0.5, 1, 0.5] }
]

const populateObjects = () => {
  objectsData.forEach(obj => {
    let geometry: THREE.BufferGeometry
    if (obj.type === 'box') {
      geometry = new THREE.BoxGeometry(1, 1, 1)
    } else {
      geometry = new THREE.SphereGeometry(1, 32, 32)
    }

    const material = new THREE.MeshStandardMaterial({ color: obj.color })
    const mesh = new THREE.Mesh(geometry, material)
    
    mesh.position.set(...obj.position)
    if (obj.scale) {
      mesh.scale.set(...obj.scale)
    }
    
    mesh.castShadow = true
    mesh.receiveShadow = true
    
    // Store metadata in the mesh for Raycaster
    mesh.userData = { ...obj, isInteractable: true }
    
    scene.add(mesh)
  })
}

const animate = () => {
  animationId = requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}

const handleResize = () => {
  if (!camera || !renderer) return
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
}

onMounted(() => {
  initScene()
  window.addEventListener('resize', handleResize)
  window.addEventListener('mousemove', onPointerMove)
  window.addEventListener('click', onClick)
})

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
  window.removeEventListener('mousemove', onPointerMove)
  window.removeEventListener('click', onClick)
  cancelAnimationFrame(animationId)
  renderer.dispose()
})
</script>

<template>
  <div ref="container" class="scene-container"></div>
  <Transition name="fade">
    <div v-if="selectedObject" class="info-card">
      <h2 class="japanese">{{ selectedObject.translation }}</h2>
      <p class="english">{{ selectedObject.name }}</p>
    </div>
  </Transition>
</template>

<style scoped>
.scene-container {
  width: 100%;
  height: 100vh;
  overflow: hidden;
}

.info-card {
  position: absolute;
  top: 30px;
  right: 30px;
  background: rgba(255, 255, 255, 0.95);
  padding: 24px;
  border-radius: 16px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.15);
  min-width: 240px;
  text-align: center;
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.5);
}

.japanese {
  margin: 0 0 8px 0;
  font-size: 2rem;
  color: #2c3e50;
  font-weight: 800;
}

.english {
  margin: 0;
  font-size: 1.2rem;
  color: #666;
  font-weight: 500;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease, transform 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
  transform: translateY(-20px);
}
</style>
