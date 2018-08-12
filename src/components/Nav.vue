<template>
  <div class="d-flex flex-column flex-md-row align-items-center p-3 px-md-4 mb-3 bg-white border-bottom box-shadow">
    <h5 class="my-0 mr-md-auto font-weight-normal">
      YNAB Category Tracker
    </h5>
    <a href="#/" @click="resetToken">Home</a>
  </div>
</template>

<script>
export default {
  props: ['resetToken']
}
</script>
