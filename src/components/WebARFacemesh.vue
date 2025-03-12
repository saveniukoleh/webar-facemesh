vue
<template>
  <div class="webcam-view" ref="container"></div>
</template>
<script>
import * as THREE from 'three' // Import Three.js
import { FaceMesh } from '@mediapipe/face_mesh' // Import FaceMesh from MediaPipe
import { Camera } from '@mediapipe/camera_utils' // Import Camera utility
import { onMounted, onBeforeUnmount, ref } from 'vue' // Vue composition API
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { TextureLoader } from 'three' // Import TextureLoader
import Stats from 'three/addons/libs/stats.module.js' // Import Stats from three.js
export default {
  name: 'WebcamView',
  setup() {
    const container = ref(null) // Reference for the container
    let scene,
      camera,
      renderer,
      video,
      faceMesh,
      faceMaskLandmarks,
      previousFaceMaskLandmarksPosition,
      landmarkThreshold = 0.1,
      stats // Declare variables
    const init = async () => {
      // Initialize Stats.js
      stats = new Stats()
      stats.showPanel(0) // 0: FPS, 1: MS, 2: Memory
      container.value.appendChild(stats.dom)
      // Set up Three.js scene
      scene = new THREE.Scene()
      camera = new THREE.PerspectiveCamera(
        75,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
      )
      renderer = new THREE.WebGLRenderer({ alpha: true })
      renderer.setSize(window.innerWidth, window.innerHeight)
      container.value.appendChild(renderer.domElement) // Add renderer to the DOM
      // Add lights to the scene
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.5) // White light, intensity 0.5
      scene.add(ambientLight)
      const directionalLight = new THREE.DirectionalLight(0xffffff, 1) // White light, intensity 1
      directionalLight.position.set(1, 1, 1).normalize() // Position the light
      scene.add(directionalLight)
      // Create video element for webcam
      video = document.createElement('video')
      video.autoplay = true
      video.muted = true // Mute to avoid feedback
      video.playsInline = true // Required for iOS
      // Start the webcam
      await startCamera()
      // Create a texture from the video
      const videoTexture = new THREE.VideoTexture(video)
      const geometry = new THREE.PlaneGeometry(16, 9) // Aspect ratio 16:9
      const material = new THREE.MeshBasicMaterial({ map: videoTexture })
      const plane = new THREE.Mesh(geometry, material)
      scene.add(plane) // Add video plane to the scene
      // Position the camera
      camera.position.z = 5
      // Prepare face mask landmarks
      prepareLandmarks()
      // Start the animation loop
      animate()
      // Initialize FaceMesh
      await initFaceMesh()
    }
    const startCamera = async () => {
      try {
        // Request access to the webcam with maximum quality settings
        const constraints = {
          video: {
            facingMode: { exact: 'user' }, // Use the front camera
            width: { ideal: 1920 }, // Set ideal width (e.g., 1920 for Full HD)
            height: { ideal: 1080 }, // Set ideal height (e.g., 1080 for Full HD)
            frameRate: { ideal: 60 } // Set ideal frame rate (e.g., 60 fps)
          }
        }
        const stream = await navigator.mediaDevices.getUserMedia(constraints)
        video.srcObject = stream // Set video source to the webcam stream
      } catch (error) {
        console.error('Error accessing webcam:', error) // Log any errors
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
      faceMesh.onResults(onFaceMeshResults) // Set results callback
      const camera = new Camera(video, {
        onFrame: async () => {
          await faceMesh.send({ image: video }) // Send video frame to FaceMesh
        },
        width: 640,
        height: 480
      })
      camera.start() // Start the camera
    }
    const onFaceMeshResults = (results) => {
      if (results.multiFaceLandmarks.length > 0) {
        const landmarks = results.multiFaceLandmarks[0] // Get the first face landmarks
        drawGlasses(landmarks) // Draw glasses instead of landmarks
        drawFaceMask(landmarks)
      }
    }
    const drawGlasses = (landmarks) => {
      // Implementation for drawing glasses (commented out for brevity)
    }
    const drawFaceMask = (landmarks) => {
      if (!faceMaskLandmarks.visible) faceMaskLandmarks.visible = true
      // Create objects at each landmark
      landmarks.forEach((landmark, index) => {
        let x = (landmark.x - 0.5) * 16 // Adjust for plane size
        let y = (0.5 - landmark.y) * 9 // Adjust for plane size
        let z = 0.1 // Slightly in front of the video plane
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
        ) // Set position based on landmark
      })
    }
    const prepareLandmarks = () => {
      faceMaskLandmarks = new THREE.Group()
      previousFaceMaskLandmarksPosition = []
      for (let i = 0; i < 478; i++) {
        const geometry = new THREE.SphereGeometry(0.05, 16, 16) // Sphere to represent the landmark
        const material = new THREE.MeshStandardMaterial({ color: 0xffff00 }) // Red color
        const mesh = new THREE.Mesh(geometry, material)
        previousFaceMaskLandmarksPosition[i] = { x: 0, y: 0, z: 0 }
        mesh.position.set(
          previousFaceMaskLandmarksPosition[i].x,
          previousFaceMaskLandmarksPosition[i].y,
          previousFaceMaskLandmarksPosition[i].z
        ) // Set position based on landmark
        mesh.scale.set(0.25, 0.25, 0.25) // Set position based on landmark
        mesh.name = 'landmark' // Name for identification
        faceMaskLandmarks.add(mesh) // Add to the scene
      }
      faceMaskLandmarks.visible = false
      scene.add(faceMaskLandmarks)
    }
    const animate = () => {
      requestAnimationFrame(animate)
      stats.begin() // Start measuring
      renderer.render(scene, camera) // Render the scene
      stats.end() // Stop measuring
    }
    const onWindowResize = () => {
      camera.aspect = window.innerWidth / window.innerHeight
      camera.updateProjectionMatrix()
      renderer.setSize(window.innerWidth, window.innerHeight)
    }
    const cleanup = () => {
      if (renderer) {
        renderer.dispose() // Clean up Three.js resources
      }
      if (video && video.srcObject) {
        video.srcObject.getTracks().forEach((track) => track.stop()) // Stop webcam tracks
      }
      window.removeEventListener('resize', onWindowResize) // Clean up resize event listener
      if (stats) {
        document.body.removeChild(stats.dom) // Remove stats from the DOM
      }
    }
    onMounted(init) // Initialize on component mount
    onBeforeUnmount(cleanup) // Clean up on component unmount
    return {
      container
    }
  }
}
</script>
<style scoped>
.webcam-view {
  position: fixed; /* Fullscreen positioning */
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden; /* Prevent scrolling */
  background: black; /* Optional: background color */
}
</style>
