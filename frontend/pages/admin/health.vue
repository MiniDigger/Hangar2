<template>
    <div>{{ $nuxt.$route.name }}</div>
</template>

<script lang="ts">
import { Component, Vue } from 'nuxt-property-decorator';

// TODO implement AdminHealthPage

@Component
export default class AdminHealthPage extends Vue {}
</script>

<style lang="scss" scoped></style>