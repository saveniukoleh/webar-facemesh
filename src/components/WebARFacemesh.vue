vue
<template>
  <div class="webcam-view" ref="container"></div>
</template>
<script>
import * as THREE from 'three'
import { FaceMesh } from '@mediapipe/face_mesh'
import { Camera } from '@mediapipe/camera_utils'
import { onMounted, onBeforeUnmount, ref } from 'vue'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import Stats from 'three/addons/libs/stats.module.js'
export default {
  name: 'WebcamView',
  setup() {
    const container = ref(null)
    let scene,
      camera,
      renderer,
      video,
      faceMesh,
      faceMaskLandmarks,
      previousFaceMaskLandmarksPosition,
      landmarkThreshold = 0.1,
      glassesMesh,
      stats
    const init = async () => {
      stats = new Stats()
      stats.showPanel(0)
      container.value.appendChild(stats.dom)
      scene = new THREE.Scene()
      camera = new THREE.PerspectiveCamera(
        75,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
      )
      renderer = new THREE.WebGLRenderer({ alpha: true })
      renderer.setSize(window.innerWidth, window.innerHeight)
      container.value.appendChild(renderer.domElement)
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.5)
      scene.add(ambientLight)
      const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
      directionalLight.position.set(1, 1, 1).normalize()
      scene.add(directionalLight)
      video = document.createElement('video')
      video.autoplay = true
      video.muted = true
      video.playsInline = true
      await startCamera()
      const videoTexture = new THREE.VideoTexture(video)
      const geometry = new THREE.PlaneGeometry(16, 9)
      const material = new THREE.MeshBasicMaterial({ map: videoTexture })
      const plane = new THREE.Mesh(geometry, material)
      plane.material.depthTest = false
      plane.material.depthWrite = false
      plane.position.z = 0
      scene.add(plane)
      camera.position.z = 5
      prepareLandmarks()
      prepareGlasses()
      animate()
      await initFaceMesh()
    }
    const startCamera = async () => {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({
          video: {
            facingMode: { exact: 'user' },
            width: { ideal: 1920 },
            height: { ideal: 1080 },
            frameRate: { ideal: 60 }
          }
        })
        video.srcObject = stream
      } catch (error) {
        console.error('Error accessing webcam:', error)
      }
    }
    const initFaceMesh = async () => {
      faceMesh = new FaceMesh({
        locateFile: (file) =>
          `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}`
      })
      faceMesh.setOptions({
        maxNumFaces: 1,
        refineLandmarks: true,
        minDetectionConfidence: 0.5,
        minTrackingConfidence: 0.5
      })
      faceMesh.onResults(onFaceMeshResults)
      const camera = new Camera(video, {
        onFrame: async () => {
          await faceMesh.send({ image: video })
        },
        width: 640,
        height: 480
      })
      camera.start()
    }
    const onFaceMeshResults = (results) => {
      if (results.multiFaceLandmarks.length > 0) {
        const landmarks = results.multiFaceLandmarks[0]
        drawFaceMask(landmarks)
        drawGlasses()
      }
    }
    const drawGlasses = () => {
      if (!glassesMesh.visible) glassesMesh.visible = true
      glassesMesh.position.set(
        previousFaceMaskLandmarksPosition[8].x,
        previousFaceMaskLandmarksPosition[8].y - 1.25,
        previousFaceMaskLandmarksPosition[8].z
      )
    }
    const prepareGlasses = () => {
      const loader = new GLTFLoader()
      loader.load(
        'glasses-1/scene.gltf',
        (gltf) => {
          glassesMesh = gltf.scene
          glassesMesh.name = 'glasses'
          glassesMesh.position.set(0, 0, 0)
          glassesMesh.scale.set(0.25, 0.25, 0.25)
          glassesMesh.rotation.y = -Math.PI / 2
          glassesMesh.visible = false
          scene.add(glassesMesh)
        },
        undefined,
        (error) => {
          console.error(
            'An error occurred while loading the glasses model:',
            error
          )
        }
      )
    }
    const drawFaceMask = (landmarks) => {
      if (!faceMaskLandmarks.visible) faceMaskLandmarks.visible = true
      landmarks.forEach((landmark, index) => {
        let x = (landmark.x - 0.5) * 16
        let y = (0.5 - landmark.y) * 9
        let z = 0.1 + landmark.z / 5
        if (
          Math.abs(previousFaceMaskLandmarksPosition[index].x - x) >
            landmarkThreshold ||
          Math.abs(previousFaceMaskLandmarksPosition[index].y - y) >
            landmarkThreshold ||
          Math.abs(previousFaceMaskLandmarksPosition[index].z - z) >
            landmarkThreshold
        ) {
          previousFaceMaskLandmarksPosition[index] = { x: x, y: y, z: z }
        } else {
          x = previousFaceMaskLandmarksPosition[index].x
          y = previousFaceMaskLandmarksPosition[index].y
          z = previousFaceMaskLandmarksPosition[index].z
        }
        faceMaskLandmarks.children[index].position.set(
          previousFaceMaskLandmarksPosition[index].x,
          previousFaceMaskLandmarksPosition[index].y,
          previousFaceMaskLandmarksPosition[index].z
        )
      })
    }
    const prepareLandmarks = () => {
      faceMaskLandmarks = new THREE.Group()
      previousFaceMaskLandmarksPosition = []
      for (let i = 0; i < 478; i++) {
        const geometry = new THREE.SphereGeometry(0.05, 16, 16)
        const material = new THREE.MeshStandardMaterial({ color: 0xffff00 })
        const mesh = new THREE.Mesh(geometry, material)
        previousFaceMaskLandmarksPosition[i] = { x: 0, y: 0, z: 0 }
        mesh.position.set(
          previousFaceMaskLandmarksPosition[i].x,
          previousFaceMaskLandmarksPosition[i].y,
          previousFaceMaskLandmarksPosition[i].z
        )
        mesh.scale.set(0.25, 0.25, 0.25)
        mesh.name = 'landmark'
        faceMaskLandmarks.add(mesh)
      }
      faceMaskLandmarks.visible = false
      scene.add(faceMaskLandmarks)
    }
    const animate = () => {
      requestAnimationFrame(animate)
      stats.begin()
      renderer.render(scene, camera)
      stats.end()
    }
    const onWindowResize = () => {
      camera.aspect = window.innerWidth / window.innerHeight
      camera.updateProjectionMatrix()
      renderer.setSize(window.innerWidth, window.innerHeight)
    }
    const cleanup = () => {
      if (renderer) {
        renderer.dispose()
      }
      if (video && video.srcObject) {
        video.srcObject.getTracks().forEach((track) => track.stop())
      }
      window.removeEventListener('resize', onWindowResize)
    }
    onMounted(init)
    onBeforeUnmount(cleanup)
    return {
      container
    }
  }
}
</script>
<style scoped>
.webcam-view {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  background: black;
}
</style>
