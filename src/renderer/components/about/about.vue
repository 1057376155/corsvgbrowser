<template>
    <div id="about">
        <h1 class="corsvgbrowser title">关于 corsvgbrowser</h1>
        <p class="describe">写 corsvgbrowser 的动机是为了解决在开发的时候想要查看svg文件，但，window 上并没有找到可以预览svg文件的软件，之前都是用把
            文件拖入chrome中进行查看，但是文件数量多了之后，想要找到是某个文件对我来说是一件困难的事情，所以corsvgbrowser就诞生了，corsvgbrowser主要是用于
            查看svg文件，并不是代替资源文件管理器
        </p>
        <h1 class="aboutCor title">关于作者</h1>
        <p class="describe pointer" @click="openweb('https://github.com/1057376155')">
            cor是一个前端开发工程师
            <br/>
            github地址 : https://github.com/1057376155
        </p>
        <h1 class="title">技术构成</h1>
        <p class="describe">该软件使用 electron + vue 构成</p>
        <h1 class="title">版本</h1>
        <p class="describe">当前版本 {{version}}</p>
        <p class="describe pointer" @click="openweb('https://github.com/1057376155/corsvgbrowser')">最新版本 {{serverVserion}} &nbsp; 点击获取最新版本</p>
        <button @click="closeFN" class="btn close">关闭</button>
    </div>
</template>

<script>
    import pjson from '../../../../package.json'
    var shell=require('electron').remote.shell
    var https = require('https');   
    export default {
        data(){
            return{
                version:pjson.version,
                serverVserion:''
            }
        },
        mounted() {
            this.getServerVersion();
        },
        methods:{
            closeFN(){
                this.$emit('colose')
            },
            getServerVersion(){
                https.get('https://raw.githubusercontent.com/1057376155/corsvgbrowser/master/package.json',(res)=>{
                    var resData = "";
                    res.on("data",function(data){
                        resData += data;
                    });
                    res.on("end",()=>{
                        this.serverVserion=JSON.parse(resData).version
                        console.log(this.serverVserion,'-=-=')
                    });
                })
            },
            openweb(url){
                shell.openExternal(url)
            }
        }
    }
</script>

<style lang='less'>
@import '../../style/theme.less';
#about{
    width: 100%;
    height: 100%;
    position: absolute;
    top:0px;
    left: 0px;
    background: @tbg;
    padding: 40px;
    box-sizing: border-box;
    .title{
        color:lighten(@tbg,80%);
        // text-align: center;
        font-size: 20px;
        margin:10px 0px;
        font-weight: 100;
        -webkit-app-region: no-drag;
    }
    .describe{
        color:lighten(@tbg,55%);
        -webkit-app-region: no-drag;
    }
    .close{
        font-size: 14px;
        color: red;
        margin: 10px 0px;
        padding:5px 0px;
        border-radius: 50px;
        position: absolute;
        right:20px;
        bottom: 0px;
        -webkit-app-region: no-drag;
    }
    .pointer{
        cursor: pointer;
    }
}
</style>