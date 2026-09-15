<template>
  <main class="lesson">
    <section class="panel">
      <p class="eyebrow">第 8 课 · GLB Animation</p>
      <h1>AnimationMixer 动画控制</h1>
      <p class="description">模型：RobotExpressive.glb。点击按钮观察动画切换，切换时会用 0.25 秒平滑过渡。</p>
      <div class="controls">
        <button :disabled="!ready" @click="playLoop('Idle')">Idle</button>
        <button :disabled="!ready" @click="playLoop('Walking')">Walk</button>
        <button :disabled="!ready" @click="playLoop('Running')">Run</button>
        <button :disabled="!ready" @click="playLoop('Dance')">Dance</button>
        <button :disabled="!ready" class="once" @click="playOnce('Punch')">Punch（攻击一次）</button>
      </div>
      <p class="status">{{ status }}</p>
      <p v-if="currentAnimation" class="current">当前动画：{{ currentAnimation }}</p>
      <details open>
        <summary>gltf.animations（{{ animationInfo.length }} 段）</summary>
        <ul><li v-for="clip in animationInfo" :key="clip.name">{{ clip.name }} · {{ clip.duration.toFixed(2) }} 秒</li></ul>
      </details>
    </section>
    <div ref="container" class="viewport"></div>
  </main>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import robotUrl from '../assets/glb/RobotExpressive.glb?url'

type ClipInfo = { name: string; duration: number }
const container = ref<HTMLDivElement | null>(null)
const ready = ref(false)
const status = ref('正在加载 RobotExpressive.glb…')
const currentAnimation = ref('')
const animationInfo = ref<ClipInfo[]>([])
let playLoop: (name: string) => void = () => {}
let playOnce: (name: string) => void = () => {}
let dispose: (() => void) | undefined

onMounted(() => {
  const host = container.value!

  const scene = new THREE.Scene()
  scene.background = new THREE.Color(0x172033)

  const camera = new THREE.PerspectiveCamera(45, 1, 0.1, 100)
  camera.position.set(4, 2.6, 5.5)

  const renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.outputColorSpace = THREE.SRGBColorSpace
  host.appendChild(renderer.domElement)

  const controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.target.set(0, 1, 0)

  scene.add(new THREE.HemisphereLight(0xffffff, 0x1d2940, 2.5))
  const keyLight = new THREE.DirectionalLight(0xffffff, 3)
  keyLight.position.set(4, 6, 4)
  scene.add(keyLight)
  scene.add(new THREE.GridHelper(10, 10, 0x506987, 0x34465d))

  const timer = new THREE.Timer()
  timer.connect(document)
  let mixer: THREE.AnimationMixer | null = null
  let currentAction: THREE.AnimationAction | null = null
  let model: THREE.Object3D | null = null
  new GLTFLoader().load(robotUrl, (gltf) => {
    model = gltf.scene
    scene.add(model)
    const box = new THREE.Box3().setFromObject(model)
    const center = box.getCenter(new THREE.Vector3())
    const size = box.getSize(new THREE.Vector3())
    const distance = Math.max(size.x, size.y, size.z) /
      (2 * Math.tan(THREE.MathUtils.degToRad(camera.fov / 2)))
    // 留出动作摆动的余量，避免 Punch、Dance 等姿势被画面边缘裁切。
    camera.position.set(center.x + distance * 0.7, center.y + size.y * 0.15, center.z + distance * 2.1)
    controls.target.copy(center)
    controls.update()
    // Level 1：查看 GLB 自带的 AnimationClip。
    animationInfo.value = gltf.animations.map((clip) => ({ name: clip.name, duration: clip.duration }))
    gltf.animations.forEach((clip) => console.log('动画名称:', clip.name, '时长:', clip.duration))

    // Level 2：AnimationMixer 管理模型上的所有 AnimationAction。
    mixer = new THREE.AnimationMixer(model)
    const actions = new Map(gltf.animations.map((clip) => [clip.name, mixer!.clipAction(clip)]))
    function switchTo(action: THREE.AnimationAction, name: string) {
      if (currentAction === action) return
      action.reset().setEffectiveWeight(1).setEffectiveTimeScale(1).play()
      // 切换动画时，平滑过渡，0.25 秒
      currentAction?.crossFadeTo(action, 0.25, false)
      currentAction = action
      currentAnimation.value = name
    }
    // Level 3、4：按名称取 AnimationAction，并用 crossFadeTo 平滑切换。
    playLoop = (name) => {
      const action = actions.get(name)
      if (!action) return console.warn(`找不到动画：${name}`)
      action.setLoop(THREE.LoopRepeat, Infinity)
      action.clampWhenFinished = false
      switchTo(action, name)
    }
    // Level 5：Punch 是该模型的攻击动作，播放一次后自动回到 Idle。
    playOnce = (name) => {
      const action = actions.get(name)
      if (!action) return console.warn(`找不到动画：${name}`)
      action.setLoop(THREE.LoopOnce, 1)
      action.clampWhenFinished = true
      switchTo(action, name)
    }
    mixer.addEventListener('finished', (event) => {
      if (event.action === actions.get('Punch')) playLoop('Idle')
    })
    ready.value = true
    status.value = '已加载。试试 Idle、Walk、Run 的平滑切换。'
    playLoop('Idle')
  }, undefined, (error) => {
    console.error('GLB 加载失败：', error)
    status.value = '模型加载失败，请检查浏览器控制台。'
  })

  const resize = () => {
    camera.aspect = host.clientWidth / host.clientHeight
    camera.updateProjectionMatrix()
    renderer.setSize(host.clientWidth, host.clientHeight)
  }
  resize()
  window.addEventListener('resize', resize)
  renderer.setAnimationLoop(() => {
    timer.update()
    mixer?.update(timer.getDelta()) // 没有这一行，action.play() 后动画不会前进。
    controls.update()
    renderer.render(scene, camera)
  })
  dispose = () => {
    renderer.setAnimationLoop(null)
    window.removeEventListener('resize', resize)
    timer.dispose()
    mixer?.stopAllAction()
    controls.dispose()
    model?.traverse((object) => {
      if (object instanceof THREE.Mesh) {
        object.geometry.dispose()
        ;(Array.isArray(object.material) ? object.material : [object.material]).forEach((material) => material.dispose())
      }
    })
    renderer.dispose()
    renderer.domElement.remove()
  }
})
onBeforeUnmount(() => dispose?.())
</script>

<style scoped>
.lesson { min-height: 100vh; display: grid; grid-template-columns: 330px 1fr; background: #172033; color: #eef4ff; }
.panel { padding: 28px; background: #202d43; box-shadow: 4px 0 18px #080d17aa; z-index: 1; }
.eyebrow { margin: 0 0 6px; color: #7ec8ff; font-size: 13px; font-weight: 700; letter-spacing: .08em; }
h1 { margin: 0; font-size: 25px; }.description, .status { color: #b8c6db; line-height: 1.65; }
.controls { display: flex; flex-wrap: wrap; gap: 8px; margin: 20px 0; }
button { border: 0; border-radius: 7px; padding: 9px 12px; background: #3e6391; color: white; cursor: pointer; }
button:hover:not(:disabled) { background: #5684ba; } button:disabled { cursor: wait; opacity: .45; }.once { background: #944f66; }
.current { color: #9fe7ba; font-weight: 700; } details { margin-top: 22px; color: #cbd8e8; } summary { cursor: pointer; font-weight: 700; } ul { padding-left: 20px; line-height: 1.7; }
.viewport { min-width: 0; min-height: 65vh; } @media (max-width: 720px) { .lesson { grid-template-columns: 1fr; } .viewport { height: 62vh; } }
</style>
