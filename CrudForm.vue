<template>
  <div class="container">
    <slot></slot>
  </div>
</template>

<script>
export default {
  name: 'CrudForm'
}
</script>
<style scoped>
  .container {
    display: flex;
    flex-flow: column;
    align-items: center;
  }
</style>
