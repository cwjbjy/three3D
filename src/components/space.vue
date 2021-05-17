<template>
  <div id="space">
    <div id="cont"></div>
  </div>
</template>

<script>
import * as THREE from "three";
import * as GEOLIB from "geolib";
import { MapControls } from "three/examples/jsm/controls/OrbitControls";
import Stats from 'three/examples/jsm/libs/stats.module.js'
import {BufferGeometryUtils} from 'three/examples/jsm/utils/BufferGeometryUtils'

export default {
  name: "space",
  data() {
    return {
      scene: null,
      camera: null,
      renderer: null,
      controls: null,
      center: [12.466816, 41.8754885],
      MAT_BUILDING: null,
      MAT_ROAD:null,
      stats:null,
      geos_building:[],
      collider_building:[],
      raycaster:null,
    };
  },
  mounted() {
    this.Awake();
    let that = this;
    window.addEventListener('resize',onWindowResize,false)
    function onWindowResize(){
      that.camera.aspect = window.innerWidth / window.innerHeight
      that.camera.updateProjectionMatrix()
      that.renderer.setSize(window.innerWidth,window.innerHeight)
    }
    onWindowResize()

    document.getElementById('cont').addEventListener('mousedown',evt=>{
      let mouse = {
        	x : ( event.clientX / window.innerWidth ) * 2 - 1,
	        y : - ( event.clientY / window.innerHeight ) * 2 + 1
      }
      let hitted = that.Fire(mouse)
      console.log(hitted)
    })
  },
  methods: {
    Awake() {
      let cont = document.getElementById("cont");

      this.scene = new THREE.Scene();

      this.camera = new THREE.PerspectiveCamera(
        45,
        window.innerWidth / window.innerHeight,
        1,
        1000
      );
      this.camera.position.set(12,8,4);

      this.raycaster = new THREE.Raycaster()

      let light0 = new THREE.AmbientLight(0xfafafa, 0.25);
      let light1 = new THREE.PointLight(0xfafafa, 0.4);
      light1.position.set(200, 90, 40);
      let light2 = new THREE.PointLight(0xfafafa, 0.4);
      light2.position.set(200, 90, -40);
      this.scene.add(light0);
      this.scene.add(light1);
      this.scene.add(light2);

      let gh = new THREE.GridHelper(
        60,
        60,
        new THREE.Color(0x555555),
        new THREE.Color(0x333333)
      );
      this.scene.add(gh);

      let axesHelper = new THREE.AxesHelper(100);
      this.scene.add(axesHelper);

      this.renderer = new THREE.WebGLRenderer({ antialias: true });
      this.renderer.setSize(window.innerWidth, window.innerHeight);
      this.renderer.setPixelRatio(window.devicePixelRatio);
      // this.renderer.setClearColor(0xb9d3ff, 1); //背景色
      cont.appendChild(this.renderer.domElement);

      this.controls = new MapControls(this.camera, this.renderer.domElement);
      this.controls.dampingFactor = 0.25; //延迟响应时间
      this.controls.maxDistance = 800;

      this.MAT_BUILDING = new THREE.MeshPhongMaterial({
        // color: new THREE.Color(0xb1e90ff),
      });

      this.MAT_ROAD = new THREE.LineBasicMaterial({
        color:new THREE.Color(0x2F9BFF)
      })

      this.stats = new Stats()
      cont.appendChild(this.stats.domElement)

      this.GetJSON();

      this.Update();
    },
    Update() {
      requestAnimationFrame(this.Update);
      this.renderer.render(this.scene, this.camera);
      this.stats.update()
    },
    GetJSON() {
      fetch("/assets/road.geojson").then((res) => {
        res.json().then((data) => {
          this.LoadBuildings(data);
        });
      });
    },
    LoadBuildings(data) {
      let features = data.features;

      for (let i = 0; i < features.length; i++) {
        let fel = features[i];
        if (!fel["properties"]) return;

        let info = fel.properties

        if (info["building"]) {
          this.addBuilding(
            fel.geometry.coordinates,
            info,
            info["building:levels"]
          );
        }else if(info['highway']){
           this.addRoad(fel.geometry.coordinates,info)
        }
      }
      this.$nextTick(()=>{
        let mergeGeometry = BufferGeometryUtils.mergeBufferGeometries(this.geos_building)
        let mesh = new THREE.Mesh(mergeGeometry,this.MAT_BUILDING)
        this.scene.add(mesh);
      })
    },
    addRoad(data,info){
      let points = [];
      for(let i=0;i<data.length;i++){
        if(!data[0][1])return;
        let el = data[i]
        if(!el[0] || !el[i])return
        let elp = [el[0],el[1]]
        elp = this.GPSRelativePosition([elp[0],elp[1]],this.center)
        points.push(new THREE.Vector3(elp[0],0.5,elp[1]))
      }
      let geometry = new THREE.BufferGeometry().setFromPoints(points)

      geometry.rotateZ(Math.PI)

      let line = new THREE.Line(geometry,this.MAT_ROAD)
      this.scene.add(line)
      line.position.y = 0.5
    },
    addBuilding(data, info, height = 1) {
      height = height ? height : 1;
      let shape,geometry
      let holes = []

      for (let i = 0; i < data.length; i++) {
        let el = data[i];

        if(i === 0){
          shape = this.genShape(el, this.center);
        }else{
          holes.push(this.genShape(el, this.center)) //定义了形状上的孔洞
        }
      }

      for(let i=0;i<holes.length;i++){
        shape.holes.push(holes[i]) 
      }

      geometry = this.genGeometry(shape, {
          curveSegments: 1,
          depth: 0.05 * height,
          bevelEnabled: false,
        });

        geometry.rotateX(Math.PI / 2);
        geometry.rotateZ(Math.PI);

        // let mesh = new THREE.Mesh(geometry, this.MAT_BUILDING);
        // this.scene.add(mesh);

        /* 性能优化，创建单个实例 */
        this.geos_building.push(geometry)

        let helper = this.genHelper(geometry)

        if(helper){
          helper.name = info['name'] ? info['name']:'Building'
          helper.info = info
          this.collider_building.push(helper)
        }
    },

    genHelper(geometry){
      if(!geometry.boundingBox) geometry.computeBoundingBox()
      let box3 = geometry.boundingBox

      if(!isFinite(box3.max.x)){
        /* 值不能无限大 */
        return false
      }
      let helper = new THREE.Box3Helper(box3,0xffff00)
      helper.updateMatrixWorld()
      return helper
    },

    genShape(points, center) {
      let shape = new THREE.Shape();

      for (let i = 0; i < points.length; i++) {
        let elp = points[i];
        elp = this.GPSRelativePosition(elp, center);

        if (i == 0) {
          shape.moveTo(elp[0], elp[1]);
        } else {
          shape.lineTo(elp[0], elp[1]);
        }
      }

      return shape;
    },

    Fire(mouse){
      this.raycaster.setFromCamera(mouse,this.camera)
      let intersects = this.raycaster.intersectObjects(this.collider_building,true)
      if(intersects.length > 0){
        return intersects[0].object
      }else{
        return false
      }
    },
    genGeometry(shape, config) {
      let geometry = new THREE.ExtrudeBufferGeometry(shape, config);
      geometry.computeBoundingBox(); //盒子信息
      return geometry;
    },

    GPSRelativePosition(objPosi, centerPosi) {
      let dis = GEOLIB.getDistance(objPosi, centerPosi);
      let bearing = GEOLIB.getRhumbLineBearing(objPosi, centerPosi);
      let x = centerPosi[0] + dis * Math.cos((bearing * Math.PI) / 180);
      let y = centerPosi[1] + dis * Math.sin((bearing * Math.PI) / 180);
      return [-x / 100, y / 100];
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
</style>
