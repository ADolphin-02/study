<template>  
  <div class="hello">  
    <h1>{{ msg }}</h1>  
    <h2>Essential Links</h2>  
    <el-button @click="See()">百度按钮</el-button>  
    <el-button type="primary" @click="See()">百度按钮

    </el-button>  
    <el-button type="text" @click="See()">百度按钮</el-button>  
  </div>  

</template>  
  
<script>  
export default {  
  name: 'hello',  
  data () {  
    return {  
      msg: 'Welcome to Your  App'  
    }  
  },
  
methods:{
  See(){
        window.location.href = "http://www.baidu.com"
  }
}
    
}  
</script>  
  
<!-- Add "scoped" attribute to limit CSS to this component only -->  
<style scoped>  
h1, h2 {  
  font-weight: normal;  
}  
  
ul {  
  list-style-type: none;  
  padding: 0;  
}  
  
li {  
  display: inline-block;  
  margin: 0 10px;  
}  
  
a {  
  color: #42b983;  
}  
</style>  