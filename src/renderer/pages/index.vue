<template>
    <div id="index">
        <div class="imgPath">
            <bFN :class="showAbout?'showAbout':''"></bFN>
            <div class="imgPath_content">
                <div :class="['pathInfo',index==selectIndex?'active':'']" @click="selectPath(index)" v-for="(item,index) in pathsList" :key="index">
                    <div class="pathRemark">{{item.remark}}</div>
                    <div class="pathsFN">
                        <span @click.stop="delPath(item,index)">删除</span>
                    </div>
                </div>
                <p v-if="!pathsList||pathsList.length==0" class="isNoData pathInfo">暂无数据</p>
            </div>
            
        </div>
        <div class="imgContetn">
            <div class="imgContetnInfo" v-if="nowPath&&!showAbout">
               <span @click="openPath()">文件路径 : {{nowPath}} </span>
            </div>
            <div class="files">
                <div @click="nextPath(item)" :style="'background:'+item.bgColor" :class="['file',item.isDirectory?'folderType':'fileType']" v-for="(item,index) in files" :key="index">
                    <div v-if="showFileType[getFileExtend(item.name)]" class="showFileType">
                        <img :src="item.path" alt=""> 
                    </div>
                    <span>{{item.name}}</span>
                </div>
            </div>
            <div class="pathFN">
                <span class="pre" v-if="showPre" @click="preFN">上一页</span>
                <!-- <span class="next" @click="nextFN">下一页</span> -->
                <span class="addPath" @click="getPath">增加文件夹</span>
                <span @click="showAbout=true">关于</span>
            </div>
            <transition name="offsetY">
                <inputPanel v-if="inputPanelIsShow" @affirm="getUserRemark"></inputPanel>
            </transition>
        </div>
        <transition name="offsetY">
            <about v-show="showAbout" @colose="showAbout=false"></about>
        </transition>
    </div>
</template>

<script>
const dialog = require('electron').remote.dialog
import bFN from '../components/bFN/bFN'
import inputPanel from '../components/inputPanel/inputPanel'
import about from '../components/about/about'
import { constants } from 'http2';
const low = require('lowdb');
const FileSync = require('lowdb/adapters/FileSync');
const adapter = new FileSync('db.json');
const db = low(adapter)
const uuidv1 = require('uuid/v1');
var shell = require('electron').remote.shell;
var fs=require('fs');
var path=require('path')
    export default {
        data(){
            return{
                inputPanelIsShow:false,
                userSelectPath:'',
                isFirst:false,
                pathsList:[],
                selectIndex:0,
                files:[],//文件列表
                showPre:false,
                nowPath:'',//当前的文件路径
                showAbout:false,//是否显示
                showFileType:{
                    '.jpg':true,
                    '.png':true,
                    '.svg':true
                }
            }
        },
        components:{
            bFN,inputPanel,about
        },
        mounted() {
             this.initPaths()
             process.on('uncaughtException', function (e) {
                /*处理异常*/
                console.log(e.message,'-=-')
            });
        },
        methods:{
            getPath(){
                //获取文件夹路径
                dialog.showOpenDialog(
                    {
                        title:"选择文件含有图片的文件夹",
                        properties:["openDirectory"]
                    },(r)=>{
                        console.log(fs.readdirSync(r[0]))
                        if(!r||!r[0])return;
                        if(fs.readdirSync(r[0]).length>500){
                            this.$corArt({str:'这里有好多文件,我就不添加了'})
                            return;
                        }
                        var paths=db.get('paths').value();
                        this.userSelectPath=r[0]
                        if(paths==undefined){
                            this.isFirst=true;
                            this.inputPanelIsShow=true
                        }else{
                            this.isFirst=false;
                            for(let item of paths){
                                if(item.path==r[0]){
                                    this.$corArt({str:'路径已存在'})
                                    return;
                                }
                            }
                            this.inputPanelIsShow=true
                        }
                    })
            },
            getUserRemark(r){
                if(r.text==''){
                    this.$corArt({str:'请输入备注名称'});
                    return;
                }
                if(this.isFirst){
                    //说明第一次使用
                    db.set('paths',[{id:uuidv1(),remark:r.text,path:this.userSelectPath}]).write()
                }else{
                    //说明不是第一次
                    db.get('paths').push({id:uuidv1(),remark:r.text,path:this.userSelectPath}).write()
                }
                this.inputPanelIsShow=false
                this.initPaths();
            },
            initPaths(){
                this.pathsList=db.get('paths').value();//获取保存的路径
                this.selectIndex=db.get('selectIndex').value();//获取保存的路径内容
                var FilsHistory=db.get('FilsHistory').value();//获取保存的浏览历史
                this.nowPath=FilsHistory[db.get('FilsHistoryIndex').value()];//获取当前路径
                this.getPathFile(this.nowPath);//获取路径内容
                this.showPre=true//显示上一页内容
                if(db.get('FilsHistory').value().length==1){
                    this.showPre=false
                }
            },
            delPath(item,index){
                //删除路径
                db.get('paths').remove({id:item.id}).write()
                
                if(this.pathsList.length==0){
                    this.files=[];
                    this.nowPath='';
                    this.showPre=false;
                    db.set('FilsHistory',[]).write()
                    return;
                }
                this.pathsList.forEach((it2,in2)=>{
                    if(it2.id==item.id){
                        this.pathsList.splice(in2,1)
                    }
                })
                if(this.selectIndex==index){
                    this.selectPath(this.pathsList.length-1);
                }else{
                    this.selectPath(this.selectIndex);
                }

                
            },
            selectPath(index){
                this.selectIndex=index;
                db.set('selectIndex',index).write()
                var filePath=this.pathsList[index].path
                this.getPathFile(filePath)
                db.set('FilsHistory',[filePath]).write()
                db.set('FilsHistoryIndex',0).write()
                this.nowPath=filePath;
                this.$hideCorArt()
            },
            openPath(){
                //打开文件
                shell.showItemInFolder(this.nowPath)
            },
            nextPath(item){
                //点击进入下一个路径
                if(!item.isDirectory){
                    if(this.showFileType[path.parse(item.name).ext]){
                        shell.showItemInFolder(item.filePath)
                    }else{
                        shell.showItemInFolder(item.path)
                    }
                    return;
                }
                if(fs.readdirSync(item.path).length>500){
                    this.$corArt({str:'这里有好多文件,我就不进去了'})
                    return;
                }
                this.getPathFile(item.path)
                this.nowPath=item.path;
                for(let pathItem of db.get('FilsHistory').value()){
                    if(pathItem==item.path){
                        return;
                    }
                }
                db.get('FilsHistory').push(item.path).write()
                db.set('FilsHistoryIndex',db.get('FilsHistoryIndex').value()+1).write();
                this.showPre=true
            },
            getPathFile(filePath){
                //获取文件类型
                if(!filePath)return;
                var fileTypeArr=[]
                var folderTypeArr=[]
                fs.readdir(filePath,async (err,files)=>{
                    if(err){
                        console.log(err)
                    }
                    var num=0;
                    console.log(files.length)
                    files.forEach((file,index)=>{
                        fs.access(path.resolve(filePath,file), fs.constants.R_OK, (err) => {
                            num+=1
                            if(!err){
                                var type=fs.lstatSync(path.resolve(filePath,file)).isDirectory()
                                if(type){
                                    //文件加类型
                                    folderTypeArr.push(
                                        {
                                            path:path.resolve(filePath,file),
                                            isDirectory:type,
                                            name:file
                                        }
                                    )
                                }else{
                                    //文件类型
                                    fileTypeArr.push(
                                        {
                                            path:path.resolve(filePath,file),
                                            isDirectory:type,
                                            name:file
                                        }
                                    )
                                }
                                if(num==files.length){
                                    this.getBase64OfFiles([...folderTypeArr,...fileTypeArr],0)
                                }
                            }else{
                                console.log(err)
                            }
                        });
                    })
                })
            },
            async getBase64OfFiles(files,index){
                //获取base64 
                if(files[index]){
                    if(this.showFileType[path.parse(files[index].name).ext]){
                        var temp=await this.getImage(files[index].path);
                        files[index].filePath=files[index].path;
                        files[index].path=temp.base64
                        files[index].bgColor=temp.bgColor
                    }else{
                        files[index].bgColor=`rgba(0,0,0,1)`
                    }
                    this.getBase64OfFiles(files,index+1)
                }else{
                    this.$hideCorArt()
                    this.files=files;
                }
            },
            preFN(){
                //返回上一页、
                var FilsHistory=db.get('FilsHistory').value()
                if(FilsHistory.length==1){
                    return;
                }
                this.nowPath=FilsHistory[db.get('FilsHistoryIndex').value()-1];
                this.getPathFile(FilsHistory[db.get('FilsHistoryIndex').value()-1])
                FilsHistory.splice(FilsHistory.length-1,1)
                this.showPre=true
                db.set('FilsHistory',FilsHistory).write()
                db.set('FilsHistoryIndex',FilsHistory.length-1).write()
                if(db.get('FilsHistory').value().length==1){
                    this.showPre=false
                }
            },
            nextFN(){
                //返回下一页
            },
            getFileExtend(name){
                //获取文件的扩展名称
                return path.parse(name).ext
            },
            async getImage(pathFile){
                //返回base64 并且 返回背景颜色
                return new Promise((resolve,reject)=>{
                    try {
                        var imageFile=fs.readFileSync(pathFile);
                        if(imageFile.length>100000){
                            this.$corArt({str:'读取文件中',isTimeOut:false})
                        }
                    } catch (error) {
                        this.$corArt({str:'出现异常'})
                        return;
                    }
                    var imageFileBase64=`data:image/${path.parse(pathFile).ext.substring(1)}+xml;base64,`+imageFile.toString('base64')
                    var img = new Image();
                    img.src = imageFileBase64;
                    img.onload =()=>{     
                        var canvas=document.createElement("canvas");
                        canvas.width=img.width;
                        canvas.height=img.height;
                        var ctx = canvas.getContext("2d");
                        ctx.drawImage(img,0,0);
                        var colorData=[]
                        var changeBg=false;
                        var sum=0;
                        for(var i=0;i<100;i++){
                            sum=0;
                            colorData=ctx.getImageData(this.$getbs(0,img.width),this.$getbs(0,img.height),1,1).data;
                            colorData.forEach((item)=>{sum=sum+item})
                            if(sum>255){
                                changeBg=true
                            }
                        }
                        if(changeBg){
                            resolve({
                                base64:imageFileBase64,
                                bgColor:`rgba(0,0,0,1)`
                            })
                        }else{
                            resolve({
                                base64:imageFileBase64,
                                bgColor:`rgba(80,80,80,1)`
                            })
                        }
                    };
                    
                })
            }
        }
    }
</script>

<style lang="less">
@import "../style/theme.less";
#index{
    width: 100%;
    height: 100%;
    display: flex;
    .imgPath{
        height: 100%;
        flex-basis: 300px;
        min-width:300px;
        background:@bg;
        padding-top: 25px;
        box-sizing: border-box;
        position: relative;
        .imgPath_content{
            width:100%;
            height:100%;
            overflow: auto;
            padding:0px 10px;
            box-sizing: border-box;
            .pathInfo{
                background:@tbg;
                color:rgb(144, 144, 144);
                margin:10px 0px;
                padding: 10px;
                border-radius: 5px;
                user-select:none;
                cursor: pointer;
                display: flex;
                align-items: center;
                &.active{
                    // background:lighten(@tbg,50%);
                    color:rgb(255, 255, 255);
                    .pathsFN{
                        // color:#ff8787;
                    }
                }
                .pathRemark{
                    flex: 1;
                    word-break: break-all;
                }
                .pathsFN{
                    flex-basis:50px;
                    color:rgb(255, 53, 53);
                    font-weight: 800;
                    font-size: 13px;
                    text-align: center;
                }
            }
        }
        
    }
    .imgContetn{
        height: 100%;
        flex:3;
        background: @bgn;
        position: relative;
        .files{
            height: 100%;
            width: 100%;
            padding-top:40px; 
            box-sizing: border-box;
            overflow: auto;
            .pre,.next{
                font-size: 18px;
                border-radius: 30px;
                margin:0px 5px;
            }
            .pre{color: #0069f3}
            .next{color: #ffffff}
            .file{
                color:white;
                padding: 10px 20px;
                background: @tbg;
                margin: 10px 30px;
                display: inline-block;
                border-radius: 10px;
                position: relative;
                user-select: none;
                cursor: pointer;
                text-align: center;
                &.folderType::after{
                    content:'文件夹';
                    display:block;
                    position: absolute;
                    padding: 4px 10px;
                    border-radius: 30px;
                    right: -30px;
                    top: -10px;
                    font-size: 10px;
                    background:#0069f3;
                }
                &.fileType::after{
                    content:'文件';
                    display:block;
                    position: absolute;
                    padding: 3px 10px;
                    border-radius: 30px;
                    right: -20px;
                    top: -10px;
                    font-size: 10px;
                    background:#ff692e;
                }
                .showFileType{
                    img{
                        // height: 100px;
                        width: 100%;
                        max-width: 100px;
                        display: block;
                        margin:10px auto;
                    }
                }
            }
        }
        .imgContetnInfo{
            background: @tbg;
            color:white;
            height: 30px;
            display: flex;
            align-items: center;
            padding: 0px 10px;
            font-size: 12px;
            position: absolute;
            width: 100%;
            top:0;
            user-select:none;
            -webkit-app-region: drag;
            z-index: 2;
            white-space: nowrap;
            span{
                -webkit-app-region: no-drag;
                cursor: pointer;
            }
        }
        .pathFN{
            position:absolute;
            bottom: 10px;
            right: 10px;
            display: flex;
            align-items: center;
            span{
                background: @tbg;
                border:lighten(@tbg,40%) solid 1px;
                color: white;
                margin:0px 5px;
                padding: 10px 20px;
                font-size: 12px;
                border-radius: 30px;
                cursor: pointer;
                user-select:none;
            }
        }
        .addPath{
            cursor: pointer;
            user-select: none;
        }
    }
    .isNoData{
        text-align: center;
    }
    .showAbout{
        background: @tbg;
        position:fixed;
        z-index: 5;
    }
}

</style>