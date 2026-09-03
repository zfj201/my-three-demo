<template>

</template>

<script setup lang="ts">
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(
  75,
  window.innerWidth / window.innerHeight,
  0.1,
  1000
)

const renderer = new THREE.WebGLRenderer()
renderer.setSize(window.innerWidth, window.innerHeight)
document.body.appendChild(renderer.domElement)

const geometry = new THREE.BoxGeometry(2, 2, 2)
const material = new THREE.MeshStandardMaterial({
  color: 0x00ff00,
})
const cube = new THREE.Mesh(
  geometry,
  material
)
cube.position.set(3, 0, 0)
scene.add(cube)
const sphereGeometry = new THREE.SphereGeometry(1, 32, 32)
const sphereMaterial = new THREE.MeshStandardMaterial({
  color: 0xff0000,
})
const sphere = new THREE.Mesh(
  sphereGeometry,
  sphereMaterial
)
sphere.position.set(-3, 0, 0)
scene.add(sphere)
camera.position.z = 5
camera.position.x = 5


const group = new THREE.Group()
scene.add(group)
group.add(cube)
group.add(sphere)
group.position.set(5, 0, 0)

// 环境光
const ambientLight = new THREE.AmbientLight(
  0xffffff,
  0.5
)
scene.add(ambientLight)
//平行光
const directionalLight = new THREE.DirectionalLight(
  0xffffff,
  1
)
directionalLight.position.set(5, 5, 5)
scene.add(directionalLight)

renderer.render(scene, camera)

const controls = new OrbitControls(
  camera,
  renderer.domElement
)
controls.enableDamping = true

function animate() {
  requestAnimationFrame(animate)

  group.rotation.x += 0.01
  group.rotation.y += 0.01
  // group.position.x += 0.01

  controls.update()

  renderer.render(scene, camera)
}

animate()

</script>
<style scoped lang="scss">

</style>
