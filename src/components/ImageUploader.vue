<template>
  <div class="uploader">
    <label for="dropzoneFile" class="font-bold">
      <div class="p-2 border-2 cursor-pointer hover:bg-gray-200"> Upload Image </div>
    </label>
    <input
      type="file"
      id="dropzoneFile"
      class="dropzoneFile"
      accept="image/*"
      onclick="this.value=null;"
    />
  </div>
</template>

<style scoped>
.uploader input {
  display: none;
}
</style>

<script>
export default {
  name: 'ImageUploader'
}
</script>
