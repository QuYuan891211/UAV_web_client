<template>
    <div class = "left-tool-bar">
        <div v-for="(item, index) in menus" :key="index" @click="selectMenu(index)">
                <div class="menus-item" >
                <div class="map-button">{{ item.title }}</div>
                            <!-- 图层切换组件 -->

                </div>
        </div>
                <div id="layer-tool" >
                    <!-- <div id="mouse-position" ></div> -->
                    <div id="layerControl" class="layerControl">
                        <div class="title"><label>图层列表</label></div>
                        <ul id="layerTree" class="layerTree"></ul>
                    </div>
                </div>
                <!-- 鹰眼控件 -->
                <div id="overview-map" ></div>
                <!-- 鼠标位置控件 -->
                <div id="mouse-tool" >
                    <div id="mouse-position" ></div>          
                                                
                </div>
        
        <!-- <div class="map-button" id="panto" @click="moveToChinaSea()"></button> -->
    </div>
    <div class="wrapper">
        <div class="map" id="olMap"></div>
    </div>

</template>
<script>
import { tsThisType } from '@babel/types';
// import { ol } from 'dist/static/libs/ol5/ol';
import bus from '../../utils'
import {baseurl} from '../../assets/js/common_data'
import {mousePositionControl} from '../../assets/js/map_control_tool'
// import MousePosition from "ol/control/MousePosition";
// import { format } from "ol/coordinate";

export default {
    data() {
        return {
             // 菜单栏数据
             menus: [
                { title: '地图复位' },

            ],

            url_load_config : 'http://' + baseurl + ':8085/config/all',
            map: null,
            point_icon_style_path:'./static/images/label/icon32.png',
            point_selected_icon_style_path:'./static/images/label/icon32_selected.png',
            parser: null,
            select_id:null,
            //map中的图层数组
            layer: new Array(),
            //图层名称数组
            layerName: new Array(),
            //图层可见属性数组
            layerVisibility: new Array(),
            //选中变换的样式
           
            //列表,在加载时导入[]
            buoyList: []
            // selected_name : null
        };
    },
    mounted() {
        // this.initBuoyData();
        this.initMap();
        this.initMarker();
        //加载图层列表数据
        this.loadLayersControl(this.map, "layerTree");
        
    },
    methods: {
           /**
        * 加载图层列表数据
        * @param {ol.Map} map 地图对象
        * @param {string} id 图层列表容器ID
        */
         loadLayersControl(map, id) {
            var layer = new Array(); //map中的图层数组
            var layerName = new Array();  //图层名称数组
            var layerVisibility = new Array(); //图层可见属性数组
            // alert(map.getLayers())
            var treeContent = document.getElementById(id); //图层目录容器

            var layers = map.getLayers(); //获取地图中所有图层
            for (var i = 0; i < layers.getLength(); i++) {
                //获取每个图层的名称、是否可见属性
                layer[i] = layers.item(i);
                layerName[i] = layer[i].get('name');
                layerVisibility[i] = layer[i].getVisible();

                //新增li元素，用来承载图层项
                var elementLi = document.createElement('li');
                treeContent.appendChild(elementLi); // 添加子节点
                //创建复选框元素
                var elementInput = document.createElement('input');
                elementInput.type = "checkbox";
                elementInput.name = "layers";
                elementLi.appendChild(elementInput);
                //创建label元素
                var elementLable = document.createElement('label');
                elementLable.className = "layer";
                //设置图层名称
                this.setInnerText(elementLable, layerName[i]);
                elementLi.appendChild(elementLable);

                //设置图层默认显示状态
                if (layerVisibility[i]) {
                    elementInput.checked = true;
                }
                this.addChangeEvent(elementInput, layer[i]);  //为checkbox添加变更事件                                         
            }
        },
                /**
        * 为checkbox元素绑定变更事件
        * @param {input} element checkbox元素
        * @param {ol.layer.Layer} layer 图层对象
        */
        addChangeEvent(element, layer) {
            element.onclick = function () {
                if (element.checked) {
                    layer.setVisible(true); //显示图层
                }
                else {
                    layer.setVisible(false); //不显示图层
                }
            };
        },
                /**
        * 动态设置元素文本内容（兼容）
        */
        setInnerText(element, text) {
            if (typeof element.textContent == "string") {
                element.textContent = text;
            } else {
                element.innerText = text;
            }
        },
        /**
            * 创建矢量标注样式函数,设置image为图标ol.style.Icon
            * @param {ol.Feature} feature 要素
            */
        createLabelStyle(feature, is_selected) {
            return new ol.style.Style({
                image: new ol.style.Icon(
                    /** @type {olx.style.IconOptions} */
                    ({
                        anchor: [0.5, 30],
                        anchorOrigin: 'top-right',
                        anchorXUnits: 'fraction',
                        anchorYUnits: 'pixels',
                        offsetOrigin: 'top-right',
                        // offset:[0,10],
                        //图标缩放比例
                        // scale:0.5,
                        //透明度
                        opacity: 1,
                        //图标的url
                        src: is_selected?this.point_selected_icon_style_path:this.point_icon_style_path
                        // 
                    })),
                text: new ol.style.Text({
                    //位置
                    textAlign: 'center',
                    //基准线
                    textBaseline: 'middle',
                    //文字样式
                    font: 'normal 14px 微软雅黑',
                    //文本内容
                    text: feature.get('name'),
                    //文本填充样式（即文字颜色）
                    fill: new ol.style.Fill({ color: 'white' }),
                    offsetY: 12,
                    scale: 1,
                    stroke: new ol.style.Stroke({ color: '#white', width: 2 })
                })
            });
        },
        /**
            * 添加一个新的标注（矢量要素）
            * @param {ol.Coordinate} coordinate 坐标点
            */
        addVectorLabel(coordinate) {
            //新建一个要素 ol.Feature
            var newFeature = new ol.Feature({
                //几何信息
                geometry: new ol.geom.Point(coordinate),
                //名称属性
                name: '新标注'
            });
            //设置要素的样式
            newFeature.setStyle(this.createLabelStyle(newFeature,false));
            //将新要素添加到数据源中
            this.vectorSource.addFeature(newFeature);
        },
        
        initMarker() {
            
            var shanghai = ol.proj.fromLonLat([121.29, 31.14]);

            //矢量标注的数据源
            var vectorSource = new ol.source.Vector({
                features: []
            });
            
            this.initFeature(vectorSource)

            //矢量标注图层
            var vectorLayer = new ol.layer.Vector({
                source: vectorSource,
                name:'观测点位矢量图层'
            });
            this.map.addLayer(vectorLayer);
            
            /**
        * 为map添加点击事件监听,渲染弹出popup
        */
            var vm=this;
            this.map.on('click', function (evt) {
                
                // vm.map.removeLayer (vm.vectorLayer)
                // var vs = vm.vectorSource.forEachFeatues()
                // var features = vm.vectorSource.getFeatures()
                
                //     for(var i=0; i< features.length;i++){
                //         features[i].setStyle(vm.createLabelStyle(feature,false));
                //     }
                //判断当前单击处是否有要素,捕获到要素时弹出popup
                var feature = this.forEachFeatureAtPixel(evt.pixel, function (feature, layer) { return feature; });
                if (feature) {

                    var layers_collection =  vm.map.getLayers()
                    var layers_arrays = layers_collection.getArray()
                    // console.log('获取layers长度'+layers_arrays.length)
                    var target_layer = layers_arrays[0]
                    // console.log('获取layers长度'+layers_arrays.length)
                    var source = target_layer.getSource()
                    var features = source.getFeatures()
                    // vm.initFeature()
                    for(var i=0; i< features.length;i++){
                        features[i].setStyle(vm.createLabelStyle(features[i],false));
                    }
                    // var targetLayer = layers.get


                    // var sources = layers[2].getSource
                    // console.log('获取sources长度'+sources.length)
                    //var coor=features[0].getGeometry().getCoordinates();
                    //var property=features[0].getProperties();
                    //var type=features[0].getGeometry().getType();
                    // console.log(feature.getStyle.getImage)
                    var name = feature.getProperties().name
                    // vm.selected_name = name
                    //向公共组件传选择的名称
                    // vm.sendSelectedNameToFa(name)
                    bus.emit('select_feature', name );
                    // bus.emit('select_feature_7', name );
                    // bus.emit('select_feature_30', name );
                    vm.updateSelectClientInfo(name)
                    // vm.initMarker()
                    feature.setStyle(vm.createLabelStyle(feature,true));
                    
                } else {
                    
                    //  vm.initMarker()
                     bus.emit('changeActive', {'active':0, 'name':''} );
                }
            });

            /**
        * 为map添加鼠标移动事件监听,当指向标注时改变鼠标光标状态
        */
            this.map.on('pointermove', function (e) {
                var pixel = this.getEventPixel(e.originalEvent);
                var hit = this.hasFeatureAtPixel(pixel);
                this.getTargetElement().style.cursor = hit ? 'pointer' : '';
            });
        },
        // before_select(){
        //     var selectClick = new ol.interaction.Select({
        //     // 事件类型
        //     condition: ol.events.condition.click,
        //     // 点击后的样式
        //     style: createLabelStyle(feature, false)
        // })
        // },
        
        
        // recoveryFeatures(vectorSource){
        //     this.vectorSource.clear()
        //     this.initFeature(vectorSource)
        // },
        sendSelectedNameToFa(name){

            this.$emit("getSelectedNameByCh",name)

        },
        initFeature(vectorSource){
            axios(
                {method: 'get',//提交方法
            url: this.url_load_config,//提交地址
            params: {}}).then((res) => {
                if(100 == res.data.commonResultCode.code){
                    this.buoyList = res.data.observeConfigList
                    // alert(this.buoyList[0].name)
                    // alert(this.buoyList[0].lat)

                // vectorSource.clear
                for (var i = 0; i < this.buoyList.length; i++) {
                    var user = this.buoyList[i];
                    var loc = [user.lon, user.lat]
                    
                    var point = ol.proj.fromLonLat(loc);
                    //实例化Vector要素,通过矢量图层添加到地图容器中
                    var iconFeature = new ol.Feature({
                        geometry: new ol.geom.Point(point),
                        //名称属性
                        name: user.name,

            //  
                });
                iconFeature.setStyle(this.createLabelStyle(iconFeature,false));
                vectorSource.addFeature(iconFeature);
            }
        }else{
                    alert(res.data.commonResultCode.message)
                }
                // console.log(res。commonResultCode);
                // alert(res.data.observeConfigList)
            })
        },
       
        //更新指定用户的数据
        updateSelectClientInfo(name){
            // console.log('选中'+name)
             for (var i = 0; i < this.buoyList.length; i++) {
                var user = this.buoyList[i];
                if(user.name == name){
                    bus.emit('changeActive', {'active':1, 'name':name} );
                    break
                }
             }

        },
        initMap() {
            var TiandiMap_vec = new ol.layer.Tile({
                name: "天地图矢量图层",
                source: new ol.source.XYZ({
                    url: "http://t0.tianditu.com/DataServer?T=vec_w&x={x}&y={y}&l={z}&tk=cd7516c53e2e5bee9bad989b63db6ce4",
                    wrapX: false
                }),
                preload:Infinity
            });
            var TiandiMap_cva = new ol.layer.Tile({
                name: "天地图矢量注记图层",
                source: new ol.source.XYZ({
                    url: "http://t0.tianditu.com/DataServer?T=cva_w&x={x}&y={y}&l={z}&tk=cd7516c53e2e5bee9bad989b63db6ce4",
                    wrapX: false
                }),
                preload:Infinity
            });
            var TiandiMap_img = new ol.layer.Tile({
                name: "天地图影像图层",
                source: new ol.source.XYZ({
                    url: "http://t0.tianditu.com/DataServer?T=img_w&x={x}&y={y}&l={z}&tk=cd7516c53e2e5bee9bad989b63db6ce4",
                    // url:"http://128.5.7.127:8082/geoserver/observer/wms",
                    params:{
                        LAYERS:"observer:geotools_coverage"
                    },
                    wrapX: false
                }),
                preload:Infinity
            });
            // var GaodeMap_img = new ol.layer.Tile({
            //     name: "高德影像图层",
            //     source: new ol.source.TileWMS({
            //         // url: "http://t0.tianditu.com/DataServer?T=img_w&x={x}&y={y}&l={z}&tk=cd7516c53e2e5bee9bad989b63db6ce4",
            //         url:"http://localhost:8082/geoserver/observer/wms",
            //         params:{
            //             'FORMAT': 'image/png',
            //             'VERSION': '1.1.1',
            //             tiled: true,
            //             STYLES: '',
            //             LAYERS:"observer:gaode_part_imgAndMark"
            //         },
            //         wrapX: false
            //     }),
            //     preload:Infinity
            // });
            var TiandiMap_cia = new ol.layer.Tile({
                name: "天地图影像注记图层",
                source: new ol.source.XYZ({
                    url: "http://t0.tianditu.com/DataServer?T=cia_w&x={x}&y={y}&l={z}&tk=cd7516c53e2e5bee9bad989b63db6ce4",
                    wrapX: false
                }),
                preload:Infinity
            });

            //实例化Map对象加载地图
            this.map = new ol.Map({

                //地图容器div的ID
                target: 'olMap',

                // layers: [GaodeMap_img],
                // layers: [TiandiMap_img],
                //地图容器中加载的图层:加载影像图
                // layers: [TiandiMap_img, TiandiMap_cia],

                //图层
                layers: [
                    TiandiMap_img, 
                    TiandiMap_cia
                ],

                //地图容器中加载的图层:加载矢量图层
                // layers: [TiandiMap_vec, TiandiMap_cva],

                //地图视图设置
                view: new ol.View({
                    //地图初始中心点
                    center: ol.proj.transform([130, 27], 'EPSG:4326', 'EPSG:3857'),
                    //地图初始显示级别
                    zoom: 5,
                    minZoom:4,
                    maxZoom:9
                }),
                


            });




           //1. 添加地图组件
            //1.1 比例尺控件
            this.map.addControl(new ol.control.ScaleLine());

             //1.2 鼠标坐标控件
            var mousePositionControl = new ol.control.MousePosition({
            //坐标格式
                coordinateFormat: function (coordinate) {
                    return ol.coordinate.format(coordinate, "经度:{x} &nbsp; 纬度:{y}", 2);
                },
                //地图投影坐标系（若未设置则输出为默认投影坐标系下的坐标）
                projection: "EPSG:4326",
                //坐标信息显示样式类名,默认是'ol-mouse-position'
                className: "custom-mouse-position",
                //显示鼠标位置信息的目标容器
                target: document.getElementById("mouse-position"),
                //未定义坐标的标记
                undefinedHTML: "&nbsp;",
            });
            this.map.addControl(mousePositionControl);
           //1.3 鹰眼控件
            //实例化鹰眼控件（OverviewMap）,自定义样式的鹰眼控件  
            var overviewMapControl = new ol.control.OverviewMap({
                className: 'ol-overviewmap ol-custom-overviewmap', //鹰眼控件样式（see in overviewmap-custom.html to see the custom CSS used）
                //鹰眼中加载同坐标系下不同数据源的图层
                layers: [ 
                    TiandiMap_img
                ],
                collapseLabel: '\u00BB', //鹰眼控件展开时功能按钮上的标识（网页的JS的字符编码）
                label: '\u00AB', //鹰眼控件折叠时功能按钮上的标识（网页的JS的字符编码）
                collapsed: false //初始为展开显示方式
            });
            // this.map.addControl(overviewMapControl);
            //1.4 地图全屏控件
            // this.map.addControl(new ol.control.FullScreen());


            var view = this.map.getView()
            var zoom = view.getZoom()
            var center = view.getCenter()
            var rotation = view .getRotation()

            


        },
        /**
         * 地图工具菜单方法集
         * @param {*} index 
         */
        selectMenu(index){
            switch (index){
            case 0:
            this.moveToChinaSea()  // 当表达式的结果等于 0 时,则执行该代码
             break;
                

            default :
            this.moveToChinaSea()  // 如果没有与表达式相同的值,则执行该代码
}
        },
        moveToChinaSea(){
            //单击平移按钮
            var view = this.map.getView()
            var wh = ol.proj.fromLonLat([130,27])
            view.setCenter(wh)
            view.setZoom(5)
            
            }
    },
};
</script>

<style scoped>
.map {
    width: 100%;
    height: 100vh;
}

/* TODO:不起作用 */
.ol-zoom .ol-zoom-in {
    display:none;
}
.left-tool-bar {
    position: absolute;
    /* left:0; */
    top: 100px;
    z-index: 2001;
    /* pointer-events: none; */
}
.left-tool-bar .menus-item {
    width: 144px;
    height: 30px;
    font-size: 18px;
    line-height: 55px;
    margin-bottom: 49px;
    cursor: pointer;

}
.left-tool-bar .map-button {
    background: url(@/assets/no-select.png);
    background-size: 100%;
    color: white;
}
.TooLib{
  float:left;
  position:absolute;
  bottom: 10px;
  z-index:2000;
}
.ButtonLib{
        width: 200px; 
        margin-left: 2px;
        opacity: 0.8; 
        padding:8px;  
        background-color: #428bca;  
        border-color: #357ebd;  
        color: #fff;  
        -moz-border-radius: 10px;  
        -webkit-border-radius: 10px;  
        border-radius: 10px; /* future proofing */  
        -khtml-border-radius: 10px; /* for old Konqueror browsers */  
        text-align: center;  
        vertical-align: middle;  
        border: 1px solid transparent;  
        font-weight: 900;  
        font-size:125%  
}

        #layer-tool{
            margin-top: 10px;
            border:none;padding:0;margin:0;
            font-size:14px;
            font-family:"微软雅黑";
            text-align: left;
            margin-top: 2%;
            width:100%;
            height:100%;
            position:absolute;
            
        }
        /* 图层控件层样式设置 */
        .layerControl{
            position:absolute;
            bottom:5px;
            min-width:200px;
            max-height:200px; 
            left:0px;  
            top:2%;    
            z-index:2001;   /*在地图容器中的层，要设置z-index的值让其显示在地图上层*/
            color:#ffffff;
            background-color:#4c4e5a;
            border-width: 10px; /*边缘的宽度*/
            border-radius: 10px;    /*圆角的大小 */ 
            border-color: #000 #000 #000 #000;  /*边框颜色*/
        }
        .layerControl .title
        {
            font-weight:bold;
            font-size:15px;
            margin:10px;
        }
        .layerTree{
            list-style-position: outside;
        }
        ul{
            border:none;padding:0;margin:0;
            font-size:14px;
            font-family:"微软雅黑";
			list-style: none;
		}
        .layerTree li
        {
            
             /* list-style:none; */
             margin:200px; 
             /* text-align:left; */
        }

        #mouse-tool{
            margin-top: 10%;
            width:100%;
            height:100%;
            position:absolute;
        }
         /* 鼠标位置控件层样式设置 */
         #mouse-position{
            color: white;
            float:left;
            position:absolute;
            bottom:5px;
            width:200px;
            height:20px;         
            z-index:2000;   /*在地图容器中的层，要设置z-index的值让其显示在地图上层*/
        }
        /* 鼠标位置信息自定义样式设置 */
        .custom-mouse-position{
            color:Red;
            font-size:50px;
            font-family:"微软雅黑";
        }

        #overview-map{
            margin-top: 10%;
            width:100%;
            height:100px;
        }
        /*=S 自定义鹰眼样式 */
        .ol-custom-overviewmap,.ol-custom-overviewmap.ol-uncollapsible {
            bottom: auto;
            left: auto;
            right: 0; /* 右侧显示 */
            top: 0;  /* 顶部显示 */
        }
        /* 鹰眼控件展开时控件外框的样式 */
        .ol-custom-overviewmap:not(.ol-collapsed)  {
            border: 1px solid black;
        }
         /* 鹰眼控件中地图容器样式 */
        .ol-custom-overviewmap .ol-overviewmap-map {
            border: none;
            width: 300px;
        }
        /* 鹰眼控件中显示当前窗口中主图区域的边框 */
        .ol-custom-overviewmap .ol-overviewmap-box {
            border: 2px solid red;
        }
        /* 鹰眼控件展开时其控件按钮图标的样式 */
        .ol-custom-overviewmap:not(.ol-collapsed) button{
            bottom: auto;
            left: auto;
            right: 1px;
            top: 1px;
        }
</style>