<script setup>
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <v-app>
    <v-container>
      <v-app-bar :elevation="2">
        <template v-slot:prepend>
          <v-app-bar-nav-icon></v-app-bar-nav-icon>
        </template>
        <v-app-bar-title>LRZTP PDF Search</v-app-bar-title>
      </v-app-bar>
    </v-container>
    <RouterView />
  </v-app>
</template>

<style>
</style>
