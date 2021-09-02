<template>
  <q-page class="container">
    <Header></Header>
    <Hub></Hub>
  </q-page>
</template>

<script>
import { defineComponent } from 'vue';
import Header from 'components/HeaderComponent.vue';
import Hub from 'components/HubComponent.vue';

export default defineComponent({
  name: 'PageIndex',
  components: {
    Header,
    Hub,
  },
});
</script>
