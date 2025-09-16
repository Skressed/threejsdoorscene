<template>
  <div class="three-container">
    <div ref="canvasContainer" class="canvas"></div>
    <div class="ui">
      <h3>Дверь</h3>
      <label>Ширина: {{ doorWidth.toFixed(2) }} м</label>
      <input type="range" min="0.5" max="2.0" step="0.01" v-model.number="doorWidth" @input="updateDoorSafe" />
      <label>Высота: {{ doorHeight.toFixed(2) }} м</label>
      <input type="range" min="1.5" max="3.0" step="0.01" v-model.number="doorHeight" @input="updateDoorSafe" />
      <button @click="toggleDoor">{{ isOpen ? 'Закрыть' : 'Открыть' }}</button>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'

export default defineComponent({
  setup() {
    const canvasContainer = ref<HTMLDivElement | null>(null)
    const doorWidth = ref(1)
    const doorHeight = ref(2)

    const roomWidthMeters = ref(6)
    const roomHeightMeters = ref(3)
    const roomDepthMeters = ref(6)

    const isOpen = ref(false)

    const textureLoader = new THREE.TextureLoader()
    const doorTexture = textureLoader.load('/tex/wood.jpg')
    const floorTexture = textureLoader.load('/tex/wood_reflective.jpg')
    const wallpaperTexture = textureLoader.load('/tex/wallpaper.jpeg')

    let scene: THREE.Scene
    let camera: THREE.PerspectiveCamera
    let renderer: THREE.WebGLRenderer
    let controls: any
    let doorGroup: THREE.Group | null = null
    let doorMesh: THREE.Mesh | null = null
    let animationId = 0
    let doorAngle = 0

    let frontWallTop: THREE.Mesh
    let frontWallLeft: THREE.Mesh
    let frontWallRight: THREE.Mesh

    function createDoor(width: number, height: number) {
      const geometry = new THREE.BoxGeometry(width, height, 0.1)
      const material = new THREE.MeshStandardMaterial({ 
        map: doorTexture, roughness: 0.9 
      })
      const mesh = new THREE.Mesh(geometry, material)
      mesh.castShadow = true
      mesh.receiveShadow = true
      return mesh
    }

    function updateDoor() {
      if (!scene || !doorGroup) return
      if (doorMesh) {
        doorGroup.remove(doorMesh)
        doorMesh.geometry.dispose()
      }

      // --- Дверь ---
      doorMesh = createDoor(doorWidth.value, doorHeight.value)
      doorMesh.position.x = doorWidth.value / 2
      doorMesh.position.y = doorHeight.value / 2
      doorMesh.position.z = 0
      doorGroup.position.set(-doorWidth.value/2, 0, roomDepthMeters.value/2)
      doorGroup.add(doorMesh)

      // --- Передние стены ---
      const leftWidth = (roomWidthMeters.value - doorWidth.value) / 2
      const rightWidth = (roomWidthMeters.value - doorWidth.value) / 2

      frontWallTop.geometry.dispose()
      frontWallTop.geometry = new THREE.BoxGeometry(doorWidth.value, 1 - (doorHeight.value - 2), 0.1)
      frontWallTop.position.set(0,doorHeight.value + (1 - (doorHeight.value - 2))/2,roomDepthMeters.value/2)

      frontWallLeft.geometry.dispose()
      frontWallLeft.geometry = new THREE.BoxGeometry(leftWidth, roomHeightMeters.value, 0.1)
      frontWallLeft.position.set(-(doorWidth.value / 2 + leftWidth / 2), roomHeightMeters.value / 2, roomDepthMeters.value/2)

      frontWallRight.geometry.dispose()
      frontWallRight.geometry = new THREE.BoxGeometry(rightWidth, roomHeightMeters.value, 0.1)
      frontWallRight.position.set(doorWidth.value / 2 + rightWidth / 2, roomHeightMeters.value / 2, roomDepthMeters.value/2)
    }

    function updateDoorSafe() {
      if (!scene || !doorGroup) return
      updateDoor()
    }

    function toggleDoor() {
      isOpen.value = !isOpen.value
    }

    onMounted(async () => {
      if (!canvasContainer.value) return

      scene = new THREE.Scene()
      scene.background = new THREE.Color(0xbfd1e5)

      const width = canvasContainer.value.clientWidth
      const height = canvasContainer.value.clientHeight

      camera = new THREE.PerspectiveCamera(60, width / height, 0.1, 1000)
      camera.position.set(5, 3, 7)

      renderer = new THREE.WebGLRenderer({ antialias: true })
      renderer.setSize(width, height)
      renderer.setPixelRatio(window.devicePixelRatio ?? 1)
      renderer.shadowMap.enabled = true
      canvasContainer.value.appendChild(renderer.domElement)

      // --- Комната ---
      const roomWidth = roomWidthMeters.value
      const roomHeight = roomHeightMeters.value
      const roomDepth = roomDepthMeters.value

      // Пол
      const floor = new THREE.Mesh(
        new THREE.PlaneGeometry(roomWidth, roomDepth),
        new THREE.MeshStandardMaterial({ 
          map: floorTexture, roughness: 0.1, side: THREE.DoubleSide
        })
      )
      floor.rotation.x = -Math.PI / 2
      floor.receiveShadow = true
      scene.add(floor)

      // Функция для стен
      function createWall(width: number, height: number, depth: number, fillSide: number) {
        const materials: THREE.Material[] = []

        for (let i = 0; i < 6; i++) {
          if (i === fillSide) {
            materials.push(new THREE.MeshStandardMaterial({
              map: wallpaperTexture,
              side: THREE.DoubleSide,
              roughness: 0.4
            }))
          } else {
            materials.push(new THREE.MeshStandardMaterial({
              color: 0xffffff,
              side: THREE.DoubleSide,
              roughness: 0.8
            }))
          }
        }

        const wall = new THREE.Mesh(new THREE.BoxGeometry(width, height, depth), materials)
        wall.receiveShadow = true
        return wall
      }

      // Задняя стена
      const backWall = createWall(roomWidth, roomHeight, 0.1, 4)
      backWall.position.set(0, roomHeight / 2, -roomDepth / 2)
      scene.add(backWall)

      // Левая стена
      const leftWall = createWall(0.1, roomHeight, roomDepth, 0)
      leftWall.position.set(-roomWidth / 2, roomHeight / 2, 0)
      scene.add(leftWall)

      // Правая стена
      const rightWall = createWall(0.1, roomHeight, roomDepth, 1)
      rightWall.position.set(roomWidth / 2, roomHeight / 2, 0)
      scene.add(rightWall)

      // Передние стены с дверью
      frontWallLeft = createWall((roomWidth - doorWidth.value)/2, roomHeight, 0.1, 5)
      frontWallLeft.position.set(-(doorWidth.value / 2 + (roomWidth - doorWidth.value)/4), roomHeight / 2, roomDepth / 2)
      scene.add(frontWallLeft)

      frontWallRight = createWall((roomWidth - doorWidth.value)/2, roomHeight, 0.1, 5)
      frontWallRight.position.set(doorWidth.value / 2 + (roomWidth - doorWidth.value)/4, roomHeight / 2, roomDepth / 2)
      scene.add(frontWallRight)

      frontWallTop = createWall(doorWidth.value, roomHeight - doorHeight.value, 0.1, 5)
      frontWallTop.position.set(0,5,0)
      scene.add(frontWallTop)

      // Куб и сфера
      const cube = new THREE.Mesh(
        new THREE.BoxGeometry(1, 1, 1),
        new THREE.MeshStandardMaterial({ color: 0x00aa00 })
      )
      cube.position.set(-2, 1.5, 1)
      cube.castShadow = true
      cube.receiveShadow = true
      scene.add(cube)

      const sphere = new THREE.Mesh(
        new THREE.SphereGeometry(0.5, 32, 32),
        new THREE.MeshStandardMaterial({ color: 0x0000ff, metalness: 0.6, roughness: 0.2 })
      )
      sphere.position.set(2, 0.5, 1)
      sphere.castShadow = true
      sphere.receiveShadow = true
      scene.add(sphere)

      // Свет
      const dirLight = new THREE.DirectionalLight(0xffffff, 0.1)
      dirLight.position.set(5, 10, 7)
      dirLight.castShadow = false
      scene.add(dirLight)

      const ambient = new THREE.AmbientLight(0xffffff, 0.1)
      scene.add(ambient)

      // Источник света в центре комнаты
      const centerLight = new THREE.PointLight(0xFFBF00, 5, 15) // цвет, интенсивность, радиус действия
      centerLight.position.set(0, roomHeightMeters.value - 0.15, 0) // чуть ниже потолка
      centerLight.castShadow = true
      scene.add(centerLight)

      // Визуальный индикатор света (необязательно)
      const lightSphere = new THREE.Mesh(
        new THREE.SphereGeometry(0.1, 16, 16),
        new THREE.MeshBasicMaterial({ color: 0xFFBF00 })
      )
      lightSphere.position.copy(centerLight.position)
      scene.add(lightSphere)

      // Дверь через группу
      doorGroup = new THREE.Group()
      scene.add(doorGroup)
      updateDoor()

      // const helper = new THREE.BoxHelper(doorGroup, 0xff0000) // красная рамка
      // scene.add(helper)

      // OrbitControls
      const { OrbitControls } = await import('three/examples/jsm/controls/OrbitControls')
      controls = new OrbitControls(camera, renderer.domElement)
      controls.enableDamping = true
      controls.dampingFactor = 0.08
      controls.target.set(0, 1, 0)
      controls.update()

      // Анимация
      const animate = () => {
        cube.rotation.x += 0.01
        cube.rotation.y += 0.01
        sphere.rotation.y -= 0.01

        // Дверь
        const targetAngle = isOpen.value ? Math.PI/2 : 0
        doorAngle += (targetAngle - doorAngle) * 0.05
        if (doorGroup) doorGroup.rotation.y = doorAngle

        controls.update()
        renderer.render(scene, camera)
        animationId = requestAnimationFrame(animate)

        // helper.update()
      }
      animate()

      window.addEventListener('resize', () => {
        const w = canvasContainer.value!.clientWidth
        const h = canvasContainer.value!.clientHeight
        camera.aspect = w / h
        camera.updateProjectionMatrix()
        renderer.setSize(w, h)
      })
    })

    onBeforeUnmount(() => {
      cancelAnimationFrame(animationId)
      if (renderer && canvasContainer.value) {
        renderer.dispose()
        canvasContainer.value.removeChild(renderer.domElement)
      }
    })

    return { canvasContainer, doorWidth, doorHeight, updateDoorSafe, isOpen, toggleDoor }
  },
})
</script>

<style scoped>
.three-container {
  display: flex;
  width: 100%;
  height: 100vh;
}
.canvas {
  flex: 1;
}
.ui {
  width: 260px;
  padding: 12px;
  background: rgba(255, 255, 255, 0.9);
  border-left: 1px solid #ccc;
  font-family: sans-serif;
  display: flex;
  flex-direction: column;
}
</style>