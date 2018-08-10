<template>
  <div class="container">
    <div class="crudForm elevation-4" >
       <slot></slot>
    </div>
  </div>
</template>

<script>
export default {
  name: 'crud-form'
}
</script>
<style scoped>
  .crudForm {
    max-width: 60%;
    width: 60%;
  }
  .container {
    display: flex;
    flex-flow: column;
    align-items: center;
  }
</style>
