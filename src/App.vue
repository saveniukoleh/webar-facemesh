<template>
  <WebARFacemesh />
</template>

<script>
import WebARFacemesh from './components/WebARFacemesh.vue'

export default {
  name: 'App',
  components: {
    WebARFacemesh
  }
}
</script>

<style lang="scss">
#app {
  margin: 0;
  padding: 0;
  height: 100vh; /* Ensure the app takes full height */
  overflow: hidden; /* Prevent overflow */
}

html,
body {
  margin: 0;
}
</style>
