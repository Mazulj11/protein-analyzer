<template>
  <q-layout>
    <q-header>
      <div
        class="text-h4 bg-white text-black q-pa-lg"
        style="
          position: absolute;
          top: 35px;
          left:50%;
          transform: translateX(-50%);
          z-index: 99;
          border-radius: 10px;
          box-shadow: rgba(0, 0, 0, 0.3) 10px 10px 0px, rgba(0, 0, 0, 0.5) 0px 6px 6px;
        "
      >
        Algoritam 3
      </div>
      <q-img src="/cover.png" style="width: 100%; height: 175px;" />
      <q-separator color="black" size="10px" />
    </q-header>

    <q-page-container>
      <router-view />
    </q-page-container>
  </q-layout>
</template>

<script setup>
defineOptions({
  name: 'MainLayout'
})
</script>
<style>
body {
  background-color: #dfdfdf;
  margin-bottom: 24px;
}
</style>
