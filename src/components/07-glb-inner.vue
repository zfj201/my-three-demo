<template>
  <div ref="container" class="three-container"></div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";

// 第一个参数用相对当前组件的路径；Vite 构建时会生成带哈希的最终 URL。
const modelUrls = {
  antiqueCamera: new URL("../assets/glb/AntiqueCamera.glb", import.meta.url)
    .href,
  chronographWatch: new URL(
    "../assets/glb/ChronographWatch.glb",
    import.meta.url,
  ).href,
  damagedHelmet: new URL("../assets/glb/DamagedHelmet.glb", import.meta.url)
    .href,
  toyCar: new URL("../assets/glb/ToyCar.glb", import.meta.url).href,
};

const container = ref<HTMLDivElement | null>(null);
let dispose: (() => void) | undefined;

onMounted(() => {
  const host = container.value!;

  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x202735);

  const camera = new THREE.PerspectiveCamera(45, 1, 0.1, 1000);
  camera.position.set(0, 4, 18);

  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  host.appendChild(renderer.domElement);

  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.target.set(0, 0, 0);

  scene.add(new THREE.HemisphereLight(0xffffff, 0x303040, 2));
  const directionalLight = new THREE.DirectionalLight(0xffffff, 3);
  directionalLight.position.set(5, 8, 6);
  scene.add(directionalLight);

  const loader = new GLTFLoader();
  const mixers: THREE.AnimationMixer[] = [];
  const timer = new THREE.Timer();
  timer.connect(document);
  async function loadModel(name: string, url: string, x: number) {
    const gltf = await loader.loadAsync(url);
    console.log('gltf',name,gltf);
    // console.log(
    //   'gltf.animations',name,gltf.animations
    // )
    const model = gltf.scene;
    const box = new THREE.Box3().setFromObject(model);
    const center = box.getCenter(new THREE.Vector3());
    const size = box.getSize(new THREE.Vector3());
    const scale = 3 / Math.max(size.x, size.y, size.z);

    const car = model.getObjectByName('ToyCar')
    if (car instanceof THREE.Mesh) {
      car.visible = false
    }

    const wheel = model.getObjectByName('Band_Carbon_Fiber')

    if (wheel instanceof THREE.Mesh) {
      // 如果想只修改这个零件，不影响其他使用同一材质的零件，需要 material.clone()
      wheel.material = wheel.material.clone()
      if (wheel.material instanceof THREE.MeshStandardMaterial) {
        wheel.material.color.set(0x00ff00)
        // 透明度
        wheel.material.transparent = true
        wheel.material.opacity = 0.3
      }
      // 如果多个mesh材质公用，下面的写法会修改所有使用该材质的零件的颜色
      // wheel.material.color.set(0x00ff00);
    }
    model.traverse((child) => {
      if (child instanceof THREE.Mesh) {
        // child.material.color.set(0x00ff00);
        console.log("mesh", name, child);
      }
      // console.log('child',name, child);
    });
    // 统一尺寸，并把每个模型的中心移动到指定的 x 位置。
    model.scale.setScalar(scale);
    model.position.set(
      x - center.x * scale,
      -center.y * scale,
      -center.z * scale,
    );
    scene.add(model);

    // ChronographWatch.glb 自带 1 个 AnimationClip。
    // Mixer 负责把动画作用到模型，update(delta) 则推动动画时间前进。
    if (name === "ChronographWatch" && gltf.animations.length > 0) {
      const mixer = new THREE.AnimationMixer(model);
      const action = mixer.clipAction(gltf.animations[0]);
      action.reset().play();
      mixers.push(mixer);
      // console.log("已播放手表动画:", gltf.animations[0].name || "第一个 AnimationClip");
    }
    console.log(`${name} 的原始尺寸:`, size);
  }

  void Promise.all([
    loadModel("AntiqueCamera", modelUrls.antiqueCamera, -6),
    loadModel("ChronographWatch", modelUrls.chronographWatch, -2),
    loadModel("DamagedHelmet", modelUrls.damagedHelmet, 2),
    loadModel("ToyCar", modelUrls.toyCar, 6),
  ]).catch((error: unknown) => console.error("GLB 加载失败：", error));

  const resize = () => {
    const width = host.clientWidth;
    const height = host.clientHeight;
    camera.aspect = width / height;
    camera.updateProjectionMatrix();
    renderer.setSize(width, height);
  };
  resize();
  window.addEventListener("resize", resize);

  function animate() {
    timer.update();
    const delta = timer.getDelta();
    mixers.forEach((mixer) => mixer.update(delta));
    controls.update();
    renderer.render(scene, camera);
  }
  renderer.setAnimationLoop(animate);

  dispose = () => {
    renderer.setAnimationLoop(null);
    window.removeEventListener("resize", resize);
    mixers.forEach((mixer) => mixer.stopAllAction());
    timer.dispose();
    controls.dispose();
    renderer.dispose();
    renderer.domElement.remove();
  };
});

onBeforeUnmount(() => dispose?.());
</script>
<style scoped>
.three-container {
  width: 100%;
  height: 100vh;
}
</style>
