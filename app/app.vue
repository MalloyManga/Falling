<script setup lang="ts">
import { BasicShadowMap, SRGBColorSpace, NoToneMapping } from 'three'
</script>

<template>
    <TresCanvas clear-color="#000510" window-size :shadows="true" :shadow-map-type="BasicShadowMap"
        :output-color-space="SRGBColorSpace" :tone-mapping="NoToneMapping">
        <Falling />
    </TresCanvas>
</template>
