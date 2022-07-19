<script>
import { defineComponent } from 'vue';
import CSVBoxButton from '@/csv-box-button.vue';

export default defineComponent({
  name: 'ServeDev',
  components: {
    CSVBoxButton
  },
  methods: {
    onImport(result, data){
      console.log("onImport", result, data);
      if(result){
        console.log("success");
        console.log(data.row_success + " rows uploaded");
        //custom code
      }else{
        console.log("fail");
        //custom code
      }
    },
    onReady(){
      console.log("onReady")
    },
    onClose(){
      console.log("onClose")
    }
  },
  mounted() {
    
  },
});
</script>

<template>
  <div id="app">
    <CSVBoxButton 
      licenseKey="aZ37XCq5U6FiZUtP3BOYoCzDyQkzT0" 
      :user="{ user_id: 'default123' }" 
      :onImport="onImport"
      :onReady="onReady"
      :onClose="onClose"
      >Import</CSVBoxButton>
  </div>
</template>
