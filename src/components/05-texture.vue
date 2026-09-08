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

const geometry = new THREE.SphereGeometry(1, 32, 32)
const planeGeometry = new THREE.PlaneGeometry(2, 2)
const textureLoader = new THREE.TextureLoader()
let texture = textureLoader.load('/src/assets/imgs/forest.png')
texture.wrapS = THREE.RepeatWrapping
texture.wrapT = THREE.RepeatWrapping
texture.repeat.set(2, 2)
const material1 = new THREE.MeshStandardMaterial({
//   color: 0xff0000,
  roughness: 0,
  metalness: 0,
  transparent: true,
  opacity: 0.5,
  wireframe: false,
  map: texture,
})

texture = textureLoader.load('/src/assets/imgs/online_088.png')
texture.wrapS = THREE.RepeatWrapping
texture.wrapT = THREE.RepeatWrapping
texture.repeat.set(2, 2)
const material2 = new THREE.MeshStandardMaterial({
//   color: 0xff0000,
//   roughness: 1,
//   metalness: 0,
//   transparent: true,
//   opacity: 0.5,
//   wireframe: true,
    map: texture,
})
texture = textureLoader.load('/src/assets/imgs/mountain_lake.png')
const material3 = new THREE.MeshStandardMaterial({
//   color: 0xff0000,
  roughness: 0.2,
  metalness: 1,
//   transparent: true,
//   opacity: 0.5,
  wireframe: true,
  map: texture,
})

const plane = new THREE.Mesh(planeGeometry, material1)
plane.position.set(0, 3, 0)
scene.add(plane)

const sphere1 = new THREE.Mesh(geometry, material1)
const sphere2 = new THREE.Mesh(geometry, material2)
const sphere3 = new THREE.Mesh(geometry, material3)
sphere1.position.set(0, 0, 0)
sphere2.position.set(3, 0, 0)
sphere3.position.set(6, 0, 0)

scene.add(sphere1)
scene.add(sphere2)
scene.add(sphere3)

camera.position.z = 5
camera.position.x = 5


const group = new THREE.Group()
scene.add(group)
// group.add(cube)
// group.add(sphere)
group.position.set(5, 0, 0)

// 环境光
const ambientLight = new THREE.AmbientLight(
  0xffffff,
  0.5
)
scene.add(ambientLight)
// //平行光
const directionalLight = new THREE.DirectionalLight(
  0xffff00,
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
  sphere1.rotation.y += 0.01
  sphere2.rotation.y += 0.01
  sphere3.rotation.y += 0.01
//   group.rotation.x += 0.01
//   group.rotation.y += 0.01
  // group.position.x += 0.01

  controls.update()

  renderer.render(scene, camera)
}

animate()

</script>
<style scoped lang="scss">

</style>
